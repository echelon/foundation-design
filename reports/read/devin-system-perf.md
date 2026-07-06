# Why is the system (and Devin IDE) lagging?

Report, 2026-07-05, from live diagnostics run at 17:48 (`ps`, `vm_stat`, `sysctl
vm.swapusage`, `memory_pressure`).

## TL;DR

Your machine has been up **39 days**, has burned through **4.7 GB of its 6 GB swap**, and the
biggest memory holders are RustRover (**6.5 GB**), Firefox (~5 GB across content processes),
and a stack of Electron apps (Discord ~1 GB, Chrome ~1 GB, Devin ~1.5–2 GB). Devin itself is
**not** the villain — it's the victim: an Electron/VS Code-fork renderer whose JS heap and
GPU caches get compressed/paged after weeks of system-wide memory churn, which is exactly
what scroll jank feels like. **Yes, reboot** — or do the three targeted restarts below.

## The numbers

| Metric                  | Value                          | Reading                                        |
| ----------------------- | ------------------------------ | ----------------------------------------------- |
| Uptime                  | 39 days                        | Long; caches/leaks accumulate                   |
| Load average            | 2.1–2.5                        | Fine for this machine — CPU is not the problem  |
| Swap                    | **4.70 / 6.00 GB used**        | The system has been under real memory pressure  |
| RAM                     | 48 GB total; ~4.3 GB free pages| "85% free" per memory_pressure counts purgeable/file-backed — pressure is OK *right now*, but the swap says it wasn't |
| Pages reactivated       | 3.59 **billion** (lifetime)    | Heavy long-term swap/compression churn          |

**Top memory holders right now:**

| Process                       | RSS       | Note                                          |
| ----------------------------- | --------- | ---------------------------------------------- |
| RustRover                     | **6.5 GB**| Also 70+ CPU-hours accumulated — indexing churn |
| Firefox (sum of processes)    | ~5+ GB    | Multiple 0.5–1.3 GB content processes           |
| Discord renderer              | ~1.0 GB   |                                                 |
| Chrome renderer (headless-ish)| ~1.0 GB   | Running against a /tmp profile since Fri        |
| Devin (renderer + helpers)    | ~1.5–2 GB | Renderer 677 MB + extension host + LSP          |
| claude (this session)         | ~0.9 GB, 38% CPU | Me — transient; three research agents were running during capture |

**Devin specifics:** its Windsurf language server is running a **workspace indexing service**
(`--enable_index_service`, 5,000-file cap) against this repo, and there are two `devin acp`
agent processes idling. Normal footprint; nothing pathological. Two `nx dev` watchers
(artcraft, artcraft-webapp) have also been running for days.

**One more suspect:** WindowServer at ~19% CPU. Sustained WindowServer load is the classic
correlate of *scrolling* lag specifically (compositing pressure from many windows/displays +
Electron apps repainting). It has 108 CPU-hours over the uptime.

## Is Devin paging to disk?

Almost certainly parts of it have been (macOS doesn't expose per-process swap cleanly, but
4.7 GB swap used + weeks of uptime + Electron heaps = yes in practice). When you scroll, the
renderer touches cold heap/texture pages → decompress/page-in stalls → jank. That's the
mechanism.

## What to do (in order of effort/payoff)

1. **Reboot.** 39 days + ¾ swap consumed — this is the cheap, complete fix. Everything below
   is the no-reboot version.
2. Quit and relaunch **RustRover** (reclaims 6.5 GB; also consider Settings → limit its
   indexing scope if artcraft is in its project set).
3. Restart **Devin** (fresh renderer heap fixes the scrolling immediately, if temporarily)
   and **Firefox** (or prune the heavyweight tabs).
4. Kill strays: the Chrome instance running against `/tmp/chrome/...` since Friday and the
   two `nx dev` watchers if you're not actively using them.
5. If scroll jank persists after ALL that on a fresh boot, then it's a Devin/Windsurf bug
   (GPU compositing in their VS Code fork) — check Devin's GPU acceleration toggle and
   whether the lag tracks having many editor tabs/webviews open.

Not the cause: overall CPU (load 2.x is idle-ish for this box), disk space wasn't flagged,
and my session's 38% CPU is transient research-agent work.
