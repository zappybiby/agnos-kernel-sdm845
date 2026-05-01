# Runtime Frag-Bank Evidence

Two runtime captures answer different parts of the frag-bank question:

- `post_flash_20260501_131549`: whether the bad layout existed on a natural crash boot, and whether the fatal FAR was a direct frag-bank read.
- `fastpath_shadow_20260501_133508`: whether ordinary fast-path TX reaches the bad ID range and whether the target-style descriptor view decodes to bad payload addresses before any natural fault occurs.

## What `post_flash` Showed

The natural crash boot reported:

```text
HTT FRAG BANK DESC_GAP: page=1 first_index=56 expected_desc=0xa0911fc0 host_desc=0xa0912000 desc_size=72 slack_per_page=64
HTT FRAG BANK SUMMARY: base=0xa0911000 pages=65 pool_elems=3600 desc_size=72 elems_per_page=56 slack_per_page=64 page_linear=1 desc_linear=0 first_page_gap_index=65535 first_desc_gap_index=56
arm-smmu 15000000.apps-smmu: Unhandled context fault: iova=0xa58c00d8, fsr=0x40000402, fsynr=0x360003, cb=5
arm-smmu 15000000.apps-smmu: FAR    = 00000000a58c00d8
arm-smmu 15000000.apps-smmu: soft iova-to-phys=0x0000000000000000
arm-smmu 15000000.apps-smmu: SID=0x40
```

The layout variables mean:

- `pool_elems=3600`: TX/frag descriptor IDs available to this pool.
- `pages=65`: coherent 4K pages used for the frag-desc pool.
- `desc_size=72`: bytes per `msdu_ext_desc_t`.
- `elems_per_page=56`: complete descriptors per 4K page.
- `slack_per_page=64`: unused tail bytes after the last complete descriptor in each page.
- `page_linear=1`: page base IOVAs happened to be contiguous.
- `desc_linear=0`: the descriptor stream was still not packed, because each page has the 64-byte unused tail.
- `first_desc_gap_index=56`: the first ID where `bank_base + id * desc_size` differs from the host's `dma_pages[]` address.

This is significant because the bad geometry was present on the same boot as the natural SMMU fault. It also shows that non-contiguous DMA page allocation is not required: even with contiguous page bases, the one-bank packed descriptor model is already wrong at ID 56.

The fatal FAR was `0xa58c00d8`. With `base=0xa0911000`, `pool_elems=3600`, and `desc_size=72`, the advertised packed bank would end near `0xa0950480`; the fatal FAR is far outside that range. That argues against the natural crash being a direct read of the frag-bank allocation and supports a later address derived from bad descriptor contents.

The relevant diagnostics are emitted by the layout checker in [`htt_tx.c#L713-L790`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L713-L790). The SMMU fault path prints the FAR, software translation result, and SID before calling the frag-bank fault matcher in [`arm-smmu.c#L1518-L1549`](drivers/iommu/arm-smmu.c#L1518-L1549).

## What `fastpath_shadow` Showed

The fast-path capture reported:

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

`msdu_id` is the TX descriptor ID published to the target. `host_iova` is where the host actually filled the frag descriptor using page/index math. `linear_iova` is where the target-style one-bank formula would look: `bank_base + msdu_id * desc_size`.

The important point is that normal fast-path TX immediately reached `msdu_id=1199`, far beyond the first broken ID 56. The host descriptor view decoded to a plausible fragment entry, while the target-style linear view decoded to fragment entries with impossible lengths and unmapped payload IOVAs.

That is strong non-fault evidence for the current theory: ordinary traffic can publish an ID whose host frag descriptor is valid, while the target-style bank/id view lands on different bytes that decode as bad payload DMA metadata.

The fast-path publish hook runs after successful `ce_send_fast()` in [`ol_tx.c#L845-L872`](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L845-L872) and [`ol_tx.c#L980-L993`](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L980-L993). The shadow logic computes host and target-style addresses, decodes both views, checks decoded linear-view payload IOVAs, and stores recent samples in [`htt_tx.c#L457-L545`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L457-L545), [`htt_tx.c#L648-L692`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L648-L692), and [`htt_tx.c#L800-L845`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L800-L845).

## Why the Pair Matters

`post_flash` ties the confirmed layout mismatch to a real natural crash boot and shows the fatal FAR is probably a derived address, not a direct frag-bank address. `fastpath_shadow` shows that ordinary TX reaches the bad ID range and that the target-style descriptor view can decode to unmapped payload IOVAs before any crash path is active.

Together, they support this chain:

```text
ordinary TX publishes high msdu_id
-> host-filled descriptor is valid
-> target-style bank/id address points elsewhere
-> wrong bytes decode as payload fragment address/length entries
-> MAC DMA may fetch from a bogus or stale payload IOVA
-> SMMU reports an unmapped SID 0x40 FAR
```

The remaining proof point is exact correlation on a future crash: the SMMU FAR should match, fall inside, or fall within the 128-byte burst envelope of a recently decoded linear-view fragment entry. The current fault hook records that as `HTT FRAG BANK SMMU_MATCH` from [`htt_tx.c#L587-L646`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L587-L646).
