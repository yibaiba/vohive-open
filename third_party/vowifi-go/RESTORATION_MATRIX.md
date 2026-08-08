# Legacy Module Restoration Matrix

This matrix tracks parity with `old/vowifi-go` and the original v1.5.5
binary. A module advances only when every listed symbol is `exact` or
`equivalent-proven` and its production call path is verified.

## 01 — `engine/bufferpool`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| Package `init` | `init` | `exact` | Installs one constructor for each of the five pools |
| `init.newBucket.func1` … `func5` | Per-class `sync.Pool.New` closures | `exact` | Each closure returns a full-length `*[]byte` for its captured class size |
| Five size-class constants | `bufferClasses` | `exact` | Original binary data at `0x03a51260`: 512, 1024, 2048, 4096, 8192 |
| `Get(int) Lease` | `Get` | `exact` | Negative sizes normalize to zero; requests above 8192 are unpooled |
| Five-word `Lease` value layout | `Lease{slot, bytes, class}` | `exact` | `*[]byte`, `[]byte`, class; architecture-sized layout test |
| `(*Lease).Release` | `(*Lease).Release` | `exact` | Restores the pooled slice header, returns its slot, and clears lease metadata |
| Current additive byte accessor | `Lease.Bytes` | `equivalent-proven` | Returns the requested-length slice without changing legacy ownership |
| Production reuse | `swu.(*Session).encapsulateInnerPacketLease` | `equivalent-proven` | The outbound userspace ESP loop leases the sized destination and releases it after synchronous transport send |

The original release path clears the lease metadata; it does not scrub the
underlying packet bytes before reuse. The reconstruction preserves that
behavior rather than adding a new security or performance policy.
