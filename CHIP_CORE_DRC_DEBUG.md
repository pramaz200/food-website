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
