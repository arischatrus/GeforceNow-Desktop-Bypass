# GeForce NOW Desktop Bypass — Incident Analysis

> **Disclaimer:** This document is for **educational and research purposes only**.
> The technique described is a **violation of NVIDIA's Terms of Service** and
> carries a risk of account termination. It is **not** a security vulnerability, nor does it contain any lateral movement techniques. Do not use this information to abuse the service.

## Summary

In **late July / early August 2026**, modders **Zortos** and **dpadGuy**
demonstrated a method to reach the **full Windows desktop** running inside a
GeForce NOW cloud session — turning a paid gaming subscription into what is
effectively a general-purpose cloud PC with access to NVIDIA's hardware
(AMD EPYC 9375F CPU, RTX 5080 GPU, up to 48 GB VRAM on the Ultimate tier).

The demonstration went viral after Zortos showed running **LM Studio** with the
**Gemma 4** model (~41 tokens/s), **Wallpaper Engine**, and almost any other normal 
desktop apps on the remote instance.

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


Before:
<img width="1920" height="1080" alt="GeForceNOW_3nHQen5qHl" src="https://github.com/user-attachments/assets/692dcd76-e851-4ba7-b0c4-d2ec28b45cd6" />


After:
<img width="1920" height="1080" alt="GeForceNOW_brL4ac3OAH" src="https://github.com/user-attachments/assets/97790880-bcbb-426f-ab24-ce920cf327d7" />
<img width="1920" height="1080" alt="GeForceNOW_zsZnsL5be5" src="https://github.com/user-attachments/assets/a2562ed2-be2d-495b-a9e2-bd7140a707d5" />



## Step by step explanation

Alliance partners are not supported. Only global GeForce NOW. Free tier is not supported. It is a **file-replacement trick** that abuses the
writable surface I2P provides:

1. **Entry point** — The user launches an Install to Play title game (e.g., *Trove*) through
   GeForce NOW. Steam's client starts on the VM, in a fresh GeForce NOW session. 

2. **Browser pivot** — During the game's first-run setup, Steam's embedded
   browser opens. This browser can navigate to `file://` URLs (e.g C:\Users), giving access
   to the VM's local file system from within the browser context. User should 
   navigate to game's original folder path and find the main *exe* file of the
   game.

4. **Executable swap** — After that, user would type `salsa.salsanowfiles.work` on their steam browser. 
   It will pop up an error and by simply pressing *ok*, user will see the file explorer. 
   They will type "C:\Program Files (x86)\Steam\steamapps\common\<your game>\<main exe file>, 
   and by pressing *okay*, it **replaces it** with a modified launcher. The
   modified executable, when invoked by Steam, does not start the game —
   it launches **`explorer.exe`**, the Windows desktop shell. 

5. **Desktop access** — On the launch, Steam thinks it is starting the
   game, but the user lands on a **full Windows desktop** (taskbar, Start
   menu, desktop icons) running on NVIDIA's hardware. 
‎ 
‎
‎ 
‎ 
**For easier guidance:**
https://www.youtube.com/watch?v=L0bNUoGcYJY&t=5s
‎ 
‎ 
‎ 

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

- **Paid tier only** — Requires an Performance (or higher) subscription with I2P support.
- **Session-scoped** — The VM is wiped when the session ends. Persistent storage (paid add-on) can retain the modified files across sessions.
- **Unprivileged** — No admin rights. Some system operations are blocked.
- **ToS violation** — Account termination is a realistic risk.
- **No confirmed NVIDIA fix** — As of mid-October 2026, NVIDIA had not
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
