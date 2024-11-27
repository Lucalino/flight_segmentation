---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.4
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# Flight segmentation HALO-20240906a

```python
import matplotlib
import yaml
import hvplot.xarray
import xarray as xr
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

from navdata import get_navdata_HALO
from utils import *

from orcestra.flightplan import bco, mindelo
cvao = mindelo
```

```python
platform = "HALO"
flight_id = "HALO-20240906a"
```

## Loading data
### Get HALO position and attitude

```python
ds = get_navdata_HALO(flight_id)
```

### Get dropsonde launch times

```python
drops = get_sondes_l1(flight_id)
ds_drops = ds.sel(time=drops, method="nearest")
```

### Defining takeoff and landing
On Barbads, the airport runway plus bumps make HALO move between 7.8-8m above WGS84, on Sal between 88.2-88.4m above WGS84. We therefore define the flight time such that altitude must be above 9m on Barbados and 90m on Sal.

```python
takeoff, landing, duration = get_takeoff_landing(flight_id, ds)
print("Takeoff time: " + str(takeoff))
print("Landing time: " + str(landing))
print(f"Flight duration: {int(duration / 60)}:{int(duration % 60)}")
```

### Get EC track and EC meeting point
**no EC meeting on that day as the flight was delayed by one day while the flight plan could not be adjusted to the new EC track due to ATC restrictions.**

```python
#ec_track = get_ec_track(flight_id, ds)
#dist_ec, t_ec = get_overpass_track(ds, ec_track)
ec_track = None
```

### Get PACE track
**loading the PACE track for the first time takes 6-7 minutes!**
Might be worth only if the flight report states a PACE coordination. Based on your decision, choose `load_pace = True` or `load_pace = False`!

```python
load_pace = False

if load_pace:
    from get_pace import get_pace_track
    _pace_track = get_pace_track(to_dt(takeoff), to_dt(landing))
    
    pace_track = _pace_track.where(
            (_pace_track.lat > ds.lat.min()-2) & (_pace_track.lat < ds.lat.max()+2) &
            (_pace_track.lon > ds.lon.min()-2) & (_pace_track.lon < ds.lon.max()+2),
            drop=True)
else:
    pace_track = None
```

### Get METEOR track
select maybe only the track from the respective flight day

```python
from orcestra.meteor import get_meteor_track

meteor_track = get_meteor_track().sel(time=slice(takeoff, landing))
```

## Overview plot: HALO track, EC meeting point, and dropsonde locations

```python
plt.plot(ds.lon.sel(time=slice(takeoff, landing)), ds.lat.sel(time=slice(takeoff, landing)), label="HALO track")
plt.scatter(ds_drops.lon, ds_drops.lat, s=10, c="k", label="dropsondes")
if ec_track: 
    plt.plot(ec_track.lon, ec_track.lat, c='C1', ls='dotted')
    plt.plot(ds.lon.sel(time=t_ec, method="nearest"), ds.lat.sel(time=t_ec, method="nearest"), marker="*", ls=":", label="EC meeting point")
if pace_track: plt.plot(pace_track.lon, pace_track.lat, c="C2", ls=":", label="PACE track")
plt.plot(meteor_track.lon, meteor_track.lat, c="C4", ls=":", label="METEOR track")
plt.xlabel("longitude / °")
plt.ylabel("latitude / °")
plt.legend();
```

## Interactive plots

```python
ds["alt"].hvplot()
```

```python
ds["roll"].hvplot()
```

```python
ds["heading"].hvplot()
```

## Segments

defined as a tuple of time slice (`start`, `end`) , segment `kind`, `name`, `remarks`.

* in case of irregularities within a circle, 1 sec before the first and after the last sonde are chosen as start and end times
* use the list of `remarks` to state any deviations, also with respective times

Alternatively, you can also define the segments as dictionaries which also allows to add further attributes to single segments, e.g. a `radius` to a `circle` segment. At the end of the following code block all segments will be normalized by the `parse_segments` function.

```python
sl1 = (
    slice("2024-09-06T10:43:06", "2024-09-06T11:05:05"),
    ["straight_leg", "ascent"],
    "leg 1 ascending and heading south-west",
    ["irregularity: roll angle deviation up to plus/minus 1.5 degree around 11:00:51"],
)

sl2 = (
    slice("2024-09-06T11:09:50", "2024-09-06T11:39:28"),
    ["straight_leg"],
    "leg 2 heading south-west",
)

sl3 = (
    slice("2024-09-06T11:39:28", "2024-09-06T11:47:55"),
    ["straight_leg", "ascent"],
    "leg 3 ascending and heading south-west",
)


sl4 = (
    slice("2024-09-06T11:47:55", "2024-09-06T12:18:27"),
    ["straight_leg"],
    "leg 4 heading south-west",
)

sl5 = (
    slice("2024-09-06T12:20:04", "2024-09-06T12:26:40"),
    ["straight_leg"],
    "leg 5 heading west",
)

sl6 = (
    slice("2024-09-06T12:28:00", "2024-09-06T12:37:19"),
    ["straight_leg"],
    "leg 6 heading west",
)

sl7 = (
    slice("2024-09-06T12:44:14", "2024-09-06T12:58:09"),
    ["straight_leg"],
    "leg 7 heading west",
)

sl8 = (
    slice("2024-09-06T13:01:59", "2024-09-06T13:07:03"),
    ["straight_leg"],
    "southmost leg 8 heading west",
    ["contains METEOR overpass"]
)

sl9 = (
    slice("2024-09-06T13:11:29", "2024-09-06T13:19:16"),
    ["straight_leg"],
    "southern leg 9",
)

sl10 = (
    slice("2024-09-06T13:19:36", "2024-09-06T14:07:18"),
    ["straight_leg"],
    "southern leg 10",
)

sl11 = (
    slice("2024-09-06T14:17:58", "2024-09-06T14:40:13"),
    ["straight_leg"],
    "southern leg 11",
)

sl12 = (
    slice("2024-09-06T14:42:58", "2024-09-06T14:54:11"),
    ["straight_leg"],
    "leg 12 heading north",
)

sl13_1 = (
    slice("2024-09-06T14:59:12", "2024-09-06T15:05:22"),
    ["straight_leg", "ascent"],
    "ascending while heading north",
)

sl13_2 = (
    slice("2024-09-06T15:05:22", "2024-09-06T15:20:20"),
    ["straight_leg"],
    "leg crossing circle",
)

c1 = (
    slice("2024-09-06T15:23:12", "2024-09-06T16:21:35"),
    ["circle"],
    "circle west",
)
# c1_1 = (
#     slice("2024-09-06T16:21:35", "2024-09-06T16:41:20"),
#     [],
#     "continued circling",
# )

sl14 = (
    slice("2024-09-06T16:44:26", "2024-09-06T17:30:05"),
    ["straight_leg"],
    "leg 14 heading north-west",
)
sl15 = (
    slice("2024-09-06T17:30:05", "2024-09-06T17:41:44"),
    ["straight_leg", "descent"],
    "leg 15 descending and heading north-west",
)

sl16 = (
    slice("2024-09-06T17:42:39", "2024-09-06T17:50:55"),
    ["straight_leg", "descent"],
    "leg 16 descending and heading west",
    ["irregularity: roll angle deviation up to -3.2 degree at 17:49:54"]
)
# add all segments that you want to save to a yaml file later to the below list
segments = [parse_segment(s) for s in [sl1, sl2, sl3, sl4, sl5,
                                       sl6, sl7, sl8, sl9, sl10,
                                       sl11, sl12, sl13_1, sl13_2, c1, sl14, sl15, sl16]] #c1_1 removed for now by commenting out
```

### Quick plot for working your way through the segments piece by piece
select the segment that you'd like to plot and optionally set the flag True for plotting the previous segment in your above specified list as well. The latter can be useful for the context if you have segments that are close or overlap in space, e.g. a leg crossing a circle.

```python
seg=parse_segment(c1_1)
add_previous_seg = True

###########################

fig = plt.figure(figsize=(12, 5))
gs = fig.add_gridspec(2,2)
ax1 = fig.add_subplot(gs[:, 0])

# extend the segment time period by 3min before and after to check outside dropsonde or roll angle conditions
seg_drops = slice(pd.Timestamp(seg["slice"].start) - pd.Timedelta("3min"), pd.Timestamp(seg["slice"].stop) + pd.Timedelta("3min"))
ax1.plot(ds.lon.sel(time=seg_drops), ds.lat.sel(time=seg_drops), "C0")

# plot the previous segment as well as the chosen one
if add_previous_seg:
    if segments.index(seg) > 0:
        seg_before = segments[segments.index(seg) - 1]
        ax1.plot(ds.lon.sel(time=seg_before["slice"]), ds.lat.sel(time=seg_before["slice"]), color="grey")
ax1.plot(ds.lon.sel(time=seg["slice"]), ds.lat.sel(time=seg["slice"]), color="C1")

# plot dropsonde markers for extended segment period as well as for the actually defined period
ax1.scatter(ds_drops.lon.sel(time=seg_drops), ds_drops.lat.sel(time=seg_drops), c="C0")
ax1.scatter(ds_drops.lon.sel(time=seg["slice"]), ds_drops.lat.sel(time=seg["slice"]), c="C1")

ax2 = fig.add_subplot(gs[0, 1])
ds["alt"].sel(time=seg_drops).plot(ax=ax2, color="C0")
ds["alt"].sel(time=seg["slice"]).plot(ax=ax2, color="C1")

ax3 = fig.add_subplot(gs[1, 1])
ds["roll"].sel(time=seg_drops).plot(ax=ax3, color="C0")
ds["roll"].sel(time=seg["slice"]).plot(ax=ax3, color="C1")

#Check dropsonde launch times compared to the segment start and end times
print(f"Segment time: {seg["slice"].start} to {seg["slice"].stop}")
print(f"Dropsonde launch times: {ds_drops.time.sel(time=seg_drops).values}")
```

### Identify visually which straight_leg segments lie on EC track

```python
seg = parse_segment(c1_1)
plt.plot(ds.lon.sel(time=slice(takeoff, landing)), ds.lat.sel(time=slice(takeoff, landing)))
plt.plot(ds.lon.sel(time=seg["slice"]), ds.lat.sel(time=seg["slice"]), color='red', label="selected segment", zorder=10)
plt.scatter(ds_drops.lon, ds_drops.lat, s=10, c="k", label="dropsondes")
if ec_track:
    plt.plot(ec_track.lon, ec_track.lat, c='C1', ls='dotted')
    plt.plot(ds.lon.sel(time=t_ec, method="nearest"), ds.lat.sel(time=t_ec, method="nearest"),
         marker="*", ls=":", label="EC meeting point", zorder=20)
if pace_track: plt.plot(pace_track.lon, pace_track.lat, c="C2", ls=":", label="PACE track")
plt.xlabel("longitude / °")
plt.ylabel("latitude / °")
plt.legend();
```

## Events
there was no EC meeting point on that day, but during the southern leg HALO deviated slightly to the south to fly over the METEOR (with 1km distance in the end)

```python
events = [
    meteor_event(ds, meteor_track),
]
events
```

## Save segments and events to YAML file

```python
yaml.dump(to_yaml(platform, flight_id, ds, segments, events),
          open(f"../flight_segment_files/{flight_id}.yaml", "w"),
          sort_keys=False)
```

## Import YAML and test it

```python
flight = yaml.safe_load(open(f"../flight_segment_files/{flight_id}.yaml", "r"))
```

```python
kinds = set(k for s in segments for k in s["kinds"])
```

```python
fig, ax = plt.subplots()

for k, c in zip(['straight_leg', 'circle', ], ["C0", "C1"]):
    for s in flight["segments"]:
        if k in s["kinds"]:
            t = slice(s["start"], s["end"])
            ax.plot(ds.lon.sel(time=t), ds.lat.sel(time=t), c=c, label=s["name"])
ax.set_xlabel("longitude / °")
ax.set_ylabel("latitude / °");
```

### Check circle radius

```python
from orcestra.flightplan import LatLon, FlightPlan, IntoCircle

for s in flight["segments"]:
    if "circle" not in s["kinds"]: continue
    d = ds.sel(time=slice(s["start"], s["end"]))
    start = LatLon(float(d.lat[0]), float(d.lon[0]), label="start")
    center = LatLon(s["clat"], s["clon"], label="center")
    FlightPlan([start, IntoCircle(center, s["radius"], 360)]).preview()
    print(f"Radius: {round(s["radius"])} m")
    plt.plot(d.lon, d.lat, label="HALO track")
    plt.legend()
```

```python

```