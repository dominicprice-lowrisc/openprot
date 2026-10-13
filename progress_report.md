# openPRoT Progress Report

13/10/2026

---

## 1. Intro to openPRoT

*This section exists primarily for the benefit of readers who are new to openPRoT*

### Problem statement

Firmware runs at the bottom of the software stack, interacting directly with the hardware. A computer or server may have several microprocessors each controlling a device (such as a network card) that is attached to the motherboard. How can we verify that it has not been tampered with by an attacker?

A **Platform Root of Trust (PRoT)** is a small, dedicated security chip on the motherboard that guards against this. For example, it may

- Verify the signed hash distributed with the firmware binary using its public key, and fail to boot if the authenticity of the binary cannot be verified.
- Similarly, a firmware update may fail if the authenticity of the binary cannot be verified.
- Providing attestation: a remote party can ask what firmware the chip is running and get an answer it can trust.

docs/src/specification/services/(firmwareupdate|attestation).md respectively provide more detail on those two bullet points.

### Solution

**openPRoT** is an rust-based open-source firmware stack. The aim is a shared, auditable, vendor-neutral implementation that hardware makers can adopt, rather than each vendor writing its own closed one.


| Layer | Description | Source |
|---|---|---|
| **Kernel** | `pw_kernel`, a small microkernel from Google's Pigweed project. It runs separate programs ("apps") in isolation and lets them communicate over IPC. | `third_party/pigweed` |
| **Hardware support** | Drivers for each supported chip: I2C, SPI, flash controller, GPIO, crypto accelerator, etc. | `target/`, `drivers/` |
| **Hardware abstraction (HAL)** | Common interfaces, so services above don't depend on a particular chip. | `hal/`, `platform/` |
| **Services** | Separate apps that do the security work: talking to other devices, running firmware updates, etc. | `services/` |
| **Orchestrator** | The state machine that decides when devices boot, when an update may proceed, and what to do on failure. | `services/orchestrator` |


### Targets

| Target | What it is | Status | Source |
|---|---|---|---|
| **ASPEED AST1060 / AST10x0** | A commercial RoT chip used on real server boards | Main development and demo platform. OpenPRoT examples support the AST1060 Test Harness board. Customer EVB for the AST1060 is not specifically supported at this time. There is also emulation support via QEMU. | `target/ast10x0`, `target/ast10x0/BOARDS.md`, `target/ast10x0/README.md` |
| **OpenTitan Earl Grey** | An open-source silicon root of trust | Substantial work on a separate branch. There is an example project of the Pigweed kernel running on this chip. Has CW310, CW340, Verilator and QEMU runners | `target/earlgrey/README.md` |
| **Caliptra / VeeR** | An open-source RoT core, run in an emulator or on an FPGA (the latter is tagged as manual). | Basic support | `target/veer/caliptra_emulator_main.rs`, `target/veer/workflow.json` |
