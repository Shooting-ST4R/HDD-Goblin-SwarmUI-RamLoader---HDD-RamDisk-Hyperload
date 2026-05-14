# HDD-Goblin 🧌  — Your Friendly Fast Model-Goblin
## SwarmUI Extension · HDD → RAM-Disk Hyperloader

> ⚠️ **Disclaimer:** This extension was vibe-coded. I don't claim ownership of the underlying code. Usage is entirely at your own risk. Functionality has been tested on **Windows 10 only**. No guarantees for other platforms.

Known Bugs: Cancelling model loading with [X] does not work. You need to wait for the model to finish loading before continuing. (Cancelling image generation works fine.)

---

## What is this?

HDD-Goblin is a [SwarmUI](https://github.com/mcmonkeyprojects/SwarmUI) extension that eliminates the painful multi-minute wait when loading large AI models from a spinning hard drive.

It works by **sequentially buffering the model from your HDD into a RAM disk** before ComfyUI touches it. HDDs hate the random I/O that PyTorch and safetensors do — but they're fast at sequential reads. RAM is fast at everything. HDD-Goblin bridges the gap.

```
HDD  ──sequential read──▶  RAM disk  ──instant──▶  ComfyUI / GPU
```

**This extension is for you if:**
- You store AI models on a spinning hard disk (HDD)
- You're tired of waiting 4–5 minutes every time you switch models
- You have spare RAM (you need free RAM equal to the model size, e.g. 7 GB free for a 7 GB model)

---

## Speed Comparison

Same machine, same model, same settings. The only difference is this extension.

| | Model load time | Total prep time |
|---|---|---|
| ❌ **Without extension** | ~5 min (random HDD I/O) | 4 min 49 sec |
| ✅ **With extension** | 20–37 sec (RAM disk) | 57 sec / 40 sec |

**Real log output — without extension:**
```
17:30:26  Self-Start ComfyUI-0 on port 7821 started.
17:30:43  User requested image with model 'Illustrious-Based/MyModel_v7.safetensors'
17:35:54  Generated image in 4.89 min (prep) and 17.14 sec (gen)
```

**Real log output — with extension (first load, cold RAM disk):**
```
  Loading model from HDD: [████████████████████] 100% | avg 184 MB/s  (6776/6776 MiB)
  Loading model from RAM-disk ... Done  (20s elapsed)
  Cleaning up ...
  RAM-disk copy deleted.
22:43:47  Generated image in 57.45 sec (prep) and 16.02 sec (gen)
```

**Second model loaded in the same session:**
```
  Loading model from HDD: [████████████████████] 100% | avg 187 MB/s  (6616/6616 MiB)
  Loading model from RAM-disk ... Done  (4s elapsed)
22:45:01  Generated image in 40.30 sec (prep) and 15.42 sec (gen)
```

> The second model loaded in just **4 seconds** from RAM disk. Load times improve as your RAM warms up within a session.

Example CMD:

<img width="1175" height="435" alt="Screenshot_6" src="https://github.com/user-attachments/assets/564c75d0-4ad1-47df-afe3-d9e78ed78ae3" />



---

## Setup

### Option A — RAM Disk (Recommended, fastest)

Windows has no built-in RAM disk. Install one of these free tools first:

| Tool | Notes |
|------|-------|
| **[ImDisk Toolkit](https://sourceforge.net/projects/imdisk-toolkit/)** | Recommended. Free, reliable, survives reboots. |
| [OSFMount](https://www.osforensics.com/tools/mount-disk-images.html) | Alternative. |

**Steps:**
1. Install ImDisk Toolkit
2. Create a RAM disk — assign it drive letter **`R:`**
3. Size it to fit your largest model + a buffer (e.g. 8 GB for a 6.6 GB model)
4. The extension will use `R:\ModelCache` by default — no config needed

**To change the drive letter or path**, edit `HDDModelLoader.cs` and update:
```csharp
public static string SettingCachePath = @"R:\ModelCache";
```
Set it to whatever path your RAM disk is mounted at. Blank (`""`) = auto-detect.

---

### Option B — M.2 / NVMe SSD (No RAM disk needed)

If you don't have enough spare RAM but have a fast SSD, you can point the cache at an SSD folder instead. This is slower than a RAM disk but still significantly faster than a spinning drive for the random I/O phase.

In `HDDModelLoader.cs`, set:
```csharp
public static string SettingCachePath = @"D:\ModelCache";  // your SSD drive
```

---

### Disable mmap in SwarmUI

For best results, disable memory-mapped file loading in SwarmUI so ComfyUI actually reads from disk (and therefore from your RAM disk) rather than memory-mapping the original file:

1. Open SwarmUI → **Server** tab → **Server Configuration**
2. Enter `--disable-mmap` in ExtraArgs

<img width="1508" height="764" alt="image" src="https://github.com/user-attachments/assets/fe3c9ce6-4263-49c3-b095-2497a660bcf0" />

Without this, ComfyUI may bypass the RAM disk entirely on some model types.

---

### After First Install

ComfyUI reads `extra_model_paths.yaml` at startup. The extension writes its paths into that file, but ComfyUI needs to restart once to pick them up.

**After installing, relaunch SwarmUI once.** You'll see this in the log when it's working:
```
[HDDModelLoader] Prepended swarm_ram_cache to extra_model_paths.yaml
[HDDModelLoader] *** Restart ComfyUI once (relaunch SwarmUI) for the new paths to take effect. ***
```

---

### Clearing the Model Cache Manually

SwarmUI caches model metadata in a `bin` folder. If models appear missing or stale after changing paths:

1. Close SwarmUI
2. Delete the `bin` folder inside your SwarmUI directory
3. Relaunch — SwarmUI will rescan and rebuild

---

## Installation

1. Go to `/SwarmUI/src/Extensions/`
2. Extract the newest `HDD-Goblin-Vx.zip` there

Then relaunch SwarmUI. It will compile and load the extension automatically.

---

## How It Works

### The problem: HDDs and AI models are a bad match

When PyTorch or safetensors loads a model, it doesn't read the file linearly from start to finish. It jumps around — reading tensor headers, seeking to specific offsets, loading layers out of order. This is called **random I/O**.

HDDs have a physical read head that must physically move to each location on the spinning platter. Random I/O means the head is constantly seeking. A typical HDD does:

- **Sequential read:** 150–200 MB/s (head stays in one place, platter spins past)
- **Random read (4K blocks):** 0.5–2 MB/s (head thrashes back and forth)

A 7 GB model loaded with random I/O at ~1 MB/s effective throughput = **~2 hours** in theory. In practice it's a few minutes because reads aren't maximally random, but you're still looking at 4–5 minutes vs. under a minute.

### What HDD-Goblin does

1. **Sequential copy to RAM** — Before the workflow is submitted to ComfyUI, the extension opens the model file with `FileOptions.SequentialScan` and a 64 MiB buffer. This tells the OS to read ahead aggressively in one direction. The HDD runs at its peak throughput (~150–200 MB/s). A 6.6 GB model copies in ~35 seconds.

2. **ComfyUI loads from RAM** — The extension registers the RAM disk as a model search path in ComfyUI's `extra_model_paths.yaml`, prepended so it is found before the HDD path. ComfyUI finds the model there and does all its random I/O against RAM, which handles it at memory speed (~10,000 MB/s effective).

3. **RAM disk copy deleted after load** — Once ComfyUI confirms the model has been loaded into VRAM, the RAM disk copy is deleted to free up RAM for the next model.

4. **Every generation is a clean slate** — The watcher waits for a confirmed `false → true` transition on the load signal rather than acting on any pre-existing state. This ensures the second, third, and subsequent model loads behave identically to the very first one.

### Sequence diagram

```
SwarmUI         HDDModelLoader          HDD            RAM disk        ComfyUI
   │                   │                 │                │                │
   │── gen request ───▶│                 │                │                │
   │                   │── seq read ────▶│                │                │
   │                   │◀── 184 MB/s ───│                │                │
   │                   │── write ───────────────────────▶│                │
   │                   │── start watcher (background)    │                │
   │── submit workflow ─────────────────────────────────────────────────▶│
   │                   │                 │                │                │
   │                   │                 │                │◀── rand read ──│
   │                   │ (spinner)       │                │   (RAM speed)  │
   │                   │                 │                │                │
   │                   │◀── load signal: false → true ────────────────────│
   │                   │── delete RAM copy ─────────────▶│                │
   │◀── image done ────────────────────────────────────────────────────── │
```

---

## Configuration

All settings are at the top of `HDDModelLoader.cs`:

| Setting | Default | Description |
|---------|---------|-------------|
| `SettingCachePath` | `""` (auto) | Override cache directory. Blank = auto-detect (`R:\ModelCache` on Windows if R: exists, `/dev/shm` on Linux). |
| `SettingCleanupOnShutdown` | `true` | Delete cache files when SwarmUI exits. Set to `false` to keep the cache warm across SwarmUI restarts (as long as the machine stays on). |
| `SettingMaxCacheMiB` | `0` | RAM cap in MiB. `0` = no limit. When exceeded, the oldest cached model is evicted. |
| `SettingLoadTimeoutSecs` | `300` | Seconds to wait for ComfyUI to load a model before giving up and cleaning up. Reduce if you want faster cleanup after a cancelled generation. |

---

## Limitations

- Tested on **Windows 10 only**. Linux should work (uses `/dev/shm` automatically) but is untested.
- Works with the **self-start ComfyUI backend**. Remote ComfyUI instances need manual `extra_model_paths.yaml` setup on the remote machine.
- Only the **primary checkpoint model** is pre-cached. LoRAs, VAE, ControlNet etc. are not (contributions welcome).
- If your cache fills up and `SettingMaxCacheMiB` is `0`, the copy will fail and ComfyUI falls back to loading from HDD normally.
- This is **vibe-coded software**. Use at your own risk.

---

## License

MIT
