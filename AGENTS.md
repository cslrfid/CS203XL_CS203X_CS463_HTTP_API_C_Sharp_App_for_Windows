# AGENTS.md

Guidance for AI coding agents working in this repo.

## What this is

A single-project C# Windows Forms demo app (`CS463_HL_CS.sln`) that talks to CSL CS203XL / CS203X / CS463 fixed RFID readers over HTTP. Shipped by Convergence Systems Limited (https://www.convergence.com.hk/) as a reference client. Not a library, not a service — it is a desktop GUI that wraps the reader's high-level HTTP API and runs a TCP listener for instant/batch tag notifications.

## Project shape

- One project, one solution. Root namespace `CS463_HL_CS`, assembly `CS463_HL_CS`, `OutputType=WinExe`, `TargetFrameworkVersion=v4.7.2`.
- `Program.cs` — entry point; starts `frmMain`.
- `frmMain.cs` (+ `.Designer.cs` + `.resx`) — main window, tags DataGridView, menu of feature dialogs, polling timer, registry-backed settings.
- `CS463_HL_API.cs` — **everything else**. ~5,700 lines in one file containing:
  - `class CS463_HL_API` (HTTP client, login/session, all reader commands).
  - `class TrustedServer` (async TCP listener for reader-pushed notifications).
  - `StateObject`, all `EventArgs` and `*EventHandler` delegates.
  - All data types: `TAG`, `READER_ID`, `OPERATION_PROFILE`, `SERVER_INFO`, `TAG_GROUP_INFO`, `TAG_FILTER_INFO`, `TRIGGER_INFO`, `RESULTANT_ACTION_INFO`, `EVENT_INFO`, `USER_INFO`, `WRITETAG_INFO`, `INVENTORY_NTF_INFO`, `GPI_*`, `EPC`, `LOG_LEVEL`, `API_MODE`, `READER_STATUS`, `HEALT_INFO`, etc.
  - All other `Form_*` / `frm*` files are thin dialogs that construct their own `CS463_HL_API` and call a handful of methods. No business logic to extract.
- No external NuGet packages. Only `System.*` references.
- No tests, no CI, no lint/format config. `.vs/`, `/obj`, `/obj/Debug` are gitignored; **`/bin` is NOT gitignored** — be careful when staging.

## Build

Open `CS463_HL_CS.sln` in Visual Studio (user is on VS 2026; the .sln header is still VS 16 format, which loads fine). Configurations: `Debug|Any CPU` → `bin\Debug\CS463_HL_CS.exe`, `Release|Any CPU` → `bin\Release\`. No command-line build is wired up — use `msbuild`/`dotnet build` against the .csproj if needed.

## Runtime assumptions the agent will get wrong

- **Hardware required.** The app cannot run end-to-end without a CSL reader reachable on the network. Every feature dialog calls `reader.connect()` which performs `isOnline → login → forceLogout → login → getAccessMode/setAccessMode`. Without a real reader at the configured URI, expect "Cannot connect to reader" or login failures.
- **Default URI** baked into every Form's `loadUserSettings()`: `http://192.168.25.208/`. Change via the `URI` textbox on `frmMain` (saved to registry on edit).
- **Default credentials** in every form: `root` / `csl`. Changeable via registry keys `LoginName` / `LoginPassword`.
- **Default TCP port** for the notification listener: **9090**. Editable on `frmMain` (`txtAlertPort` / `txtPollingPort`); saved to registry.
- **Session ID.** Almost every API call requires a non-empty `session_id`. `CS463_HL_API.connect()` is the only path that handles login + session. If a new code path calls other methods directly, it must login first.
- **HTTP timeout default** is 30 s (`http_timeout` property, registry key `HttpTimeout`).

## Three different HTTP transports live in `CS463_HL_API.cs`

Don't assume one shape. They are:

1. Standard GET — `http://<host>/API?command=<cmd>&session_id=<sid>&...` used by ~95% of methods.
2. **Web-server access-mode workaround** — `setAccessMode_WS()` hits `http://<host>/setAccessMode?oper=setAccessMode&access_mode=...` (note: different endpoint, no `API?command=`). Used only as a workaround for `2.2.0 SP18 RC43` firmware when `setAccessMode` fails.
3. **POST form-encoded** — `importTagGroupCSV()` POSTs `application/x-www-form-urlencoded` to `http://<host>/importTagGroupCSV` with `session_id`, `tagGroupFilename`, `tagGroupContent` in the body. It bypasses `sendHTTPRequest()` and opens its own `HttpWebRequest`.

The reader replies are XML rooted at `<CSL>`. Successful acks start with `OK:` (or `OK`); errors come as `<CSL><Error msg="..." code="..."/></CSL>` or `<CSL><Ack>Error: ...</Ack></CSL>`. `isCSLResponse()` + `parseErrorCode()` handle this; `sendHTTPRequest()` accepts any TLS cert via `ServicePointManager.ServerCertificateValidationCallback`.

## `TrustedServer` (notification listener)

Async TCP server in the same file. Reads newline-delimited `cmd=evtNtf&...`, `cmd=blogNtf&...`, `cmd=errNtf&...`, `cmd=antNtf&...` lines from the reader and fires .NET events (`TagReceiveEvent`, `TagListEvent`, `BatchEnd`, `AntennaEvent`, `BufferOverflowEvent`, `InventoryEvent`, `GpiEvent`). `frmMain` wires these events and pushes tag data into a `DataTable` (primary key on `Tag ID`) with cross-thread `BeginInvoke`. `Stop()` calls `serverThread.Abort()` — that's the only shutdown.

`frmMain` has two modes that share the same port: **Instant Alert** shows every received tag live, **Batch Alert** waits for `batchEnd=yes` and flushes the buffer. They are mutually exclusive in the UI (radio buttons disable each other).

## Logging

- `CS463_HL_API.saveToLog()` → `Application.StartupPath\api.log`.
- `TrustedServer.saveToLog()` → `Environment.CurrentDirectory\tcp.log`.
- Log levels in `LOG_LEVEL` class; set via `reader.api_log_level` or the `cbLogLevel` combobox on `frmMain` (registry key `LogLevel`).

## Settings storage

All persistent UI state lives in `Application.UserAppDataRegistry` (HKEY_CURRENT_USER\Software\Convergence Systems Limited\CS463_HL_CS\...\<version>\). There are **no `app.config`/`Settings.settings` round-tripped values at runtime** — `Properties\Settings.settings` exists but is empty. Keys in use: `URI`, `LoginName`, `LoginPassword`, `HttpTimeout`, `TcpPort`, `LogLevel`, `AutoPollingEnable`, `PollingPeriod`.

## Conventions

- `CopySource` returns a single `CS463_HL_API` instance per form; forms are short and pattern-uniform: `loadUserSettings` → `connect()` → call API → show `error_msg` on failure → `logout()`. Follow the same shape when adding dialogs.
- API methods follow a strict template: build `sbReq` URL string, `sendHTTPRequest`, parse XML, return `false`/`null` + populate `ErrorCode`/`ErrorMsg`. Add new commands by mirroring this template; do not introduce `HttpClient` or async patterns — the codebase is `WebRequest` + sync parsing on purpose.
- The string `1.0.0.2` in `ApiVersion` and the assembly `1.0.0.6` in `Properties\AssemblyInfo.cs` are intentionally out of sync; do not "fix" one without checking `changelog.txt`.
- `changelog.txt` is the only release notes file. New entries go at the top, dated, with a one-line bullet per change. The top entry is currently `v1.0.0.2 — change PC time to reader time`.

## What NOT to touch casually

- The `Application.UserAppDataRegistry` keys — they are the user's saved credentials and connection. Renaming breaks upgrades.
- The hardcoded `192.168.25.208` default — it is intentional for the demo; the user overrides it on first run.
- The TLS-cert-bypass callback in `sendHTTPRequest()` — readers commonly have self-signed certs; removing it will break HTTPS to most devices.
- The `// for CS463` comment on `data_format_id=CS461 TCP Format` in `addResultantAction`/`modResultantAction` — it is correct, do not "modernize" it to "CS463".