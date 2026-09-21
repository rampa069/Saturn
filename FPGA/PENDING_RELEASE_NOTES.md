# Pending FPGA release (V28 candidate)

RTL changes from the Verilog review (branches `fix/fpga-fase-1` … `fix/fpga-fase-6`).
All changes are verified in RTL simulation only (`FPGA/sources/testbenches/prerelease_check.sh`,
21 self checking unit tests). **Nothing has been synthesised or tested on hardware yet.**

## Steps that need Vivado 2023.1

1. Open `FPGA/saturn_project/saturn_project.xpr`.
2. In each block design: *Refresh Changed Modules* (or accept the prompt), then *Report IP Status*
   and upgrade if asked. The CODEC_IQMOD_IP copies were synced with `verilogmodules/`.
3. Check these connections after the refresh:
   - `Wideband_Collect_0/m_axis` now has **TREADY**. It should join the existing interface
     connection to `byteswap_64_0/s_axis`. If it shows as unconnected, connect it: an unconnected
     TREADY is tied to 0 and every wideband word would be flagged as lost (data still flows).
   - `AXIS_DDC_Multiplexer_0` has a new parameter `SHUTDOWN_TIMEOUT` (default 16384): leave it.
   - `cw_key_ramp` has a new parameter `KEYDOWN_TIMEOUT_MS` (default 60000 = 60 s continuous key
     down; 0 disables). Change it on the block if a different limit is wanted.
   - `debounce` has a new parameter `INITIAL_LEVEL` (default 1 = input idles high): leave it.
4. Set the new firmware version: `xlconstant_swversion` in `saturn_top.bd` (V28).
5. Generate bitstream; check timing is met (WNS/WHS >= 0). New logic is small; the only new
   combinational path is TREADY -> stage-1 ready inside `AXIS_Deinterleaver`.
6. Generate `saturnprimary2026V28.bin` and add the README entry below.

## Hardware test list

- RX: all DDC rates incl. interleaved pairs; enable/disable the RX stream repeatedly.
- TX: SSB, CW (iambic A/B, straight key, IO8 external key in straight **and** iambic mode),
  EER/envelope mode on and off, PureSignal.
- CW: hold the key > 60 s -> RF must stop; release and key again -> works.
- Alex filter/antenna switching RX and TX; check relays after power-up/reset.
- Codec audio in/out and sidetone (volume at max: no wrap-around clicks).
- Wideband data collection (ADC0 and ADC1).
- ADC overload indication and peak values (Thetis ADC overload display).
- Codec SPI (codec init after power-up).

## README entry (draft)

V28. dd/mm/2026. Fixes from Verilog review: codec SPI and several AXI-Lite registers no longer hang
the bus on stray writes; TX I/Q can't get stuck on the envelope output after EER is turned off;
no lost FIFO/ADC overflow flags; ADC peak -32768 fixed; Alex TX word bits 31:16 no longer cause
endless SPI shifting and reset shift uses the normal SPI clock; iambic keyer dash length at high
weight and speed 0 fixed, IO8 keys in straight mode; 60 s CW key down limit; keys no longer read
as pressed at power-up; PWM drive 0 = no output; saturating sidetone arithmetic; DMA write
response fix; RX DDC mux can always shut down; wideband lost-data flag (control reg bit 31).

## Register / behaviour changes visible to software

| Register | Change |
|---|---|
| Wideband control 0xD000 (read) | bit 31 = words lost in last record (read only). Status 0xD00C unchanged. |
| Codec SPI 0x8 (busy) | also set between the write response and the start of the shift |
| FIFO monitor thresholds | threshold 0 = no threshold (software always sets the FIFO depth) |
| ADC overflow 0x5000 | reading 0x4/0x8 no longer clears overflow bits; 0x4/0x8 upper 16 bits always 0 |
| Any read-only register block | writes are now acknowledged (OKAY) and ignored instead of hanging |

No software change is required.
