# sdr-mesh — a mesh modem for a constellation of long-range UAVs

An IP mesh radio built on the **Neptune SDR** board — Zynq-7020 plus AD9361 —
with the same hardware and the same software on the ground and in the air.
The network carries IP; what rides on top of it is not the modem's business.

This repository currently holds **the plan**, not an implementation. The plan
rests on two existing projects: `sdr-ground`, which brought this exact board
up and got a picture out of it, and `wy-lora`, which worked out an anti-jam
FHSS protocol in simulation.

## What is decided

| | |
|---|---|
| Payload | A universal IP pipe, 20 kbps … 4 Mbps, adaptive |
| Topology | Full mesh: several aircraft, **several ground stations** |
| Node hardware | Neptune SDR on both ends — a symmetric node |
| Band | One window per constellation, **at least 1 GHz wide**, chosen to match the available power amplifiers, set by configuration |
| Time | An atomic reference on the ground, crystal in the air, epoch over the air |
| Prototype | **Three boards** |

The 1 GHz window is the load-bearing decision. It forces a jammer to choose
between width and density: spreading 20 W across the window instead of into
our channel costs it 30 dB of density, and 20 dB even against the sensible
middle strategy ([docs/03](docs/03-band-and-link-budget.md), §3.1).

## What this board gives us that nothing else did

The AD9361 on Neptune has **two coherent receive chains**, proven on hardware
in `sdr-ground`. Two coherent elements make a two-element adaptive array: one
steerable null, 20–30 dB against a jammer, whatever its bandwidth. TDMA even
hands us the quiet intervals needed to estimate the interference covariance.
That mechanism is unavailable on the 1R1T Pluto that `wy-lora` targets, and it
is the single strongest anti-jam lever in the design
([docs/07](docs/07-antijam.md), §7.2).

## How the work is split

Two tracks, deliberately ([docs/04](docs/04-phy.md), §4.1):

- **Track A — openwifi.** 802.11a/g/n on Zynq + AD9361 with our pinout
  patched in, plus Babel. Gives a working IP mesh on real hardware in weeks,
  which is what every *other* question needs in order to be answered: the
  amplifier, the antennas, thermals, routing, the IP model, the bench rig.
  It is scaffolding, not the product.
- **Track B — our own PHY.** Single-carrier TDMA with a spread control layer,
  adaptive MCS, FHSS inside the window, and the two-channel beamformer.
  This is the product, and it is about half the project.

## Repository layout (planned)

| Path | What goes in it |
|---|---|
| `docs/` | The plan and the knowledge base — **in Ukrainian** |
| `hdl/` | PL sources (Amaranth plus ADI HDL) |
| `software/sdrmesh-d/` | The node daemon (Rust): TDMA, metrics, MCS, fragmentation |
| `software/sdrmesh-httpd/` | On-board HTTP/WS service: API, metrics |
| `software/sdrmesh-kmod/` | Kernel module for the PL interface |
| `board/neptune/` | Buildroot board support, forked from `sdr-ground` |
| `app/` | Constellation monitor (Rust, egui) |
| `bench/` | Bench rig: channel model, host-side demodulator, measurement scripts |
| `tools/` | Build, deploy, measurement and serial-console scripts |

## The plan

`docs/` is the project's working memory, and it is **written in Ukrainian**.
Everything else, including code comments, is in English.

| Document | About |
|---|---|
| [00-project-brief.md](docs/00-project-brief.md) | Goal, the three sentences that decide the rest, scope |
| [01-platform-neptune.md](docs/01-platform-neptune.md) | What we inherit proven, and what was never touched |
| [02-requirements.md](docs/02-requirements.md) | Requirements and the numbers that follow from them |
| [03-band-and-link-budget.md](docs/03-band-and-link-budget.md) | **Why ≥1 GHz**, link budgets, radio horizon, the band profile |
| [04-phy.md](docs/04-phy.md) | **The two-track decision**, single-carrier vs OFDM, the MCS ladder |
| [05-mac-tdma-time.md](docs/05-mac-tdma-time.md) | Superframe, slots, the three levels of time, rendezvous |
| [06-mesh-routing.md](docs/06-mesh-routing.md) | The IP model, Babel with a radio-aware metric, several ground stations |
| [07-antijam.md](docs/07-antijam.md) | Five mechanisms, and the two-channel null in detail |
| [08-architecture.md](docs/08-architecture.md) | The whole node, PS/PL split, where each part comes from |
| [09-fpga-budget.md](docs/09-fpga-budget.md) | **DSP does not fit**, and the four levers that make it fit |
| [10-reuse-inventory.md](docs/10-reuse-inventory.md) | What we take from each donor — including its failures |
| [11-test-rig.md](docs/11-test-rig.md) | The bench, the raw-IQ harness, metrics you can trust |
| [12-roadmap.md](docs/12-roadmap.md) | Phases with exit criteria; what can start today |
| [13-open-questions.md](docs/13-open-questions.md) | Questions, risks, and what will be settled by measurement |

Markers used throughout: **[V]** verified in code, docs or on hardware;
**[?]** needs checking; **[!]** risk or blocker. Inherited from `sdr-ground`,
along with the discipline behind them.

## A note on the two donors

`sdr-ground` earned its facts on this hardware: the AD9361 pinout on this
board matches **none** of the six published ones, and recovering it cost a
week. Its `docs/` records what was measured, what failed, and which
conclusions were later retracted.

`wy-lora` is the opposite case, and useful for it. Its architecture is
carefully thought through and much of it is carried over here. Its results,
however, come from a **UDP socket emulator**, not from the air — so the
claimed packet delivery rates and jamming resilience are properties of the
emulator. We take the decisions and leave the numbers
([docs/10](docs/10-reuse-inventory.md), §10.4). The six failures in its
development log are reproduced here as a checklist of what not to do.
