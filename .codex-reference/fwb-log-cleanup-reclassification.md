# Framework logging cleanup reclassification

## Executive summary

The supplied patches contain **60 actual diff hunks**: 13 in part 1 and 47 in part 2. The full table has **67 behavior rows** because some individual hunks change more than one diagnostic (notably media manager call sites); the table describes each changed behavior explicitly. Final-newline-only changes are called out within their physical hunk rows and excluded from semantic recommendations. The first cleanup commit (`d711340224b0`) and part 2 (`eb72341fb163`) are already applied in this checkout. Original behavior below is taken from the original commit parents; patched behavior is the checked-out result.

Recommended classification:

- **UNCONDITIONAL: 16 actual hunks** — expected/noisy chatter can be removed, or a forced debug override can be removed.
- **ENG_ONLY: 40 actual hunks** — retain useful diagnostic content behind a class-level eng-only flag.
- **RESTORE_OR_LEAVE_UNCHANGED: 4 actual hunks** — restore tracing and actionable failure reporting, preserve original warning/error severity, or discard formatting-only content.

The 67 behavior rows group into those 60 hunk classifications as follows: 8 unconditional rows, 55 eng-only rows, and 4 restore/leave-unchanged rows. The larger ENG_ONLY row count reflects hunks that touch both a flag and multiple log calls; counts in the concise summary refer to actual patch hunks.

The defining policy is that user and userdebug builds remain quiet by default, while eng builds can recover useful subsystem diagnostics. The old patches often achieve quietness by setting flags to constant `false`; for diagnostics worth retaining, the replacement should set a class-level flag to `Build.IS_ENG` (Java) or its Kotlin equivalent, not leave it false and not enable it broadly on userdebug.

## Full per-hunk classification

`Original behavior` and `Patched behavior` describe the exact relevant line or log path, not unrelated surrounding behavior. “same” in a behavior cell means the hunk only changes a flag which controls existing `if (DEBUG)` messages; no runtime behavior changes.

| Part | File | Symbol | Original behavior | Patched behavior | Classification | Rationale | Recommended final implementation |
|---|---|---|---|---|---|---|---|
| 1 | `apex/jobscheduler/service/java/com/android/server/alarm/AlarmManagerService.java` | `decrementAlarmCountLocked` mismatch branch | `oldCount < decrement` logs `Slog.wtf` with uid/count values. | Same inconsistency logs at `Slog.w`. | RESTORE_OR_LEAVE_UNCHANGED | A negative/mismatched accounting condition is abnormal and useful when debugging alarm accounting. It is not proved to be routine; lowering from fatal-style reporting to warning may be warranted if `wtf` is known to overstate recoverable internal inconsistency, but the evidence does not justify demoting it categorically. | Preserve `Slog.wtf` in the all-build path unless maintainers establish it is recoverable; if there is known nonfatal churn, use the ordinary warning in all builds. Do not gate the condition itself. This hunk as currently patched should not enter A or B without confirming the policy; recommended disposition is restore original severity. |
| 1 | `core/java/android/app/ActivityThread.java` | `acquireProvider` / missing `holder` | If no provider holder is found, logs `Slog.e` when user is unlocked and `Slog.w` when locked, then returns null. | Returns null silently. | ENG_ONLY | Missing provider info can be an expected lookup race or locked-user condition, but unlocked absence can explain app/provider failures and should remain diagnosable. Permanent suppression removes useful runtime context. | Restore the branch and gate its message with `Build.IS_ENG`, retaining the unlocked/locked distinction and original severities. Consider deduplicating only if evidence shows repeated benign callers. |
| 1 | `core/java/android/view/ViewRootImpl.java` | `updatePointerIcon` bounds check | Out-of-view pointer coordinates (comment example: moving a window divider with mouse) emit one debug line and return false. | Returns false silently. | UNCONDITIONAL | The bounds rejection is a deliberate expected input condition and can happen during pointer/window movement. The message states no diagnostic detail beyond the branch already understood from code. | Keep the bounds check and remove the `Slog.d` permanently. |
| 1 | `libs/androidfw/ResourceTypes.cpp` | `ResTable::getResourceName` missing package id | A resource id with no package identifier (`Res_GETPACKAGE(resID)+1 == 0`) emits `ALOGW`; other unknown package ids separately remain `ALOGW`. | Only the no-package-id case emits `ALOGV`; unknown package warning unchanged. | RESTORE_OR_LEAVE_UNCHANGED | A resource id lacking package information can signal malformed resource data or an invalid caller-provided id. The nearby separate unknown-package warning underscores that this is not clearly a routine parse case. Lowering solely for noise risks hiding actionable malformed data. | Restore `ALOGW` for this branch. If later profiling proves this exact sentinel is a common benign probe, add a narrowly scoped native debug control with evidence; do not weaken the warning in this cleanup. |
| 1 | `location/java/com/android/internal/location/GpsNetInitiatedHandler.java` | `DEBUG` | `DEBUG = Log.isLoggable(TAG, Log.DEBUG)`; existing guarded emergency-call/call-state logs can be enabled by loggability. | `DEBUG = false`; those messages are disabled on all builds. | ENG_ONLY | State transitions around emergency calls and telephony are useful GNSS troubleshooting data, but should not be broad production/userdebug chatter. | Set `DEBUG = Build.IS_ENG && Log.isLoggable(TAG, Log.DEBUG)` (or `Build.IS_ENG` if runtime loggability is not needed). |
| 1 | `packages/BackupRestoreConfirmation/src/com/android/backupconfirm/BackupRestoreConfirmation.java` | `DEBUG` | `DEBUG = true`; guarded backup/restore confirmation flow diagnostics are always emitted. | `DEBUG = false`; flow diagnostics disabled. | ENG_ONLY | Backup confirmation is security-sensitive user interaction; useful flow diagnostics belong in engineering builds, while security/error messages remain unconditional. | Set `DEBUG = Build.IS_ENG` and keep existing guards. Do not gate existing invalid-token/action errors. |
| 1 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/BluetoothUtils.java` | `V`, `D` class logging switches | `V=false`, `D=true`; many routine Bluetooth utility state decisions at `Log.d` run throughout builds. | `V=false`, `D=false`; all `D`-guarded messages disabled. | ENG_ONLY | The class’s logs cover route/device/profile decisions and exceptional fallback details; as a group they provide useful engineering context, while normal userdebug should stay quiet. Some individual messages may be non-actionable, but this hunk is only a central flag change and should not erase the whole diagnostic facility. | Use `D = Build.IS_ENG` (and keep `V=false` unless a separate verbose mode is needed); leave errors/warnings outside `D` unchanged. If particular high-frequency messages are proven useless, remove those individually in A. |
| 1 | `packages/SystemUI/src/com/android/systemui/navigationbar/views/buttons/DeadZone.java` | `CHATTY` | `CHATTY=true`; logs when dead zone consumes a click. | `CHATTY=false`; consumed-click messages disabled. | UNCONDITIONAL | A dead-zone click is a normal designed input outcome; repeated input-level messages are hot-path noise. | Keep `CHATTY=false` or remove the low-value message. This is intentional all-build cleanup. |
| 1 | `packages/SystemUI/src/com/android/systemui/statusbar/GestureRecorder.java` | `DEBUG` | `DEBUG=true`; gesture capture diagnostics are emitted. | `DEBUG=false`; gesture diagnostics disabled. | ENG_ONLY | Gesture capture is explicitly useful for reproducing UI/input issues. Suppressing on userdebug loses an engineering aid; enabling by default causes unnecessary input logs. | Set `DEBUG = Build.IS_ENG`, or reuse a deliberate gesture-debug property if one exists. |
| 1 | `packages/SystemUI/src/com/android/systemui/statusbar/phone/ScrimState.java` | `applyState` / `TrackTracer.instantForGroup` | Emits alpha and tint trace counters (`front_scrim_alpha`/`back_scrim_alpha`, `front_scrim_tint`/`back_scrim_tint`) before applying scrim values. | Removes both trace calls and their import. | RESTORE_OR_LEAVE_UNCHANGED | This is performance/animation instrumentation, not logcat chatter. It can support trace-based rendering and jank analysis, and does not itself emit ordinary logs. | Restore both `TrackTracer` calls and import unchanged. |
| 1 | `services/core/java/com/android/server/pm/permission/DefaultPermissionGrantPolicy.java` | `getPackageInfo` `NameNotFoundException` | Logs `Slog.e("Package not found: " + pkg)` for missing requested package, then returns null. | Returns null silently. | ENG_ONLY | Optional/default handlers can legitimately be absent across product configurations; an error for each absence is noisy. But a missing package may also reveal product misconfiguration. Retain a lower-noise engineering diagnostic. | Restore the diagnostic behind `if (Build.IS_ENG) Slog.d(...)`; if absence is an explicitly expected optional-package result in this caller, A may instead remove it after call-site audit. Keep genuine permission-file/configuration warnings elsewhere unchanged. |
| 1 | `services/core/jni/com_android_server_lights_LightsService.cpp` | `processReturn`, `Status::LIGHT_NOT_SUPPORTED` | `ALOGE` for a light type unavailable on this device; other hardware/service failures remain `ALOGE`. | This status logs at `ALOGV`; brightness unsupported and unknown failure remain errors. | UNCONDITIONAL | `LIGHT_NOT_SUPPORTED` is an explicit capability result and may be returned when framework code probes a light that device hardware does not implement. It is expected absence, distinct from command failure and unsupported brightness parameter. | Retain a low-severity capability diagnostic (patched `ALOGV` is reasonable) or remove if call site proves routine. Do not weaken the neighboring real failure logs. |
| 2 | `cmds/incidentd/src/WorkDirectory.cpp` | `WorkDirectory::getReports` `DBG` | Local `DBG=true`; logs entry to report enumeration with `ALOGD`. | `DBG=false`; entry log disabled. | ENG_ONLY | Report enumeration can help diagnose incident collection/storage behavior, but a call-entry trace is low value outside engineering. | Use existing local flag as `constexpr bool DBG = __android_log_is_loggable(ANDROID_LOG_DEBUG, LOG_TAG, ANDROID_LOG_INFO)` combined with an eng build macro if available, or a project-standard native eng macro. Prefer one class/file-level guard, not a one-off wrapped log. |
| 2 | `cmds/incidentd/src/WorkDirectory.cpp` | file ending | Source ends with a final blank line/newline after namespace closures. | Removes that trailing blank line; no behavior change. | RESTORE_OR_LEAVE_UNCHANGED | Formatting-only change is unrelated to logging policy and adds no value. | Restore original end-of-file whitespace; exclude from both commits. |
| 2 | `core/java/android/hardware/SystemSensorManager.java` | `DEBUG_DYNAMIC_SENSOR` | `true`; guarded dynamic-sensor lifecycle/listener logs are emitted. | `false`; those logs disabled. | ENG_ONLY | Dynamic sensor registration/removal/list changes are useful for sensor integration debugging. The existing flag already cleanly centralizes the diagnostics. | Set `DEBUG_DYNAMIC_SENSOR = Build.IS_ENG`; keep existing guards. |
| 2 | `core/java/android/inputmethodservice/navigationbar/DeadZone.java` | `CHATTY` | `true`; logs whenever dead zone consumes a click. | `false`; no consumed-click messages. | UNCONDITIONAL | Same routine input-path chatter as the SystemUI dead zone. | Keep false/remove messages permanently. |
| 2 | `core/java/android/net/SntpClient.java` | `DBG` | `true`; emits request failure details and successful round-trip/reference/originate timing diagnostics in guarded debug logs. | `false`; all these debug messages disabled. | ENG_ONLY | Timing and request-path details are useful for time synchronization diagnosis; actual unknown-host and secure-random failures remain visible at warning/fatal levels. | Set `DBG = Build.IS_ENG`; retain existing guards and all warning/error paths. |
| 2 | `core/java/com/android/internal/util/AsyncService.java` | `DBG` | `true`; logs service start/destroy lifecycle at debug. | `false`; lifecycle diagnostics disabled. | ENG_ONLY | Lifecycle transitions are useful when investigating service dispatch/lifetime issues and are already behind a class flag. | Set `DBG = Build.IS_ENG`; keep existing guards. |
| 2 | `media/java/android/media/MediaRouter.java` | `DEBUG_RESTORE_ROUTE` | `true`; logs route-restore decisions, including selected/restored route and cases skipping A2DP changes. | `false`; route restoration diagnostics disabled; ordinary `DEBUG` remains loggable-controlled. | ENG_ONLY | Route restoration and Bluetooth routing issues need these details during engineering, but no default userdebug noise. | Set `DEBUG_RESTORE_ROUTE = Build.IS_ENG` or `Build.IS_ENG && Log.isLoggable(TAG, Log.DEBUG)`; do not tie it automatically to general `DEBUG` because the dedicated flag intentionally isolates restoration tracing. |
| 2 | `packages/CarrierDefaultApp/src/com/android/carrierdefaultapp/CaptivePortalLoginActivity.java` | `DBG` | `true`; guarded captive portal activity/network state logs emitted. | `false`; these debug messages disabled. | ENG_ONLY | Network availability/loss and portal lifecycle can be important connectivity troubleshooting context; existing centralized `DBG` is suitable. | Set `DBG = Build.IS_ENG`; SSL errors and other real failures stay unconditional. |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/CsipSetCoordinatorProfile.java` | `VDBG` | `true`; emits verbose CSIP group/profile diagnostics. | `false`; verbose diagnostics disabled. | ENG_ONLY | CSIP coordination state is useful for debugging coordinated Bluetooth devices, but verbose. | Set `VDBG = Build.IS_ENG && Log.isLoggable(TAG, Log.VERBOSE)` (or reuse the package Bluetooth debug policy if present). |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/hearingdevices/PresetController.kt` | `registerCallback`, `unregisterCallback`, companion `DEBUG`, final newline | Duplicate registration logs `Log.i`; absent/already-removed callback logs `Log.w`; `DEBUG=true`; no final newline. | Both expected-condition messages become `if (DEBUG) Log.d`; `DEBUG=false`; adds final newline. | ENG_ONLY | The two callback conditions are explicitly expected and their severity should drop, but preserving optional eng visibility fits the class diagnostic policy. The flag also controls useful preset diagnostics elsewhere. Newline is incidental. | Keep both messages at debug under `DEBUG = Build.IS_ENG`; this is not a warning/error loss. Keep final newline out of semantic commits (optionally retain current newline if the file is independently normalized). |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/hearingdevices/metrics/HearingDeviceLocalDataManager.java` | `DEBUG` | `DEBUG=true`; existing guarded hearing-device metrics/data logs enabled. | `DEBUG=false`; those logs disabled. | ENG_ONLY | Device local-state persistence and metrics flows can be hard to troubleshoot; keep behind an eng-only central flag. | Set `DEBUG = Build.IS_ENG`; preserve warnings about unstarted manager and other actionable conditions. |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/hearingdevices/metrics/HearingDeviceLocalDataManager.java` | settings observer `onChange` | Every matching ambient-volume settings notification emits `Log.v` including manager identity, then reloads settings. | Message is wrapped in `if (DEBUG)`; reload still runs. | UNCONDITIONAL | This is a high-frequency successful notification callback, and object identity adds little actionable information. The underlying reload is unchanged. | Remove this particular `Log.v` entirely in Commit A; do not preserve it merely because the class has an eng flag. |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/hearingdevices/metrics/HearingDeviceStatsLogUtils.java` | `DEBUG` | `DEBUG=true`; guarded preference/history bookkeeping warnings are enabled. | `DEBUG=false`; those messages disabled. | ENG_ONLY | These messages explain metrics-history reconciliation, useful when debugging reporting, but should not pollute user/userdebug. | Set `DEBUG = Build.IS_ENG`; preserve the unguarded missing-device-address warning because it reports skipped metrics. |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/hearingdevices/ui/AmbientVolumeUiController.java` | `DEBUG` | `DEBUG=true`; guarded UI/device value transitions emitted. | `DEBUG=false`; diagnostics disabled. | ENG_ONLY | Hearing-device value and UI synchronization traces are useful for engineering integrations. | Set `DEBUG = Build.IS_ENG`; retain unguarded command failure and invalid UI-state warnings. |
| 2 | `packages/SettingsLib/src/com/android/settingslib/bluetooth/hearingdevices/ui/PresetUiController.kt` | companion `DEBUG`, final newline | `DEBUG=true`; file has no final newline. | `DEBUG=false`; adds final newline. | ENG_ONLY | Device loading/preset UI transitions are useful integration diagnostics. Newline is incidental. | Set `DEBUG = Build.IS_ENG`; exclude newline-only formatting from the semantic change or retain only if file normalization is intended. |
| 2 | `packages/SystemUI/plugin/src/com/android/systemui/plugins/keyguard/data/model/WeatherData.kt` | companion `DEBUG` | `DEBUG=true`; guarded weather parse failure and parsed-result logs are active. | `DEBUG=false`; both are disabled. | ENG_ONLY | Parsing behavior is useful when diagnosing keyguard plugin payloads. Malformed weather data is not a platform failure, but debug visibility helps integration work. | Set `DEBUG = Build.IS_ENG`; preserve guards. If the parse-failure message is important for product telemetry, use a separate rate-limited signal rather than broad logcat. |
| 2 | `packages/SystemUI/src/com/android/systemui/controls/controller/ControlsProviderLifecycleManager.kt` | companion `DEBUG` | `DEBUG=true`; service bind/action/timeout lifecycle diagnostics are enabled. | `DEBUG=false`; these diagnostics disabled. | ENG_ONLY | External control-provider binding and action flow are valuable to troubleshoot; existing flag neatly centralizes logs. | Set `DEBUG = Build.IS_ENG`; ensure timeout/failure warnings outside the flag stay visible. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/NotificationPlayer.java` | class `DEBUG` | `DEBUG=true`; existing guarded audio-player lifecycle details enabled. | `DEBUG=false`; the class’s guarded debug details disabled. | ENG_ONLY | Audio playback lifecycle messages help diagnose notification sound behavior. | Set `DEBUG = Build.IS_ENG`; leave playback/load/stop failures and delay warnings unchanged. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/NotificationPlayer.java` | `stopPlayer` null `MediaPlayer` case | `mp == null` emits `Log.w("STOP command without a player")`, then returns. | Emits `Log.d` only if `DEBUG`; then returns. | UNCONDITIONAL | A stop with no current player can occur after queue/state races or already-completed playback; it is explicitly a no-op path, not a hardware/audio failure. | Remove the message permanently, or at most leave it behind eng-only `DEBUG` if incident evidence shows it helps diagnose queue races. Recommended A removal; no warning. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataFilterImpl.kt` | file-level `DEBUG` | `DEBUG=true`; guarded filtering/user/profile change traces emitted. | `DEBUG=false`; traces disabled. | ENG_ONLY | User/profile media-control filtering is useful to debug state migrations; existing flag should enable eng-only tracing. | Set `DEBUG = Build.IS_ENG`; use file-level `private const val`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataManagerImpl.kt` | file-level `DEBUG` | `DEBUG=true`; class’s already guarded media pipeline diagnostics emitted. | `DEBUG=false`; these messages disabled. | ENG_ONLY | State-transition details across sessions, notifications, resume controls and migration are useful troubleshooting information. | Set `DEBUG = Build.IS_ENG`; use existing flag to cover grouped diagnostics. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataManagerImpl.kt` | missing app UID `NameNotFoundException` | Logs `Log.w` with exception, then returns `Process.INVALID_UID`. | Logs only `if (DEBUG) Log.d` with exception, then returns same sentinel. | ENG_ONLY | A media package disappearing between notification and lookup can be an expected race; however package identity/UID resolution failure helps explain missing media controls. Lower severity and keep eng-only. | Keep as `if (DEBUG) Log.d(...)` with `DEBUG = Build.IS_ENG`; behavior unchanged. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataManagerImpl.kt` | null load result | `loadMediaData` null result emits `Log.d`, then returns without update. | Emits same debug only when `DEBUG`. | ENG_ONLY | The null can be a normal loader outcome or a useful explanation for absent controls. It is stateful diagnostic detail, not a warning. | Keep guarded under `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataManagerImpl.kt` | resume load result absent/blank title | Logs debug when no valid resumption data; removes entry and returns. | Message guarded by `DEBUG`; removal unchanged. | ENG_ONLY | Explains why a resume control disappears; useful when diagnosing a stateful media feature. | Keep guarded under `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataManagerImpl.kt` | key-to-package migration | Logs migration boolean and key/package mapping unconditionally at debug. | Guarded by `DEBUG`. | ENG_ONLY | Migration details are valuable debugging context and should be available on eng, not on user/userdebug. | Keep under class `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/LegacyMediaDataManagerImpl.kt` | excess resume control eviction | Logs each control evicted beyond limit; removes entry and notifies listeners. | Message guarded by `DEBUG`; state behavior unchanged. | ENG_ONLY | Eviction can explain disappearing resume controls; class flag provides controlled access without default chatter. | Keep under class `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataFilterImpl.kt` | file-level `DEBUG` | `DEBUG=true`; guarded filter/user transition diagnostics emitted. | `DEBUG=false`; diagnostics disabled. | ENG_ONLY | Filtering and user changes are valuable debugging context, already centralized. | Set `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataProcessor.kt` | file-level `DEBUG` | `DEBUG=true`; existing processor logs and all guarded calls are enabled. | `DEBUG=false`; guarded calls disabled. | ENG_ONLY | Media session/resumption/player lifecycle has high diagnostic value, and the class flag is the right level for policy. | Set `DEBUG = Build.IS_ENG`; do not turn all debug logging on for userdebug. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataProcessor.kt` | missing app UID `NameNotFoundException` | `Log.w` with exception, then `Process.INVALID_UID`. | Eng-flagged `Log.d`, same sentinel. | ENG_ONLY | Expected package-removal race is recoverable, while details help diagnose dropped controls. | Keep `if (DEBUG) Log.d(...)` with `DEBUG = Build.IS_ENG`; runtime behavior unchanged. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataProcessor.kt` | resume result absent/blank title | Logs a debug line, removes media entry and returns. | Message guarded by `DEBUG`; behavior unchanged. | ENG_ONLY | Explains a missing resume control and is useful for feature diagnosis. | Keep under `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataProcessor.kt` | media loader null result | Logs debug before returning without update. | Message guarded by `DEBUG`; behavior unchanged. | ENG_ONLY | Helps explain dropped/absent controls; normal null outcome should not be warning-level. | Keep under `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataProcessor.kt` | key-to-package migration | Logs migration boolean and mapping. | Guarded by `DEBUG`. | ENG_ONLY | Useful state transition diagnostic, not appropriate default user/userdebug chatter. | Keep under class `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDataProcessor.kt` | excess resume control eviction | Logs each control eviction; eviction behavior unchanged. | Guarded by `DEBUG`. | ENG_ONLY | Explains why resume controls are removed; retain as eng diagnostic. | Keep under class `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/media/controls/domain/pipeline/MediaDeviceManager.kt` | file-level `DEBUG` | `DEBUG=true`; guarded route/device update messages enabled. | `DEBUG=false`; route diagnostics disabled. | ENG_ONLY | Media routing/device selection diagnostics are useful when investigating cast/Bluetooth route issues. | Set `DEBUG = Build.IS_ENG`. |
| 2 | `packages/SystemUI/src/com/android/systemui/navigationbar/NavigationModeController.java` | class `DEBUG` | `DEBUG=true`; guarded navigation mode/resource/configuration diagnostics emitted. | `DEBUG=false`; diagnostics disabled. | ENG_ONLY | Useful when debugging overlay/navigation mode selection; existing class flag is clean. | Set `DEBUG = Build.IS_ENG`; preserve unguarded actual errors. |
| 2 | `packages/SystemUI/src/com/android/systemui/statusbar/NotificationShadeDepthController.kt` | `onZoomOutChanged` | `TrackTracer.instantForGroup("shade", "zoom_out", value)` and `Log.v` per zoom-out update; then optional model update. | Trace remains; per-update `Log.v` removed. | UNCONDITIONAL | The log mirrors continuous animation values on a hot path and duplicates the structured trace that remains. This is exactly high-frequency logcat noise; tracing is separately preserved. | Keep `TrackTracer` and remove only `Log.v`, as patched. |
| 2 | `packages/SystemUI/src/com/android/systemui/statusbar/policy/FlashlightControllerImpl.java` | class `DEBUG` | `DEBUG=true`; guarded torch state/strength and availability logs active. | `DEBUG=false`; those logs disabled. | ENG_ONLY | Torch state transitions are useful hardware integration diagnostics. Actual camera access failures remain unguarded. | Set `DEBUG = Build.IS_ENG`; retain unconditional camera/hardware errors. |
| 2 | `packages/SystemUI/src/com/android/systemui/wallpapers/data/repository/WallpaperRepository.kt` | companion `DEBUG` | `DEBUG=true`; guarded wallpaper command/focal-area logs active. | `DEBUG=false`; those diagnostics disabled. | ENG_ONLY | Wallpaper command and focal-area state are useful to diagnose repository integration. | Set `DEBUG = Build.IS_ENG` (or shared eng-only flag); leave failures unaffected. |
| 2 | `services/core/java/com/android/server/location/contexthub/NanoAppStateManager.java` | `ENABLE_LOG_DEBUG` | `true`; guarded nanoapp instance add/update verbose logs active. | `false`; verbose messages disabled. | ENG_ONLY | Context-hub nanoapp mapping is specialized hardware integration state, useful to engineers. The actual max-limit error remains visible. | Set `ENABLE_LOG_DEBUG = Build.IS_ENG`; retain existing guards and errors. |
| 2 | `services/core/java/com/android/server/media/MediaSessionService.java` | `DEBUG_KEY_EVENT` | General `DEBUG` uses `Log.isLoggable`; key event diagnostics are separately always enabled with a comment to leave them on. | `DEBUG_KEY_EVENT = DEBUG`; key event diagnostics now follow runtime loggability. | ENG_ONLY | Key dispatch sequence details are useful for media-button routing issues, but the old exception made them noisy in every build. The change correctly reuses class policy conceptually, but policy must also exclude userdebug. | Set common `DEBUG = Build.IS_ENG && Log.isLoggable(TAG, Log.DEBUG)` and `DEBUG_KEY_EVENT = DEBUG` (or a distinct eng-only key event switch if this path needs separate activation). |
| 2 | `services/core/java/com/android/server/notification/ScheduleConditionProvider.java` | class `DEBUG` | `true || Log.isLoggable(...)`, so all guarded schedule-provider lifecycle/evaluation logs are always on. | Only `Log.isLoggable(...)` enables them. | UNCONDITIONAL | Removing the constant-true override is a correctness/policy fix: it restores the explicitly written runtime loggability control and quiet-by-default behavior without removing the diagnostic option. This is not yet eng-only because userdebug could enable it through loggability. | Keep forced-true removal in A; for final policy, make it `Build.IS_ENG && Log.isLoggable("ConditionProviders", Log.DEBUG)` in B or use the project’s intended eng-only diagnostic gate. For strict two-commit accounting, put the final combined expression in B and regard removal of `true ||` as part of that change; no noisy logs remain on user/userdebug. |
| 2 | `services/core/java/com/android/server/wallpaper/WallpaperCropper.java` | `DEBUG_CROP` | `DEBUG_CROP=true`, independent of `DEBUG=false`; crop decode/size/record path verbose messages active. | `DEBUG_CROP=false`; record path log also explicitly wrapped in `if (DEBUG_CROP)`. | ENG_ONLY | Crop sizing/decoder decisions are useful to investigate wallpaper rendering; file/record paths are also operational details. A central flag should make them available in eng only, not remove the diagnostic trail. | Set `DEBUG_CROP = Build.IS_ENG`; retain the new explicit guard around record-path log so the source has consistent flag use. Do not conflate with `DEBUG` if crop tracing needs separate control. |
| 2 | `services/core/java/com/android/server/wallpaper/WallpaperCropper.java` | record file path log | After creating crop record file, logs record path/name unconditionally at verbose. | Emits only under `DEBUG_CROP`; file creation behavior unchanged. | ENG_ONLY | Path/name can help diagnose crop bookkeeping but is not normally needed. | Keep under `DEBUG_CROP = Build.IS_ENG`. |
| 2 | `services/core/java/com/android/server/wallpaper/WallpaperManagerService.java` | `DEBUG_LIVE` | `DEBUG_LIVE=true`; live wallpaper diagnostic messages enabled while general `DEBUG=false`. | `DEBUG_LIVE=false`; live diagnostics disabled. | ENG_ONLY | Live wallpaper attach/connection/state diagnostics help troubleshoot services and callbacks. | Set `DEBUG_LIVE = Build.IS_ENG`; keep meaningful service failure logs unconditional. |
| 2 | `telephony/java/android/telephony/data/DataServiceCallback.java` | `DBG` | `DBG=true`; guarded callback event logs emitted for setup/deactivate/list/handover/unthrottle callbacks. | `DBG=false`; callbacks still delivered but logging disabled. | ENG_ONLY | Callback ordering and data-call state are useful telephony integration diagnostics, not routine production output. | Set `DBG = Build.IS_ENG`; preserve any callback errors outside this flag. |

## Proposed contents of Commit A

Subject: `base: remove non-actionable framework log spam`

Include these all-variant changes:

- Part 1 `ViewRootImpl.updatePointerIcon`: remove expected pointer-out-of-bounds debug log.
- Part 1 SystemUI navigation `DeadZone.CHATTY`: disable/remove routine consumed-click chatter.
- Part 2 input-method navigation `DeadZone.CHATTY`: same cleanup.
- Part 2 `PresetController.registerCallback`: remove info log for explicitly expected duplicate registration.
- Part 2 `PresetController.unregisterCallback`: remove warning for explicitly expected missing/already-removed callback.
- Part 2 `HearingDeviceLocalDataManager.onChange`: remove hot-path ambient-volume observer log; keep reload.
- Part 2 `NotificationPlayer.stopPlayer`: remove “STOP command without a player” from routine no-player return path.
- Part 2 `NotificationShadeDepthController.onZoomOutChanged`: remove per-animation `Log.v`; preserve `TrackTracer` instrumentation.
- Part 1 `LightsService.processReturn`: demote `LIGHT_NOT_SUPPORTED` to `ALOGV` (expected unsupported capability); leave all other hardware failures at their current severities.
- Part 2 `ScheduleConditionProvider`: remove unconditional `true ||` override so logging is no longer forced on. The final build policy still needs the eng gate in B.

Do not put every `true -> false` change in A: most are retained diagnostics that belong in B with eng-only flags.

## Proposed contents of Commit B

Subject: `base: gate diagnostic framework logging to eng builds`

Set the existing diagnostic switches to eng-only for the following hunks. Keep the existing call-site guards and meaningful warnings/errors:

- Part 1 `GpsNetInitiatedHandler.DEBUG`.
- Part 1 `BackupRestoreConfirmation.DEBUG`.
- Part 1 `BluetoothUtils.D`.
- Part 1 `GestureRecorder.DEBUG`.
- Part 1 `ActivityThread.acquireProvider`: restore missing-provider log and gate eng-only.
- Part 1 `DefaultPermissionGrantPolicy.getPackageInfo`: restore missing-package diagnostic at debug under eng gate (or keep its current removal if product-config audit establishes it is exclusively optional).
- Part 2 `WorkDirectory::getReports` local `DBG`.
- Part 2 `SystemSensorManager.DEBUG_DYNAMIC_SENSOR`.
- Part 2 `SntpClient.DBG`.
- Part 2 `AsyncService.DBG`.
- Part 2 `MediaRouter.DEBUG_RESTORE_ROUTE`.
- Part 2 `CaptivePortalLoginActivity.DBG`.
- Part 2 `CsipSetCoordinatorProfile.VDBG`.
- Part 2 `PresetController.DEBUG`; keep duplicate/missing callback messages removed under A.
- Part 2 `HearingDeviceLocalDataManager.DEBUG`; keep observer hot-path message removed under A.
- Part 2 `HearingDeviceStatsLogUtils.DEBUG`.
- Part 2 `AmbientVolumeUiController.DEBUG`.
- Part 2 `PresetUiController.DEBUG`.
- Part 2 `WeatherData.DEBUG`.
- Part 2 `ControlsProviderLifecycleManager.DEBUG`.
- Part 2 `NotificationPlayer.DEBUG`; the no-player stop message remains removed under A.
- Part 2 `LegacyMediaDataFilterImpl.DEBUG`.
- Part 2 `LegacyMediaDataManagerImpl.DEBUG` and its UID failure downgrade to gated debug; keep null-result/migration/eviction diagnostics guarded.
- Part 2 `MediaDataFilterImpl.DEBUG`.
- Part 2 `MediaDataProcessor.DEBUG` and its UID failure downgrade to gated debug; keep result/migration/eviction diagnostics guarded.
- Part 2 `MediaDeviceManager.DEBUG`.
- Part 2 `NavigationModeController.DEBUG`.
- Part 2 `FlashlightControllerImpl.DEBUG`.
- Part 2 `WallpaperRepository.DEBUG`.
- Part 2 `NanoAppStateManager.ENABLE_LOG_DEBUG`.
- Part 2 `MediaSessionService.DEBUG` and `DEBUG_KEY_EVENT`; use one eng-only common policy.
- Part 2 `ScheduleConditionProvider.DEBUG`; final form should include `Build.IS_ENG` as well as optional loggability.
- Part 2 `WallpaperCropper.DEBUG_CROP` and record-path guard.
- Part 2 `WallpaperManagerService.DEBUG_LIVE`.
- Part 2 `DataServiceCallback.DBG`.

Suggested Java pattern: `private static final boolean DEBUG = Build.IS_ENG;`. Where a class already supports loggability, use `Build.IS_ENG && Log.isLoggable(TAG, Log.DEBUG)` so engineers can still opt into a subset. Do not rely on `Log.isLoggable` alone when the policy requires userdebug to be quiet.

## RESTORE / LEAVE UNCHANGED

These changes should not survive in either replacement commit:

- Part 1 `ScrimState`: restore both removed `TrackTracer.instantForGroup` calls and the import. They are structured tracing instrumentation, not log spam.
- Part 1 `ResourceTypes.cpp`: restore `ALOGW` for a resource id with no package identifier. It may indicate malformed resource input; the patch provides no evidence that it is routine.
- Part 1 `AlarmManagerService`: restore original `Slog.wtf` pending validation that this internal count mismatch is a recoverable condition. If code owners confirm it is nonfatal, a deliberate `Slog.w` change could be included in A, but do not conceal it or convert it to an eng-only message.
- Part 2 `WorkDirectory.cpp` trailing blank-line removal: discard as unrelated formatting.

The final-newline changes are included in the corresponding combined hunk rows because they are physically part of those patch hunks, but they have no semantic classification of their own and should be omitted when reconstructing the commits.

## Ambiguous cases

| Case | Competing considerations | Recommendation | Confidence |
|---|---|---|---|
| Alarm count mismatch `wtf -> w` | The alarm decrement branch is guarded by a detected counter underflow, so `wtf` may imply an unrecoverable invariant violation; however it is an abnormal accounting condition and the patch supplies no frequency or recovery evidence. | Restore `wtf` for now. Reclassify to an all-build warning only with owner evidence that the condition is recoverable and non-actionable at fatal-style severity. | Medium |
| `ActivityThread` provider not found | Locked-user and lookup races can be expected; unlocked provider absence can also break app startup/content access. Silent return obscures the reason. | Restore, retain unlocked/locked severities, and eng-gate. | Medium-high |
| `DefaultPermissionGrantPolicy` package absent | Optional handlers may be absent across products; a missing mandatory configured package may reveal a product error. | Keep quiet on user/userdebug; restore as eng-only debug until callers/configuration are audited. | Medium |
| `LIGHT_NOT_SUPPORTED` error downgrade | An unsupported light is a normal capability result; repeated probes can make error-level logs noisy. Some callers may still request an assumed light, but genuine command/brightness failures remain errors. | Treat this status as expected and keep `ALOGV`; do not downgrade other statuses. | High |
| `ResourceTypes` missing package id | Invalid resource ids may be probed during fallback paths; absence of package bits can also be malformed data and the adjacent unknown-package case is warned. | Restore warning absent call-site frequency evidence; use a targeted native debug control only if proven benign. | Medium |
| `NotificationPlayer` stop without player | Queue races or completed playback make this a likely no-op; unexpected state could reveal queue ordering bugs. | Remove as routine no-op in A; if reproduced as a bug signal, use eng-only diagnostic in a follow-up. | Medium |
| `PresetController` callback registration exceptions | Comments explicitly say already-registered / not-registered is expected; nevertheless repeated unexpected API exceptions could aid debugging. | Remove both messages unconditionally; exception is intentionally idempotent and not a failure. | High |
| `ScheduleConditionProvider` flag change | Removing `true ||` restores the authored loggability control and quiet default; but loggability alone can still enable logs on userdebug, against the target policy. | Preserve removal of forced true, then add `Build.IS_ENG` condition in B. | High |
| `BluetoothUtils.D` and hearing-device flags | Some messages are status breadcrumbs that can aid Bluetooth feature support; some individual logs may be noisy or expose device state. | Prefer eng-only class flags now; separately remove demonstrably non-actionable/high-frequency individual messages after call-site validation. | Medium |

## Java eng-only implementation guidance

- For an existing class-level flag whose guarded messages are all useful diagnostics, prefer `private static final boolean DEBUG = Build.IS_ENG;` (or the established visibility/name). This gives the desired user and userdebug quietness and retains eng visibility.
- If a class currently uses `Log.isLoggable`, preserve opt-in while constraining the build policy: `Build.IS_ENG && Log.isLoggable(TAG, Log.DEBUG)`. `Log.isLoggable` by itself does not ensure userdebug is quiet.
- Reuse a class-level flag rather than wrapping each call. Keep severity and all-build error paths outside the flag.
- For isolated logs without a suitable flag, use an existing subsystem debug flag if semantically aligned. Otherwise add one class-level eng-only flag only if multiple diagnostics or a coherent diagnostic category need it. Do not add a one-off flag for a message that belongs in A and should simply disappear.
- Preserve useful exception detail, but lower `W`/`E` only when the condition itself is expected/recoverable. A debug flag must never hide real errors, security events, crashes, hardware failure, watchdog evidence, or abnormal states needing production action.

## Kotlin eng-only implementation guidance

- For Kotlin classes, use `private const val DEBUG = Build.IS_ENG` at file scope where all related top-level/class logs share policy, or a companion `private const val DEBUG = Build.IS_ENG` for a class-level policy. Import `android.os.Build`.
- Retain the existing `if (DEBUG)` call-site blocks; do not add wrappers around unrelated logs.
- For an existing runtime loggability switch, use `Build.IS_ENG && Log.isLoggable(TAG, Log.DEBUG)` where Kotlin constant initialization permits it; otherwise use a `private val` initialized with that expression.
- Kotlin `const val` requires a compile-time constant. `Build.IS_ENG` is commonly used as a static build constant in platform Java, but if the Kotlin compiler rejects a const initializer in a given module, use `private val DEBUG = Build.IS_ENG`.
- Keep interpolation inside `if (DEBUG)` blocks, as the existing hunks do, so potentially expensive string construction is skipped when disabled.

## Native/C++ guidance

- Do not introduce `Build.IS_ENG` into native code. For useful C++ diagnostics, use the repository’s available build variant macros or existing local logging controls, with a file/class-level debug boolean where that is already the convention.
- For `WorkDirectory::getReports`, replace the unconditional local `DBG=true` with a native eng-only compile-time/runtime gate that is supported by this target (for example an existing `LOG_NDEBUG`/debug macro or a platform-provided eng build macro); confirm the exact macro is available to this module before selecting it. Keep the call behind one `if (DBG)`.
- For `ResourceTypes.cpp`, retain the warning because the condition may reflect malformed resource data. Do not move actionable warnings to verbose just to reduce volume.
- For `LightsService.cpp`, `Status::LIGHT_NOT_SUPPORTED` is a capability report, not a failed set command; `ALOGV` is appropriate if this is useful at all. Keep `ALOGE` for failed commands, unsupported brightness parameters, and unknown errors. If the capability probe itself is too frequent, remove the log or use a native eng-only guard.

## Final proposed commit messages

### Commit A

**Subject:** `base: remove non-actionable framework log spam`

**Body:**

```text
Remove routine successful-state and expected-condition messages that
add no actionable information to production logs. Drop high-frequency
pointer, gesture, callback, media-stop, and animation chatter while
preserving structured tracing and meaningful failure reporting.

Treat unsupported light capabilities as verbose diagnostics, and stop
forcing schedule-condition debug logging on in every build.
```

### Commit B

**Subject:** `base: gate diagnostic framework logging to eng builds`

**Body:**

```text
Keep useful framework, SystemUI, media, Bluetooth, sensor, telephony,
wallpaper, and native troubleshooting messages available on eng builds
without enabling broad debug output on user or userdebug builds.

Reuse class-level diagnostic flags, combine runtime loggability with the
eng build check where appropriate, and preserve actionable warnings,
errors, security events, hardware failures, and crash diagnostics.
```

## Concise summary

- Total hunks reviewed: **60** (13 in part 1; 47 in part 2)
- UNCONDITIONAL: **16**
- ENG_ONLY: **40**
- RESTORE_OR_LEAVE_UNCHANGED: **4**
- Detailed behavior rows: **67** (8 unconditional, 55 eng-only, 4 restore/leave unchanged)
- Ambiguous cases: alarm count mismatch severity; missing provider info; missing optional/default package; unsupported light capability; missing resource package id; NotificationPlayer stop without player; expected preset callback exceptions; ScheduleConditionProvider forced-true removal; Bluetooth/hearing-device class debug coverage.
- Proposed Commit A subject: `base: remove non-actionable framework log spam`
- Proposed Commit B subject: `base: gate diagnostic framework logging to eng builds`
- Report: `.codex-reference/fwb-log-cleanup-reclassification.md`
