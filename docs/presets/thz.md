# THZ preset (Stiebel Eltron LWZ, Tecalor THZ; serial interface)

Source: the [THZ integration](https://github.com/bigbadoooff/thz) (HACS, domain `thz`, GPL v3, based on the FHEM module 00_THZ.pm by Immi), release 0.7.2-beta.1. It reads the heat pump directly over its serial service interface (USB or ser2net), without an ISG. The preset is derived from the integration source: the register maps per firmware (`register_maps/`), `entity_translations.py`, `value_maps.py` and `strings.json`. It was first built from the source only; the additions marked "real installation" below come from a THZ on firmware 5.x read through the integration (entity registry and live states).

Units: the integral units LWZ 303/403/404/504 and THZ 303/304/403/404/504 (also SOL), and the LWZ 5/8 CS family. Heat pumps with an ISG and the `stiebel_eltron_isg` integration use the [`stiebel_isg`](stiebel-isg.md) preset instead.

Every entity of the integration carries a translation key, so the preset finds it through the entity registry (platform `thz`). Entity ids, the integration's entity-id style option (FHEM names or translated names) and the Home Assistant language do not matter.

## Setup

```yaml
type: custom:heatpump-flow-card
preset: thz
```

What you get depends on the firmware the integration detects:

| Firmware | Extra fields |
|---|---|
| 2.06, 2.14 | temperatures, flags, booster, room values |
| 4.19, 4.39 | + electrical and thermal power, electricity and heat counters per use, the integration's COP |
| 5.09, 5.39, 7.59 | + flow rate, dew points, compressor relative power, room temperature of circuit 2, cooling heat counter |

Some entities are disabled by default in the integration and are then not found:
- circuit 2: switch on the option "Heating circuit 2" of the integration;
- the booster flag (`booster_hc`): enable the entity by hand, else the card shows no booster.

With the integration's option "Split into sub-devices" (on for new entries) the entities belong to several devices (heating, hot water, compressor …). Do not set `device` then: it would restrict the preset to one of them.

## How the card gets its state

The integration has no operating-state sensor that the card can read as a mode: the select `op_mode` is the program the user chose, and `heat_request` is a raw number whose values are only partly documented. The preset therefore maps no `mode` entity, and the card derives the state from three flags: the compressor (`compressor`), the defrost flag (`defrost`) and the DHW heating flag of the operating status (`heating_dhw`). A running compressor means heating, with the DHW flag hot water, with the defrost flag defrost. **Cooling is not shown.**

## Mapped

- `compressor`, `defrost`, `valve`: binary sensors `compressor`, `defrost`, `heating_dhw` (block pxx0A0176).
- `flow` / `return`: `flow_temp` / `return_temp` (the heat pump). `volume`: `flow_rate` (l/min, 5.x only). See "Open questions" for the second `flow_rate` sensor.
- `hc_return` (circuit 1): `return_temp` as well. The THZ reports the return of the heat pump only, and circuit 1 is fed by the heat pump directly (real installation, underfloor circuit).
- `fan_speed`: `main_ventilator_speed` (Hz) (real installation).
- `dhw_mode`: `dhw_op_mode` (block pxxF3, e.g. `setback`), shown as text. Its states are the hot water program, never "heating hot water now", so the flow logic still follows `heating_dhw` (real installation).
- `cop_heating_today` / `cop_dhw_today`: the integration's `daily_cop_hc` / `daily_cop_dhw` (real installation).
- `power` / `heat`: `actual_power_pel` / `actual_power_qc` (W, from 4.19 on). With both the card shows the live COP.
- `outdoor`: `outside_temp`.
- `dhw_top` / `dhw_target`: `dhw_temp` / `dhw_set_temp`.
- Heating circuit 1: `hc_flow` from `heat_temp`, `hc_setpoint` from `heat_set_temp` (block pxxF4), `hc_pump` from `heating_circuit_pump`, else `pump_hc`. `hc_pressure`: `pressure_hc` where it reports.
- `room` / `room_target`: `inside_temp_rc` (remote control FEK / FE7), else `inside_temp`, and `room_set_temp`. `room_humidity`: `rel_humidity`. `dew_point`: `dew_point_hc1`, else `dew_point`.
- Heating circuit 2 (mixed circuit): `flow_temp_hc2`, `hc2_setpoint_temp`, the mixer position `hc2_mixer_valve` and the motor outputs `mixer_open` / `mixer_closed`; the room temperature `inside_temp_rc_hc2`, humidity `rel_humidity_hc2`, dew point `dew_point_hc2`. Circuit 2 counts when `flow_temp_hc2` reports a number; set `hc_count: 2`.
- `heater`: the booster flag `booster_hc` (block pxx0A0176) only. The stage fields `hc_booster_stage` (block pxxF4) and `heat_stage_control_modul` (block pxxF2) are not the electric inline heater: on a real installation both read 2 whenever the compressor ran, while the whole unit drew 0.55 to 0.74 kW and `booster_hc` stayed off. Two heater stages alone would draw several kW. Both stage fields are in the details.
- Solar thermal: `solar_collector` from `solar_collector_temp` and `solar_pump`, only when the solar block (pxx16) reports a value. The collector field of the status block exists on every unit and is not used.
- Refrigerant circuit: `hotgas_temp`, `evaporator_temp`, `condenser_temp`, `pressure_hd`, `pressure_nd`, `compr_rel_power` (%, 5.39) as compressor speed.
- `program`: the select `op_mode`; `setback` shows the night chip.
- `errors`: the sensor `fault_status` (`ok` / `fault`); `error_ok: ['ok']`.
- Quick controls: the select `op_mode`, the water heater `dhw` and the climate entity `heating_circuit`.
- Details: the compressor and booster runtimes, the booster energy, the recovered heat, the integration's current COP, season mode, the operating mode of the circuit, the two stage fields `hc_booster_stage` and `heat_stage_control_modul`, the latest fault; each where it reports.
- Tiles: electricity and heat per use (`electr_hc_*`, `electr_dhw_*`, `heat_hc_*`, `heat_dhw_*`): the day counters (Wh, corrected by the integration at the device's midnight reset) give "today", the totals (kWh) the other periods from long-term statistics. `heat_cooling` from `cool_hc_total`. `cop_total` from the integration's `daily_cop_total` and `lifetime_cop_total`.

## Deliberately not mapped

- `mode` and cooling: see above. The binary sensor `cooling` and the climate's `hvac_action` know about cooling, but one mode entity cannot combine them with the flags used for heating.
- `buffer_*`: the read map has no buffer temperature.
- `circulation`: the read map has no circulation pump output. `dhw_pump` is the unit's one pump while the diverter valve sends the flow into the DHW tank (real installation), not a circulation pump.
- `hc_volume`: see "Open questions".
- `hours`, `aux_hours`: the integration splits the runtimes into heating, hot water and cooling; they are in the details. Mapping one part would show it as the total.
- `energy_total` / `energy_today` and heat totals over all uses: the integration has them per use only; they are in the tiles.
- The ventilation of the integral units (fan stages, air flows, filters): no matching card key.

## Open questions

- Two sensors carry the translation key `flow_rate`: block pxx0A033B (`sflowrate`) is the heat pump flow and reports while the compressor runs; block pxxFB offset 55 (`flowrate`) is the heating circuit flow and drops to 0 when the circuit pump stops (real installation). The display registry has no `unique_id`, so the preset cannot tell them apart and `volume` takes the first one found. Once the integration gives the circuit flow its own key, it becomes `hc_volume`.

- Does `heating_dhw` switch off as soon as the diverter valve returns to heating? The binary sensor `diverter_valve` is the alternative.
- Is `heat_temp` the flow of circuit 1 on every unit, or the heat pump flow on units without a mixer?
- Do the units without solar thermal answer the solar block (pxx16) with plausible numbers?

## Evidence

- Integration source as listed in [`upstream/thz.json`](../../upstream/thz.json); the per-firmware entity list comes from the register maps merged by `RegisterMapManager` (`FIRMWARE_MAPS`) and from the snapshot test `tests_ha/test_firmware_matrix.py` of the integration.
- Field meanings of the blocks follow FHEM's `00_THZ.pm` (`sGlobal`, `sHC1`, `sDHW`, `F2ctrl`).
