
# Tutorial 1: Loading and Sampling Trajectory Data

Real-world mobility files vary widely in structure and formatting. Timestamps may be recorded as UNIX integers or ISO-formatted strings, with or without timezone offsets. Coordinate columns may follow different naming conventions, and files may be stored either as flat CSVs or as partitioned Parquet directories. This notebook demonstrates how `nomad.io.base` standardizes data loading across these variations using two example datasets: a CSV file (`gc-data.csv`) and a partitioned Parquet directory (`gc-data/`).

## Inspecting schemas
Let's start by inspecting the schemas of the datasets we will use with the nomad helper function `table_columns` from the `io` module. This method reports column names for both flat files and partitioned datasets without reading the full content into memory.


```python
from nomad.io import base as loader

print(loader.table_columns("gc-data.csv", format="csv"))
print(loader.table_columns("gc-data/", format="parquet"))
```

    Index(['identifier', 'unix_timestamp', 'device_lat', 'device_lon', 'date',
           'offset_seconds', 'local_datetime'],
          dtype='object')
    Index(['user_id', 'timestamp', 'latitude', 'longitude', 'tz_offset',
           'datetime', 'date'],
          dtype='object')


## Loading data 

Reading data with `pandas` or Parquet readers does not enforce any particular schema, but spatiotemporal data often contains columns that must follow specific formats. The `from_file` function applies consistent type casting, converting temporal fields to `datetime` objects, ensuring coordinates are numeric, and optionally creating a `tz_offset` column to store timezone offsets when parsing datetime strings. This enables compatibility with engines like Spark, in which `Timestamp` objects cannot store timezone information. When column names differ from expected defaults, `from_file` accepts a `traj_cols` dictionary that maps standard names to the dataset’s column names, allowing downstream functions to locate required fields without renaming or altering the data.



```python
traj_cols = {
    "user_id": "identifier",
    "timestamp": "unix_timestamp",
    "latitude": "device_lat",
    "longitude": "device_lon",
    "datetime": "local_datetime",
    "tz_offset": "offset_seconds",
    "date": "date"
}
df_mapped = loader.from_file("gc-data.csv", format="csv", traj_cols=traj_cols)
df_mapped.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>identifier</th>
      <th>unix_timestamp</th>
      <th>device_lat</th>
      <th>device_lon</th>
      <th>date</th>
      <th>offset_seconds</th>
      <th>local_datetime</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>wizardly_joliot</td>
      <td>1704119340</td>
      <td>38.321711</td>
      <td>-36.667334</td>
      <td>2024-01-01</td>
      <td>0</td>
      <td>2024-01-01 14:29:00</td>
    </tr>
    <tr>
      <th>1</th>
      <td>wizardly_joliot</td>
      <td>1704119700</td>
      <td>38.321676</td>
      <td>-36.667365</td>
      <td>2024-01-01</td>
      <td>0</td>
      <td>2024-01-01 14:35:00</td>
    </tr>
    <tr>
      <th>2</th>
      <td>wonderful_swirles</td>
      <td>1704121560</td>
      <td>38.321017</td>
      <td>-36.667869</td>
      <td>2024-01-01</td>
      <td>-7200</td>
      <td>2024-01-01 13:06:00</td>
    </tr>
    <tr>
      <th>3</th>
      <td>youthful_galileo</td>
      <td>1704098820</td>
      <td>38.321625</td>
      <td>-36.666612</td>
      <td>2024-01-01</td>
      <td>0</td>
      <td>2024-01-01 08:47:00</td>
    </tr>
    <tr>
      <th>4</th>
      <td>youthful_galileo</td>
      <td>1704103140</td>
      <td>38.321681</td>
      <td>-36.666841</td>
      <td>2024-01-01</td>
      <td>0</td>
      <td>2024-01-01 09:59:00</td>
    </tr>
  </tbody>
</table>
</div>



This mapping makes the dataset compatible with nomad tools without modifying its original structure. Algorithms expecting standard names like timestamp, latitude, or user_id will work correctly, thanks to the dictionary.


```python
# This dataset has default column names, so no traj_cols argument is necessary
df_pq = loader.from_file("gc-data/", format="parquet", parse_dates=True)
df_pq.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>timestamp</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>tz_offset</th>
      <th>datetime</th>
      <th>date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>wizardly_joliot</td>
      <td>1704119340</td>
      <td>38.321711</td>
      <td>-36.667334</td>
      <td>0</td>
      <td>2024-01-01 14:29:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>wizardly_joliot</td>
      <td>1704119700</td>
      <td>38.321676</td>
      <td>-36.667365</td>
      <td>0</td>
      <td>2024-01-01 14:35:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>2</th>
      <td>wonderful_swirles</td>
      <td>1704121560</td>
      <td>38.321017</td>
      <td>-36.667869</td>
      <td>-7200</td>
      <td>2024-01-01 13:06:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>3</th>
      <td>youthful_galileo</td>
      <td>1704098820</td>
      <td>38.321625</td>
      <td>-36.666612</td>
      <td>0</td>
      <td>2024-01-01 08:47:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>4</th>
      <td>youthful_galileo</td>
      <td>1704103140</td>
      <td>38.321681</td>
      <td>-36.666841</td>
      <td>0</td>
      <td>2024-01-01 09:59:00</td>
      <td>2024-01-01</td>
    </tr>
  </tbody>
</table>
</div>



Even when GPS data is stored in partitioned directories (e.g. date=2024-01-01/), from_file seamlessly handles it, allowing users familiar with Pandas to simplify the inspection of partitioned datasets in parquet formats without worrying about data casting. 

## Working on smaller samples and persistence 
Large mobility datasets should typically not be fully loaded into the memory of a machine during interactive analysis, so subsampling by user is a common step in early analyses. nomad's `sample_users` selects a reproducible subset of user IDs, and `sample_from_file` filters the input dataset to include only those records. The resulting sample can be written to disk using `to_file`, partitioned by date in `hive` format to preserve compatibility with distributed engines. Reading the output back with `from_file` confirms that the sample was saved correctly and remains compatible with the same loading functions.


```python
users = loader.sample_users("gc-data/", format="parquet", size=10, seed=42)
sample_df = loader.sample_from_file("gc-data/", users=users, format="parquet")

loader.to_file(sample_df, "/tmp/nomad_sample", format="parquet", partition_by=["date"], existing_data_behavior='delete_matching')

round_trip = loader.from_file("/tmp/nomad_sample", format="parquet")
round_trip.head()
```

    C:\Users\pacob\Documents\notebooks\daphme\nomad\io\base.py:613: UserWarning: The 'datetime' column has timezone-naive records consider localizing or using unix timestamps.
      warnings.warn(f"The '{col}' column has timezone-naive records consider localizing or using unix timestamps.")





<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user_id</th>
      <th>timestamp</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>tz_offset</th>
      <th>datetime</th>
      <th>date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>wizardly_joliot</td>
      <td>1704119340</td>
      <td>38.321711</td>
      <td>-36.667334</td>
      <td>0</td>
      <td>2024-01-01 14:29:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>wizardly_joliot</td>
      <td>1704119700</td>
      <td>38.321676</td>
      <td>-36.667365</td>
      <td>0</td>
      <td>2024-01-01 14:35:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>2</th>
      <td>competent_torvalds</td>
      <td>1704114840</td>
      <td>38.320659</td>
      <td>-36.667228</td>
      <td>-7200</td>
      <td>2024-01-01 11:14:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>3</th>
      <td>competent_torvalds</td>
      <td>1704117060</td>
      <td>38.322056</td>
      <td>-36.667541</td>
      <td>-7200</td>
      <td>2024-01-01 11:51:00</td>
      <td>2024-01-01</td>
    </tr>
    <tr>
      <th>4</th>
      <td>competent_torvalds</td>
      <td>1704117120</td>
      <td>38.322075</td>
      <td>-36.667592</td>
      <td>-7200</td>
      <td>2024-01-01 11:52:00</td>
      <td>2024-01-01</td>
    </tr>
  </tbody>
</table>
</div>




```python

```
