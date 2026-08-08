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

## 02 — `engine/logger`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| `fixedWidthColorLevelEncoder` | `fixedWidthColorLevelEncoder` | `exact` | Pads to five visible characters, then applies the original magenta/blue/yellow/red/bright-red ANSI mapping |
| `Init(string, string) error` and `Init.func1` | `Init` and its `sync.Once` closure | `exact` | First call wins; later calls return without replacing the process logger |
| `initLogger` encoder selection | `initLogger` | `exact` | Lowercase `json` selects production JSON; every other value selects the customized console encoder |
| Level parsing | `parseLevel` | `exact` | Exact lowercase debug/info/warn/error mapping; unknown values select info |
| JSON encoding | `initLogger` JSON branch | `exact` | Production keys with ISO8601 `time` replacing epoch `ts` |
| Console encoding | `initLogger` console branch | `exact` | `[2006-01-02 15:04:05]`, 28-column caller, single-space separator, fixed color level |
| Caller and stack options | `zap.AddCaller` and `zap.AddStacktrace(ErrorLevel)` | `exact` | Caller tests point at the external caller; errors include stack traces |
| `Debug`, `Info`, `Warn`, `Error` | Same exported wrappers | `exact` | Each lazily initializes and applies `AddCallerSkip(1)` before writing fields |
| `With` | `With` | `exact` | Child loggers retain structured context without the wrapper skip |
| Logger and sugared logger globals | Atomic `global` and `globalSugar` | `equivalent-proven` | Same initialized values with race-safe lazy reads |
| Current additive APIs | `L`, `InitFile`, option helpers | `equivalent-proven` | Direct access, file output, and zap option constructors remain available without changing legacy `Init` |
| Production adapter and packet fields | `engine/ipsec/logging.go` | `equivalent-proven` | IPsec debug/info/warn calls now flow through this logger; binary and byte-string packet fields survive console and JSON encoding |
