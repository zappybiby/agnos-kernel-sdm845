# HTT Frag-Descriptor Bank Geometry Bug

This driver programs HELIUMPLUS/WIFI2.0 frag descriptors as `sizeof(struct msdu_ext_desc_t)` ([`htt_tx.c#L211-L216`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L211-L216)). In this tree, that object is 72 bytes: `qdf_tso_flags_t` is six 32-bit words ([`qdf_types.h#L568-L604`](drivers/staging/qca-wifi-host-cmn/qdf/inc/qdf_types.h#L568-L604)), and `msdu_ext_desc_t` adds six 8-byte payload fragment entries ([`htt_types.h#L173-L194`](drivers/staging/qcacld-3.0/core/dp/htt/htt_types.h#L173-L194)).

The allocator does not allocate one packed descriptor array. `htt_tx_frag_desc_attach()` asks `qdf_mem_multi_pages_alloc()` for frag descriptors ([`htt_tx.c#L960-L974`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L960-L974)), and that helper computes descriptors-per-page as `PAGE_SIZE / element_size`, then allocates one coherent `PAGE_SIZE` DMA object per page ([`qdf_mem.c#L1276-L1295`](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_mem.c#L1276-L1295), [`qdf_mem.c#L1324-L1337`](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_mem.c#L1324-L1337)).

With a 72-byte descriptor, each 4K page holds 56 descriptors:

```text
56 * 72 = 4032 = 0xfc0
4096 - 4032 = 64 = 0x40 unused bytes at the end of each page
```

The host accounts for that page boundary. When assigning a frag descriptor for TX descriptor `index`, it uses:

```text
page  = index / num_element_per_page
slot  = index % num_element_per_page
addr  = dma_pages[page] + slot * desc_size
```

That is exactly what `htt_tx_desc_alloc()` does before storing the virtual and DMA addresses ([`htt_tx.c#L1020-L1040`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L1020-L1040)).

The firmware-facing bank message is different. `FRAG_DESC_BANK_CFG` carries a global `desc_size`, per-bank base addresses, and per-bank min/max ID ranges ([`htt.h#L8944-L8964`](drivers/staging/fw-api/fw/htt.h#L8944-L8964), [`htt.h#L9016-L9042`](drivers/staging/fw-api/fw/htt.h#L9016-L9042)). This driver hardcodes `num_banks = 1`, publishes only `dma_pages[0].page_p_addr`, and gives that one bank the full `0..pool_elems - 1` ID range ([`htt_h2t.c#L162-L188`](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L162-L188)).

That describes this layout as if it were:

```text
target_addr = bank_base + id * 72
```

But the real host layout is:

```text
host_addr = dma_pages[id / 56] + (id % 56) * 72
```

Those formulas agree only for IDs 0 through 55. At ID 56, the bank formula lands at `bank_base + 0xfc0`, which is the 64-byte unused tail of page 0; the real descriptor is at the start of page 1. Even if the DMA pages happen to be physically contiguous, the advertised packed-bank geometry is still wrong because it ignores the 64 bytes that are unused at the end of every page.

The firmware API makes this mismatch relevant: the HTT TX descriptor `id` is documented as being used inside the target to calculate the fragmentation descriptor pointer from the configured base address ([`htt.h#L770-L777`](drivers/staging/fw-api/fw/htt.h#L770-L777)). The descriptor also contains an explicit `frags_desc_ptr`, but the same API describes that pointer as telling MAC DMA where the TX frame fragments reside ([`htt.h#L779-L787`](drivers/staging/fw-api/fw/htt.h#L779-L787)), and the driver still sends the bank configuration described above.

The source-level bug is therefore the advertised geometry, not merely random non-contiguous DMA placement. For any `msdu_id >= 56`, target-side bank/id lookup can select bytes that are not the host descriptor for that ID; after each 4K page, the bank-derived address drifts another 64 bytes behind the host's page-indexed descriptor address.
