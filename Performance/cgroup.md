# `cgroups.json` and `task_profiles.json`

Configuration files for **libprocessgroup**, the Android component that owns cgroup
mounting and per-task/per-process resource policy. Together they replace what used to
be hardcoded C++ in `SchedPolicy` and scattered `mkdir`/`mount` lines in `init.rc`.

| File | Answers |
|---|---|
| `cgroups.json` | *Which cgroup controllers exist, where are they mounted, who owns them?* |
| `task_profiles.json` | *Given a named policy, which files do I write and which cgroups do I join?* |

Source of truth in AOSP: `system/core/libprocessgroup/profiles/`.

---

## 1. Layering and load order

Both files are layered. The platform ships a base copy; vendors may add or override
entries without touching the system image.

```
/etc/cgroups.json                    # platform (system/etc)
/etc/cgroups_<api_level>.json        # platform, compat variant for older vendor images
/vendor/etc/cgroups.json             # vendor additions
/odm/etc/cgroups.json                # ODM additions

/etc/task_profiles.json
/etc/task_profiles_<api_level>.json
/vendor/etc/task_profiles.json
/odm/etc/task_profiles.json
```

The `<api_level>` variant is selected from the device's shipping API level
(`ro.product.first_api_level` / vendor API level) so that a new platform can keep
old cgroup layouts alive for a vendor image built against an earlier release.

Merge semantics: later files win. A vendor profile with the same `Name` as a platform
profile **replaces** it wholesale — it is not merged action-by-action. Same for a
controller entry with the same `Controller` name.

> Exact filenames and the property used to pick the versioned variant have shifted
> between releases. Check `system/core/libprocessgroup/cgroup_map.cpp` and
> `task_profiles.cpp` for the branch you are on.

---

## 2. `cgroups.json`

### Purpose

Read by `android::CgroupSetup()`, which `init` calls very early — before most services
start. It mounts the controllers described here and then writes a compact binary
description to:

```
/dev/cgroup_info/cgroup.rc
```

Every later libprocessgroup client (system_server via JNI, native services,
`vendor` code linking `libprocessgroup`) memory-maps that file instead of re-parsing
JSON. That is why the file is parsed exactly once per boot and why editing it requires
a reboot to take effect.

### Schema

```json
{
  "Cgroups": [
    {
      "Controller": "cpu",
      "Path": "/dev/cpuctl",
      "Mode": "0755",
      "UID": "system",
      "GID": "system"
    },
    {
      "Controller": "cpuset",
      "Path": "/dev/cpuset",
      "Mode": "0755",
      "UID": "system",
      "GID": "system"
    },
    {
      "Controller": "blkio",
      "Path": "/dev/blkio",
      "Mode": "0755",
      "UID": "system",
      "GID": "system"
    }
  ],
  "Cgroups2": {
    "Path": "/sys/fs/cgroup",
    "Mode": "0755",
    "UID": "system",
    "GID": "system",
    "Controllers": [
      {
        "Controller": "freezer",
        "Path": ".",
        "Mode": "0755",
        "UID": "system",
        "GID": "system"
      },
      {
        "Controller": "memory",
        "Path": ".",
        "Mode": "0755",
        "UID": "system",
        "GID": "system",
        "NeedsActivation": true
      }
    ]
  }
}
```

### Field reference

| Field | Meaning |
|---|---|
| `Controller` | Logical name. This is the string `task_profiles.json` refers to — it need not equal the kernel controller name. |
| `Path` | Mount point (v1) or path relative to the v2 root (v2 controllers, usually `"."`). |
| `Mode` | Octal permissions on the mount point directory, as a string. |
| `UID` / `GID` | Owner of the directory, by name. Controls who may create sub-cgroups and move tasks. |
| `NeedsActivation` | v2 only. Controller is not enabled in `cgroup.subtree_control` at mount time; libprocessgroup activates it lazily on first use. |
| `Optional` | Boot does not fail if the controller is unavailable (e.g. kernel built without it). Essential for vendor kernels with a reduced config. |

`Cgroups` is the **v1** list — one mount per controller, each at its own path.
`Cgroups2` is the **v2** unified hierarchy — a single mount whose `Controllers` array
just names what is enabled inside it. Android runs a hybrid: `cpuset`, `cpuctl`,
`blkio`, `stune` on v1; `freezer` and `memory` on v2.

### Runtime inspection

```bash
adb shell cat /proc/cgroups           # what the kernel supports
adb shell cat /proc/mounts | grep cgroup
adb shell ls /dev/cpuset /dev/cpuctl
adb shell cat /sys/fs/cgroup/cgroup.controllers
adb shell cat /proc/<pid>/cgroup      # where a given task actually sits
```

---

## 3. `task_profiles.json`

### Purpose

Defines named, device-independent **profiles**. Callers ask for a profile by name;
libprocessgroup translates that into concrete writes. This is the indirection that lets
`system_server` say "put this process in `HighPerformance`" without knowing whether the
device uses `cpuset`, `uclamp`, `schedtune`, or nothing at all.

### Top-level structure

```json
{
  "Attributes": [ ... ],
  "Profiles":   [ ... ],
  "AggregateProfiles": [ ... ]
}
```

### 3.1 `Attributes`

A named handle for "a file inside a controller". Decouples profile actions from
kernel-specific filenames.

```json
{
  "Attributes": [
    { "Name": "UClampMin",  "Controller": "cpu",    "File": "cpu.uclamp.min" },
    { "Name": "UClampMax",  "Controller": "cpu",    "File": "cpu.uclamp.max" },
    { "Name": "MemSoftLimit","Controller": "memory","File": "memory.soft_limit_in_bytes",
      "Optional": true },
    { "Name": "FreezerState","Controller": "freezer","File": "cgroup.freeze" }
  ]
}
```

Some releases add `FileV2` so one attribute can name both the v1 and v2 spelling of the
same knob.

### 3.2 `Profiles`

```json
{
  "Profiles": [
    {
      "Name": "HighPerformance",
      "Actions": [
        {
          "Name": "JoinCgroup",
          "Params": { "Controller": "schedtune", "Path": "foreground" }
        }
      ]
    },
    {
      "Name": "TimerSlackHigh",
      "Actions": [
        { "Name": "SetTimerSlack", "Params": { "Slack": "40000000" } }
      ]
    },
    {
      "Name": "ProcessCapacityHigh",
      "Actions": [
        {
          "Name": "JoinCgroup",
          "Params": { "Controller": "cpuset", "Path": "foreground" }
        }
      ]
    },
    {
      "Name": "Frozen",
      "Actions": [
        { "Name": "SetAttribute", "Params": { "Name": "FreezerState", "Value": "1" } }
      ]
    }
  ]
}
```

#### Action types

| Action | Params | Effect |
|---|---|---|
| `JoinCgroup` | `Controller`, `Path` | Writes the tid/pid into `<controller_mount>/<Path>/cgroup.procs` (or `tasks`). |
| `SetAttribute` | `Name`, `Value`, `Optional` | Writes `Value` into the file named by the `Attributes` entry. |
| `SetTimerSlack` | `Slack` (ns) | Writes `/proc/<tid>/timerslack_ns`. Larger slack = coarser wakeups = less power. |
| `WriteFile` | `FilePath`, `Value`, `LogFailures`, `UseFd` | Escape hatch for an absolute path outside any cgroup. |
| `SetClamps` | `Boost`, `Clamp` | Legacy schedtune boost/clamp shim. |

Path substitution is available in `JoinCgroup` params: `<uid>` and `<pid>` expand at
apply time, which is how per-app cgroups get created:

```json
{ "Name": "JoinCgroup",
  "Params": { "Controller": "memory", "Path": "system/uid_<uid>/pid_<pid>" } }
```

Actions run in array order and are **not** transactional — a failure partway through
leaves earlier actions applied. Mark genuinely optional writes with `"Optional": true`
so a missing kernel knob logs instead of erroring.

### 3.3 `AggregateProfiles`

A profile that is just a list of other profiles. Used to give the legacy `SP_*`
scheduler policies a name that maps to several concrete profiles.

```json
{
  "AggregateProfiles": [
    {
      "Name": "SCHED_SP_FOREGROUND",
      "Profiles": ["HighPerformance", "TimerSlackNormal"]
    },
    {
      "Name": "SCHED_SP_BACKGROUND",
      "Profiles": ["HighEnergySaving", "LowIoPriority", "TimerSlackHigh"]
    }
  ]
}
```

### Conventional profile names

Platform code depends on these existing. If your vendor file removes one, things break
silently and oddly.

- **Scheduling / capacity** — `HighEnergySaving`, `NormalPerformance`,
  `HighPerformance`, `MaxPerformance`, `RealtimePerformance`
- **CPU set** — `ProcessCapacityDefault`, `ProcessCapacityRestricted`,
  `ProcessCapacitySystem`, `ProcessCapacityMax`
- **Service capacity** — `ServiceCapacityLow`, `ServiceCapacityRestricted`,
  `CameraServiceCapacity`
- **I/O** — `LowIoPriority`, `NormalIoPriority`, `HighIoPriority`, `MaxIoPriority`
- **Memory** — `LowMemoryUsage`, `HighMemoryUsage`, `SystemMemoryProcess`
- **Timer slack** — `TimerSlackNormal`, `TimerSlackHigh`
- **Freezer** — `Frozen`, `Unfrozen`

---

## 4. How it is consumed

```c
// system/core/libprocessgroup/include/processgroup/processgroup.h
bool SetTaskProfiles(pid_t tid, const std::vector<std::string>& profiles,
                     bool use_fd_cache = false);
bool SetProcessProfiles(uid_t uid, pid_t pid,
                        const std::vector<std::string>& profiles);
```

Typical callers:

- **`system_server`** — `ActivityManagerService` / `OomAdjuster` map process state
  (foreground app, cached, service) to profile names via `Process.setProcessGroup()`
  and `Process.setProcessFrozen()`.
- **`init`** — service definitions in `.rc` files carry `task_profiles <name>`.
- **Native services** — call `SetTaskProfiles()` directly, e.g. audio and camera HALs
  pinning worker threads.
- **`libcutils` `set_sched_policy()`** — the old `SP_FOREGROUND` / `SP_BACKGROUND` API
  now routes through the `SCHED_SP_*` aggregate profiles.

---

## 5. Worked example: AMS starting an app process

This is the path from "user taps an icon" to "bytes land in `cgroup.procs`". It is worth
tracing once, because the cgroup placement happens in **two separate phases** that people
routinely confuse: the zygote does an initial placement at fork time, and AMS corrects it
a few milliseconds later once oom_adj has been computed.

### Phase 1 — AMS decides to start, and passes a hint

`ActivityManagerService` → `ProcessList.startProcessLocked()`. The only cgroup-relevant
decision made here is whether the new process is the top app:

```java
// frameworks/base/services/core/java/com/android/server/am/ProcessList.java
boolean isTopApp = hostingRecord.isTopApp();

startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
        app.info.dataDir, invokeWith, app.info.packageName,
        zygotePolicyFlags,
        isTopApp,                       // <-- the hint
        app.getDisabledCompatChanges(), ...);
```

`isTopApp` rides through `ZygoteProcess` over the zygote socket as
`--is-top-app`, into `Zygote.forkAndSpecialize()`.

### Phase 2 — Zygote: create the per-app cgroup, apply an initial policy

After the fork, the child runs `SpecializeCommon()` in
`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`. Two things matter:

```cpp
// (a) create this app's own cgroup directories
if (!is_system_server && getuid() == 0) {
    const int rc = createProcessGroup(uid, getpid());
    if (rc == -errno) {
        fail_fn(CREATE_ERROR("createProcessGroup(%d, %d) failed: %s",
                             uid, /* pid= */ 0, strerror(errno)));
    }
}

// (b) initial scheduling policy
SetSchedulerPolicy(fail_fn, is_top_app);
```

`createProcessGroup(uid, pid)` lives in libprocessgroup and uses the mount points that
came from **`cgroups.json`**. It creates and populates:

```
/dev/cpuctl/uid_10123/pid_4711/cgroup.procs        # cpu controller (v1)
/sys/fs/cgroup/uid_10123/pid_4711/cgroup.procs     # unified v2 (freezer, memory)
```

This per-app directory is the thing that makes two later operations cheap and reliable:

- **freezing** the app — one write to `cgroup.freeze`
- **killing** the app — `killProcessGroup()` signals every task in the directory, so
  forked children and orphaned threads die too, instead of leaking

`SetSchedulerPolicy()` then applies a coarse initial policy:

```cpp
static void SetSchedulerPolicy(fail_fn_t fail_fn, bool is_top_app) {
  SchedPolicy policy = is_top_app ? SP_TOP_APP : SP_DEFAULT;
  if (set_sched_policy(0, policy) == -1) {
    fail_fn(CREATE_ERROR("set_sched_policy(0, %d) failed", policy));
  }
}
```

`set_sched_policy()` (in `libprocessgroup/sched_policy.cpp`) is where the enum finally
becomes profile names from **`task_profiles.json`**:

```cpp
int set_sched_policy(int tid, SchedPolicy policy) {
    if (tid == 0) tid = GetThreadId();
    policy = _policy(policy);

    switch (policy) {
        case SP_BACKGROUND:
            return SetTaskProfiles(tid, {"HighEnergySaving", "TimerSlackHigh"}, true)
                   ? 0 : -1;
        case SP_FOREGROUND:
        case SP_AUDIO_APP:
        case SP_AUDIO_SYS:
            return SetTaskProfiles(tid, {"HighPerformance", "TimerSlackNormal"}, true)
                   ? 0 : -1;
        case SP_TOP_APP:
            return SetTaskProfiles(tid, {"MaxPerformance", "TimerSlackNormal"}, true)
                   ? 0 : -1;
        case SP_SYSTEM:
            return SetTaskProfiles(tid, {"ServiceCapacityLow", "TimerSlackNormal"}, true)
                   ? 0 : -1;
        ...
    }
}
```

The exact profile-name lists differ between releases — check the branch you are on. The
structure does not: an `SP_*` enum maps to a fixed set of profile names, and everything
device-specific lives in JSON below that line.

### Phase 3 — AMS corrects the placement after oom_adj

The app calls back with `attachApplication()`. AMS runs `updateOomAdjLocked()`, which
computes both `curAdj` and `curSchedGroup`. `applyOomAdjLSP()` then acts on the sched
group:

```java
// frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
if (state.getCurrentSchedulingGroup() != state.getSetSchedGroup()) {
    int processGroup;
    switch (state.getCurrentSchedulingGroup()) {
        case ProcessList.SCHED_GROUP_BACKGROUND:
            processGroup = THREAD_GROUP_BACKGROUND;
            break;
        case ProcessList.SCHED_GROUP_TOP_APP:
        case ProcessList.SCHED_GROUP_TOP_APP_BOUND:
            processGroup = THREAD_GROUP_TOP_APP;
            break;
        case ProcessList.SCHED_GROUP_RESTRICTED:
            processGroup = THREAD_GROUP_RESTRICTED;
            break;
        default:
            processGroup = THREAD_GROUP_DEFAULT;
            break;
    }
    mProcessGroupHandler.sendMessage(mProcessGroupHandler.obtainMessage(
            0 /* unused */, app.getPid(), processGroup, app.processName));
    // handler → Process.setProcessGroup(pid, group)
}
```

The handler thread exists because `setProcessGroup()` walks every thread of the process
and does synchronous `write()` calls into sysfs — you do not want that on the AMS lock.

`Process.setProcessGroup()` → JNI `android_os_Process_setProcessGroup()` in
`android_util_Process.cpp` → `SetProcessProfiles(uid, pid, {...})` in libprocessgroup.

### Phase 4 — actions get applied

For `THREAD_GROUP_TOP_APP` the resolved profile set is roughly
`{"MaxPerformance", "ProcessCapacityMax", "MaxIoPriority", "TimerSlackNormal"}`.
libprocessgroup looks each name up in the parsed `task_profiles.json` and executes its
actions in order. Concretely, for a device using cpuset + uclamp:

| Profile | Action | Resulting write |
|---|---|---|
| `ProcessCapacityMax` | `JoinCgroup` cpuset `top-app` | `4711` → `/dev/cpuset/top-app/cgroup.procs` |
| `MaxPerformance` | `SetAttribute` `UClampMin` | `"max"` → `/dev/cpuctl/top-app/cpu.uclamp.min` |
| `MaxIoPriority` | `SetAttribute` blkio weight | `"1000"` → `/dev/blkio/.../blkio.weight` |
| `TimerSlackNormal` | `SetTimerSlack` | `50000` → `/proc/<tid>/timerslack_ns` |

Nothing in AMS or the zygote knows any of those paths. Swap the vendor
`task_profiles.json` and the same Java code produces entirely different writes.

### Phase 5 — lifecycle changes reuse the same machinery

- App goes to background → `SCHED_GROUP_BACKGROUND` → `SetProcessProfiles(uid, pid,
  {"HighEnergySaving", "ProcessCapacityLow", "LowIoPriority", "TimerSlackHigh"})`
- App becomes cached and eligible for freezing → `CachedAppOptimizer.freezeProcess()` →
  `Process.setProcessFrozen()` → `SetProcessProfiles(uid, pid, {"Frozen"})` → `1` into
  `/sys/fs/cgroup/uid_10123/pid_4711/cgroup.freeze`
- App is killed → `Process.killProcessGroup(uid, pid)` → libprocessgroup signals every
  task listed in that directory, then removes it

### Tracing it on a real device

```bash
PID=$(adb shell pidof com.example.app)

# where the zygote and AMS actually put it
adb shell cat /proc/$PID/cgroup
# 0::/uid_10123/pid_4711            <- v2, per-app dir from createProcessGroup()
# 4:cpuset:/top-app                 <- v1, from ProcessCapacityMax
# 2:cpu:/top-app                    <- v1, from MaxPerformance

# confirm membership from the other side
adb shell cat /dev/cpuset/top-app/cgroup.procs | grep $PID

# watch it move when you background the app
adb shell 'while true; do cat /proc/'$PID'/cgroup | grep cpuset; sleep 1; done'

# the timer slack action landed here
adb shell cat /proc/$PID/timerslack_ns

# and the oom_adj/sched group decision that drove it
adb shell dumpsys activity oom | grep -A2 com.example.app
```

If the process sits in the right cpuset but performance is still wrong, the profile
applied fine and the problem is in the *contents* of `task_profiles.json` — that is the
file to edit, not the framework.

---

## 6. Working with these files on a device

```bash
# What the platform shipped
adb shell cat /system/etc/task_profiles.json
adb shell cat /vendor/etc/task_profiles.json

# Where a process ended up
adb shell cat /proc/<pid>/cgroup
adb shell cat /dev/cpuset/foreground/cgroup.procs

# Apply a profile by hand for a quick experiment
adb shell cmd activity ...        # or a small binary calling SetTaskProfiles()

# Parse errors show up here at boot
adb logcat -b all | grep -iE 'libprocessgroup|task_profiles|cgroup'
```

---

## 7. References

- `system/core/libprocessgroup/` — implementation and the shipped JSON
- `system/core/libprocessgroup/cgroup_map.cpp` — `cgroups.json` parsing, `cgroup.rc`
- `system/core/libprocessgroup/task_profiles.cpp` — profile parsing and action classes
- `system/core/libprocessgroup/include/processgroup/processgroup.h` — public API
- `system/core/libprocessgroup/sched_policy.cpp` — `SP_*` enum → profile-name mapping
- `frameworks/base/core/jni/com_android_internal_os_Zygote.cpp` — `SpecializeCommon()`,
  `createProcessGroup()`, `SetSchedulerPolicy()`
- `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java` —
  `applyOomAdjLSP()`, sched group → `THREAD_GROUP_*`
- `frameworks/base/core/jni/android_util_Process.cpp` — `setProcessGroup()` JNI
- Kernel: `Documentation/admin-guide/cgroup-v1/`, `Documentation/admin-guide/cgroup-v2.rst`
