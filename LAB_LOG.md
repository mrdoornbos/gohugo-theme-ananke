# Lab Log

## 2026-05-10 — KIM-1-style 6809 board design

### Question: How hard is a KIM-1-like SBC around the 6809?

Not hard. The 6809 is hobbyist-friendly: clean bus, on-chip clock generator
(plain MC6809; the 6809E variant takes external E/Q clocks — pick one and
stick with it). Standard recipe:

- 74HC138 or small GAL/ATF22V10 for memory decode
- 32K SRAM (62256) + 8–16K EPROM/flash, skip DRAM entirely
- 6850 ACIA or 6551 for serial console (USB-serial bridge to host)
- 6522 VIA or 6821 PIA to scan a hex keypad and multiplex 6 × 7-seg LEDs
  (the original KIM-1's 6530 RRIOT is unobtainium; PIA + glue replicates it)

Real work is the **monitor ROM**: keypad scan, display refresh, examine/modify,
single-step, breakpoints, S-record load. ASSIST09 (Motorola) is the canonical
starting point. Open-source 6809 monitors from Grant Searle, Daryl Rictor,
Dave Dunfield exist to crib from.

**Chip notes**: NMOS MC6809 is scarce and runs hot — prefer **HD63C09**
(Hitachi CMOS, undocumented extra instructions, runs faster than spec).

**Effort estimate**: weekend on paper, few evenings in KiCad, ~$30–50 for
small-batch PCBs from JLCPCB. 2–4 weekends to first blink for a hobbyist
with prior digital hardware experience.

### Reference designs (note: Corsham Tech is gone — Bob Applegate passed away)

Living references:

- Grant Searle's "Simple 6809" — schematic + ROM still on his site
- Daryl Rictor's SBC pages (6502-focused but bus/monitor patterns translate)
- Dave Dunfield's archive (lots of 6809 monitor source incl. ASSIST09)
- Jeff Tranter's blog — recent 6809 builds
- 6809.org.uk wiki, groups.io 6809 lists
- CoCo (Tandy Color Computer) community — deep 6809 expertise, lwasm/lwtools
- Corsham docs mirrored on archive.org — grab before link rot

### Buyable today (clickable)

**Closest to KIM-1 experience (hex keypad + 7-seg, assembled or kit)**

- Wichit Sirichote 6809 Microprocessor Kit, ~$130 from Thailand
  - <https://kswichit.net/6809/6809.htm>
  - eBay: <https://www.ebay.com/itm/286592731740>

**Modular RC2014-style (serial console, no keypad)**

- Digicool Things MECB 6809 CPU Card (v2.3), kit form, ships from NZ
  - <https://www.tindie.com/products/digicoolthings/minimalist-europe-card-bus-6809-cpu-card/>
  - 6309 PLCC variant: <https://www.tindie.com/products/digicoolthings/minimalist-europe-card-bus-6309-cpu-plcc-card/>
  - Backplane: <https://www.tindie.com/products/digicoolthings/minimalist-ecb-mecb-backplane/>
  - Design notes: <https://digicoolthings.com/minimalist-europe-card-bus-mecb-6809-or-6309-cpu-card-v2-3-update/>

**PCB-only (source chips yourself)**

- Land Boards "Simple 6809" (Grant Searle lineage, MS BASIC)
  - <https://www.tindie.com/products/land_boards/simple-6809-cpu-pcb-only/>
- Computer For All — Bare Retro 6809 SBC
  - <https://www.tindie.com/products/akund/bare-retro-6809-sbc-pcb-only/>
  - <https://github.com/akund/retro-6809-SBC>
- Generic MC6809 SBC bare PCBs on eBay (~$10–20)
  - <https://www.ebay.com/itm/255111275907>

**Real chip without committing to a board**

- 8-Bit-Force RetroShield 6809 for Arduino Mega (Mega emulates RAM/ROM/IO)
  - <https://www.tindie.com/products/8bitforce/retroshield-6809-for-arduino-mega/>
  - Note: seller was on hiatus until 2026-04-30

**Recommendation**: Wichit Sirichote kit for the actual KIM-1 experience.
Digicool MECB for a flexible modular system to grow into.

### MiSTer FPGA core for a 6809 KIM-1

Very tractable — most plumbing already exists.

**1. CPU core — don't write from scratch.** Use an existing one already
proven on Cyclone V:

- John Kent's `cpu09` (VHDL, used in System09)
- Greg Estabrooks' 6809 (used in MiSTer Vectrex + CoCo cores)
- MAME-derived Verilog cores (Miller / Riddle / Giles)

The MiSTer CoCo3 and Vectrex cores both contain working 6809 — lift from
those rather than integrating fresh.

**2. Start from `MiSTer-devel/Template_MiSTer`** — gives HPS↔FPGA bridge,
OSD, scaler, HDMI/VGA, SDRAM, SD via HPS, USB controllers, audio. The
**Apple-I MiSTer** core is the best concrete template: 6502 + monitor +
character display + keyboard, almost identical scope.

**3. System composition**

- 6809 core, clocked via clock-enable off master (MiSTer convention — no
  gated clocks)
- 32–64K RAM in BRAM
- 8–16K monitor ROM in BRAM, initialized from .mif or loaded at runtime
  via `ioctl_download` from SD
- 6850 ACIA in HDL (~100 lines)
- 6821 PIA in HDL for keypad/display, or memory-map directly

**4. "Front panel"** — render virtual hex keypad + 7-seg as HDMI overlay
(~200 lines Verilog), map USB keyboard 0–9 / A–F / GO / ST to the matrix
the monitor ROM scans. Audio out can emulate KIM-1 cassette encoding so
programs save/load as .wav on SD.

**5. Monitor ROM** — port ASSIST09, assemble with **lwasm** (lwtools),
output Intel HEX → .mif or runtime-loaded.

**6. Build env** — Quartus Prime Lite **17.1** (MiSTer requires this exact
version; newer breaks things). Free.

**7. Effort**

- W1: Apple-I core building unmodified, flashed to DE10-Nano
- W2: Swap CPU → 6809, get ASSIST09 talking over UART (key milestone)
- W3–4: Virtual front panel, keypad mapping, polish
- W5+: Cassette audio, expansion (6840 timer, 6845 CRTC, etc.)

**8. Gotchas**

- Synchronous design with clock enables only — no gated clocks
- SDRAM not needed for 64K machine, stay in BRAM
- HDMI is fixed-rate; render low-res framebuffer, let scaler upscale

**Shortcut path**: **Multicomp** (Grant Searle / Neal Crook) already has a
working 6809 SoC with monitor + video + keyboard + SD on Cyclone II/IV.
Wrapping it in the MiSTer framework is the fastest route — couple of
weekends for a determined hobbyist.
