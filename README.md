# GeForce NOW Desktop Bypass — Incident Analysis

> **Disclaimer:** This document is for **educational and research purposes only**.
> The technique described is a **violation of NVIDIA's Terms of Service** and
> carries a risk of account termination. It is **not** a security vulnerability
> or a host/container escape. Do not use this information to abuse the service.

## Summary

In **late July / early August 2026**, modders **Zortos** and **dpadGuy**
demonstrated a method to reach the **full Windows desktop** running inside a
GeForce NOW cloud session — turning a paid gaming subscription into what is
effectively a general-purpose cloud PC with access to NVIDIA's hardware
(AMD EPYC 9375F CPU, RTX 5080 GPU, up to 48 GB VRAM on the Ultimate tier).

The demonstration went viral after Zortos showed running **LM Studio** with the
**Gemma 4** model (~41 tokens/s) and **Wallpaper Engine** on the remote
instance.

## What Actually Happened (Technical Context)

### The Architecture

Each GeForce NOW session runs a **single-tenant Windows virtual machine** on
NVIDIA's infrastructure. The VM is provisioned per session and is **wiped
between sessions** (unless the user has a persistent-storage add-on).  The
user runs under an **unprivileged Windows account**.

On top of that VM, NVIDIA layers a **game-launcher shell** that restricts the
user to launching supported titles from linked storefronts (Steam, Epic, etc.).
This shell is what most users see — it is **not** a container in the
Docker/Kubernetes sense, and it is **not** the VM boundary. 

### The Key Enabler: Install-to-Play (I2P)

GeForce NOW's **Ultimate tier** supports **I2P (Install-to-Play)** games —
titles that are downloaded to the VM's local disk and launched through the
**Steam client** rather than being streamed from a fixed image. 

This is the critical difference from the older streaming model:

| Aspect | Legacy streaming | I2P (Ultimate) |
|---|---|---|
| Game files | Pre-baked, read-only | Downloaded to local disk, **writable** |
| Launcher | NVIDIA's proprietary shell | **Steam client** (with embedded browser) |
| Writable surface | Minimal | Steam's file system, game install dirs |
| Browser access | None | **Steam's built-in web browser** (can navigate `file://` paths) |

The I2P model creates a **larger writable surface** and exposes Steam's
browser, which can resolve local file-system paths. 

### The Mechanism (Conceptual)

The bypass does **not** exploit a memory corruption, privilege escalation,
or hypervisor flaw. It is a **file-replacement trick** that abuses the
writable surface I2P provides:

1. **Entry point** — The user launches an I2P title (e.g., *Trove*) through
   GeForce NOW. Steam's client starts on the VM. 

2. **Browser pivot** — During the game's first-run setup, Steam's embedded
   browser opens. This browser can navigate to `file://` URLs, giving access
   to the VM's local file system from within the browser context. 

3. **Executable swap** — Using the browser's file access, the user locates
   the game's primary executable in the Steam install directory and
   **replaces it** with a modified launcher (developed by dpadGuy).  The
   modified executable, when invoked by Steam, does not start the game —
   it launches **`explorer.exe`**, the Windows desktop shell. 

4. **Desktop access** — On the next launch, Steam thinks it is starting the
   game, but the user lands on a **full Windows desktop** (taskbar, Start
   menu, desktop icons) running on NVIDIA's hardware. 

The open-source tool **SalsaNOW** (by dpadGuy) packages and automates parts
of this process.

### What This Is NOT

| Claim | Reality |
|---|---|
| "VM escape" | ❌ The user stays **inside** their own VM |
| "Host access" | ❌ No access to the hypervisor, host OS, or physical hardware management |
| "Other users' data" | ❌ No cross-tenant access demonstrated |
| "NVIDIA network access" | ❌ No lateral movement into NVIDIA's internal network |
| "Security vulnerability" | ❌ Zortos explicitly stated: *"We did not touch the host, another user's session, or NVIDIA's network."* |
| "Privilege escalation" | ❌ The user remains an **unprivileged** Windows account |

It is, in Zortos's own words, a **terms-of-service violation** — the
equivalent of using a game console to run homebrew. The "sandbox" being
bypassed is the **launcher UI**, not the VM or the host. 

### Limitations

- **Paid tier only** — Requires an Ultimate (or equivalent) subscription with I2P support.
- **Session-scoped** — The VM is wiped when the session ends. Persistent storage (paid add-on) can retain the modified files across sessions.
- **Unprivileged** — No admin rights. Some system operations are blocked.
- **Not universal** — Some users report GeForce NOW closing the session when
  File Explorer is opened, or blocking third-party file downloads.
- **ToS violation** — Account termination is a realistic risk.
- **No confirmed NVIDIA fix** — As of mid-August 2026, NVIDIA had not
  published a patch or official statement. 

## Why This Matters (Security Perspective)

While not a vulnerability in the traditional sense, the incident highlights
a **session-control gap** in cloud-gaming architecture:

- **I2P expands the attack surface.** The writable file system + Steam's
  browser with `file://` access is a significantly larger surface than the
  read-only streaming model.
- **Launcher ≠ isolation.** The game-launcher shell is a UI restriction,
  not a security boundary. Anyone with write access to the game directory
  can replace the entry-point executable.
- **Cloud gaming is not a container.** Unlike Docker/K8s containers, a
  cloud-gaming VM is a full OS image. "Escaping the launcher" is analogous
  to escaping a kiosk mode on a local PC — it doesn't breach the VM, but
  it defeats the intended usage model.
- **Monitoring gap.** NVIDIA's infrastructure did not detect or block the
  file replacement in real time (at least for Zortos's account).

### Comparison with Actual Container Escapes

For contrast, **real** container-to-host escapes in the NVIDIA ecosystem
involve the **NVIDIA Container Toolkit** (NCT) and are genuine security
vulnerabilities:

| CVE | CVSS | Year | Impact |
|---|---|---|---|
| CVE-2024-0132 | 9.0 | 2024 | Full host takeover from a container via NCT |
| CVE-2025-23359 | 8.3 | 2025 | Privilege escalation via NCT |
| CVE-2025-23266 ("NVIDIAScape") | 9.0 | 2025 | Container escape to host |

These affect **AI/GPU workloads in cloud and on-premises container
environments**, not GeForce NOW. The GeForce NOW desktop bypass is a
**different class of issue** — a ToS/UX bypass, not a security breach.

## Timeline

| Date | Event |
|---|---|
| ~Jul 31, 2026 | Zortos & dpadGuy publish walkthrough video |
| Aug 1, 2026 | Tutorial posted on X (Twitter) |
| Aug 3, 2026 | Zortos demonstrates LM Studio + Gemma 4 on the cloud desktop |
| Aug 8–10, 2026 | Widespread media coverage (Tom's Hardware, TechSpot, VideoCardz, WccfTech, etc.) |
| Aug 2026 | NVIDIA has **not** confirmed a fix or issued a statement |

## References

- [Tom's Hardware — GeForce NOW exploit lets you access the full Windows desktop](https://www.tomshardware.com/service-providers/streaming/geforce-now-exploit-lets-you-access-the-full-windows-desktop-through-a-simple-file-swap-modder-runs-local-ai-models-on-ultimate-tier-with-48gb-of-vram-and-no-restrictions)
- [TechSpot — GeForce Now exploit unlocks full Windows desktop access](https://www.techspot.com/news/113414-geforce-now-exploit-unlocks-full-windows-desktop-access.html)
- [WccfTech — Modder Successfully Escapes NVIDIA's GeForce NOW Sandbox](https://wccftech.com/geforce-now-sandbox-escape-windows-desktop-exploit/)
- [VideoCardz — Modders gain full Windows desktop access on GeForce NOW](https://videocardz.com/newz/modders-gain-full-windows-desktop-access-on-geforce-now)
- [MLQ.ai — GeForce NOW desktop escape exposes a cloud-session control gap](https://mlq.ai/news/geforce-now-desktop-escape-exposes-a-cloud-session-control-gap-nvidia-has-not-confirmed-a-fix/)
- [WindowsForum — GeForce NOW Desktop Bypass Risks Bans, Not a Cloud PC](https://windowsforum.com/news/geforce-now-desktop-bypass-risks-bans-not-a-cloud-pc.442316/)
- [r/nvidia discussion](https://www.reddit.com/r/nvidia/comments/1vj4kun/modders_gain_full_windows_desktop_access_on/)
- [r/Steam discussion (Zortos comment)](https://www.reddit.com/r/Steam/comments/1vjxdfm/modder_bypasses_geforce_now_interface_to_reach/) 

## License

This document is for educational purposes. All trademarks belong to their
respective owners.   
