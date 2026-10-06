# CS203XL/CS203X/CS463 HTTP API CS Demo (C#)

A C# Windows Forms reference client for **CSL (Convergence Systems Limited)** fixed RFID readers — **CS203XL**, **CS203X**, and **CS463**. It exercises the reader's high-level HTTP API and runs an internal TCP listener for instant / batch tag notifications.

Maintained by Convergence Systems Limited — https://www.convergence.com.hk/

## Requirements

- **Hardware:** a CS203XL / CS203X / CS463 reader reachable on the network. The demo cannot be exercised end-to-end without one.
- **OS:** Windows with **.NET Framework 4.7.2**.
- **IDE:** Visual Studio (the project file loads cleanly in VS 2026).
- **Default connection** (overridable on first run):
  - Reader URL: `http://192.168.25.208/`
  - Username: `root`
  - Password: `csl`
  - Notification listener TCP port: `9090`

## Build

Open `CS463_HL_CS.sln` in Visual Studio and build:

| Configuration | Output |
| --- | --- |
| `Debug \| Any CPU`   | `bin\Debug\CS463_HL_CS.exe` |
| `Release \| Any CPU` | `bin\Release\CS463_HL_CS.exe` |

There is no command-line build script; use `msbuild` or `dotnet build` against `CS463_HL_CS.csproj` if you prefer the shell.

## Run

1. Launch the built executable. The main window loads with the default URL and credentials.
2. On first run, edit the **URI** field to point at your reader (any change is saved to the Windows registry under `HKCU\Software\Convergence Systems Limited\CS463_HL_CS`).
3. Update **Username** / **Password** if your reader uses non-default credentials.
4. Use the menu (Reader ID, Operation Profile, Trigger, Resultant Action, Event, Trusted Server, Tag Group, UCODE DNA, Capture Point Name) to open the feature dialogs.
5. On the main window:
   - **Start Read Alert** opens the TCP listener on the chosen port and displays tags pushed by the reader in real time.
   - **Start Polling Trigger** polls the reader at a fixed interval and reads all selected memory banks in one go.
   - **Save Tags to File** exports the in-memory tag table to CSV.

## What it covers

The demo drives a large subset of the reader's HTTP API surface:

- Connection management (`isOnline`, `login`, `forceLogout`, `logout`, `setAccessMode`, `getReaderStatus`, `healthCheck`).
- Reader configuration (Reader ID, Operation Profile, Capture Point Name, Trusted Server list).
- Inventory modes (polling trigger, autonomous trigger, instant alert, batch alert).
- Tag capture (EPC, Bank0, Bank2, Bank3, PC bits, RSSI, frequency).
- Tag groups (create, add/delete members, bulk import via CSV or list).
- Triggering logic, resultant actions, event configuration, user management.
- NXP UCODE DNA (read / write / activate key, authenticate).

Two notification channels are supported by the embedded `TrustedServer` TCP listener:

- **Instant Alert** — every tag is shown as soon as it arrives (`cmd=evtNtf`).
- **Batch Alert** — tags accumulate in memory and are flushed when the reader sends `batchEnd=yes`.

## Logs

- API HTTP traffic: `%USERPROFILE%\AppData\...\<exe folder>\api.log`.
- TCP listener traffic: `<current directory>\tcp.log`.

## Project layout

```
CS463_HL_CS.sln
CS463_HL_CS.csproj
Program.cs                       Application entry point -> frmMain
frmMain.cs (+ .Designer/.resx)   Main window, tag table, polling timer
CS463_HL_API.cs                  All API methods, TrustedServer, and structs (~5,700 lines)
frmReaderID.cs                   Dialog: Reader ID
frmOperationProfile.cs           Dialog: Operation Profile
frmCapturePointName.cs           Dialog: Capture Point Name
frmTrustedServer.cs              Dialog: Trusted Server list
frmTrigger.cs                    Dialog: Triggering Logic
frmResultantAction.cs            Dialog: Resultant Actions
frmEvent.cs                      Dialog: Event
Form_NewTagGroup.cs              Dialog: New Tag Group
Form_AddMember.cs                Dialog: Add Tag Group Member
Form_DelMember.cs                Dialog: Delete Tag Group Member
Form_DelAllMember.cs             Dialog: Delete All Members
Form_ImportMembers.cs            Dialog: Import Members
Form_UCODEDNA.cs                 Dialog: UCODE DNA
changelog.txt                    Release notes (newest entry at the top)
LICENSE                          MIT License
```

## License

MIT — see `LICENSE`.