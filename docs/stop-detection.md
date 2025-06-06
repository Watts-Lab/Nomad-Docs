# Tutorial 3: Stop detection in trajectories

This notebook shows how to process device-level trajectory data, in different formats, to detect stops using ```nomad```. Stop detection is an important step in
pre-processing trajectory data and in making sense of trajectories by grouping together pings that reflect stationary behavior. The output of stop-detection algorithms is commonly a "stop table", indicating when a stop started, its duration, and a pair of coordinates that approximates the location of the group of pings (typically the centroid). Alternatively, ```nomad``` allows users to retrieve a cluster label for each ping (useful for plotting, for example).


```python
import pandas as pd
import numpy as np
from datetime import timedelta
import pygeohash as gh
import geopandas as gpd
from matplotlib import cm
import matplotlib.pyplot as plt
import matplotlib.colors as mcolors
from pyproj import Transformer
```


```python
import nomad.io.base as loader
import nomad.constants as constants
import nomad.stop_detection.ta_dbscan as DBSCAN
import nomad.stop_detection.lachesis as Lachesis
import nomad.filters as filters
import nomad.city_gen as cg
```

## Load data sample

For these examples we load some test data from ```nomad``` which has the following trajectory columns. Defining this dictionary beforehands makes the handling of parameters more concise and helps the algorithms know which columns to use. 


```python
traj_cols = {'user_id':'uid',
             'datetime':'local_datetime',
             'latitude':'latitude',
             'longitude':'longitude'}

data = loader.from_file("../nomad/data/gc_sample.csv")
```


```python
data.head()
```

This synthetic data has records for 100 users for a 1 week period, with spherical coordinates (lat, lon) and datetime format for the time component of each ping. 

### Additional columns 

Nomad allows a degree of flexibility on the input trajectory data used for stop detection (and other algorithms), including common cases like datetime64[ns] formats for the time variable, ISO8601 string formats, or a pandas series with pandas.Timestamp objects. Similarly, it is often the case (and it can speed up stop-detection algorithms) that trajectory data has non-spherical coordinates with units in meters. These are useful for local analyses so that Euclidean distance can be used. 

To demonstrate this flexibility, we create some of these columns with alternative formats. 


```python
# We create a time offset column with different UTC offsets (in seconds)
data['tz_offset'] = 0
data.loc[data.index[:5000],'tz_offset'] = -7200
data.loc[data.index[-5000:], 'tz_offset'] = 3600

# create datetime column as a string
data['local_datetime'] = loader._unix_offset_to_str(data.timestamp, data.tz_offset)
data['local_datetime'] = pd.to_datetime(data['local_datetime'], utc=True)

# create x, y columns in web mercator
gdf = gpd.GeoSeries(gpd.points_from_xy(data.longitude, data.latitude),
                        crs="EPSG:4326")
projected = gdf.to_crs("EPSG:3857")
data['x'] = projected.x
data['y'] = projected.y

data.sample(5)
```

## Stop detection algorithms

The stop detection algorithms in ```nomad``` are applied to each user's trajectories separately. Thus, we demonstrate first by sampling a single user's data.


```python
user_sample = data.loc[data.uid == "angry_spence"]
user_sample.head()
```

For this user, the trajectory data has 1696 rows (pings) and covers a period of 15 days (start date: 2024-01-01, end date: 2024-01-15). We can visualize this trajectory below:


```python
%matplotlib inline

plot_df = user_sample.copy()
#transformer = Transformer.from_crs("EPSG:4326", "EPSG:3857", always_xy=True)
#plot_df['x'], plot_df['y'] = transformer.transform(plot_df['latitude'].values, plot_df['longitude'].values)
plot_df['x'] = (plot_df['x'] - 4265699)/15
plot_df['y'] = (plot_df['y'] + 4392976)/15

fig, ax = plt.subplots(figsize=(6, 6))
plt.box(on=False)

# Plotting Pings
ax.scatter(x=plot_df['x'], 
           y=plot_df['y'],
           s=6,
           color='black',
           alpha=1,
           zorder=2)

# Plotting Garden City Map
city = cg.load('garden-city.pkl')
city.plot_city(ax, doors=True, address=False)

ax.set_yticklabels([])
ax.set_xticklabels([])
ax.set_xticks([])
ax.set_yticks([])

plt.tight_layout()
plt.show()
```

### Sequential stop detection

The first stop detection algorithm implemented in ```nomad``` is a sequential algorithm insipired by the one in _Project Lachesis: Parsing and Modeling Location Histories_ (Hariharan & Toyama). This algorithm for extracting stays is dependent on two parameters: the roaming distance and the stay duration. 

* Roaming distance represents the maximum distance an object can move away from a point location and still be considered to be staying at that location.
* Stop duration is the minimum amount of time an object must spend within the roaming distance of a location to qualify as a stop.

The algorithm identifies stops as contiguous sequences of pings that stay within the roaming distance for at least the duration of the stop duration.

![png](filtering_files/lachesis_algo.png)

This algorithm has the following parameters, which determine the size of the resulting stops:
* ```dur_min```: Minimum duration for a stay in minutes.
* ```dt_max```: Maximum time gap permitted between consecutive pings in a stay in minutes (dt_max should be greater than dur_min).
* ```delta_roam```: Maximum roaming distance for a stay in meters.


```python
DUR_MIN = 60
DT_MAX = 120
DELTA_ROAM = 50
```

The Lachesis algorithm can output a complete table of attributes for identified stops, including the start time, end time, the medoid coordinates, duration, number of pings in the stop, and diameter.


```python
%%time
lachesis_stop_df = Lachesis.lachesis(traj=user_sample,
                                     dur_min=DUR_MIN,
                                     dt_max=DT_MAX,
                                     delta_roam=DELTA_ROAM,
                                     traj_cols=traj_cols,
                                     complete_output=True,
                                     keep_col_names = False,
                                     datetime='local_datetime',
                                     latitude= 'latitude',
                                     longitude='longitude')

lachesis_stop_df.head()
```


```python
lachesis_stop_df.columns
```

An additional argument, ```complete_output```, can be passed to only output the stop start time, duration, and medoid coordinates.


```python
%%time
Lachesis.lachesis(traj=user_sample,
                  dur_min=DUR_MIN,
                  dt_max=DT_MAX,
                  delta_roam=DELTA_ROAM,
                  traj_cols=traj_cols,
                  complete_output=False,
                  keep_col_names = False,
                  datetime='local_datetime',
                  latitude='latitude',
                  longitude='longitude').head()
```

We can also get the final cluster label for each of the pings, including those who were identified as noise.


```python
%%time
sample_labels_lach = Lachesis._lachesis_labels(traj=user_sample,
                                               dur_min=DUR_MIN,
                                               dt_max=DT_MAX,
                                               delta_roam=DELTA_ROAM,
                                               traj_cols=traj_cols,
                                               datetime='local_datetime')
sample_labels_lach.sample(n=5)
```

The data could also come with different formats for spatial and temporal variables, the algorithm can handle those situations as well.


```python
%%time
# Lachesis with x, y, and timestamp
Lachesis.lachesis(traj=user_sample,
                  dur_min=DUR_MIN,
                  dt_max=DT_MAX,
                  delta_roam=DELTA_ROAM,
                  traj_cols=traj_cols,
                  complete_output=False,
                  timestamp='timestamp',
                  x='x',
                  y='y').head()
```

Applying these stop detection algorithms to multiple users is straightforward with pandas' ```groupby``` and ```apply``` methods:


```python
mult_users = data.loc[data.uid.isin(["angry_spence", "stoic_almeida", "relaxed_colden", "dazzling_bassi"])]
mult_users.sample(10)
```


```python
%%time
mult_users.groupby(['uid']).apply(lambda x: Lachesis.lachesis(x.reset_index(),
                                                                  dur_min=DUR_MIN,
                                                                  dt_max=DT_MAX,
                                                                  delta_roam=DELTA_ROAM,
                                                                  traj_cols=traj_cols,
                                                                  complete_output=False),include_groups=False)
```

We can visualize the identified stops within the city detected by Lachesis for the sample user. Where pings of the same color represent pings belonging to the same cluster/stop and pings in grey are noise.

We can visualize the identified stops within the city detected by Lachesis for the sample user. Where pings of the same color represent pings belonging to the same cluster/stop and pings in grey are noise.


```python
%matplotlib inline

# Merging sample data with labels
merged_data_lach = user_sample.merge(sample_labels_lach.to_frame(name='cluster'), left_on='local_datetime', right_index=True)
#transformer = Transformer.from_crs("EPSG:4326", "EPSG:3857", always_xy=True)
#merged_data_lach['x'], merged_data_lach['y'] = transformer.transform(merged_data_lach['latitude'].values, merged_data_lach['longitude'].values)
merged_data_lach['x'] = (merged_data_lach['x'] - 4265699)/15
merged_data_lach['y'] = (merged_data_lach['y'] + 4392976)/15

fig, ax = plt.subplots(figsize=(6, 6))
plt.box(on=False)

# Plotting Garden City Map
city = cg.load('garden-city.pkl')
city.plot_city(ax, doors=True, address=False)

# Getting colors for clusters
unique_clusters = np.sort(merged_data_lach['cluster'].unique())
cluster_mapping = {cluster: i for i, cluster in enumerate(unique_clusters)}
mapped_clusters = merged_data_lach['cluster'].map(cluster_mapping).to_numpy()
cmap_base = plt.get_cmap('turbo', len(unique_clusters) - (1 if -1 in unique_clusters else 0))
colors = ['gray'] + list(cmap_base.colors)
extended_cmap = mcolors.ListedColormap(colors)

# Plotting Pings
ax.scatter(merged_data_lach['x'], 
           merged_data_lach['y'], 
           c=mapped_clusters, 
           cmap=extended_cmap, 
           s=6,
           alpha=1,
           zorder=2)

ax.set_yticklabels([])
ax.set_xticklabels([])
ax.set_title("Lachesis Stops for Sample User")
ax.set_xticks([])
ax.set_yticks([])

# plt.savefig('gc_empty.png')
plt.show()
```

### Density based stop detection (Temporal DBSCAN)

The second stop detection algorithm implemented in ```nomad``` is a time-augmented density-based algorithm, Temporal DBSCAN. This algorithm for clustering user pings combines temporal and spatial dimensions, relying on three key parameters: the time threshold, the distance threshold, and the minimum number of points.

* The time threshold defines the maximum time difference (in minutes) between two consecutive pings for them to be considered neighbors within the same cluster.
* The distance threshold specifies the maximum spatial distance (in meters) between two pings for them to be considered neighbors.
* The minimum points parameter sets the minimum number of points required for a dense region to form a cluster.

If a region contains fewer than minimum number of points required, it is treated as noise. The algorithm identifies clusters by grouping contiguous pings that meet both the temporal and spatial criteria, while also ensuring that each cluster has enough density to be considered valid. Our implementation of Temporal DBSCAN recursively processes the clusters obtained from DBSCAN to address the issue of some clusters overlapping in time.

![png](filtering_files/dbscan_algo.png)

This algorithm has the following parameters, which determine the size of the resulting stops:
* ```time_thresh```: Time threshold in minutes for identifying neighbors.
* ```dist_thresh```: Distance threshold in meters for identifying neighbors.
* ```min_pts```: Minimum number of points required to form a dense region (core point).


```python
TIME_THRESH = 100
DIST_THRESH = 40
MIN_PTS = 10
```

Similarly to Lachesis, the Temporal DBSCAN algorithm can output a complete table of attributes for identified stops, including the start time, end time, the medoid coordinates, duration, number of pings in the stop, and diameter.


```python
%%time
DBSCAN.temporal_dbscan(user_sample,
                       time_thresh=TIME_THRESH,
                       dist_thresh=DIST_THRESH,
                       min_pts=MIN_PTS,
                       traj_cols=traj_cols,
                       complete_output=True,
                       datetime='local_datetime',
                       latitude='latitude',
                       longitude='longitude').head()
```

The additional argument ```complete_output``` can also be passed to only output the stop start time, duration, and medoid coordinates.


```python
%%time
DBSCAN.temporal_dbscan(user_sample,
                       time_thresh=TIME_THRESH,
                       dist_thresh=DIST_THRESH,
                       min_pts=MIN_PTS,
                       traj_cols=traj_cols,
                       complete_output=False,
                       datetime='local_datetime',
                       latitude='latitude',
                       longitude='longitude').head()
```

We can also get the final cluster and core labels for each of the pings.


```python
%%time
sample_labels_dbscan = DBSCAN._temporal_dbscan_labels(user_sample,
                                                      time_thresh=TIME_THRESH,
                                                      dist_thresh=DIST_THRESH,
                                                      min_pts=MIN_PTS,
                                                      traj_cols=traj_cols,
                                                      datetime='local_datetime',
                                                      latitude='latitude',
                                                      longitude='longitude')

sample_labels_dbscan.sample(5)
```

The Temporal DBSCAN algorithm also handles data that comes with different formats for spatial and temporal variables.


```python
%%time
# Temporal DBSCAN with x, y, and timestamp
DBSCAN.temporal_dbscan(user_sample,
                       time_thresh=TIME_THRESH,
                       dist_thresh=DIST_THRESH,
                       min_pts=MIN_PTS,
                       traj_cols=traj_cols,
                       complete_output=True,
                       timestamp='timestamp',
                       x='x',
                       y='y').head()
```

We can also visualize the identified stops within the city detected by DBSCAN for the sample user. Again, pings of the same color represent pings belonging to the same cluster/stop and pings in grey are noise.


```python
%matplotlib inline

# Merging sample data with labels
merged_data_dbscan = user_sample.merge(sample_labels_dbscan[['cluster']], left_on='local_datetime', right_index=True)
#transformer = Transformer.from_crs("EPSG:4326", "EPSG:3857", always_xy=True)
#merged_data_dbscan['x'], merged_data_dbscan['y'] = transformer.transform(merged_data_dbscan['latitude'].values, merged_data_dbscan['longitude'].values)
merged_data_dbscan['x'] = (merged_data_dbscan['x'] - 4265699)/15
merged_data_dbscan['y'] = (merged_data_dbscan['y'] + 4392976)/15

fig, ax = plt.subplots(figsize=(6, 6))
plt.box(on=False)

# Plotting Garden City Map
city = cg.load('garden-city.pkl')
city.plot_city(ax, doors=True, address=False)

# Getting colors for clusters
unique_clusters = sorted(merged_data_dbscan['cluster'].unique())
cluster_mapping = {cluster: i for i, cluster in enumerate(unique_clusters)}
mapped_clusters = merged_data_dbscan['cluster'].map(cluster_mapping).to_numpy()
cmap_base = plt.get_cmap('turbo', len(unique_clusters) - (1 if -1 in unique_clusters else 0))
colors = ['gray'] + list(cmap_base.colors)
extended_cmap = mcolors.ListedColormap(colors)

# Plotting Pings
ax.scatter(merged_data_dbscan['x'], 
           merged_data_dbscan['y'], 
           c=mapped_clusters, 
           cmap=extended_cmap, 
           s=6,
           alpha=1,
           zorder=2)

ax.set_yticklabels([])
ax.set_xticklabels([])
ax.set_title("DBSCAN Stops for Sample User")
ax.set_xticks([])
ax.set_yticks([])

# plt.savefig('gc_empty.png')
plt.show()
```
