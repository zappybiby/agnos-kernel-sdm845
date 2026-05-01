# The `0x80` Fault-Stride Question

Several natural SMMU fault captures did not look like a simple walk through `msdu_ext_desc_t` records. The repeated pattern was closer to:

```text
...0xd8 -> ...0x100 -> later +0x80 -> +0x80 -> +0x80
...0xc8 -> ...0x100 -> later +0x80
```

That was initially awkward for the frag-bank theory because the confirmed descriptor geometry problem is based on 72-byte records, not 128-byte records. HELIUMPLUS/WIFI2.0 sets the frag descriptor size to `sizeof(struct msdu_ext_desc_t)` ([`htt_tx.c#L211-L216`](drivers/staging/qcacld-3.0/core/dp/htt/htt_tx.c#L211-L216)), and `msdu_ext_desc_t` is a TSO header plus six 8-byte fragment entries ([`htt_types.h#L173-L194`](drivers/staging/qcacld-3.0/core/dp/htt/htt_types.h#L173-L194)).

A direct descriptor-walk explanation would predict `0x48`-sized movement, or page-boundary drift caused by the 64 unused bytes at the end of each 4K descriptor page. It would not naturally predict repeated `+0x80` reads after an initial alignment step.

The next possibility was a source-visible 128-byte TX object. The TX/HTT source was checked for rings, tables, descriptor pools, flow-control objects, CE state, IPA/uC-offload state, and target-shared coherent allocations with a natural 128-byte stride. That search did not produce a strong host-visible object matching the repeated fault tail, which weakened the idea that firmware was directly walking some 128-byte host table.

The useful pivot was that a frag descriptor is not just a descriptor ID record. Its fragment entries are payload DMA address/length tuples: the source describes each entry as a 48-bit physical address plus a 16-bit length ([`htt_types.h#L173-L189`](drivers/staging/qcacld-3.0/core/dp/htt/htt_types.h#L173-L189)). The TX path fills those entries from SKB fragment DMA addresses and lengths ([`ol_tx.c#L626-L664`](drivers/staging/qcacld-3.0/core/dp/txrx/ol_tx.c#L626-L664)), and `htt_tx_desc_frag()` writes the fragment DMA address and length into the descriptor ([`ol_htt_tx_api.h#L664-L721`](drivers/staging/qcacld-3.0/core/dp/ol/inc/ol_htt_tx_api.h#L664-L721)).

That changes what the fault address means. If target-side bank/id lookup selects the wrong `msdu_ext_desc_t`, the next access may no longer be "read the frag descriptor." It can become "fetch payload from the DMA address stored in the wrongly decoded fragment entry." In that case the SMMU FAR is a derived payload IOVA, not the descriptor IOVA.

The `0x80` tail then has a source-backed explanation. The target defaults document MAC DMA burst size as `0: 128B` ([`target_if_def_config.h#L51-L52`](drivers/staging/qca-wifi-host-cmn/target_if/core/inc/target_if_def_config.h#L51-L52)). That default is placed into the target resource config ([`wma_main.c#L195-L222`](drivers/staging/qcacld-3.0/core/wma/src/wma_main.c#L195-L222)) and copied into the WMI resource configuration sent to firmware ([`wmi_unified_tlv.c#L10450-L10455`](drivers/staging/qca-wifi-host-cmn/wmi/src/wmi_unified_tlv.c#L10450-L10455)).

Under this interpretation, the fault shape is:

```text
wrong frag descriptor selected by bank/id lookup
-> bogus or stale payload fragment IOVA decoded from that descriptor
-> MAC DMA starts fetching at the payload offset in that IOVA
-> DMA advances on 128-byte burst boundaries
-> SMMU reports ...0xd8 or ...0xc8, then alignment to ...0x100, then +0x80 steps
```

This also explains why the observed FARs do not need to be inside the advertised frag-bank range. The descriptor read can succeed from mapped coherent memory, while the later payload fetch faults because the decoded payload IOVA is stale, unmapped, or unrelated to the current packet.

The evidence therefore points away from a direct raw `0x48` descriptor-walk fault and toward an indirect payload-DMA fault caused by a bad `0x48` descriptor lookup. The remaining proof point is runtime correlation: a future SID `0x40` SMMU FAR should match, or fall within the 128-byte burst envelope of, a payload IOVA recently decoded from the target-style linear view of a high-ID frag descriptor.
