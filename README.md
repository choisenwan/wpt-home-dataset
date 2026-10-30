# wpt-home-dataset

Appliance usage traces from real homes, prepared for simulating a mobile robot that recharges battery-powered household appliances by wireless power transfer.

Usage times (when each appliance is on) come from plug-level power measurements in four public datasets: LEEDR, REFIT and UK-DALE from the UK, and DARCK from Germany. Each measured appliance is matched to a battery-powered product of the same kind that is sold today, and the simulation uses that product's published battery capacity and runtime. Appliances without a battery-powered equivalent, such as kettles and microwave ovens, are left out.

Version 2, released 2026-10-29.

## Contents

| Path | Content |
|---|---|
| `multihouse/households.csv` | 37 simulation windows in 24 households and the train / validation / test split |
| `multihouse/channels.csv` | all 767 plug channels of the 44 converted households, and which ones were kept |
| `multihouse/battery_models.csv` | the 30 battery models used for appliances |
| `multihouse/sessions/<house>.csv` | every usage session of the kept appliances (228,348 in total) |
| `multihouse/observed/<house>.csv` | periods in which each kept appliance has valid measurements |
| `multihouse/npz/<house>.npz` | minute-level arrays that our simulator reads directly |

House names join the source dataset and the original house number, for example `leedr_H09`, `refitb2_house6` (REFIT, cleaned release), `ukdale_1` and `darck_1`. Times are local time of the household (Europe/London or Europe/Berlin).

## Files

### households.csv

One row per simulation window. Most households have one primary window of 60 or 120 days; a few also have a secondary or a long window.

- `role`: train, validation, test or stress. The split is by household, so all windows of one house have the same role. Two REFIT houses marked `—` are not used.
- `window_start_local`, `window_days`, `warmup_days_not_counted`: the first 7 days of a window are a warm-up period and are not scored.
- `demand_wh_per_day`, `demand_share_of_robot_limit`: average energy the appliances use per day in the simulation, and its share of 331 Wh, the most our robot model can deliver in one day.
- `class`, `class_detail`: how hard the window is for three reference policies. In a normal window a hand-designed scoring rule (HS) never lets an appliance run empty; in a tight window HS does, but a planner that knows the future usage (oracle) does not; in a hard window the oracle fails as well.
- `baseline_HS_dead_min`, `oracle_dead_min`, `baseline_R1_dead_min`: appliance-minutes with an empty battery under HS, the oracle and a simple threshold rule (R1).
- `window_channels`, `window_models`: the appliances in the window and their battery models, in the same order.

### channels.csv

One row per plug channel of the source data.

- `decision` is keep, drop or exclude, with the reason in `reason_if_not_kept`. Drop means the appliance type has no battery-powered equivalent that a 15 W charger could keep going. Exclude covers whole circuits, always-on devices, channels with less than 60 days of data and channels whose power does not match their label.
- `battery_model_if_kept` is the model code used in battery_models.csv.
- The remaining columns are the parameters and results of the in-use rule described below.

### battery_models.csv

One row per battery model.

- `real_product` and `source`: the product the model is based on and where its specifications were taken from. Models with `origin` = own house were defined in our earlier single-household study; the others were added for these datasets.
- `battery_wh`, `drain_mode`, `runtime_min_full_battery`, `uses_per_full_charge`: battery size and how fast it drains. A `time` model loses 100 % / runtime per minute of use. An `event` model loses 100 % / uses per charge for each use.
- `charge_pct_per_min_at_15w`: charging speed when the robot delivers 15 W.
- `survives_min_if_charged_while_used`: how long a full battery lasts if the appliance is charged while in use; "unlimited" when its use power is 15 W or less.
- `critical_line_pct`: battery level below which the simulator treats the appliance as critical. For the added models it was copied from the closest own-house model, named in `critical_line_source`.

### sessions/ and observed/

`sessions/<house>.csv` lists `channel`, `model`, `start_local`, `end_local` (exclusive) and `minutes` for every usage session. `observed/<house>.csv` lists the periods with valid data for each appliance; outside them the appliance state is unknown, not off.

### npz/

| Array | Shape and type | Meaning |
|---|---|---|
| `active` | minutes x appliances, bool | appliance in use |
| `valid` | minutes x appliances, bool | measurement available |
| `intensity` | minutes x appliances, float32 | use intensity, kept for the simulator (1 while in use, 0 otherwise) |
| `models` | appliances, str | battery model of each column |
| `channels` | appliances, str | source channel of each column |
| `tz` | str | time zone |
| `day0_local_date` | str | local date of row 0 |

Row t is minute t counted from local midnight of `day0_local_date`.

```python
import numpy as np
import pandas as pd

z = np.load("multihouse/npz/leedr_H09.npz")
active = z["active"]        # (minutes, appliances)
models = list(z["models"])
day0 = str(z["day0_local_date"])

sessions = pd.read_csv("multihouse/sessions/leedr_H09.csv",
                       parse_dates=["start_local", "end_local"])
```

## How "in use" was derived (rule R3.2)

For each channel we first find its standby level, the lowest power it returns to every day (a steady level below 5 % of the working power also counts as standby). A minute is in use when the power is above standby x 1.3 + 2 W.

A stretch of at least 4 hours with almost constant power below 40 % of the usual working power is treated as a second standby level. This removes set-top boxes, idle desktop computers and consoles in rest mode. It is not applied to lamps, rechargeable lights, ceiling lights or fans.

Gaps of one minute are bridged, and only where data exist. Washing machines (30 min) and coffee makers (10 min) get longer bridges so that one programme or one brew is one use. Appliances counted per use must reach 10 % of their working power. Runs that are far too long are trimmed for washing machines and coffee makers and dropped for everything else (a tool battery charger on a drill socket, for example).

Any power above standby counts as use. This is a proxy derived from the power signal, not a record of whether someone was actually using the appliance.

## Simulation conventions

These are the settings our simulator uses with the data.

- A `time` model drains while in use. An `event` model turns each use into a fixed 5-minute block that uses 100 % / uses per charge, so the raw session length does not equal the energy used.
- When not in use, batteries self-discharge by 0.0001 % per minute.
- The robot charges an appliance at 15 W.
- Charging an appliance while it is in use counts as disturbing the user.

## Known limitations

- There is no ground truth for who used what. We checked the rule against hand-set thresholds in one household measured by our group: for 10 of 14 appliances the in-use minutes agree within 10 %.
- Working modes of set-top boxes and computers left on count as use, for example the TV socket of `refitb2_house17` (in use 43 % of the time) and the desktop in `ukdale_5` (49.8 %).
- Low-power idle periods drain the battery at the normal use rate, which is conservative.
- REFIT and DARCK come from cleaned releases in which short gaps were filled. On the two daylight-saving changeover days, durations can be off by up to one hour.
- The vacuum cleaner, hair dryer, hair iron and desk lamp models use assumed battery sizes from our earlier study rather than one specific product. 15 W is optimistic for small batteries.
- Only UK and German homes are included.

## Sources and license

The files in `multihouse/` are derived from four datasets released under the Creative Commons Attribution 4.0 International license (CC BY 4.0).

Changes made:
- appliance channels relabelled and matched to battery-powered products;
- channels removed when they are circuits, always-on devices, appliances without a battery-powered equivalent, or have less than 60 days of data;
- power converted to 0/1 in-use minutes on a 1-minute local-time grid (UK data converted from UTC to Europe/London);
- simulation windows and a train / validation / test split defined.

This derived dataset is released under the same license, CC BY 4.0 (see `LICENSE`). If you use it, please cite the original datasets:

- LEEDR: R. Buswell, L. Webb, P. Cosar-Jorda, D. Marini, S. Brownlee, M. Thomson, S.-H. Yang and R. Kalawsky. LEEDR project home energy dataset. Loughborough University, 2018. https://doi.org/10.17028/rd.lboro.6176450.v1
- REFIT: D. Murray, L. Stankovic and V. Stankovic. An electrical load measurements dataset of United Kingdom households from a two-year longitudinal study. Scientific Data 4, 160122, 2017. https://doi.org/10.1038/sdata.2016.122 (data: D. Murray, REFIT: Electrical Load Measurements (Cleaned), University of Strathclyde, 2016, https://doi.org/10.15129/9ab14b0e-19ac-4279-938f-27f643078cec)
- UK-DALE: J. Kelly and W. Knottenbelt. The UK-DALE dataset, domestic appliance-level electricity demand and whole-house demand from five UK homes. Scientific Data 2, 150007, 2015. https://doi.org/10.1038/sdata.2015.7 (data: https://doi.org/10.5286/UKERC.EDC.000001)
- DARCK: J. Breyer, K. Gützlaff, L. Pompe and K. Wehrle. Dataset: Device Activity Report with Complete Knowledge (DARCK) for NILM. In Proc. ACM BuildSys 2025. https://doi.org/10.1145/3736425.3771959 (data: https://doi.org/10.5281/zenodo.17159850)

The battery specifications in `battery_models.csv` come from the manufacturers' public product pages listed in the `source` column.

## Contact

Jintao Zhao, Graduate School of Science and Engineering, Hosei University. Questions and corrections are welcome as GitHub issues.
