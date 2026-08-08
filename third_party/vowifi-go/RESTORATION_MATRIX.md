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

## 06 — `engine/ikev2`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| `IKEHeader`, `Encode`, header decoding and string form | `IKEHeader`, `(*IKEHeader).Encode`, `DecodeHeader`, and `String` | `exact` | Restores the 28-byte big-endian header, SPI integers, payload/exchange types, flags, message ID, and declared length; an exact byte vector locks the wire format |
| `IKEPacket`, `NewIKEPacket`, `(*IKEPacket).Encode`, and `DecodePacket` | Same symbols in `packet.go` | `equivalent-proven` | Valid-packet framing and chaining match exactly; the restored path also preserves explicit encoding failures and rejects malformed declared lengths instead of accepting ambiguous trailing/truncated input, then stops correctly at an outer SK payload |
| `Payload`, `PayloadHeader`, and `RawPayload` | Same original body-only interface and structures | `exact` | Concrete payloads encode only their bodies; the packet layer adds four-byte generic headers, while unknown payload bodies remain available as raw bytes |
| KE and nonce payloads | `EncryptedPayloadKE`, `EncryptedPayloadNonce`, and structured decoders | `exact` | DH group/reserved bytes and nonce bodies match recovered encodings; decoded fields are used directly by SWu key exchange |
| ID, AUTH, and EAP payloads | `EncryptedPayloadID`, `EncryptedPayloadAuth`, and `EncryptedPayloadEAP` | `exact` | Restores initiator/responder type selection, reserved bytes, authentication method framing, and EAP body framing with exact byte tests |
| Notify payload and notification names | `EncryptedPayloadNotify`, `DecodePayloadNotify`, and `NotifyTypeToString` | `exact` | Protocol/SPI/type/data framing and the recovered error/status name table are restored, including distinct unknown error and status formatting |
| Delete payload | `EncryptedPayloadDelete` and `DecodePayloadDelete` | `exact` | Restores protocol, SPI size/count, exact SPI bytes, and strict length checks; SWu now emits zero-SPI IKE deletes and one four-byte CHILD_SA SPI correctly |
| CP payload and attributes | `EncryptedPayloadCP`, `CPAttribute`, `DecodePayloadCP`, and `decodeCPAttribute` | `exact` | CFG type, reserved bytes, TLV attributes, ordering, and truncated-value errors match the original packet model |
| `ParseCPConfig`, `HasIPv4`, and `HasIPv6` | Same symbols and original address slices | `exact` | IPv4/IPv6 address, DNS, P-CSCF, and IPv6 prefix extraction preserve source order; regression tests cover both address families |
| TS payload, selectors, and constructors | `EncryptedPayloadTS`, `TrafficSelector`, `NewTrafficSelectorIPV4`, `NewTrafficSelectorIPV6`, and `DecodePayloadTS` | `exact` | Restores TSi/TSr selection, IPv4/IPv6 range widths, ports, protocol byte, selector count, and malformed/trailing-byte rejection |
| SA, Proposal, Transform, and TransformAttribute | Same original structures and encode/decode methods | `exact` | Proposal/transform chaining, SPI and transform counts, and TV/TLV key-length attributes match RFC 7296 and recovered byte vectors |
| Multi-proposal IKE/ESP builders | `CreateMultiProposalIKE` and `CreateMultiProposalESP` original SPI form | `exact` | Restores four ordered IKE and four ordered ESP proposals, including AES key lengths, integrity/PRF families, DH choices, AEAD handling, and ESN transforms |
| Proposal matching | `ProposalMatcher`, `DefaultProposalMatcher`, and `SelectBestProposal` | `equivalent-proven` | Selection follows configured local priority per transform class, distinguishes IKE and ESP requirements, and accepts AEAD without a separate integrity transform |
| `CalculateNATDetectionHash` | Original four-argument form in `nat.go` | `exact` | Computes SHA-1 over network-order SPIi, SPIr, IP bytes, and port; a fixed digest vector is locked at `d798d986143f878f70765e0e869c80bbc375f701` |
| IKE and CHILD SA key containers | `IKESAKeys` and `ChildSAKeys` | `exact` | Restores the original exported key-slice field surface used by the protocol state machine |
| Current additive compatibility surface | Superset compatibility fields, range helpers, algorithm config builders, and checked payload-chain helpers | `equivalent-proven` | Existing reconstructed callers remain source-compatible where Go permits; ambiguous legacy helpers dispatch explicitly, and the impossible first-type-free decode form returns a real error instead of fabricating a payload chain |
| Production SWu packet path | IKE_SA_INIT, IKE_AUTH, CREATE_CHILD_SA, rekey, DPD, and delete paths | `equivalent-proven` | Production constructors use the restored header and payload fields, responses decode into structured payloads, proposal/TS validation reads original fields, and packet/payload encoding failures propagate to callers |

## 07 — `engine/driver`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| `NetToolError{Op, Args, Err}`, `Error`, and `Unwrap` | Same exported type and methods in `nettools.go` | `exact` | Restores both original Chinese formatting branches and error unwrapping; regression tests lock both forms |
| `NetTools` construction and link operations | `NewNetTools`, `GetLink`, `SetLinkUp`, `SetLinkDown`, `SetMTU`, and `DeleteLink` | `equivalent-proven` | Original calls retain their behavior; an additive optional default interface preserves the interim bound-tool call form without changing explicit legacy operations |
| IPv4/IPv6 address lifecycle | `AddAddress`, `DelAddress`, `AddAddress6`, and `DelAddress6` | `exact` | Restores netlink parsing, contextual errors, IPv6 `IFA_F_NODAD`, five attempts at 80 ms, and the original IPv6 delete delegation |
| Routes and policy-routing operations | `AddRoute*`, `DelRoute*`, `AddRule`, `DelRule`, `AddInputRule`, `DelInputRule`, `FlushRules`, and `CleanConflictRoutes` | `equivalent-proven` | Original route/table/rule construction, EEXIST/ESRCH handling, family selection, stale-rule cleanup, and conflict-route matching are retained; checked cleanup is additive |
| Sysctl helpers and runtime IPv6 enablement | `sysctlPath`, `readSysctlValue`, `SetSysctl`, and `EnsureIPv6Enabled` | `equivalent-proven` | Binary strings and valid `/proc/sys` key construction match; all/default/interface values accept only 0/1, writes are read back, changed keys are returned, permission/unsupported errors remain explicit, and path separators/traversal keys are rejected |
| `NetTxn` immediate operation and rollback model | `Begin`, operation methods, `Commit`, and `Rollback` | `exact` | Operations execute immediately, inverse closures are retained, commit clears them, and rollback executes every inverse in reverse order while joining errors; tests lock ordering and clearing |
| Additive policy-route transaction support | `AddRouteTable`, `AddRule`, `AddInputRule`, and `EnsureIPv6Enabled` on `NetTxn` | `equivalent-proven` | The production TUN/XFRMI path applies routes and sysctls transactionally and restores changed IPv6 keys; checked rule cleanup exposes every rollback failure |
| `TUNDevice` layout and lifecycle | `TUNDevice`, `NewTUNDevice`, `Read`, `Write`, `Close`, and `DeviceName` | `equivalent-proven` | Uses the original `songgao/water` TUN, requested-name pre-delete, kernel-assigned name, direct packet IO, and real close behavior; unlike the legacy best-effort pre-delete, non-not-found failures propagate before creation |
| XFRM algorithm descriptors and mapping | `XFRMCryptAlgo`, `XFRMAuthAlgo`, `XFRMAeadAlgo`, and `IKEv2AlgToXFRM*` | `exact` | DES/3DES/AES-CBC/AES-CTR, MD5/SHA families, GCM/CCM salt and ICV lengths, defaults, unsupported errors, and AEAD classification are locked by table tests |
| XFRM SA and SP configuration layouts | `XFRMSAConfig` and `XFRMSPConfig` | `exact` | Field order and netlink types follow the recovered source, including NAT-T ports, limits, replay window, SA direction, ESN, template SPI, and interface ID |
| XFRM interface, SA, and compatibility installation | `AddXFRMInterface`, `DelXFRMInterface`, `AddSA`, `DelSA`, and `addStateCompat` | `equivalent-proven` | Restores same-name cleanup, optional parent link, replay default 32, tunnel AF_UNSPEC, limits/algorithms/encapsulation, idempotent not-found/ESRCH deletes, and ordered EINVAL retries; interface lookup/delete errors that legacy code ignored now propagate |
| XFRM policy and update behavior | `AddSP`, `DelSP`, `UpdateSA`, `UpdateSP`, `buildXfrmState`, and `buildXfrmPolicy` | `exact` | Uses policy update semantics, IPv4 template normalization on add, update replay default 128, recovered add/update algorithm differences, and explicit update failures; Linux-only builder tests lock the structures |
| XFRM inspection and cleanup | `FlushAll`, `FlushByIP`, `GetSALastUsed`, `Cleanup`, and `UndoFuncs` | `exact` | Original public best-effort/void contracts remain; additive checked variants aggregate errors, and production uses checked reverse-order cleanup rather than hiding failures |
| Linux and non-Linux split | Platform-tagged netlink, TUN, XFRM, and socket-facing adapters | `equivalent-proven` | Linux builds against the repository's original `iniwex5/netlink` fork; Darwin tests compile and non-Linux operations return explicit unsupported errors instead of fake success |
| Production SWu driver path | `DataplaneModeTUN`, `DataplaneModeXFRMI`, TUN packet loops, XFRM install, and policy-route setup | `equivalent-proven` | The default userspace mode is unchanged; explicit TUN creates a real device and bridges ESP, while XFRMI resolves the kernel outbound route, enables UDP encapsulation, installs interface/SAs/SPs/routes, and rolls back every completed step on failure or shutdown |

## 08 — `engine/ipsec`

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| `SecurityAssociation` layout and constructors | `SecurityAssociation`, `NewSecurityAssociation`, and `NewSecurityAssociationCBC` | `equivalent-proven` | Recovered exported and private field order is locked by amd64 offset tests; the legacy encrypter/integrity forms and current transform-ID forms resolve to the same prepared ciphers without changing existing callers |
| ESP sequence and cipher preparation | `NextSequenceNumber`, `reserveSequenceNumber`, and `cipher` | `equivalent-proven` | Prepared ciphers are cached at construction as in the binary; the current path additionally surfaces preparation failure and rejects sequence exhaustion rather than wrapping to a reused nonce |
| ESP encryption and layout | `Encapsulate`, `EncapsulateInto`, `EncapsulateWithNextHeaderInto`, and `EncapsulationLayout` | `equivalent-proven` | Legacy and current call forms share the same SPI/sequence/IV/padding/next-header/ICV wire layout; destination prefixes are preserved and CBC integrity excludes those prefixes exactly |
| ESP authentication and replay | `Decapsulate` and `DecapsulateWithNextHeaderInto` | `equivalent-proven` | CBC and GCM packets authenticate and decrypt with the recovered errors and next-header result; strict padding and a synchronized 64-packet replay window reject tampering, duplicates, and stale sequences before delivery |
| NAT-T IKE classification | `parseIKEPayload` | `exact` | Port-4500 non-ESP markers are stripped while four-byte NAT keepalives and ESP packets remain distinguishable; capacity ownership behavior is covered by packet tests |
| UDP address resolution | `ResolveUDPAddrAll` and resolver helpers | `equivalent-proven` | Restores direct IP/system DNS/custom DNS resolution, three-second bounds, candidate deduplication, IPv4 preference, and explicit resolution errors; a real local DNS server test verifies configured-server use |
| `SocketManager` construction and public layout | `NewSocketManager` and `SocketManager` | `exact` | Restores device/local/remote/DNS arguments, recovered channel capacities, exported channel fields, candidate addresses, and key amd64 offsets; empty local address binds an ephemeral wildcard socket |
| UDP 500/4500 send and receive | `Start`, `readLoop`, `SendIKE`, `SendESP`, and `SendNATKeepalive` | `equivalent-proven` | Uses a real reusable UDP socket, source allowlisting, ordered candidate retry, RFC 3948 marker insertion on 4500, one-byte keepalive, exact short-write failures, and owned receive buffers |
| NAT rebinding and socket statistics | `reportPortChange`, `Stats`, and packet delivery helpers | `exact` | A changed accepted peer port atomically updates the remote endpoint and emits the recovered event fields; IKE/ESP received and dropped counters are synchronized and regression-tested |
| Socket lifecycle and raw descriptor surface | `Start`, `Stop`, `RawFD`, and address/accessor methods | `equivalent-proven` | Start and stop are concurrency-safe and idempotent, terminal channels close once, start-after-stop is explicit, and the real descriptor and resolved address reach production callers |
| Linux UDP encapsulation | `SetUDPEncap`, `DisableUDPEncap`, and platform `setUDPEncap` | `equivalent-proven` | Linux programs `UDP_ENCAP_ESPINUDP` through the real socket; the additive disable operation supports transactional XFRM rollback, while non-Linux returns an explicit unsupported error |
| Linux ICMP error queue | `ParseSockExtError`, `startErrorListener`, and `netEventFromExtendedError` | `equivalent-proven` | Restores the native extended-error structure, IPv4/IPv6 `RECVERR`, nonblocking `MSG_ERRQUEUE` draining, PMTU updates, and unreachable events; malformed/control errors surface and non-Linux support is explicit |
| SOCKS5 protocol encoding | Handshake, username/password auth, request/reply, address, and UDP datagram helpers | `exact` | Restores method ordering, RFC 1928 IPv4/IPv6/domain frames, reply mapping, UDP fragment rejection, and exact destination/data representation; byte-level tests cover every address family |
| `Socks5Transport` construction and public layout | `NewSocks5Transport`, connection/association helpers, and `Socks5Transport` | `equivalent-proven` | Restores config fields, recovered queue capacities and key offsets, real TCP handshake/UDP associate, unspecified relay-address replacement, and the original control-local UDP association address; the interim constructor form remains source-compatible |
| SOCKS5 transport IO, lifecycle, and statistics | `Start`, `Stop`, `sendUDP`, `readLoop`, `tcpKeepalive`, and `Stats` | `equivalent-proven` | A real local proxy integration test proves bidirectional IKE/ESP framing, marker handling, NAT keepalive, fragment drops, all ten counters, synchronized shutdown, and control-connection failure events |
| Production SWu transport path | `swu.Transport`, IKE exchange, data plane, rekey, NAT keepalive, and runtime host construction | `equivalent-proven` | The restored direct and SOCKS transports satisfy the recovered interface; every production send returns real failures through retransmit/control/data-plane loops, and runtime requests now propagate the device ID |

## 09 — `engine/swu`

### 09.1 — initialization and proposals

| Legacy symbol or behavior | Current mapping | Status | Evidence |
| --- | --- | --- | --- |
| `NegotiationError` and algorithm policy constants | Original fields/constants plus additive `Context` compatibility field | `equivalent-proven` | `Class`, `Reason`, and `Retryable` retain the recovered prefix and exact `<class>: <reason>` behavior; strict, balanced, legacy-prefer, error-class constants, and nil receiver are regression-tested |
| `buildAlgorithmPlan`, normalizers, and legacy allowlist | Same symbols in `algorithm_policy.go` | `exact` | Policy only gates DES/3DES as in the binary; it no longer replaces negotiation with a fabricated single fixed suite, and effective algorithm labels retain sorted original values |
| IKE/ESP configured proposal parsers | `parseIKEProposal`, `parseESPProposal`, and primitive parsers | `exact` | AES-128/256, GCM-16, DES/3DES, SHA families, explicit/derived PRF, MODP/ECP, normalization, policy rejection, and original error branches match the validated v0.2 source |
| IKE/ESP capability filters and cloning | Original filter/clone symbols in `proposal_filter.go` | `equivalent-proven` | Unsupported encryption, PRF, integrity, and DH transforms are removed through the restored crypto factories; required transform classes and non-AEAD integrity are enforced, with explicit nil rejection added at malformed boundaries |
| `buildIKEProposals` | Same original config/SPI/profile-offset signature | `equivalent-proven` | Empty configuration emits the recovered four ordered proposals; configured strings retain order, retry offsets slice and renumber them, and the current explicit transform-ID API maps to one equivalent proposal |
| `buildESPProposals` | Same original config/SPI signature | `equivalent-proven` | Empty configuration emits the recovered four ordered ESP proposals, including GCM-256 first; configured ordered proposals, SPI ownership, AEAD integrity omission, and capability filtering are covered |
| Proposal summaries and DH prioritization | `summarizeIKEProposal`, `firstDHGroupFromProposals`, and `prioritizeDHGroup` | `exact` | Recovered lowercase profile labels, default DH group, stable non-DH ordering, and preferred-DH-first behavior are locked by tests |
| `detectOutboundIPv4` | Same IP/port signature plus hostname adapter for current configuration | `exact` | Uses a real two-second UDP dial and requires an IPv4 local address; the production transport resolver invokes it after hostname resolution |
| COOKIE state and retry | `ErrCookieRequired`, `handleCookie`, and `buildIKESAInitPacket` | `exact` | COOKIE is copied, marked, emitted as the first payload, and retried with identical SPIi, Ni, and DH public key; errors are real and the original Chinese sentinel is restored |
| IKE_SA_INIT request construction | Original byte-returning `buildIKESAInitPacket` and object helper | `equivalent-proven` | Production emits ordered COOKIE/SA/KE/Ni/NAT-D/fragmentation payloads, preserves configured multi-proposal order, generates real random SPI/nonce/DH material, and propagates every encode/random/DH failure |
| IKE_SA_INIT response handling | Original byte-taking `handleIKESAInitResp` and structured helper | `equivalent-proven` | Mandatory payloads, COOKIE, NO_PROPOSAL, INVALID_KE, REDIRECT, fragmentation, NAT-D, selected transforms, DH secret, and IKE key derivation are validated; selected fallback proposals update the real crypto parameters |
| INVALID_KE, proposal fallback, and redirect production loop | `ErrInvalidKEGroup`, `advanceIKEProfileOffset`, `selectRequestedDHGroup`, and `Connect` redirect path | `equivalent-proven` | INVALID_KE creates fresh DH/SPI/nonce material, retryable negotiation errors advance the ordered profile, and redirects replace the actual endpoint before the next transport is built |
| Production proposal path | `runIKESAInit` and initial IKE_AUTH CHILD_SA offer | `equivalent-proven` | The real connect loop invokes the restored byte builders/handlers; IKE_AUTH now offers the restored ordered ESP set while rekey helpers retain the already-negotiated single transform set |
