# chip_core signoff — PROACTIVE workflow (fix first, run once, then debug residuals)

Goal: never spend a 5–6 h chip_core run (DRC alone ~2 h) on an unfixed design. So we **bake in the
high-probability fixes first**, run, and only then debug what's left. All paths use **step directories**
(`runs/<tag>/NN-<step>/…`) because `final/` does not exist until the whole flow finishes.

Run on the institute PC (≥32 GB), inside `$PROJ=$HOME/fabulous_self`, with
`source $HOME/fabulous_offline_bundle/activate.sh` done. Define once:
```bash
export PROJ=$HOME/fabulous_self PDK_ROOT=$PROJ/gf180mcu
TAIL="--pdk gf180mcuD --pdk-root $PDK_ROOT --manual-pdk --design-dir $PROJ \
  --skip Checker.SetupViolations --skip Checker.HoldViolations \
  --skip Checker.MaxSlewViolations --skip Checker.MaxCapViolations"
```

## The strategy (read this)
- **21466 Magic errors is huge → the bulk is almost certainly INSIDE `eFPGA_top`** (76 tiles, deferred /
  never DRC-signed-off), not chip-level. The dominant rule "via2 can't abut between subcells" is what a dense
  76-tile stitch produces. **No amount of chip_core tuning fixes macro-internal DRC** — so we confirm where
  the bulk is (Phase 0) before committing the long chip_core run.
- **DV (Dualgate)** is fixed once, chip-level, by a die-sized Dualgate blanket (covers eFPGA_top too).
- **LVS 5** is a separate chip_core pin/power fix.
- **XOR 112 (all Via3)** is benign tool rendering — waive.

---

# PHASE 0 — Localize the bulk: DRC `eFPGA_top` ALONE (cheaper than chip_core, decisive)

You still have the macro at `work/ip/efpga/eFPGA_top.gds`. DRC just that (≥32 GB handles it):
```bash
cd $PROJ
klayout -b -r $PDK_ROOT/gf180mcuD/libs.tech/klayout/tech/drc/gf180mcu.drc \
  -rd input=work/ip/efpga/eFPGA_top.gds -rd topcell=eFPGA_top -rd report=/tmp/efpga_drc.lyrdb \
  -rd thr=4 -rd run_mode=deep -rd metal_top=11K -rd mim_option=B -rd metal_level=5LM \
  -rd feol=true -rd beol=true -rd offgrid=true -rd conn_drc=false
echo "eFPGA_top violations by rule:"
grep -oE "<category>[^<]+</category>" /tmp/efpga_drc.lyrdb | sed -E 's#</?category>##g' | sort | uniq -c | sort -rn
```
**Branch:**
- **eFPGA_top shows the V2.1/V2.2a/V2.2b (and DF/CO) in the thousands** → the macro is the source → do
  **PHASE 1A + PHASE 2** (re-stitch it clean) before chip_core.
- **eFPGA_top is clean (only a handful or only DV)** → the bulk is chip-level → **skip Phase 2**, go to
  PHASE 1B + PHASE 3. (DV-only here is expected; the chip blanket fixes it.)

---

# PHASE 1 — Apply ALL proactive fixes now (before any long run)

## 1A — `eFPGA_top` Via2 fix in `work/gen_stitch.py` (only if Phase 0 said the macro is dirty)
Edit `work/gen_stitch.py`:
- **Line 16**, widen routing channels so the inter-tile router stops hugging tile edges:
  ```python
  GX=GY=160.0; MARGIN=150.0
  ```
  →
  ```python
  GX=GY=240.0; MARGIN=150.0
  ```
- In the config-emit list (the `L+=[...]` block, currently ends ~line 46 with `"MACROS:"]`), add a macro
  keep-out + push inter-tile routing off Via2. Change the line that begins `"RUN_CTS: false",…` to also emit:
  ```python
  "FP_MACRO_HORIZONTAL_HALO: 4","FP_MACRO_VERTICAL_HALO: 4","GRT_MACRO_EXTENSION: 0","RT_MIN_LAYER: Metal3",
  ```
  (Tile pins are Metal4/5, so Metal3-min routing still connects but eliminates M2↔M3 Via2 near tiles. If a
  later stitch run congests, drop `RT_MIN_LAYER: Metal3` and keep only the wider gap + halo.)

## 1B — chip_core config fixes in `work/librelane/chip_core_recon/config.yaml`
Insert **after line 25** (`GRT_ALLOW_CONGESTION: true`):
```yaml
FP_MACRO_HORIZONTAL_HALO: 6
FP_MACRO_VERTICAL_HALO: 6
GRT_MACRO_EXTENSION: 0
FP_TAP_HORIZONTAL_HALO: 2
FP_TAP_VERTICAL_HALO: 2
```
Insert **after line 17** (`CLOCK_PERIOD: 40`) for the LVS miso/pin fix:
```yaml
FP_PIN_ORDER_CFG: dir::work/librelane/chip_core_recon/pin_order.cfg
```
Create `work/librelane/chip_core_recon/pin_order.cfg` (puts spi_miso_o and fpga_miso_o on **different edges**
so netgen can't swap them):
```
#N
clk_i
rst_ni
fetch_enable_i
#S
spi_sclk_i
spi_cs_n_i
spi_mosi_i
spi_miso_o
#E
fpga_sclk_i
fpga_cs_n_i
fpga_mosi_i
fpga_miso_o
fpga_mode_i
config_busy_o
#W
core_sleep_o
boot_addr_i.*
```
**SRAM PDN bridge** (`work/librelane/chip_core/pdn_cfg.tcl`): SRAM VDD is on **Metal2**, VSS on **Metal1**.
Lines 198–199 are:
```tcl
add_pdn_connect -grid macro -layers "Metal2 $::env(PDN_VERTICAL_LAYER)"
add_pdn_connect -grid macro -layers "Metal3 $::env(PDN_VERTICAL_LAYER)"
```
Line 199 (Metal3) is dead (no SRAM M3 power pin) — leave it. Line 198 (Metal2) is needed for VDD but is the
Via2-on-SRAM-pin source. **Leave it for now** (dropping it would float SRAM VDD); the Via2 at the 8 SRAMs is a
small count — we’ll only revisit it in Phase 5 if those specific markers remain. Also add the Metal1 path for
VSS (currently only rails carry it) — add **after line 199**:
```tcl
add_pdn_connect -grid macro -layers "Metal1 $::env(PDN_VERTICAL_LAYER)"
```

## 1C — Dualgate blanket script (used in Phase 4, prepare now)
Create `work/add_dualgate.py`:
```python
import pya, sys
src, dst = sys.argv[1], sys.argv[2]
ly = pya.Layout(); ly.read(src); top = ly.top_cell()
dg = ly.layer(55, 0)                                  # Dualgate
bb = top.bbox(); m = int(round(2.0 / ly.dbu))         # +2 um margin (>> DV.6 0.24)
top.shapes(dg).insert(pya.Box(bb.left-m, bb.bottom-m, bb.right+m, bb.top+m))
ly.write(dst); print("dualgate blanket added ->", dst)
```

---

# PHASE 2 — Re-stitch `eFPGA_top` clean (ONLY if Phase 0 flagged the macro)

```bash
cd $PROJ
RECON=$PROJ/cv32e40x_updated_with_coproc_working_verilog_recon_efpga
python3 work/gen_stitch.py "$RECON/rtl/efpga/fabric/eFPGA.v" work/ip/efpga/tiles work/librelane/efpga_stitch/config.yaml
grep -E "RT_MIN_LAYER|GX=GY|FP_MACRO" work/gen_stitch.py work/librelane/efpga_stitch/config.yaml | head
rm -rf runs/efpga_stitch
eval librelane work/librelane/efpga_stitch/config.yaml $TAIL --run-tag efpga_stitch
# verify the macro's Via2 is now clean BEFORE using it:
klayout -b -r $PDK_ROOT/gf180mcuD/libs.tech/klayout/tech/drc/gf180mcu.drc \
  -rd input=runs/efpga_stitch/*magic-streamout*/eFPGA_top.gds -rd topcell=eFPGA_top \
  -rd report=/tmp/efpga_v2.lyrdb -rd table_name=via2 -rd run_mode=deep \
  -rd metal_top=11K -rd mim_option=B -rd metal_level=5LM -rd feol=true -rd beol=true -rd conn_drc=false
grep -c "<item>" /tmp/efpga_v2.lyrdb      # want ~0
# adopt the clean macro:
cp runs/efpga_stitch/*magic-streamout*/eFPGA_top.gds work/ip/efpga/eFPGA_top.gds
cp runs/efpga_stitch/*detailedrouting*/eFPGA_top.nl.v work/ip/efpga/eFPGA_top.nl.v
# LEF: if Magic.WriteLEF ran use it, else OpenROAD:
cp runs/efpga_stitch/*writelef*/eFPGA_top.lef work/ip/efpga/eFPGA_top.lef 2>/dev/null || \
  openroad -no_init -exit <<EOF
read_db runs/efpga_stitch/*fillinsertion*/eFPGA_top.odb
write_abstract_lef work/ip/efpga/eFPGA_top.lef
EOF
```
> If Via2 is still high after a 240 µm gap, that means tiles' OWN edge Via2 is the culprit → re-harden the
> offending tile type with a clamp/keep-out (Phase 5, §T) and re-stitch. Do Phase 0's per-tile check first:
> DRC one tile GDS (`work/ip/efpga/tiles/LUT4AB/LUT4AB.gds`) the same way — each tile should be 0.

---

# PHASE 3 — Run chip_core, but STOP at streamout (skip the 2 h DRC for now)

```bash
cd $PROJ
rm -rf runs/chip_core_recon
eval librelane work/librelane/chip_core_recon/config.yaml $TAIL --run-tag chip_core_recon --to KLayout.StreamOut
# this is route + GDS only (~3-4 h); the GDS lands here:
ls -la runs/chip_core_recon/*magic-streamout*/chip_core.gds runs/chip_core_recon/*klayout-streamout*/chip_core.gds
# also check power got connected (no point continuing if it didn't):
grep -ch PSM-0069 runs/chip_core_recon/*generatepdn*/*.log    # want 0
```
If `--to KLayout.StreamOut` errors before streamout (e.g. a disconnected-pins or RCX/STA abort), apply the
same skips we used for the macro: add `--skip Checker.DisconnectedPins --skip OpenROAD.RCX
--skip OpenROAD.STAPostPNR --skip OpenROAD.IRDropReport` and re-run. (RCX writes an empty SPEF on this PDK
and STA then fails; those are timing-only, irrelevant to DRC/LVS.)

---

# PHASE 4 — Patch the GDS (Dualgate) and run DRC+LVS ONCE on the fixed GDS

```bash
cd $PROJ
SO=$(ls -d runs/chip_core_recon/*magic-streamout* | head -1)        # streamout step dir
# 1) blanket Dualgate over the streamed GDS, overwriting in place so the flow's DRC uses it:
cp "$SO/chip_core.gds" "$SO/chip_core.beforeDG.gds"                  # backup
python3 work/add_dualgate.py "$SO/chip_core.beforeDG.gds" "$SO/chip_core.gds"
# (do the same for the klayout-streamout GDS so XOR/KLayoutDRC see it too)
KO=$(ls -d runs/chip_core_recon/*klayout-streamout* | head -1)
cp "$KO/chip_core.gds" "$KO/chip_core.beforeDG.gds"
python3 work/add_dualgate.py "$KO/chip_core.beforeDG.gds" "$KO/chip_core.gds"
# 2) resume the back end on the patched GDS — DRC + LVS in one ~2 h pass:
eval librelane work/librelane/chip_core_recon/config.yaml $TAIL --last-run \
  --from Magic.WriteLEF --to Checker.LVS
```
> If the resume refuses (state hash on the overwritten GDS), fall back: let the streamout-overwrite stand and
> run **standalone** DRC on the patched GDS (`gf180mcu.drc` per-table, §grep below) + standalone netgen LVS
> using `$SO/../*spice*`/the netlist. The in-flow resume is just the convenience path.

---

# PHASE 5 — Read the residual reports (STEP dirs) and debug what remains

```bash
cd $PROJ; R=runs/chip_core_recon
# Magic DRC residual by rule:
grep -ohE "via2|Diffusion|M3 |DV\.|DF\.|CO\.|[A-Z]+\.[0-9]+[a-z]?" $R/*checker-magicdrc*/*.{rpt,log} 2>/dev/null \
  | sort | uniq -c | sort -rn | head -30
# KLayout DRC residual by rule:
grep -oE "<category>[^<]+</category>" $R/*klayout-drc*/*.lyrdb 2>/dev/null | sed -E 's#</?category>##g' \
  | sort | uniq -c | sort -rn
# LVS:
sed -n '/Subcircuit summary/,/match/p' $R/*netgen-lvs*/*.log 2>/dev/null | head -120
```

### Residual branch — Via2 still high AND inside eFPGA_top → Phase 2 didn't fully clean it
Per-tile DRC to find the dirty tile type, then re-harden it:
```bash
for t in LUT4AB CIO W_IO N_term_single S_term_single; do
  klayout -b -r $PDK_ROOT/gf180mcuD/libs.tech/klayout/tech/drc/gf180mcu.drc \
    -rd input=work/ip/efpga/tiles/$t/$t.gds -rd topcell=$t -rd report=/tmp/$t.lyrdb \
    -rd table_name=via2 -rd run_mode=deep -rd metal_top=11K -rd mim_option=B -rd metal_level=5LM \
    -rd feol=true -rd beol=true -rd conn_drc=false >/dev/null 2>&1
  echo "$t via2: $(grep -c '<item>' /tmp/$t.lyrdb)"
done
```
- Tile with via2>0 → re-harden it: edit `work/tiles/<T>/config.yaml`, add `GRT_MACRO_EXTENSION: 0` and raise
  `PL_TARGET_DENSITY_PCT` down by 5; re-run `librelane work/tiles/<T>/config.yaml $TAIL --run-tag <tag>`,
  recollect into `work/ip/efpga/tiles/<T>/`, redo Phase 2+3+4.

### Residual branch — Via2 at the 8 SRAM VDD pins (markers cluster at SRAM locations)
The Metal2 bridge (pdn_cfg.tcl line 198) stacks Via2 on the SRAM M2 VDD pin. Replace the chip-added Via2 with
an offset: edit `work/librelane/chip_core/pdn_cfg.tcl`, change line 198 to add an x/y offset grid so the via
doesn't land exactly on the pin (or accept 8 small violations for a research run and waive them). Re-run
`--from OpenROAD.GeneratePDN --to Checker.KLayoutDRC`.

### Residual branch — DV.x still present after the blanket
Re-confirm the blanket landed: `klayout -b -rd g=$SO/chip_core.gds -r work/check55.py` (count layer 55/0 area).
If 0, the blanket script wrote the wrong file — point it at the actual streamout GDS path printed in Phase 3.
If DV.7 ("partial COMP overlap") appeared, increase the margin in `add_dualgate.py` (`2.0`→`5.0`) and redo.

### Residual branch — DF.9 / DF.3a / CO.4 dots at macro edges
Raise `FP_MACRO_*_HALO` (1B) from 6 to 10 and `FP_TAP_*_HALO` to 4; re-run `--from Floorplan`-onward (new tag,
since halo changes placement).

### Residual branch — LVS still fails
Read the off-by-one net (`sed` above). If a power net is split → fix `PDN_MACRO_CONNECTIONS` (config lines
36–38, exact instance path from `grep efpga_i $R/*/chip_core.nl.v`). If a miso net → the pin_order.cfg (1B)
should have fixed the swap; if still swapped, the net is genuinely open → `--from OpenROAD.GlobalRouting`.

### XOR (112 Via3) — waive
Add to config.yaml after line 35: `PRIMARY_GDSII_STREAMOUT_TOOL: klayout` and treat the KLayout GDS as
signoff; the 112 are via-cut rendering differences (confirmed all on Via3 40/0).

---

# Appendix — exact file/line map + step IDs
- `chip_core_recon/config.yaml`: 17 CLOCK_PERIOD (add FP_PIN_ORDER_CFG after) · 25 GRT_ALLOW_CONGESTION (add
  halos after) · 36-38 PDN_MACRO_CONNECTIONS · 39 PDN_CFG · 46 eFPGA location · 54-61 SRAM locations.
- `pdn_cfg.tcl`: 185 define_pdn_grid -macro · 192-194 add_pdn_connect Metal4/Metal5 · 198 Metal2 bridge ·
  199 Metal3 bridge.
- `gen_stitch.py`: 16 `GX=GY=…` · ~34-46 the config-emit `L+=[...]` list.
- Step IDs: Magic.StreamOut, KLayout.StreamOut, Magic.WriteLEF, Magic.DRC, KLayout.DRC, Magic.SpiceExtraction,
  Checker.MagicDRC, Checker.KLayoutDRC, Checker.LVS, Checker.XOR, Odb.CheckDesignAntennaProperties.
- SRAM: 431.86×484.88 µm, VDD=Metal2, VSS=Metal1. Grid 0.005 µm. Dualgate 55/0, Via2 38/0, Via3 40/0.



































# chip_core signoff — CONVERGE THE ROUTE FIRST, then DRC/LVS (corrected)

This supersedes the earlier version, which was built on three assumptions now **proven wrong**
against the actual LEFs and the LibreLane source:

1. **eFPGA_top is clean on its own** (you verified: 0 DRC / 0 LVS / 0 antenna). So the 21466 Magic
   errors are **NOT** inside the macro — there is **nothing to re-stitch** and **no Dualgate blanket
   needed**. Every chip_core error is *integration-level*.
2. **The SRAM has VDD/VSS power pins on Metal3** (a full-width Metal3 rail at the top edge, plus
   fingers), not "Metal2-only." So `Metal3→Metal4` is the *correct* PDN bridge and `Metal2→Metal4` is
   the one that creates the Via2 violations.
3. **`MAGIC_EXT_USE_GDS_BLACKBOX` does not exist** in LibreLane (unknown keys are silently ignored —
   it did nothing). The real lever is **`MAGIC_DRC_USE_GDS`** (default `true`).

Two further hard rules learned the hard way:
- **Never hand-edit `eFPGA_top.nl.v`'s port list.** Prepending `VDD, VSS` shifts the positional net
  order → that is what produced the `VSS↔VDD` and `spi_miso↔fpga_miso` swaps. If you touched it, revert it.
- **200↔1000 detailed-routing oscillation is a CONGESTION (floorplan) problem, not a DRC-rule problem.**
  Halos / pin-order / Dualgate do nothing for it. It must be fixed before any signoff discussion.

The strategy is therefore: **(A) get global routing to pass cleanly in a ~30-min loop → (B) full route +
streamout → (C) the three real signoff fixes → (D) residual branches.**

---

## Setup (institute PC, ≥32 GB)

```bash
source $HOME/fabulous_offline_bundle/activate.sh
export PROJ=$HOME/fabulous_self PDK_ROOT=$PROJ/gf180mcu
cd $PROJ
TAIL="--pdk gf180mcuD --pdk-root $PDK_ROOT --manual-pdk --design-dir $PROJ \
  --skip Checker.SetupViolations --skip Checker.HoldViolations \
  --skip Checker.MaxSlewViolations --skip Checker.MaxCapViolations"
CFG=work/librelane/chip_core_recon/config.yaml
PDN=work/librelane/chip_core/pdn_cfg.tcl
```

If you applied any of the institute-chat edits, **revert first**:
```bash
git checkout work/ip/efpga/eFPGA_top.nl.v 2>/dev/null   # or restore from prebuilt/
git checkout cv32e40x_updated_with_coproc_working_verilog*/rtl/efpga/xif_copro_efpga.v 2>/dev/null
# in $CFG: delete MAGIC_EXT_USE_GDS_BLACKBOX, any NETGEN_SETUP pointing at lvs_exclude/ignore files,
#          and put the eFPGA location back to its original (do NOT use [1300,120]).
# in $PDN: delete any define_pdn_grid sram_pdn / add_pdn_stripe Metal3 block you added.
```

---

# PHASE A — Make global routing converge (do this FIRST, ~30 min per iteration)

The oscillation happens because `GRT_ALLOW_CONGESTION: true` lets global routing "pass" while reporting
overflow, then hands detailed routing an infeasible problem (hours later you find out). We flip that: make
GRT the **fast, honest gate**, and only spend the full route once GRT is clean.

## A1 — Floorplan changes in `$CFG`

Your CPU + 8 SRAMs + the eFPGA's 590-pin edge are crammed into a ~1450 µm-wide left strip (eFPGA is
2716×6144, so it eats the right side). Give that strip room:

```yaml
GRT_ALLOW_CONGESTION: false          # ← the key change: fail fast instead of oscillating in DRT
DIE_AREA:  [0, 0, 4900, 6320]        # widen X by ~650
CORE_AREA: [20, 20, 4880, 6300]
PL_TARGET_DENSITY_PCT: 30            # keep low; congestion here is local, not global density
FP_MACRO_HORIZONTAL_HALO: 10
FP_MACRO_VERTICAL_HALO: 10
```

Move the eFPGA **right** (widens the logic strip 1450 → ~1900; the institute advice to move it *left* was
backwards), and spread the SRAMs so their Metal2 signal pins have escape room (channels ~68 µm → ~150 µm):

```yaml
MACROS:
  eFPGA_top:
    # ...gds/lef/nl unchanged...
    instances:
      "u_cpu.xif_copro_i.xif_copro_verilog_i.efpga_i": {location: [1900, 75], orientation: N}
  gf180mcu_fd_ip_sram__sram512x8m8wm1:
    # ...
    instances:
      "u_mem.g_bank[0].g_lane[0].u_sram": {location: [120, 120],  orientation: N}
      "u_mem.g_bank[0].g_lane[1].u_sram": {location: [700, 120],  orientation: N}
      "u_mem.g_bank[0].g_lane[2].u_sram": {location: [120, 720],  orientation: N}
      "u_mem.g_bank[0].g_lane[3].u_sram": {location: [700, 720],  orientation: N}
      "u_mem.g_bank[1].g_lane[0].u_sram": {location: [120, 1320], orientation: N}
      "u_mem.g_bank[1].g_lane[1].u_sram": {location: [700, 1320], orientation: N}
      "u_mem.g_bank[1].g_lane[2].u_sram": {location: [120, 1920], orientation: N}
      "u_mem.g_bank[1].g_lane[3].u_sram": {location: [700, 1920], orientation: N}
```
> The 8 SRAMs occupy only the bottom ~1900 µm of a 6320 µm-tall strip, so the upper strip is free for CPU
> logic — that's fine. The congestion is local to the SRAM channels + the eFPGA pin edge, which is exactly
> what A1 relieves.

## A2 — PDN fix in `$PDN` (also de-clutters the SRAM region for routing)

Delete the Metal2 bridge; keep only Metal3 (the SRAM's real wide Metal3 rail takes a clean Via3 to Metal4):
```tcl
# DELETE this line:
#   add_pdn_connect -grid macro -layers "Metal2 $::env(PDN_VERTICAL_LAYER)"
# KEEP only this:
add_pdn_connect -grid macro -layers "Metal3 $::env(PDN_VERTICAL_LAYER)"
```
The SRAM stays fully powered through its Metal3 pins (and it bridges Metal2↔Metal3 internally).

## A3 — Fast GRT-only run (placement + global route, NO detailed route, NO streamout)

```bash
rm -rf runs/chip_core_recon
eval librelane $CFG $TAIL --run-tag chip_core_recon --to OpenROAD.GlobalRouting
```
This is ~30 min, not 5 h. Then read the verdict (no pasting needed — you decide locally):
```bash
R=$(ls -td runs/chip_core_recon* | head -1)
grep -iE "congestion|overflow|GRT-00|tiles\b" $R/*globalrouting*/*.log | tail -40
```

## A4 — Branch on the GRT result

- **GRT finishes with no overflow / "congestion: 0"** → the route will converge. Go to **PHASE B**.
- **GRT errors with overflow, hotspot near the SRAM coordinates (x≈120–1130, y≈120–2400)** → SRAM
  pin-escape congestion. In `$CFG` widen the channels more (col2 `700`→`760`, row pitch `600`→`660`) and/or
  bump `FP_MACRO_*_HALO` to 14. Re-run A3.
- **GRT overflow along the eFPGA edge (x≈1900, full height)** → the 590-pin macro edge is saturating the
  strip. Widen the strip further: `DIE_AREA` X `4900`→`5400`, eFPGA `[1900,75]`→`[2300,75]`. Re-run A3.
- **GRT overflow spread globally (not localized)** → genuine density. Lower `PL_TARGET_DENSITY_PCT` `30`→`22`
  and re-run A3.
- **GRT passes only because you left `GRT_ALLOW_CONGESTION: true`** → it will still oscillate in DRT. Set it
  `false` and actually resolve the hotspot above; do not proceed on a masked pass.

Iterate A1–A4 (each ~30 min) until GRT is clean. **Only then** spend the long run.

---

# PHASE B — Full route + streamout (only after GRT is clean)

```bash
rm -rf runs/chip_core_recon
eval librelane $CFG $TAIL --run-tag chip_core_recon --to KLayout.StreamOut
R=$(ls -td runs/chip_core_recon* | head -1)
# detailed routing should now converge to 0 (or a handful), not oscillate:
grep -iE "number of violations|drc viol|completing" $R/*detailedrouting*/*.log | tail -10
# power must be connected:
grep -ch PSM-0069 $R/*generatepdn*/*.log    # want 0
ls -la $R/*magic-streamout*/chip_core.gds $R/*klayout-streamout*/chip_core.gds
```
If detailed routing *still* oscillates here even though GRT passed clean, that's a real DRT/Via issue, not
congestion — go to PHASE D §"DRT oscillates after clean GRT".

If `--to KLayout.StreamOut` aborts before streamout on a timing/extraction step, add the same skips we used
for the macro and re-run: `--skip OpenROAD.RCX --skip OpenROAD.STAPostPNR --skip OpenROAD.IRDropReport`
(RCX writes an empty SPEF on this PDK; those are timing-only, irrelevant to DRC/LVS).

---

# PHASE C — The three real signoff fixes

These are applied to `$CFG`/`$PDN` and then you run DRC+LVS once. C2 is already done if you did A2.

## C1 — Magic DRC 21466 → `MAGIC_DRC_USE_GDS: false`

Those 21466 are the **SRAM's 5 V devices** checked *flat*: `DV6+DV3` (5/6 V diffusion to 3.3 V tap),
"LV and MV dev can't be in the same well", `DF.8`, `CO.4`, `DF.3a`, `DF.9`. They are **inside PDK IP** that
GF signed off and waived; Magic re-flags them because it loses the SRAM's dualgate/waiver context when it
flattens the GDS. (KLayout doesn't fire them — that's why KLayout is 664, not 21k.)

Add to `$CFG`:
```yaml
MAGIC_DRC_USE_GDS: false      # Magic DRCs the abstract/DEF view → SRAM & eFPGA internals not flattened
```
Magic then checks only chip_core-level geometry (std cells, routing, PDN). KLayout (`run_mode: deep`,
hierarchy-aware) remains your authoritative signoff and already black-boxes the IP correctly — **leave
KLAYOUT_DRC_OPTIONS as-is; do NOT switch run_mode to hierarchical** (that won't remove real violations).
This is a legitimate signoff combo: each macro was DRC-clean on its own (eFPGA verified, SRAM is GF IP).

## C2 — KLayout/Magic Via2 (V2.1:384, V2.2b:60, M3.2a:4 = 448) → drop the Metal2 PDN bridge

Already done in **A2**. Mechanism: `add_pdn_connect "Metal2 Metal4"` forces a Via2→thin-Metal3-stub→Via3
stack; the auto stub is min-width (~0.26 µm) but Via2 needs ≥0.28 µm of surrounding metal (V2.1 = cut +
2×V2.3 enclosure) → 384 fails. The SRAM's real Metal3 rail makes the stub unnecessary, so removing the
Metal2 line eliminates all 448 at the source. (If you skipped A2, do it now.)

## C3 — LVS 5–6 errors → revert the hacks, then handle eFPGA power the supported way

Most of what you saw was self-inflicted by the nl.v/RTL edits. Order of operations:

1. **Revert** `eFPGA_top.nl.v` and `xif_copro_efpga.v` (Setup section). This alone clears the `VSS↔VDD` and
   `spi_miso↔fpga_miso` swaps — they were positional-port corruption, not real opens.
2. Run LVS and read the report (you decide locally):
   ```bash
   eval librelane $CFG $TAIL --last-run --from Magic.WriteLEF --to Checker.LVS
   R=$(ls -td runs/chip_core_recon* | head -1)
   sed -n '/Subcircuit summary/,/Cell pin lists/p' $R/*netgen-lvs*/*.log | head -150
   ```
3. **Branch on what the report shows:**
   - **Only `eFPGA_top` VDD/VSS unmatched (layout has the connection, schematic doesn't)** → expected for a
     hardened black box. Add the eFPGA to managed PDN so the connection is declared, in `$CFG`:
     ```yaml
     PDN_MACRO_CONNECTIONS:
       - ".*u_sram VDD VSS VDD VSS"
       - ".*efpga_i VDD VSS VDD VSS"
     ```
     and remove `eFPGA_top` from `IGNORE_DISCONNECTED_MODULES`. Re-run step 2. If 2 power-pin mismatches
     still remain, they are the known black-box supply pins and are waivable for a research tapeout —
     **do not** patch nl.v to "fix" them (that reintroduces the swap).
   - **A miso/signal net still mismatched after the revert** → now it's potentially a real connectivity
     issue. Check the RTL nets, don't guess:
     ```bash
     grep -nE "miso|fpga_miso|spi_miso" \
       cv32e40x_updated_with_coproc_working_verilog*/soc/chip_core.sv \
       cv32e40x_updated_with_coproc_working_verilog*/rtl/efpga/xif_copro_efpga.v
     ```
     If the two ports are driven by the same wire with an ambiguous name, give the chip-level wire its own
     explicit name in `chip_core.sv`. If they're genuinely separate, the layout net is open → re-run
     `--from OpenROAD.GlobalRouting` (it routed before the floorplan change; confirm it still connects).
   - **An off-by-one net count (e.g. 22171 vs 22172) with a power net split** → a supply got split by the PDN
     change. Re-check `PDN_MACRO_CONNECTIONS` instance globs match the real paths:
     ```bash
     grep -oE "u_sram|efpga_i" $R/*/chip_core.nl.v | sort | uniq -c
     ```

## C4 — KLayout residual (~216 after C2: density + antenna)

```yaml
RUN_HEURISTIC_DIODE_INSERTION: true
HEURISTIC_ANTENNA_THRESHOLD: 90
RUN_FILL_INSERTION: true
```
Antenna: the long CPU→eFPGA-pin routes need diodes (same fix that worked for the stitch run). Density: GF180
wants 20–80 % metal density per window; fill insertion covers the sparse strips. The wider strip from
PHASE A also helps fill meet the floor near the macro boundary.

## C5 — XOR 112 (all Via3, 40/0) → waive

These are via-cut rendering differences between the Magic and KLayout streamout views of a black-box macro —
not physical. Treat the KLayout GDS as signoff:
```yaml
PRIMARY_GDSII_STREAMOUT_TOOL: klayout
```

After C1–C5, run the back end once:
```bash
eval librelane $CFG $TAIL --last-run --from Magic.WriteLEF --to Checker.LVS
```
Expected: Magic DRC → small chip-level count, KLayout DRC → ~0–50 (residual density), XOR → 0, LVS → 0
(or 2 waived eFPGA supply pins).

---

# PHASE D — Residual branches (read the step dirs; decide locally)

```bash
R=$(ls -td runs/chip_core_recon* | head -1)
# Magic DRC residual by rule:
grep -ohE "[A-Z]+\.[0-9]+[a-z]?|via2|Diffusion|Dualgate" $R/*checker-magicdrc*/*.{rpt,log} 2>/dev/null \
  | sort | uniq -c | sort -rn | head -30
# KLayout DRC residual by rule:
grep -oE "<category>[^<]+</category>" $R/*klayout-drc*/*.lyrdb 2>/dev/null | sed -E 's#</?category>##g' \
  | sort | uniq -c | sort -rn
```

### DRT oscillates even after a clean GRT pass
Not congestion then — it's a Via/obstruction conflict. Check *where* the markers are:
```bash
grep -iE "Metal[0-9]|Via[0-9]|\( *[0-9]+ +[0-9]+ *\)" $R/*detailedrouting*/*.log | tail -40
```
- Clustered on **Via2/Metal3 at SRAM coords** → the Metal2 PDN bridge is still present; redo **A2/C2**.
- On **Metal4/Metal5 over the eFPGA** → over-the-macro routing is colliding with the macro's top-metal PDN.
  Add `GRT_MACRO_EXTENSION: 0` to `$CFG` and set `GRT_ALLOW_CONGESTION: false`; re-run from GlobalRouting.

### Via2 still high and the markers are INSIDE the SRAM footprint
That's SRAM-internal geometry leaking into KLayout — confirm `run_mode: deep` with `topcell: chip_core` is
set (it is in the baseline) so KLayout black-boxes the IP. If markers persist inside the SRAM, point the SRAM
`gds` in `$CFG` at a LEF-derived stub (boundary+pins only) — but only if C2 didn't clear them, which it
should.

### Magic DRC still ~thousands after C1
`MAGIC_DRC_USE_GDS: false` didn't take effect — confirm it's spelled exactly and is a top-level key in `$CFG`
(grep the resolved config): 
```bash
grep -i MAGIC_DRC_USE_GDS $R/resolved.json $R/*/resolved.json 2>/dev/null
```
If it shows `true`, the key is in the wrong scope/misspelled.

### LVS still fails after C3
Re-read the `Subcircuit summary`. The only acceptable residual is the 2 eFPGA supply pins (waivable). Any
signal-net mismatch is a real open/short — fix it in RTL or routing, **never** by editing nl.v port order.

---

# Appendix — verified facts (checked against the LEFs + LibreLane source)

- **eFPGA_top**: 2716.69 × 6144.52 µm; 590 signal pins on Metal3, on **both** left (x≈0.7) and right
  (x≈2715) edges; **no power pins in the OpenROAD-abstract LEF**, but the Magic-generated LEF (institute
  build) does carry Metal5 VDD/VSS. Verified 0 DRC/LVS/antenna standalone.
- **SRAM** `gf180mcu_fd_ip_sram__sram512x8m8wm1`: 431.86 × 484.88 µm. VDD/VSS pins on **Metal1, Metal2 AND
  Metal3**. Metal3 VDD/VSS is a full-width rail (`RECT 0 479.880 431.860 480.880`) → clean Via3 to Metal4.
  Contains 5 V devices → fires DV/DF/CO when checked flat (use C1).
- **gf180 PDN default layers**: RAIL=Metal1, VERTICAL=Metal4, HORIZONTAL=Metal5.
- **Real LibreLane variables** (the ones that exist): `MAGIC_DRC_USE_GDS` (default true),
  `GRT_ALLOW_CONGESTION`, `GRT_MACRO_EXTENSION`, `DRT_OPT_ITERS`, `PDN_MACRO_CONNECTIONS`,
  `PDN_CONNECT_MACROS_TO_GRID`, `LVS_FLATTEN_CELLS`, `RUN_HEURISTIC_DIODE_INSERTION`,
  `HEURISTIC_ANTENNA_THRESHOLD`, `RUN_FILL_INSERTION`, `PRIMARY_GDSII_STREAMOUT_TOOL`.
  **Do not exist** (silently ignored if used): `MAGIC_EXT_USE_GDS_BLACKBOX`, `lvs_exclude_port`,
  `ignore class` (netgen).
- **Step IDs**: OpenROAD.GlobalRouting, OpenROAD.DetailedRouting, OpenROAD.GeneratePDN, Magic.StreamOut,
  KLayout.StreamOut, Magic.WriteLEF, Magic.DRC, KLayout.DRC, Checker.MagicDRC, Checker.KLayoutDRC,
  Checker.LVS, Checker.XOR, OpenROAD.RCX, OpenROAD.STAPostPNR, OpenROAD.IRDropReport.
- **Layers**: Dualgate 55/0, Via2 38/0, Via3 40/0, COMP 22/0, Metal3 42/0. Grid 0.005 µm, DBU 2000/µm.
- **gf180 DRC rules referenced**: V2.1 (Via2 enclosure), V2.2a/V2.2b (Via2 spacing/array), M3.2a (Metal3
  spacing 0.28), DF.3a/DF.8/DF.9 (diffusion), CO.4 (contact), DV.3/DV.6 (dualgate 5 V spacing to 3.3 V tap).
```

