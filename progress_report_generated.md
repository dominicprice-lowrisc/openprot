# openPRoT Progress Report

*Snapshot as of 12 October 2026*

This report is for readers who are new to openPRoT. It explains what the project is,
what people are working on now, what has been tried along the way, and what is still
missing. A glossary of terms is at the end.

*Source note: the report was built from the project's git branches and commit history
as of 12 October 2026. Pull requests and issue discussions were not available, so
whether something is "under review" is inferred from commit messages.*

---

## 1. What openPRoT is

### The problem: who checks the firmware?

A modern server holds many processors besides its main CPUs: a management controller
(the BMC), network cards, storage controllers, and so on. Each one runs its own
**firmware**, low-level software that starts the device and controls it. An attacker
who can tamper with that firmware can take control of the machine in a way that is
very hard to detect or remove, because the firmware runs before, and below, everything
else.

A **Platform Root of Trust (PRoT)** is a small, dedicated security chip on the
motherboard that guards against this. Its jobs are to:

- **Check firmware before it runs** (secure boot). The RoT verifies that each
  device's firmware is genuine and has not been tampered with.
- **Supervise firmware updates.** New firmware is checked before it is installed. A
  device is not allowed to go back to an older, vulnerable version (anti-rollback).
- **Recover from failures.** If an update breaks a device, the RoT can restore a
  known-good copy.
- **Prove the platform's state to others** (attestation). A remote party can ask
  "what firmware are you running?" and get an answer it can trust.
- **Monitor the hardware.** For example, it watches traffic on the flash chips that
  hold other devices' firmware.

### What openPRoT provides

**openPRoT** is an open-source firmware stack, written in Rust, that runs *on* that
security chip. The aim is a shared, auditable, vendor-neutral implementation that
hardware makers can adopt, rather than each vendor writing its own closed one. The
work is associated with the Open Compute Project (OCP) community, and two of its
demo milestones are named after OCP summits.

The software is built in layers, from the hardware up:

| Layer | What it is | Where it lives |
|---|---|---|
| **Kernel** | `pw_kernel`, a small microkernel from Google's Pigweed project. It runs separate programs ("apps") in isolation and lets them talk through messages (IPC). | `pw/`, `third_party/pigweed` |
| **Hardware support** | Drivers for each supported chip: I2C, SPI, flash controller, GPIO, crypto accelerator, and so on. | `target/`, `drivers/` |
| **Hardware abstraction (HAL)** | Common interfaces, so services above don't depend on a particular chip. | `hal/`, `platform/` |
| **Services** | Separate apps that do the security work: talking to other devices, running firmware updates, crypto, and more. | `services/` |
| **Orchestrator** | The decision-maker: a state machine that decides when devices boot, when an update may proceed, and what to do on failure. | `services/orchestrator` |

### Which chips it runs on

| Target | What it is | Status |
|---|---|---|
| **ASPEED AST1060 / AST10x0** | A commercial RoT chip used on real server boards | Main development and demo platform |
| **OpenTitan Earl Grey** | An open-source silicon root of trust | Substantial work on a separate branch (§4) |
| **Caliptra / VeeR** | An open-source RoT core, run in an emulator or on an FPGA | Basic support |
| **QEMU** | An emulated AST1060 | Used for automated testing without hardware |

### How the RoT talks to other devices

The RoT has to communicate with the devices it protects. openPRoT uses three
industry-standard protocols from the DMTF standards body. Each one runs on top of the
one below:

```
  SPDM   – "prove who you are and what you're running"   (attestation, secure sessions)
  PLDM   – "here is a firmware update; install it"         (Type 5 = firmware update)
  ─────────────────────────────────────────────────────────
  MCTP   – the message transport that carries both of the above
  ─────────────────────────────────────────────────────────
  I2C / I3C – the physical wires between chips
```

Most of the past year's work has been building this stack from the bottom up: the
I2C driver, then MCTP, then SPDM and PLDM, then the orchestrator that uses them.

### How development is organised

- **`main`** is the shared trunk. Work reaches it after review.
- Bigger efforts happen on **feature branches**. Several of them were built to hit a
  **demo deadline**: a working end-to-end scenario shown at a conference. Demo
  branches sometimes contain shortcuts that are explicitly *not* meant for `main`.

---

## 2. The big picture right now

- **What most people are working on: the "OCP Global" demo.**
  - The scenario is a complete, end-to-end firmware update. The RoT receives a new
    firmware image for another device over PLDM, writes it to that device's flash
    chip, resets the device, and watches it boot the new firmware.
  - It runs on two real AST1060 boards controlled by a Raspberry Pi, and in emulation.
- **This work has not yet reached `main`.**
  - About 150 commits are spread across three related branches, two of which have
    diverged from each other. They will need reconciling before merging.
  - Nothing new has landed on `main` since 24 September.
- **The demo shows the *flow* of a secure update, but not yet the *security*.**
  - There is no crypto service yet, so firmware signatures are not actually checked.
  - The step that decides "should this update be allowed?" currently approves
    everything.
  - These are the most significant missing pieces.
- **A separate line of work on the OpenTitan Earl Grey chip has paused.** It includes
  USB-based updates and ownership transfer. Its branch has not been touched since
  July and is falling further behind `main`.
- **Small fixes are waiting for review.** Several are from an external contributor
  (Jabil).

---

## 3. Who is working on what

| Person | Working on | In plain terms |
|---|---|---|
| **Christina Quast** | Orchestrator, update state machine, automated emulator tests, design documentation | The "brain" that decides what happens during boot and update, and the tests that prove it behaves correctly, including in failure cases |
| **JesseMelon** | AST1060 hardware demo | Making the update work on real boards: writing to real flash chips, resetting the real target device, and the Raspberry Pi that runs the demo |
| **Anthony Rocha** | PLDM firmware update, I2C/MCTP performance, flash controller, fitting images into memory | Communication plumbing and the update protocol, plus making everything fit on the chip |
| **Chris Frantz**, **Anthony Chen** | OpenTitan Earl Grey support | Running openPRoT on the open-source OpenTitan chip, including updates over USB (paused since July) |
| **Evan Chuchen**, **PojuiLiu** (Jabil) | External contributions | Protocol library upgrade; bug fixes to the I2C and SPI-monitor drivers |
| **HaiLin Wu** | AST1060 flash controller and SPI monitor | Driver for the chips that hold firmware (merged) |
| **Will Marone** | SPDM, first conference demo, security disclosure policy | Earlier work (Feb–Apr); since taken over or merged by others |
| Steven Lee, linlin.xu, Courtney Drant, Miguel Osorio, Dave Roth | Other drivers (I3C, GPIO, OTP, crypto accelerator), FPGA target, build system and CI | Merged to `main` over the summer |

---

## 4. Current work streams

### 4.1 OCP Global demo: end-to-end firmware update (active)

**Goal:** show the RoT safely updating another device's firmware, end to end.

**The demo scenario:**

1. An update agent offers a new firmware image to the RoT over PLDM.
2. The RoT's **orchestrator** decides whether to accept it.
3. The image is streamed into the target device's flash chip.
4. The RoT resets the target device (a second board standing in for a server's BMC).
5. The RoT watches the device come back up running the new firmware.
6. The new version is then committed, which prevents rolling back to the old one.

**The branches:**

| Branch | Owner | What it is |
|---|---|---|
| `cq-demo-integration` | Christina Quast | A shared base collecting everyone's in-flight pieces: flash service, flash-chip discovery, GPIO reset/ready lines, I2C/MCTP fixes |
| `ocp-global-demo` | JesseMelon | The **hardware** version: two AST1060 boards and a Raspberry Pi |
| `ocp-global-demo-wip` | Christina Quast | The **emulated** version: automated tests in QEMU, including many failure scenarios, plus the start of the crypto service |
| `i2c-mctp-throughput` | Anthony Rocha | A performance test for the communication stack |
| `mctp-lib-main` | Evan Chuchen | An upgrade to the MCTP library, adding standard control messages |

The two demo branches share a common base but have each added work the other lacks.
Combining them is the next big integration task.

**What works:**

- **Orchestrator:**
  - supervises boot
  - handles an update from offer through to commit
  - keeps a spare copy of the firmware up to date
  - refuses updates it cannot handle
  - reports faults
- **PLDM firmware update:** runs as its own service, with commands to accept or
  refuse an offer.
- **Hardware demo, start to finish, on real boards:**
  - The image is written to shared flash, and the target board actually boots it.
  - The target can then send its *own* image back round the loop as a further
    update.
  - The whole demo runs from a Raspberry Pi.
- **Emulated tests cover many failure cases:**
  - the device hangs, or never comes back up
  - a corrupt image
  - the update is refused
  - a transfer error
  - the update is cancelled midway

### 4.2 OpenTitan Earl Grey support (paused)

**Branch:** `earlgrey-hwe` (Chris Frantz, Anthony Chen; last activity late July).

This branch runs openPRoT on the OpenTitan chip. It includes:

- a USB stack, with firmware updates over USB (DFU)
- an update manager that writes new firmware to internal and external flash
- **ownership transfer**: handing control of the chip to a new owner, with tests
  across all combinations of A/B firmware slots

Very little of this has reached `main`. A bot is meant to merge `main` into this
branch daily to keep it current, but the last merge was on 30 June, and the branch is
now 261 commits behind. The longer it waits, the harder it will be to bring back.

### 4.3 Earlier work, now finished or merged

The **OCP EMEA demo** (Feb–Apr 2026) was the first end-to-end demonstration. Two
AST1060 boards proved their identity to each other using SPDM over MCTP over I2C.
Its branches (`ocp-emea-demo*`, `spdm-standup`) are now historical. Their contents
were rebuilt and merged into `main` between May and August.

The **flash controller and SPI monitor** drivers for the AST1060 (`smc-*` branches)
are also largely merged.

---

## 5. What was tried, what didn't work, and what was done instead

This section lists the main changes of direction in the history. Commit references
are given in brackets for anyone who wants to look up the detail.

### Communication stack (first demo, spring 2026)

**I2C interrupts → polling → interrupts again.**

- **Tried:** the I2C driver was first built to react to hardware *interrupts*. This is
  efficient, but adding it overflowed a fixed-size kernel table, and interrupts
  misbehaved elsewhere in the system.
- **Instead:** for the demo, the team fell back to *polling*, checking the hardware
  repeatedly in a loop (`ac0125c`). A later rewrite of the I2C service on `main`
  restored interrupts.

**A buffer too small for security messages.**

- **Problem:** the AST1060's I2C hardware has a 32-byte buffer, and some SPDM
  messages are much larger.
- **Demo-only hack:** to get the demo working on time, the code sent a truncated
  message and filled in the rest from a hard-coded copy on the other side
  (`468bf41`). It was labelled "for demo purposes only, should not be merged".
- **Proper fix:** switch the I2C hardware to **DMA mode**, where it reads and writes
  memory directly. This needed a special uncached memory region (`897ea84`). `main`
  later got its own, cleaner DMA implementation.

**Hard-coded device addresses.** The first version assumed a fixed peer address. It
was made configurable, but a proper lookup table mapping protocol addresses to wire
addresses is still to do.

**Dependency on a vendor driver library.** The first demo used ASPEED's own driver
kit. It was later removed, and the drivers were rewritten inside openPRoT.

### Flash controller (summer 2026)

**Unreliable DMA reads from flash.**

- **Problem:** reading flash via DMA was unreliable, and the cause was suspected to
  be where the buffer sat in memory (`bd2140d`).
- **Fix:** the same uncached memory region. A faster DMA-based read path is still a
  TODO.

**Calibration redesign.**

- **Problem:** flash timing calibration relied on a shared global buffer, which is
  unsafe in Rust's terms.
- **Proposal:** a redesign that encodes "not yet calibrated" versus "calibrated" in
  the type system, so the compiler prevents misuse (`7bebd28`). This is **not
  merged**.

**SPI monitor testing.**

- **Problem:** the RoT's own flash controller can't be used to test the SPI monitor,
  because the monitor only sees traffic from *other* devices.
- **Instead:** testing moved to an emulated external master.
- **Open issue:** Jabil has since reported that some of the monitor's settings on
  `main` are wrong, including a reversed switch direction. A fix is waiting
  (`d396e7a`).

### Orchestrator and update demo (Aug–Oct 2026)

**Tests that passed without proving anything.** Several early automated tests passed
for the wrong reason:

- One counted a stuck device as a success.
- One passed even though the device never rebooted.
- In another, several programs could each declare the test passed, so the result
  depended on which one spoke first.

Each was tightened, so that the RoT alone decides the result and only after the
device has truly booted (`01d4aa6`, `5fb39f3`, `7687d2b`).

**A made-up test protocol replaced with the real one.** Early tests used a simplified
messaging scheme invented for the test. They now use the same client/server code the
product ships with, which immediately exposed a bug: the RoT was being asked about
the same offer twice (`7687d2b`).

**Staging updates in RAM → in flash.** Holding the incoming image in memory wasn't
realistic. It is now written to an emulated 64 MiB flash chip, as real hardware would
do (`9deba1b`). Along the way the team found that a program whose memory stack is too
small dies silently, with no error message.

**A flaky test.**

- **Problem:** the update test failed about 6 times in 10 because of a race when the
  two sides first discover each other.
- **Fix:** retry discovery, and stop the bus answering too early (`508aa74`).

**Working around a library bug.** The PLDM library reports a normal "please wait"
situation as an error. Until the library is fixed, openPRoT treats that error as
"wait and retry" (`08763db`).

**Simplifying the orchestrator.** It originally kept an internal queue of events.
This was removed: results now flow back directly, and the rule is that hardware
actions must never block (`7080f99`). The hardware demo currently breaks this rule.
It waits up to 30 seconds for the target to reboot.

**Fitting in memory.** The firmware images exceeded the chip's 640 KiB of fast memory
and were repacked to fit. They have been built but not yet run on hardware.

### Project infrastructure

**Keeping the Earl Grey branch in sync.** Switching the daily sync bot from merging to
rebasing was tried and reverted the next day (`992a1b3`). The bot now opens a daily
pull request with conflicts resolved by hand, but nobody has merged one since June.

**An automatic Pigweed updater** was set up but went unused. Updates to Pigweed are
done by hand.

---

## 6. What's missing

### Security essentials (the most important gaps)

- **No crypto service.**
  - Nothing actually verifies firmware signatures yet. Every place that should do so
    has a placeholder.
  - The low-level crypto interfaces also have 21 unimplemented functions, including
    signature validation.
- **The update approval step approves everything.** It does not yet check:
  - whether the image is for the right device
  - whether it is a newer version
  - whether the device can safely be updated now

  When it does refuse, the refusal is recorded but not enforced, and the sender isn't
  told.
- **Limited internal protection.**
  - By design, the services inside the RoT currently trust one another completely.
  - The orchestrator never sees the firmware bytes itself, so it cannot confirm that
    the image that was checked is the one that was installed.
  - These limits are documented, but there is no plan yet for addressing them.
- **The "lockdown latch" is undefined.** This is the mechanism for locking the
  system down after a serious failure.

### Shortcuts in the hardware demo

- The orchestrator runs inside the update service, not as its own isolated program.
- Boot verification is skipped, and the RoT's own checking and staging steps do
  nothing yet.
- Self-update of the RoT's own firmware is written but not connected.

### Scenarios not yet tested

The project documents these itself, in `target/ast10x0/tests/integration/GAPS.md`.

- **PLDM update standard (DSP0267):**
  - activation without an explicit command
  - the reset handshake
  - lost (rather than failed) messages and their timeouts
  - re-requesting data
  - a second update after an aborted one
- **Orchestrator:** security-version commits, commit timeouts, and recovery
  interrupting an update.
- **Emulated tests:** several stand in for real behaviour. For example, the checker
  reads a synthetic pattern rather than the actual installed bytes, and reset lines
  are simulated as messages.

### Drivers and platform

- No address lookup table for talking to more than one device over MCTP.
- Open TODOs in the AST1060 drivers:
  - SPI monitor: reset handling, violation log.
  - Flash controller: clock and calibration details, DMA reads.
  - I2C: some receive modes.

### Specification and services

The specification is at version 0.5, Work in Progress.

- The **attestation** chapter is mostly empty: 8 sections are marked TODO.
- **Firmware resiliency**, **recovery**, **secure boot** and **policy management**
  are marked TBD.
- The `storage` and `telemetry` services exist only as empty placeholders.
- There is no published roadmap.

---

## 7. Work waiting to be merged

| What | Branch | Notes |
|---|---|---|
| The OCP Global demo (about 150 commits) | `ocp-global-demo`, `ocp-global-demo-wip`, `cq-demo-integration` | The two demo branches must be reconciled first |
| I2C/MCTP throughput test | `i2c-mctp-throughput` | A single commit |
| MCTP library upgrade | `mctp-lib-main` | Small API changes |
| I2C fix: bus could hang on certain probes | `fix/ast10x0-i2c` | External (Jabil) |
| SPI-monitor settings fix | `fix/ast10x0-spim` | External (Jabil); likely fixes a real bug on `main` |
| Flash calibration redesign | `smc-typestate` | The rest of that branch is merged |
| SPDM stress test (about 2,100 lines) | `spdm-measurements` | The rest of that branch is merged |
| Security vulnerability disclosure policy | `psirt-security-policy` | Copied from Caliptra; needs adapting |
| Earl Grey USB updates, update manager, ownership transfer | `earlgrey-hwe` | Large; needs a decision on how to bring it back |

**Branches that can be cleaned up:**

- `pldm-fd-run-once`: already merged.
- `revert-261-synchronize-bot` and `automation/roll-pigweed`: obsolete.
- The `ocp-emea-demo*` family: historical, once anything worth keeping is saved.

---

## 8. Activity on `main` during 2026

| Month | Commits | What landed |
|---|---|---|
| Feb | 22 | Switched entirely to the Bazel build system; CI overhaul |
| Mar | 9 | Pigweed updates; Earl Grey chip definitions |
| Apr | 45 | AST1060 support, first MCTP pieces, documentation refresh |
| May | 41 | MCTP subsystem, AST1060 I2C driver, Earl Grey emulation |
| Jun | 96 | Flash controller and SPI monitor, MCTP server, SPDM tests, I3C |
| Jul | 71 | Device configuration tables, boot control, DMA memory region |
| Aug | 135 | **The orchestrator arrives**: state machine, fuzz testing, PLDM basics |
| Sep | 48 | Recovery and trial-boot support, PLDM firmware device, FPGA test runner |
| Oct | 0 | All activity is on the demo branches |

---

## 9. Suggested next steps

1. **Bring the demo work back to `main`.** Reconcile the two demo branches, then merge
   in reviewable pieces. Activity has been off `main` for three weeks, and the gap is
   growing.
2. **Build the crypto service and real update approval.** These turn the demo from a
   working *flow* into an actually *secure* update.
3. **Decide what happens to the Earl Grey branch.** Either resume regular syncs, or
   move its features to `main` piece by piece.
4. **Review the small pending fixes**, especially the SPI-monitor fix, which likely
   corrects a live bug.
5. **Rescue the SPDM stress test and security policy**, then delete stale branches.

---

## Glossary

| Term | Meaning |
|---|---|
| **Attestation** | Proving to a remote party what firmware a device is running, in a way it can trust |
| **BMC** | Baseboard Management Controller: a small computer on a server board for remote management, and a common target that the RoT protects |
| **Commit (of an update)** | Declaring new firmware good after it boots successfully. Usually also raises the minimum allowed version, so the old one can't be reinstalled. |
| **DFU** | Device Firmware Upgrade, a standard for updating firmware over USB |
| **DMA** | Direct Memory Access: hardware moving data to or from memory without the CPU copying it |
| **DMTF** | The standards body behind SPDM, MCTP and PLDM |
| **eRoT** | "Embedded" or "external" RoT: a root of trust chip guarding a specific device |
| **FPGA** | Reprogrammable hardware, used here to run a chip design before it exists as silicon |
| **I2C / I3C** | Simple two-wire buses connecting chips on a board |
| **IPC** | Inter-process communication: how isolated programs on the RoT send messages to each other |
| **MCTP** | Management Component Transport Protocol: carries management messages between chips |
| **Orchestrator** | openPRoT's central state machine that makes boot and update decisions |
| **OCP** | Open Compute Project, an industry community for open data-centre hardware |
| **Pigweed** | A Google open-source toolkit for embedded software; provides openPRoT's kernel and build tooling |
| **PLDM (Type 5)** | Platform Level Data Model; Type 5 (DSP0267) is the firmware-update protocol |
| **QEMU** | A hardware emulator, used to test without physical boards |
| **RoT / PRoT** | Root of Trust / Platform Root of Trust: the security chip this firmware runs on |
| **SPDM** | Security Protocol and Data Model: device identity, attestation and secure sessions |
| **SPI / SPI monitor** | SPI is the bus to flash memory chips. The SPI monitor watches and filters other devices' access to that flash. |
| **SVN** | Security Version Number, used to stop rollback to older vulnerable firmware |
