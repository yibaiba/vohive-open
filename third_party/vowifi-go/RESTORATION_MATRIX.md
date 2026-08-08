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

## 04 — `engine/crypto`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| MODP group constants initialized by package `init` | `prime768` … `prime8192` in `dh_groups.go` | `exact` | All eight hexadecimal strings were recovered byte-for-byte from the v1.5.5 binary; regression tests lock the resulting historical bit lengths, including its malformed group 1/15–18 values |
| `DiffieHellman` layout and `NewDiffieHellman` | Same exported type and constructor | `exact` | Restores public group/key/prime fields, private ECDH fields, groups 1/2/5/14–20, and the original unsupported-group error |
| `GenerateKey`, `ComputeSharedSecret`, `PublicKeyBytes` | Same methods | `exact` | MODP and P-256/P-384 paths, left padding, peer boundary rejection, uninitialized ECDH error, and shared-key retention match the recovered implementation |
| `Encrypter`, `AppendEncrypter`, `CipherPreparer`, `PreparedCipher` | Same interfaces | `exact` | Method sets and error-bearing append/prepare contracts match recovered type metadata |
| AES/DES/3DES CBC transforms | `aesCBC`, `desCBC`, `tripleDESCBC`, `preparedCBC` | `exact` | Raw block-aligned encryption/decryption, key sizes, IV/block sizes, append behavior, preparation, and original Chinese errors are covered by vectors and failure tests |
| NULL encryption transform | `nullEncryption` | `exact` | Transform ID 11, zero key/IV, four-byte block size, and append-preserving pass-through are restored |
| AES-GCM transform and preparation | `aesGCM`, `preparedGCM` | `exact` | Restores K\|salt parsing, eight-byte explicit IV, twelve-byte nonce, sixteen-byte tag, AAD authentication, and the recovered short-IV zero-fill/long-IV truncation behavior |
| Encryption factories | `GetEncrypter`, `GetEncrypterWithKeyLen` | `exact` | IDs 2/3/11/12/18/19/20, bit-to-byte key sizing, mandatory explicit GCM key length, and original errors match the binary; IDs 18/19/20 are locked as GCM-8/12/16 |
| Cipher preparation and append fallback | `PrepareCipher`, `EncryptTo`, `DecryptTo`, `fallbackPreparedCipher` | `equivalent-proven` | The original encrypter selector and the current additive transform-ID selector share the same prepared implementations while preserving destination prefixes and real errors |
| HMAC PRF registry | `PRF_HMAC_*`, `GetPRF`, `computeHMAC` | `exact` | MD5, SHA-1, SHA2-256/384/512 and their recovered key lengths/IDs match the original globals and factory |
| AES-XCBC PRF/MAC | `xcbcPRF`, `aesXCBCPRF128`, `aesXCBCMAC` | `exact` | Restores short/long key normalization, subkey generation, padding, and truncation; RFC 3566 and RFC 4434 vectors pass |
| `PrfPlus` | Same exported function | `exact` | Restores chained blocks, one-byte counter, exact overflow error, and the original post-block-255 failure boundary |
| FIPS 186-2 SHA-1 PRF | `FIPS1862PRFSHA1` | `exact` | Restores right-aligned XKEY/XSEED, two SHA-1 compression outputs per block, modulo-160-bit state updates, arbitrary output lengths, and stateful generation |
| Integrity algorithms | `IntegrityAlgorithm`, factories and concrete HMAC/XCBC/NULL types | `exact` | IDs, key/output sizes, truncation, verification, and NULL behavior match the original contract |
| HMAC compatibility helpers | `ComputeHMAC`, `VerifyHMAC` | `exact` | Preserves full-digest computation and the recovered expected-prefix slicing semantics |
| Random generation and key erasure | `RandomBytes`, `Wipe` | `exact` | Uses full reads from `crypto/rand`; wipe clears every byte, and SWu key derivation now invokes the shared erasure path |
| Current additive crypto APIs | `Cipher`, `NewCipher`, `SizedPRF`, `NewPRF`, `NewIntegrity` | `equivalent-proven` | Existing stateful/padded callers remain source-compatible without narrowing the restored legacy interfaces |
| Production encryption and derivation paths | SWu IKE/ESP derivation and protection, EAP-AKA, userspace IPsec, and XFRM mapping | `equivalent-proven` | Seal/PRF+ failures propagate through real session paths, EAP-AKA calls the restored FIPS PRF, GCM-16 uses transform 20 end-to-end, and kernel mapping recognizes all three GCM IDs |

## 05 — `engine/eap`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| EAP code, method, subtype, and SIM/AKA attribute constants | Original `Code*`, `Type*`, `Subtype*`, and `AT_*` names in `types.go` | `exact` | Restores AKA/AKA' method IDs, reauthentication subtype 13, and the complete attribute registry through AT_BIDDING; regression tests lock previously incorrect padding/client-error/KDF values |
| `EAPPacket` layout | `EAPPacket{Code, Identifier, Type, Subtype, Data}` | `exact` | Redress projects the same four bytes followed by one byte slice; the rebuilt amd64 layout is locked at 32 bytes |
| `Parse([]byte)` | Variadic-compatible `Parse` | `equivalent-proven` | The original one-argument call and later capacity form both compile; exact short/declared-length errors, AKA header rules, terminal behavior, trailing-buffer handling, and zero-copy data views are restored |
| `(*EAPPacket).Encode` | Same method | `exact` | Restores four-, five-, and eight-byte header selection, big-endian declared length, reserved zeros, and direct type/subtype/data encoding |
| `Attribute` layout and `(*Attribute).Encode` | Same type and method | `exact` | Redress projects Type/Length/Value at the original offsets; encoding updates Length, pads to four-byte words, and leaves zero padding from allocation |
| `ParseAttributes([]byte)` | Variadic-compatible `ParseAttributes` | `equivalent-proven` | Restores map overwrite behavior, zero-copy values, silent trailing single byte, exact zero/overflow errors, and the original slice capacities while retaining the later call form |
| IANA and 3GPP notification tables | `akaNotificationCodeTextsIANA` and `akaNotificationCodeTexts3GPP` | `exact` | Binary map initialization confirms all 6 IANA and 9 3GPP keys; exact bilingual text, source prefix, and unknown S/P-bit formatting are tested |
| `NotificationCodeToString` | Same exported function | `exact` | Known IANA/3GPP and unknown classification output matches the recovered implementation byte-for-byte |
| `ReauthState` source API | Same exported type | `exact` | Restores the original next-ID, counter, MK/TEK/MSK/EMSK state surface retained by the source package |
| `FastReauthContext` layout | Same eight exported fields | `exact` | Redress projects Enabled, ReauthID, Counter, NonceS, CounterSmall, KEncr, KAut, and MK in the same order; amd64 size is locked at 136 bytes |
| `NewFastReauthContext` | Same constructor | `exact` | Returns the original disabled zero-value context |
| `SaveReauthData` | Dual-form dispatcher with original state update | `equivalent-proven` | Original `(string, mk, kEncr, kAut)` retains supplied slices, enables the context, and resets Counter; the later three-slice source form remains callable |
| `CanUseReauth` | Same method | `exact` | Requires both Enabled and a non-empty ReauthID |
| `BuildReauthResponse` | Dual-form dispatcher with original builder | `equivalent-proven` | Original nonce/counter/counter-too-small call updates state and emits AT_COUNTER, optional AT_COUNTER_TOO_SMALL, and zeroed AT_MAC exactly; the later explicit-MAC call remains supported with explicit validation |
| Current compatibility aliases | `EAPAttribute`, `SubtypeAKA*`, and `AttrAT*` | `equivalent-proven` | Existing reconstructed callers retain their names while resolving to the corrected original values and layouts |
| Production packet path | `engine/swu/eapaka` framing and attribute adapter | `equivalent-proven` | Every SWu EAP marshal/parse now uses the restored packet and attribute implementations while retaining ordered, strict AKA validation |
| Production fast reauthentication path | SWu Session fast context, challenge capture, identity selection, MAC verification, response signing, and MSK update | `equivalent-proven` | Config-restored and server-issued contexts reach real IKE_AUTH handling; signed AKA/AKA' requests reject tampering, counter rollback emits AT_COUNTER_TOO_SMALL, and newly issued identities reach the persistence callback |
