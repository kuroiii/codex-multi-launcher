# MSIX identity launch (local experimental patch)

## Proven input vs. new implementation

The user's 2026-09-27 Windows experiment on OpenAI.Codex 26.924.2738.0 produced fresh SQLite, config and Chromium files in separate test directories after temporary HKCU CODEX_HOME / CODEX_SQLITE_HOME changes, Environment broadcast, and IApplicationActivationManager activation with --user-data-dir.

That result is the basis for this patch. It is NOT a test of this implementation, concurrent instances, long-term isolation, automatic self-relaunch, all plugin behavior, or future application builds.

## Launch transaction

Acquire the existing profile-operation gate, then the common per-user MSIX environment file lock. Recover a prior incomplete environment transaction before proceeding. Take raw registry snapshots, durably write the recovery journal, change only CODEX_HOME and CODEX_SQLITE_HOME, broadcast WM_SETTINGCHANGE, allow a propagation grace interval, and activate the registered package. No IPackageDebugSettings, runtime copy, direct executable fallback, --no-sandbox or singleton-disabling flag is used.

A new launch must return a previously unseen PID. Verify the exact registered package full name with GetPackageFullName, read the process command line using WMI, and require the expected absolute --user-data-dir. Poll for new file metadata in BOTH the expected Codex Home and Electron profile. Existing files alone are insufficient. The nominal verification timeout is 45 seconds; native Windows calls and per-window broadcast timeouts can extend wall time.

Restore the original raw registry values and value kinds (or remove values which were originally absent), broadcast again, and only then return a successful launch result. An in-memory snapshot supports ordinary exception cleanup; a disk journal supports recovery on the next launcher start after forced termination. This does not provide an immediate watchdog after a hard kill.

The environment restore is compare-before-write. If the current value is neither the temporary value nor the saved original, preserve that external edit and retain the journal. This is intentionally a diagnostic stop, not an automatic registry overwrite.

## Important limitations

1. This modifies per-user Windows environment, not just a child process. The lock coordinates copies of this patched launcher only. An unrelated Codex/CLI started while the transaction is active can inherit the temporary home. Do not launch from the taskbar, Start, another terminal or an older launcher during this interval. Other Windows sessions/cached environment consumers and application self-relaunch are not guaranteed isolated.
2. File activity + PID/arguments/package checks are evidence of the selected local paths, not a complete audit of backend credentials, plugins, SQLite handles or every subprocess. Run the manual multi-account/concurrency acceptance test before real work. An independent existing daemon may write files; file metadata is not a process-attributed file-access trace.
3. MSIX activation may reuse an existing application process. A new-profile request that returns an existing PID is rejected, not registered against the new profile. A deep link that returns a different PID is reported as unverified; the app may already have received the activation. This cannot undo an activation delivered by Windows.
4. WMI-unreadable or custom unregistered desktop instances are classified as unresolved and block managed launches and mutations. Close the temporary CodexMultiIdentityTest* windows before using this patch. PID records are matched with process creation time and executable path; directory recovery requires the registry and profile marker to agree.
5. Some global credential/endpoint overrides are rejected without logging their values, because MSIX does not inherit the old ProcessStartInfo.Environment scrub. The patch does not delete global API keys to work around this. Existing per-profile auth.json/config files remain untouched by the launch service.
6. Focus does not relaunch a process if the window cannot be raised. Use its tray/taskbar entry after the environment transaction has ended. Runtime cache icon badges are not supported by this path.
7. Successful startup is not a promise of future Codex compatibility. No arbitrary downgrade/reinstall or profile deletion is performed.

## Read-only state recovery

ProcessInventory keeps IsRuntimeCache as the legacy physical-path flag, and IsManaged also recognizes ProfileId obtained from an exact registered Electron directory. Empty ProfileId explicitly means unresolved. Unknown processes are never treated as definitely personal. The UI derives state but no longer saves its normalization during polling; actual mutations still hold the original operation gate.

## Tests and validation status

The added xUnit tests cover Windows argument quoting (5 cases), control-character rejection (3), two accepted flag forms, absent/duplicate/relative arguments, old-file readiness rejection, registry restoration decisions including REG_EXPAND_SZ/absent values, preservation of concurrent external edits, unresolved ownership, and directory/registry/marker agreement: 16 test cases total.

These tests do NOT activate Codex or edit real HKCU environment. They need Windows for CommandLineToArgvW. This preparation environment lacks Windows/.NET SDK, so neither compilation nor those tests were run here. The installer runs the repository's restore/build/test and rolls source back on failure. The manual acceptance checklist is in the patch README-FIRST.md.

## Provenance

Inspected baseline: kuroiii/codex-multi-launcher-local at b87e4fa1e11ce5a6bd3e99ff8a61b36e9aae753a; source originally by yyyyyp233, MIT license retained. The shipped ProfileCoordinator baseline matched uploaded Git blob dadcb85aa795b321c12e47bdb0d654811eaee00e byte-for-byte.

Primary references consulted:
- `https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nf-shobjidl_core-iapplicationactivationmanager-activateapplication`
- `https://learn.microsoft.com/en-us/windows/win32/procthread/environment-variables`
- `https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-settingchange`
- `https://learn.microsoft.com/en-us/windows/win32/api/appmodel/nf-appmodel-getpackagefullname`
- `https://www.nuget.org/packages/System.Management/10.0.10`
- `https://developers.openai.com/codex/environment-variables`

Remote mutation status: branch creation returned 403 Resource not accessible by integration. No branch, commit or pull request was created by this patch preparation. The zip contains source changes and local application scripts only, not third-party application binaries, profiles or credentials.
