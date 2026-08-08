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

## 03 — `internal/vowifi/logging`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| `Info` and `Debug` structured wrappers | Same exported wrappers through the process `slog` sink | `equivalent-proven` | The host installs its configured zap-backed slog handler before startup; key/value fields and levels reach the same production logger |
| `RunInfo` and `RunDebug` | Same exported wrappers | `equivalent-proven` | Both are suppressed outside `go run`; inside it they emit at info and debug respectively, matching the original levels |
| `WarnRate` and `InfoRate` | Compatible dual-form wrappers | `equivalent-proven` | The legacy `(key, duration, message, fields...)` and current `(key, message, fields...)` source forms both work; the current form retains its five-second interval |
| `shouldEmitRateLimited` | `shouldEmitRateLimited` | `exact` | Lowercases and trims level, trims key, isolates levels with `level:key`, bypasses empty/invalid boundaries, and compares `now.Sub(last)` under a mutex |
| Rate limiter lifecycle | `rateLimiter` and `pruneRateLimitEntries` | `exact` | Initial map capacity 256; more than 4096 entries triggers deletion of entries older than 24 hours |
| `IsGoRun` and its once closure | `IsGoRun` | `exact` | `sync.Once` caches exactly one detection result and is race-safe |
| `detectGoRunMode` | `detectGoRunMode` | `exact` | Parsed `VOHIVE_FORCE_GO_RUN_LOG` overrides both true and false; then `/go-build` executable and `command-line-arguments` build metadata are checked in order |
| `envEnabled` | `envEnabled` | `exact` | Accepts only the original normalized values `1`, `true`, `yes`, and `on` |
| SIP digit pattern and `maskLongDigits` | `longDigitPattern` and `maskLongDigits` | `exact` | Original `\\d{8,}` pattern; keeps the first three and last two digits with a length-preserving mask |
| `RedactSIPRaw` | `RedactSIPRaw` | `exact` | Line-wise Authorization and Proxy-Authorization replacement uses the original uppercase marker and CR suffix; other lines receive digit masking |
| `RedactSMSContent` | `RedactSMSContent` | `exact` | `VOHIVE_SMS_LOG_CONTENT` opt-in, trimmed Unicode rune count, and exact `[REDACTED len=N]` output |
| `SlogAdapter` layout | `SlogAdapter{logger, callerChain}` | `exact` | Redress type recovery and the rebuilt binary both show `*zap.Logger` followed by `[]string` |
| `NewSlogHandler` and `Enabled` | Same symbols | `equivalent-proven` | The standalone module uses `zap.L()` for the original host-global fallback; explicit loggers, disabled automatic caller inference, and deferred core filtering match the original production path |
| `Handle` and its attribute closure | `(*SlogAdapter).Handle` | `exact` | Empty keys are ignored, caller attributes become a deduplicated chain, error text is retained as a zap field, and `Record.PC` becomes `Entry.Caller` |
| SIP read-error normalization | `normalizeReadError` in the `Handle` path | `exact` | The six original network error fragments map to `SIP TCP 通道读异常`; closed/EOF use debug and the other four use warn |
| `WithAttrs`, `WithGroup`, `dedupeNonEmpty`, `callerFromPC`, `writeWithCaller` | Same handler paths and helper symbols | `exact` | Preset fields and cloned caller chains are immutable across children; writes use a custom zap entry and core check, preserving the legacy lightweight group behavior |
| Current additive handler option | `WithCaller` | `equivalent-proven` | Retains the current `slog.HandlerOptions{AddSource: ...}` helper without altering the legacy adapter |
| Production logging path | IMS REGISTER, voice INVITE, netstack, SMS, and keepalive call sites | `equivalent-proven` | Existing production callers compile through the restored wrappers; `cmd/vohive` installs the configured zap-backed default slog handler before runtime startup |
