# Security, performance, and cross-platform packaging audit

Date: 2026-03-23

## Scope

This report covers the current `kiro-claude-proxy` repository, its relationship to `bestK/kiro2cc`, and the work needed to turn it into a secure, easy-to-install package for Windows, macOS, and Linux.

## Translated summary of `bestK/kiro2cc`

The upstream `bestK/kiro2cc` README describes a **Go command-line tool for managing Kiro auth tokens and providing an Anthropic API proxy service**. Its documented features are:

- read a token file from the user's AWS SSO cache
- refresh the token
- export environment variables
- start an HTTP server that proxies Anthropic-style requests

That translation matters because this repository has the same overall purpose and command surface.

## Is this project using `bestK/kiro2cc` as a Go package?

**No.** This repository does **not** import `github.com/bestk/kiro2cc` as a module dependency.

- Local `go.mod` declares only `module github.com/alexandeism/kirolink` and has no external requirements (`/home/runner/work/kiro-claude-proxy/kiro-claude-proxy/go.mod:1-3`).
- The codebase contains no import of `github.com/bestk/kiro2cc`.

## Does this project look copied or adapted from `bestK/kiro2cc`?

**Yes, it looks adapted from it rather than consumed as a package.**

The strongest signals are structural and naming similarities between `bestK/kiro2cc/main.go` and local `kirolink.go`:

- same command set: `read`, `refresh`, `export`, `claude`, `server`
- same token location: `~/.aws/sso/cache/kiro-auth-token.json` (`kirolink.go:840-849`)
- same core data structures and translation goal: `TokenData`, `AnthropicRequest`, `CodeWhispererRequest`, request/response translation
- same overall execution model: read local token, proxy Anthropic-compatible requests, translate for CodeWhisperer

So the current repo should be treated as a **fork/adaptation lineage**, not as a package consumer.

## 1) Does `kiro-claude-proxy` work for Windows users who can only install the Kiro IDE?

**Not reliably as-is.**

### Short answer

- **Partially yes** for the basic proxy flow, **if** the Kiro IDE already populates `~/.aws/sso/cache/kiro-auth-token.json`, because the main token path uses `os.UserHomeDir` + `filepath.Join` (`kirolink.go:840-849`).
- **No, not as a polished Windows-ready package** for users who only have the IDE and no extra tooling, because key workflows still assume Unix-style or CLI-specific behavior.

### Why it is not Windows-ready yet

1. `refresh` depends on the **Kiro CLI SQLite database**, not just the IDE token cache (`kirolink.go:890-933`).
2. `refresh` shells out to an external `sqlite3` executable (`kirolink.go:897-898`), which many Windows users will not have installed.
3. `getKiroDBPath` only special-cases macOS; Windows falls through to the Linux/XDG-style path `~/.local/share/kiro-cli/data.sqlite3` (`kirolink.go:876-887`).
4. The helper command `claude` assumes `~/.claude.json` exists, so it is only useful when Claude Code is already installed (`kirolink.go:980-1015`).

### Practical conclusion

Today the repo is best described as **developer-oriented and experimentally cross-platform**, not as an easy Windows package for “Kiro IDE only” users.

## 2) Security audit

### Highest-priority findings

| Severity | Finding | Evidence | Risk | Recommended fix |
| --- | --- | --- | --- | --- |
| High | Proxy listens on all interfaces by default | `http.ListenAndServe(":"+port, mux)` (`kirolink.go:1208`) | Exposes a token-backed local proxy to the LAN instead of localhost only | Bind to `127.0.0.1` by default and make non-local bind opt-in |
| High | Sensitive data is logged to stdout | full token printing in `readToken` (`kirolink.go:867-872`), partial token printing in `refreshToken` (`kirolink.go:932-933`), full request body logging (`kirolink.go:1106`), full upstream response logging (`kirolink.go:1724`), SSE event logging (`kirolink.go:1758-1759`) | Token leakage, prompt leakage, tool-input leakage, accidental credential disclosure in CI/terminal logs | Default to redacted logs; add debug mode for explicit opt-in diagnostics |
| High | Unbounded request-body read | `io.ReadAll(r.Body)` (`kirolink.go:1098`) | Memory exhaustion and trivial DoS from oversized requests | Wrap with `http.MaxBytesReader` and reject oversized bodies |
| Medium | No HTTP server timeouts | `http.ListenAndServe` with no configured `http.Server` (`kirolink.go:1208`) | Slowloris/resource exhaustion risk | Use an explicit `http.Server` with `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` |
| Medium | No outbound HTTP client timeout | `client := &http.Client{}` in both request paths (`kirolink.go:1574`, `kirolink.go:1706`) | Hanging upstream calls can stall requests indefinitely | Configure `Timeout` or a custom `Transport` with dial/TLS/response-header timeouts |
| Medium | Panic details are returned to clients | `Internal panic: %v` in HTTP error body (`kirolink.go:1151`) | Internal details may leak to clients | Return a generic error and log a redacted internal incident ID instead |
| Medium | External command dependency for token refresh | `exec.Command("sqlite3", dbPath, ...)` (`kirolink.go:897-898`) | Fragile on Windows, extra supply-chain/runtime dependency, harder error handling | Read SQLite directly from Go using a maintained driver or read a documented IDE-owned cache instead |
| Low | Hardcoded upstream region endpoint | `https://codewhisperer.us-east-1.amazonaws.com/generateAssistantResponse` (`kirolink.go:1560`, `kirolink.go:1583`, `kirolink.go:1692`) | Regional lock-in, harder failover and testing | Make upstream base URL configurable with safe defaults |

### Additional security notes

- Token file writes already use mode `0600` (`kirolink.go:927`), which is a good baseline.
- The repo currently has **no external dependencies**, which reduces supply-chain surface area, but it also means missing standard ecosystem tools for packaging and vulnerability scanning.
- The current logging style is the largest immediate confidentiality problem because the proxy handles prompts, tool inputs, and credentials.

## 3) Performance and operability audit

| Priority | Finding | Evidence | Impact | Recommended fix |
| --- | --- | --- | --- | --- |
| High | “Streaming” path buffers the full upstream response before emitting events | `respBody, err := io.ReadAll(resp.Body)` inside `handleStreamRequest` (`kirolink.go:1647-1656`) | Added latency, higher memory use, not true end-to-end streaming | Parse and forward events incrementally from the upstream body |
| Medium | Heavy debug printing on hot paths | request, response, and event logging in `kirolink.go:1106`, `1724`, `1758-1759` | Throughput loss, large logs, secret leakage | Replace with leveled, redacted logging |
| Medium | Request and response handling are fully in-memory | `io.ReadAll` on inbound and outbound bodies (`kirolink.go:1098`, `1648`, `1717`) | Higher memory pressure under concurrent use | Stream where possible and cap request sizes |
| Medium | `/v1/models` response order is map-iteration order | models built directly from `ModelMap` iteration (`kirolink.go:1171-1185`) | Non-deterministic output; minor UX/test friction | Sort model IDs before encoding |
| Low | Build script is Windows-specific and assumes UPX | `build.bat` uses `upx --best --lzma` | Not reproducible across platforms and may fail on systems without UPX | Move release packaging into CI and make compression optional |

## 4) Research-backed best practices relevant to this project

### Go and local proxy security

- Use explicit HTTP server/client timeouts and avoid unbounded body reads for resilience and DoS protection. OWASP's Go secure coding guidance and current Go security references align on this direction.
- Do not log secrets, raw prompts, or tool payloads by default; use redaction and opt-in debug logs.
- Prefer secure-by-default local binding (`127.0.0.1`) for developer proxies.

### Cross-platform filesystem behavior

- Prefer OS-native data/config directories instead of ad-hoc path assumptions:
  - Go provides `os.UserConfigDir`, `os.UserCacheDir`, and `os.UserHomeDir`.
  - Linux should follow XDG base-directory conventions.
  - Windows should use `APPDATA`/`LOCALAPPDATA`-style locations rather than Linux fallback paths.
- Avoid shelling out to `sqlite3`; use a Go SQLite library if SQLite remains the source of truth.

### Packaging and distribution

- For a Go CLI with Windows/macOS/Linux targets, the standard release path is **GoReleaser** plus package-manager integrations.
- Homebrew, Chocolatey, and Linux packages should be generated from signed release artifacts with checksums.
- Version metadata should be embedded at build time and exposed through a `--version` command.

## 5) Recommended project plan

### Phase 0 - stabilize the current app

1. Bind the server to `127.0.0.1` by default.
2. Add request size limits and HTTP timeouts.
3. Remove sensitive default logging; replace with redacted debug logging behind a flag or env var.
4. Stop returning raw panic text to clients.

### Phase 1 - make token handling truly cross-platform

1. Replace `sqlite3` shelling with direct SQLite access from Go.
2. Detect Kiro token/config/database locations using OS conventions:
   - Windows: `LOCALAPPDATA`/`APPDATA`
   - macOS: `~/Library/Application Support`
   - Linux: `XDG_DATA_HOME` or `~/.local/share`
3. Separate “IDE token cache” support from “CLI database refresh” support and document both clearly.
4. Add explicit diagnostics that tell the user which token source was found and which one failed.

### Phase 2 - improve UX and reliability

1. Add `--listen`, `--port`, `--upstream-url`, and `--log-level` flags.
2. Add a `--version` command and embed build metadata.
3. Make streaming actually stream.
4. Add deterministic `/v1/models` output and clearer health/config diagnostics.

### Phase 3 - package for Windows, macOS, and Linux

1. Add a release pipeline based on GoReleaser.
2. Publish:
   - Homebrew tap for macOS
   - Chocolatey package for Windows
   - `.deb` and `.rpm` packages for Linux
3. Generate checksums for every artifact.
4. Sign release artifacts and package repositories.

### Phase 4 - harden and maintain

1. Add CI for `go test ./...`, `go vet`, and vulnerability scanning.
2. Add smoke tests for token discovery and startup behavior on Windows, macOS, and Linux.
3. Add integration tests for streaming and non-streaming proxy paths.
4. Add a support matrix in the README describing exactly which features work with:
   - Kiro IDE only
   - Kiro CLI only
   - Kiro + Claude Code

## Suggested target architecture for a production package

- **Language:** Go is still a good fit for this project because it produces single-binary releases and works well for local proxies.
- **CLI structure:** Keep the current commands, but consider `cobra` only if the command surface grows. For the current size, a small stdlib CLI is still acceptable.
- **Config:** support flags + environment variables first; optionally add a config file later.
- **Logging:** structured, redacted, opt-in debug logs.
- **Transport:** reusable `http.Client` with timeouts and a configurable upstream base URL.
- **Storage:** prefer documented token sources; if SQLite is required, access it natively from Go.

## Bottom line

The repository already proves the proxy concept, but it is **not yet a secure, polished cross-platform package**. The main blockers are:

1. sensitive logging
2. non-local default bind
3. missing HTTP safety limits/timeouts
4. Windows-incompatible refresh behavior
5. release/distribution automation not yet in place

Once those are addressed, Go remains a strong implementation choice for shipping a single-binary proxy across Windows, macOS, and Linux.
