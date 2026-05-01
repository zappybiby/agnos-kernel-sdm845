# HTT Frag Bank Fault Chain

Status: the source supports a real host/target contract mismatch in the HELIUMPLUS/WIFI2.0 frag-desc bank setup. Current runtime evidence shows that ordinary TX can publish an ID whose target-style linear descriptor address differs from the host-filled descriptor and decodes to unmapped payload IOVAs. The natural crash capture also shows that the fatal FAR is outside the advertised frag-bank range, which supports a later derived payload fetch rather than a direct frag-bank read. The remaining proof point is an exact match between a future SMMU FAR and a recently decoded bogus payload address, range, or burst envelope.

1. WIFI2.0/HELIUMPLUS uses `sizeof(struct msdu_ext_desc_t)` for each frag descriptor. That structure carries six payload fragment entries; each entry stores a DMA address plus a length.
   Source: [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L212), [htt_types.h](drivers/staging/qcacld-3.0/core/dp/htt/htt_types.h#L181), [htt_types.h](drivers/staging/qcacld-3.0/core/dp/htt/htt_types.h#L192)

2. The pool allocator does not allocate one packed DMA object. It allocates one coherent `PAGE_SIZE` object per page and keeps the real DMA address for each page in `dma_pages[]`. That matters because only `dma_pages[0]` is later advertised to firmware. It also means the bank model can be wrong even when the page IOVAs happen to be contiguous, because each page still has unused tail bytes after its last complete descriptor.
   Source: [qdf_mem.c](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_mem.c#L1269), [qdf_mem.c](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_mem.c#L1324), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L972)

3. `FRAG_DESC_BANK_CFG` is sent with one bank, global `desc_size`, bank base from `dma_pages[0]`, min index `0`, and max index `pool_elems - 1`.
   Source: [htt_h2t.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L167), [htt_h2t.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L170), [htt_h2t.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L171), [htt_h2t.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L176), [htt_h2t.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L184), [htt_h2t.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_h2t.c#L187)

4. The firmware API documents the target-side lookup: the HTT TX descriptor `id` is used to calculate the fragmentation descriptor pointer from the configured base. The same API says WIFI2.0 hardware performs the mapping/translation instead of relying only on `frags_desc_ptr`.
   Source: [htt.h](drivers/staging/fw-api/fw/htt.h#L773), [htt.h](drivers/staging/fw-api/fw/htt.h#L774), [htt.h](drivers/staging/fw-api/fw/htt.h#L775), [htt.h](drivers/staging/fw-api/fw/htt.h#L8877), [htt.h](drivers/staging/fw-api/fw/htt.h#L8880), [htt.h](drivers/staging/fw-api/fw/htt.h#L8881)

5. Pool setup uses the same loop index `i` for the HTT TX descriptor, the frag descriptor, and `tx_desc.id`. `htt_tx_desc_init()` then writes that same `msdu_id` into the HTT TX descriptor. In the inspected source, there is no remap between the host TX ID and the frag-desc pool index before TX publish.
   Source: [ol_txrx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_txrx.c#L1777), [ol_txrx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_txrx.c#L1789), [ol_txrx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_txrx.c#L1815), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L555), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L559), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L2647)

6. The host finds the real descriptor through the page table: `dma_pages[id / elems_per_page] + (id % elems_per_page) * desc_size`. A target using the advertised single-bank contract would instead evaluate `bank_base + id * desc_size`.
   Source: [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L250), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L253), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L278), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L282)

7. With `desc_size = 0x48`, a 4096-byte page holds 56 complete descriptors. The remaining `0x40` bytes at the end of each page are unused tail space, not a valid descriptor slot. The first descriptor-address mismatch is therefore ID 56: the target-style formula points to `base + 0xfc0`, while the host uses the first descriptor in `dma_pages[1]`. The natural crash boot showed `page_linear=1` but `desc_linear=0`, so random page-to-page IOVA gaps are not required for the mismatch.
   Source: [qdf_mem.c](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_mem.c#L1286), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L737), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L739), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L748), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L751), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L768)

8. During TX preparation, the host clears the real `msdu_ext_desc_t`, terminates the fragment list, and fills payload fragment address/length entries from SKB fragments or TSO metadata.
   Source: [ol_tx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L596), [ol_tx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L626), [ol_tx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L637), [ol_tx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L643), [ol_tx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L654), [ol_tx.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L663), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L631), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L683)

9. The explicit `frags_desc_ptr` is still written in the HTT descriptor, but the firmware API above says WIFI2.0 hardware may use the ID-to-bank mapping. That leaves the single-bank configuration relevant even though the host also writes the direct pointer.
   Source: [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L1395), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L1410), [htt.h](drivers/staging/fw-api/fw/htt.h#L780), [htt.h](drivers/staging/fw-api/fw/htt.h#L8880), [htt.h](drivers/staging/fw-api/fw/htt.h#L8881)

10. For IDs at or above 56, an ID-based target lookup can read the unused tail bytes after page 0, shifted bytes from another descriptor, or a different mapped coherent page. On a page-linear boot, that wrong descriptor read may still hit mapped coherent memory and therefore not fault immediately. The diagnostic code computes both the host address and the target-style linear address, classifies where the linear address lands, and decodes any fragment address/length fields found there.
    Source: [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L258), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L281), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L313), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L457), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L534)

11. If those wrong bytes decode as non-zero payload fragment entries, the MAC DMA has plausible source addresses and lengths to fetch. The diagnostic checks those decoded payload IOVAs against the WLAN SMMU domain and keeps recent decoded entries for later SMMU fault correlation.
    Source: [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L665), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L667), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L678), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L373), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L518), [htt_tx.c](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L587)

12. TX completion releases payload DMA mappings. A delayed target read from a stale or shifted frag descriptor can therefore produce a WLAN SMMU read fault on a payload IOVA that was valid for an earlier packet but is no longer mapped.
    Source: [ol_tx_send.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx_send.c#L401), [ol_tx_desc.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx_desc.c#L795), [ol_tx_desc.c](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx_desc.c#L797), [qdf_nbuf.c](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_nbuf.c#L779), [qdf_nbuf.c](drivers/staging/qca-wifi-host-cmn/qdf/linux/src/qdf_nbuf.c#L780)

13. The observed `+0x80` FAR progression fits a payload-DMA fault better than a raw frag-desc table walk. A direct descriptor walk should be governed by the `0x48` descriptor size or by fields inside that descriptor. The proposed failure is two-stage instead: the bad ID-based lookup selects the wrong `0x48` bytes, those bytes decode as payload fragment address/length entries, and the MAC DMA then fetches from the decoded payload IOVA. At that point, the fault address follows payload-fetch behavior, not descriptor stride. With the default 128B MAC DMA burst setting, a fetch from an unmapped payload IOVA can fault at an initial packet offset such as `...c8` or `...d8`, then advance to the next 128-byte boundary and continue in `0x80` steps.
    Source: [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L665), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L667), [ol_htt_tx_api.h](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L678), [target_if_def_config.h](drivers/staging/qca-wifi-host-cmn/target_if/core/inc/target_if_def_config.h#L51), [target_if_def_config.h](drivers/staging/qca-wifi-host-cmn/target_if/core/inc/target_if_def_config.h#L52), [wmi_unified.h](drivers/staging/fw-api/fw/wmi_unified.h#L2613), [wmi_unified.h](drivers/staging/fw-api/fw/wmi_unified.h#L2616)

14. An unmapped WLAN IOVA reaches the ARM SMMU context-fault path. This path prints the fault IOVA, failed software translation, SID, and then BUGs for a fatal unhandled fault. In the natural crash capture, `base=0xa0911000`, `pool_elems=3600`, and `desc_size=72`, so the packed advertised bank would end near `0xa0950480`; the fatal FAR `0xa58c00d8` is outside that range. That rules out a direct read of the advertised frag bank for that crash and points to a later address derived from bad descriptor contents.
    Source: [arm-smmu.c](drivers/iommu/arm-smmu.c#L1520), [arm-smmu.c](drivers/iommu/arm-smmu.c#L1523), [arm-smmu.c](drivers/iommu/arm-smmu.c#L1537), [arm-smmu.c](drivers/iommu/arm-smmu.c#L1539), [arm-smmu.c](drivers/iommu/arm-smmu.c#L1548), [arm-smmu.c](drivers/iommu/arm-smmu.c#L1549)

Natural crash capture (`post_flash_20260501_131549/pstore/console-ramoops-0`):

```text
HTT FRAG BANK DESC_GAP: page=1 first_index=56 expected_desc=0xa0911fc0 host_desc=0xa0912000 desc_size=72 slack_per_page=64
HTT FRAG BANK SUMMARY: base=0xa0911000 pages=65 pool_elems=3600 desc_size=72 elems_per_page=56 slack_per_page=64 page_linear=1 desc_linear=0 first_page_gap_index=65535 first_desc_gap_index=56
arm-smmu 15000000.apps-smmu: Unhandled context fault: iova=0xa58c00d8, fsr=0x40000402, fsynr=0x360003, cb=5
arm-smmu 15000000.apps-smmu: FAR    = 00000000a58c00d8
arm-smmu 15000000.apps-smmu: soft iova-to-phys=0x0000000000000000
arm-smmu 15000000.apps-smmu: SID=0x40
```

Fast-path runtime evidence (`fastpath_shadow_20260501_133508/htt_after.txt`):

```text
first_desc_gap_index=56
msdu_id=1199
host_iova=0xa090f678
linear_iova=0xa090f138
linear_class=known_frag_page
SHADOW_HOST frag0=0xa1516c02/135
SHADOW_LINEAR frag0=0xffc319ee2150/65535 frag1=0xffc319ee2150/65535
SHADOW_LINEAR_MAP frag0=unmapped/0x0 frag1=unmapped/0x0
```

This proves that normal traffic can reach the bad ID range and that the target-style descriptor view can decode to unmapped payload IOVAs. The remaining proof point is a future SID `0x40` SMMU FAR matching a recently decoded linear-view fragment address, range, or 128-byte burst envelope.

Implications:

1. Reaching the bad ID range is no longer a weak assumption for this build. The natural crash boot had a 3600-entry pool with the first broken descriptor at ID 56, and the fast-path capture logged `msdu_id=1199` on the first publish observed after attach.

2. Reset, SSR, and teardown paths can still amplify stale ownership problems, but they are not required to create the bad descriptor view. The bad bank geometry exists during ordinary attach, and steady-state TX already produces target-style descriptors that decode to unmapped payload IOVAs.

3. Earlier descriptor-unmap injection results should be interpreted as reproducing the same SMMU fault class at an earlier stage. The natural crash shape appears to get one stage further: the wrong descriptor read can succeed, then the payload DMA fetch derived from that descriptor faults.
