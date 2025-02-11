# Approach 1: Estimate stop arrival by projecting onto trip shape.


This approach estimates stop arrival time from vehicle positions by
projecting both onto the trip shape line. This approach has two
assumptions:

1.  Projecting using the shortest line to the trip shape is reasonable.
2.  The projection produces vehicle positions that are monotonically
    increasing over time along the shape line.

In this case, the distance along the line of each stop can be used to
linearly interpolated arrival time.

## Loading data

``` python
import polars as pl
import polars.selectors as cs
import polars_st as st
from pathlib import Path
from plotnine import *

# for geo plotting and some geo func workarounds ----
import pandas as pd
import geopandas as gpd


def from_wkb(wkb: pd.Series) -> gpd.GeoSeries:
    return gpd.GeoSeries.from_wkb(wkb)


# ----
# see https://github.com/Oreilles/polars-st/issues/1
import os
import pyproj

os.environ["PROJ_LIB"] = pyproj.datadir.get_data_dir()
# ---

CRS = "EPSG:3310"
```

``` python
tbl_shapes = pl.read_parquet("data/0-shapes.parquet")
tbl_trip_stops = pl.read_parquet("data/0-trip_stops.parquet")
tbl_trip_types = pl.read_parquet("data/0-trip_types.parquet")

tbl_vp = pl.read_parquet("data/0-vp.parquet").with_columns(
    location_timestamp_ms=pl.coalesce(
        (
            pl.col("location_timestamp_local")
            - pl.col("location_timestamp_local").first().over("trip_id")
        ).dt.total_milliseconds(),
        0,
    ),
)
```

## Projecting stops and vehicle positions onto trip shape line

We know stop and vehicle positions, along with the path for the trip.
The code below projects stop and vehicle position onto the trip line. It
then calculates its distance along the trip line (i.e. how many meters
into the trip is it?).

``` python
from shapely import from_wkb, line_locate_point


def cardinal_direction(
    distance_east: pl.Expr,
    distance_north: pl.Expr,
) -> str:
    """
    We can determine the primary cardinal direction by looking at the
    delta_x (distance_east) and delta_y (distance_north).
    From shared_utils.rt_utils
    """
    return (
        pl.when((distance_east > 0) & (distance_east.abs() > distance_north.abs()))
        .then(pl.lit("Eastbound"))
        .when((distance_east < 0) & (distance_east.abs() > distance_north.abs()))
        .then(pl.lit("Westbound"))
        .when((distance_north > 0) & (distance_east.abs() < distance_north.abs()))
        .then(pl.lit("Northbound"))
        .when((distance_north < 0) & (distance_east.abs() < distance_north.abs()))
        .then(pl.lit("Southbound"))
        .otherwise(pl.lit("Unknown"))
    )


def augment_stop_geometry(
    data: pl.DataFrame,
    df_shapes: pl.DataFrame,
    col_stop: st.GeoExpr,
    partition_by="trip_id",
    prefix="",
) -> pl.DataFrame:
    """Adds columns for the distance of the point (e.g. vehicle position, stop) along its shape line.

    Columns added:

    * `{prefix}meters`: distance along shape line in meters
    * `{prefix}cardinal_direction`: cardinal direction of point relative to previous point
    """
    name_meters = f"{prefix}meters"
    return (
        data
        # assumes df_shapes has a geometry_shape column
        .join(df_shapes, on="shape_id")
        .with_columns(geometry_prev=col_stop.shift(1).over(partition_by))
        #
        .with_columns(
            cardinal_direction(
                col_stop.st.x() - st.geom("geometry_prev").st.x(),
                col_stop.st.y() - st.geom("geometry_prev").st.y(),
            ).alias(f"{prefix}cardinal_direction"),
            # calculate distance within linestring to stop point
            # note that this method is not yet in polars-st, so we
            # just use shapely directly
            pl.struct(pl.col("geometry_shape"), col_stop)
            .struct.rename_fields(["shape", "stop"])
            .map_batches(
                lambda x: line_locate_point(
                    from_wkb(x.struct["shape"]),
                    from_wkb(x.struct["stop"]),
                )
            )
            .alias(name_meters),
        )
        .drop("geometry_shape")
    )


tbl_stop_times_enhanced = augment_stop_geometry(
    tbl_trip_stops,
    tbl_shapes,
    pl.col("geometry_stop"),
    partition_by="trip_id",
    prefix="stop_",
)

tbl_vp_enhanced = augment_stop_geometry(
    tbl_vp.join(
        tbl_trip_stops["trip_id", "shape_id"].unique(),
        "trip_id",
    ),
    tbl_shapes,
    pl.col("geometry_vp"),
    partition_by="trip_id",
    prefix="vp_",
)
```

    /Users/machow/repos/2025-transit-hackathon/.venv/lib/python3.11/site-packages/shapely/linear.py:88: RuntimeWarning: invalid value encountered in line_locate_point

### Visualizing vehicle distance over time

``` python
(
    ggplot(
        tbl_vp_enhanced.join(tbl_trip_types, "trip_id"),
        aes("location_timestamp_ms", "vp_meters", color="trip_type"),
    )
    + geom_point(size=0.1)
    + geom_line()
    + facet_wrap("~trip_id")
)
```

<img
src="1a-project-onto-shape_files/figure-commonmark/cell-5-output-1.png"
width="336" height="240" />

## Estimating stop arrival times for 1 trip

``` python
A_LOOPY_TRIP = "183-04u26szx9"
A_SIMPLE_TRIP = "47-3pbila5j7"

filtered_trips = tbl_trip_types.filter(
    pl.col("trip_id").is_in([A_SIMPLE_TRIP, A_LOOPY_TRIP])
).select("trip_id", "trip_type")

filtered_vp = (
    tbl_vp_enhanced
    #
    .join(filtered_trips, "trip_id")
    #
    .with_columns(
    ).select(
        "trip_type",
        "trip_id",
        "location_timestamp_local",
        "location_timestamp_ms",
        "vp_meters",
        "vp_cardinal_direction",
    )
)

filtered_stop_times = (
    tbl_stop_times_enhanced.join(filtered_trips, "trip_id")
    #
    .filter(pl.col("trip_id").is_in([A_SIMPLE_TRIP, A_LOOPY_TRIP]))
    #
    .select("trip_id", "stop_id", "stop_meters", "stop_cardinal_direction")
)
```

``` python
filtered_nested = (
    filtered_vp.group_by("trip_id")
    .agg("location_timestamp_ms", "vp_meters")
    .join(filtered_stop_times.group_by("trip_id").agg("stop_meters"), "trip_id")
)

filtered_nested
```

<div><style>
.dataframe > thead > tr,
.dataframe > tbody > tr {
  text-align: right;
  white-space: pre-wrap;
}
</style>
<small>shape: (2, 4)</small>

| trip_id | location_timestamp_ms | vp_meters | stop_meters |
|----|----|----|----|
| str | list\[i64\] | list\[f64\] | list\[f64\] |
| "47-3pbila5j7" | \[0, 16000, … 4335000\] | \[0.0, 0.0, … 36773.929228\] | \[73.114312, 347.803, … 36969.012661\] |
| "183-04u26szx9" | \[0, 40000, … 3680000\] | \[12718.714445, 12718.714445, … 12682.569115\] | \[0.0, 296.056419, … 0.0\] |

</div>

### Visualizing stop distance over time

``` python
import numpy as np

filtered_stop_arrivals = (
    filtered_nested.with_columns(
        stop_ms=pl.struct(
            "location_timestamp_ms", "vp_meters", "stop_meters"
        ).map_elements(
            lambda x: np.interp(
                x["stop_meters"], x["vp_meters"], x["location_timestamp_ms"]
            ),
            return_dtype=pl.List(pl.Float64),
        )
    )
    .drop("location_timestamp_ms", "vp_meters")
    .explode(["stop_meters", "stop_ms"])
)
```

``` python
(
    ggplot(
        filtered_stop_arrivals.join(tbl_trip_types, "trip_id"),
        aes("stop_meters", "stop_ms"),
    )
    + geom_line()
    + geom_point()
    + facet_wrap("~trip_type")
)
```

<img
src="1a-project-onto-shape_files/figure-commonmark/cell-9-output-1.png"
width="336" height="240" />
