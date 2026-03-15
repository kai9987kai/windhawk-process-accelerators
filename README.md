# Force Process CPU/GPU Preferences

A Windhawk mod that applies per-process hardware and scheduling policy on Windows.
It can tune CPU placement, priority, memory behavior, Windows graphics settings,
and best-effort ONNX Runtime NPU preferences from a single ruleset.

This is aimed at workloads that benefit from different system policy than the
Windows default, such as games, emulators, media tools, background services,
and AI apps.

## Features

- CPU affinity
- Topology-aware CPU Set selection
- CPU priority class
- Dynamic CPU priority boost
- CPU execution-speed throttling / EcoQoS
- Timer resolution control
- Working set locking or trimming
- Hybrid CPU preference helper
- I/O priority
- Page priority
- Memory priority
- Windows per-app GPU preference
- Auto HDR toggle
- Windowed Optimizations toggle
- Variable Refresh Rate (VRR) toggle
- Conditional matching based on other running processes
- Running-process picker inside Windhawk
- Best-effort ONNX Runtime NPU modes

## Important NPU Limitation

This mod cannot force an arbitrary Windows application to start using the NPU.
Windows does not expose a generic external switch for that.

The NPU functionality here is intentionally narrow:

- It only affects apps that load `onnxruntime.dll`.
- It only changes new ONNX Runtime session options created after the hook is active.
- It can prefer NPU-capable execution providers or append supported providers.
- It cannot invent NPU support for apps, models, drivers, or provider stacks that do not already support it.

## Installation

1. Install Windhawk.
2. Create a new mod from source in Windhawk.
3. Paste the contents of [`force-process-accelerators.wh.cpp`](force-process-accelerators.wh.cpp).
4. Compile and install the mod.
5. Configure one or more rules in the Windhawk settings UI.

## How Rule Matching Works

Each rule matches one of:

- An executable name such as `notepad.exe`
- A full path such as `C:\Apps\Foo\foo.exe`
- A wildcard pattern such as `*\foo.exe`
- The special token `@picked`, which resolves to the last value chosen in the running-process picker

The first enabled matching rule wins.

Rules can also be gated by other running processes:

- `whenProcessesRunning`: the rule matches only if at least one listed process is running
- `whenProcessesNotRunning`: the rule matches only if none of the listed processes are running

These conditions are evaluated when the target process starts or when Windhawk reloads the settings.

## Running-Process Picker

Windhawk settings do not provide a native live process dropdown, so the mod
includes a small picker hosted inside `windhawk.exe`.

Use it like this:

1. Toggle `picker.launch` on.
2. Select a running process from the list.
3. The chosen value is copied to the clipboard and stored as `@picked`.

`picker.output` controls whether `@picked` stores:

- `full-path`
- `exe-name`

If you use `match: @picked`, restart the target app or reload/touch the settings
after choosing a new process so the rule is evaluated again.

## Settings Overview

### CPU and Scheduling

- `cpuCores`: `unchanged`, `all`, or a comma-separated list/range such as `0-3,8`
- `cpuSets`: `unchanged`, `all`, `performance`, `efficiency`, or explicit CPU Set IDs such as `0,4-7`
- `cpuSetNumaNode`: optional NUMA node filter for CPU Sets
- `cpuSetLastLevelCache`: optional last-level-cache filter for CPU Sets
- `cpuSetLimit`: optional cap on the number of surviving CPU Sets
- `cpuSetPlacement`: `compact` for locality or `spread` for contention avoidance when `cpuSetLimit` trims a larger set
- `cpuSetAvoidSmt`: keep one logical CPU Set per physical core when possible
- `cpuPriority`: process priority class
- `dynamicPriorityBoost`: enable or disable Windows dynamic priority boosting
- `cpuPowerMode`: leave unchanged, disable throttling, enable throttling, or use `eco-qos-full`
- `cpuSetEfficiencyPreference`: convenience helper for hybrid CPUs; when `cpuSets` is not explicitly set, it can steer selection toward performance or efficiency cores
- `timerResolution`: optionally request max timer resolution
- `workingSetMode`: `lock-minimum` or `trim`
- `profile`: composite preset, currently `low-latency`
- `ioPriority`: process I/O priority
- `pagePriority`: process page priority
- `memoryPriority`: process memory priority

### Graphics

These settings are written to:

`HKCU\Software\Microsoft\DirectX\UserGpuPreferences`

Available controls:

- `gpuPreference`: `system-default`, `power-saving`, or `high-performance`
- `autoHdr`: enable or disable Auto HDR for the executable
- `windowedOptimizations`: enable or disable Windowed Optimizations
- `variableRefreshRate`: enable or disable VRR optimizations

### NPU / ONNX Runtime

- `npuMode`
  - `unchanged`
  - `onnxruntime-prefer-npu`
  - `onnxruntime-qnn`
  - `onnxruntime-openvino`
- `npuDisableCpuFallback`: disables ONNX Runtime CPU EP fallback for new sessions
- `npuQnnPerformanceMode`: optional Qualcomm HTP performance hint
- `npuOpenVinoDeviceType`: choose the OpenVINO device string
- `npuOpenVinoEnableFastCompile`: enables OpenVINO NPU fast compile

## Low-Latency Profile

`profile: low-latency` fills in default values for any setting still left as
`unchanged`.

Current defaults are:

- `cpuPriority: high`
- `cpuPowerMode: disable-throttling`
- `dynamicPriorityBoost: enable`
- `ioPriority: normal`
- `timerResolution: max-resolution`
- `gpuPreference: high-performance`
- `windowedOptimizations: enable`
- `variableRefreshRate: enable`
- `workingSetMode: lock-minimum`

Explicit per-field settings still override the profile defaults.

## Example Rules

### Game / Emulator

```yaml
rules:
  - match: game.exe
    enabled: true
    profile: low-latency
    cpuSets: performance
    cpuSetAvoidSmt: true
```

### Prefer a Rule Only When Something Else Is Not Running

```yaml
rules:
  - match: blender.exe
    enabled: true
    whenProcessesNotRunning: game.exe
    cpuPriority: above-normal
    gpuPreference: high-performance
```

### ONNX Runtime OpenVINO NPU

```yaml
rules:
  - match: ai-app.exe
    enabled: true
    npuMode: onnxruntime-openvino
    npuOpenVinoDeviceType: npu
    npuOpenVinoEnableFastCompile: true
```

## Notes and Warnings

- `realtime` priority is dangerous and can make the system unstable.
- CPU affinity uses logical CPU indexes from the current processor group.
- CPU Sets are more topology-aware than affinity and can span processor groups.
- CPU Set selection skips parked CPUs and CPUs allocated to other target processes.
- If both `cpuCores` and `cpuSets` are configured, Windows effectively uses the intersection of both restrictions.
- If you add a rule for a process that is already running and the mod was not loaded in it yet, restart that process.
- If DXGI or D3D modules were already loaded before graphics settings were written, restart the process to guarantee the new settings are used.
- If you change NPU options, recreate the app's ONNX Runtime sessions or restart the app.
- `whenProcessesRunning` and `whenProcessesNotRunning` are only re-evaluated when the target process starts or when settings reload.
- `eco-qos-full` also ignores timer resolution requests for maximum power savings.

## Logging

`logVerbose` controls whether the mod logs:

- The matched rule
- Applied overrides
- Restored original process state
- Hooking and provider-setup details for ONNX Runtime

## Repository Layout

- [`force-process-accelerators.wh.cpp`](force-process-accelerators.wh.cpp): Windhawk mod source
- `force-process-accelerators-test.dll`: test/build artifact
