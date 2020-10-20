## Self-defined Subfunction


```python
### number of pick-up and drop-off passengers per hours

def draw_hours(df_taxi, graph_title):
    # get pick-up and drop-off time only
    df_taxi_time = df_taxi.loc[:,['tpep_pickup_datetime','tpep_dropoff_datetime']]
    df_taxi_time['tpep_pickup_datetime'] = pd.to_datetime(df_taxi_time['tpep_pickup_datetime'])
    df_taxi_time['tpep_dropoff_datetime'] = pd.to_datetime(df_taxi_time['tpep_dropoff_datetime'])
    df_taxi_time['tpep_pickup_datetime'] = [time.time() for time in df_taxi_time['tpep_pickup_datetime']]
    df_taxi_time['tpep_dropoff_datetime'] = [time.time() for time in df_taxi_time['tpep_dropoff_datetime']]
    #display(df_taxi_time)
    
    # group by hours
    grp_PU = df_taxi_time['tpep_pickup_datetime']
    grp_PU = grp_PU.groupby(by=[grp_PU.map(lambda x : (x.hour))])
    grp_DO = df_taxi_time['tpep_dropoff_datetime']
    grp_DO = grp_DO.groupby(by=[grp_DO.map(lambda x : (x.hour))])
    display(grp_PU.count())
    display(grp_DO.count())
    
    # define the line
    grp_PU.count().plot(kind='line', label='Pick-up', marker='o', color='blue')
    grp_DO.count().plot(kind='line', label='Drop-off', marker='o', color='green')
    mean = plt.axhline(y=grp_PU.count().mean(), label='Mean', linestyle='--', color='tomato')
    
    # graph config
    plt.title(graph_title) 
    plt.legend(bbox_to_anchor=(1.05, 1), loc='upper left')
    plt.xlabel("Time (Hour)")
    plt.ylabel("Number of Passengers")
    plt.xticks(np.arange(0,24,4))
    #display(plt.xticks())
    plt.grid()
    
    plt.show()
    return

def zone_table(df_loc, loc_info):
    
    # group by location id
    df_loc_zone = df_loc.groupby(['LocationID']).size().reset_index(name = "count")
    df_loc_zone['percentage'] = 100 * df_loc_zone['count'] / df_loc_zone['count'].sum()

    # join the info of location id
    df_loc_zone = df_loc_zone.join(loc_info.set_index('LocationID'), on='LocationID', how='left')
    #df_loc_zone = df_loc_zone.merge(loc_info, left_on='LocationID', right_on='LocationID', how='left')

    # sort by count
    df_loc_zone = df_loc_zone.sort_values(by = ['count'], ascending = False).reset_index(drop=True)
    
    return df_loc_zone

def borough_table(df_loc_zone):
    df_loc_bor = df_loc_zone.loc[:,['count', 'borough']]
    df_loc_bor = df_loc_bor.groupby(['borough']).sum().sort_values(by = ['count'], ascending = False).reset_index()
    df_loc_bor['percentage'] = 100 * df_loc_bor['count'] / df_loc_bor['count'].sum()
    return df_loc_bor

def display_percent(df):
    format_dict = {'percentage':'{0:,.4f}%'}
    display(df.head(10).style.format(format_dict).hide_index())
    return

### sub function
def get_lat_lon(sf):
    content = []
    for sr in sf.shapeRecords():
        shape = sr.shape
        rec = sr.record
        loc_id = rec[shp_dic['LocationID']]
        
        x = (shape.bbox[0]+shape.bbox[2])/2
        y = (shape.bbox[1]+shape.bbox[3])/2
        
        content.append((loc_id, x, y))
    return pd.DataFrame(content, columns=["LocationID", "longitude", "latitude"])

```

# Load data


```python
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
import math

import shapefile
from shapely.geometry import Polygon
from descartes.patch import PolygonPatch
```


```python
### Load taxi
#df_taxi = pd.read_csv('green_tripdata_2020-01.csv')
df_taxi = pd.read_csv('https://s3.amazonaws.com/nyc-tlc/trip+data/yellow_tripdata_2020-02.csv')
#display(df_taxi)

### Load location
df_loc = pd.read_csv('https://s3.amazonaws.com/nyc-tlc/misc/taxi+_zone_lookup.csv')
#display(df_loc)
```

    /home/silverwing1997/anaconda3/envs/jupyter/lib/python3.7/site-packages/IPython/core/interactiveshell.py:3058: DtypeWarning: Columns (6) have mixed types. Specify dtype option on import or set low_memory=False.
      interactivity=interactivity, compiler=compiler, result=result)



```python
df_taxi = df_taxi[((df_taxi.loc[:,['trip_distance']] >= 0) & (df_taxi.loc[:,['trip_distance']] <= 1000)).all(1)]
display(df_taxi)
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
      <th>VendorID</th>
      <th>tpep_pickup_datetime</th>
      <th>tpep_dropoff_datetime</th>
      <th>passenger_count</th>
      <th>trip_distance</th>
      <th>RatecodeID</th>
      <th>store_and_fwd_flag</th>
      <th>PULocationID</th>
      <th>DOLocationID</th>
      <th>payment_type</th>
      <th>fare_amount</th>
      <th>extra</th>
      <th>mta_tax</th>
      <th>tip_amount</th>
      <th>tolls_amount</th>
      <th>improvement_surcharge</th>
      <th>total_amount</th>
      <th>congestion_surcharge</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>1.0</td>
      <td>2020-02-01 00:17:35</td>
      <td>2020-02-01 00:30:32</td>
      <td>1.0</td>
      <td>2.60</td>
      <td>1.0</td>
      <td>N</td>
      <td>145</td>
      <td>7</td>
      <td>1.0</td>
      <td>11.00</td>
      <td>0.50</td>
      <td>0.5</td>
      <td>2.45</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>14.75</td>
      <td>0.0</td>
    </tr>
    <tr>
      <td>1</td>
      <td>1.0</td>
      <td>2020-02-01 00:32:47</td>
      <td>2020-02-01 01:05:36</td>
      <td>1.0</td>
      <td>4.80</td>
      <td>1.0</td>
      <td>N</td>
      <td>45</td>
      <td>61</td>
      <td>1.0</td>
      <td>21.50</td>
      <td>3.00</td>
      <td>0.5</td>
      <td>6.30</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>31.60</td>
      <td>2.5</td>
    </tr>
    <tr>
      <td>2</td>
      <td>1.0</td>
      <td>2020-02-01 00:31:44</td>
      <td>2020-02-01 00:43:28</td>
      <td>1.0</td>
      <td>3.20</td>
      <td>1.0</td>
      <td>N</td>
      <td>186</td>
      <td>140</td>
      <td>1.0</td>
      <td>11.00</td>
      <td>3.00</td>
      <td>0.5</td>
      <td>1.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>15.80</td>
      <td>2.5</td>
    </tr>
    <tr>
      <td>3</td>
      <td>2.0</td>
      <td>2020-02-01 00:07:35</td>
      <td>2020-02-01 00:31:39</td>
      <td>1.0</td>
      <td>4.38</td>
      <td>1.0</td>
      <td>N</td>
      <td>144</td>
      <td>140</td>
      <td>1.0</td>
      <td>18.00</td>
      <td>0.50</td>
      <td>0.5</td>
      <td>3.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>24.80</td>
      <td>2.5</td>
    </tr>
    <tr>
      <td>4</td>
      <td>2.0</td>
      <td>2020-02-01 00:51:43</td>
      <td>2020-02-01 01:01:29</td>
      <td>1.0</td>
      <td>2.28</td>
      <td>1.0</td>
      <td>N</td>
      <td>238</td>
      <td>152</td>
      <td>2.0</td>
      <td>9.50</td>
      <td>0.50</td>
      <td>0.5</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>10.80</td>
      <td>0.0</td>
    </tr>
    <tr>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <td>6299349</td>
      <td>NaN</td>
      <td>2020-02-28 14:28:00</td>
      <td>2020-02-28 14:54:00</td>
      <td>NaN</td>
      <td>8.60</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>259</td>
      <td>247</td>
      <td>NaN</td>
      <td>30.81</td>
      <td>2.75</td>
      <td>0.5</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>34.36</td>
      <td>0.0</td>
    </tr>
    <tr>
      <td>6299350</td>
      <td>NaN</td>
      <td>2020-02-28 14:03:00</td>
      <td>2020-02-28 14:11:00</td>
      <td>NaN</td>
      <td>3.01</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>212</td>
      <td>31</td>
      <td>NaN</td>
      <td>22.59</td>
      <td>2.75</td>
      <td>0.5</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>26.14</td>
      <td>0.0</td>
    </tr>
    <tr>
      <td>6299351</td>
      <td>NaN</td>
      <td>2020-02-28 14:55:00</td>
      <td>2020-02-28 15:02:00</td>
      <td>NaN</td>
      <td>0.71</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>42</td>
      <td>42</td>
      <td>NaN</td>
      <td>13.43</td>
      <td>2.75</td>
      <td>0.5</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>16.98</td>
      <td>0.0</td>
    </tr>
    <tr>
      <td>6299352</td>
      <td>NaN</td>
      <td>2020-02-28 14:29:00</td>
      <td>2020-02-28 14:51:00</td>
      <td>NaN</td>
      <td>12.40</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>38</td>
      <td>53</td>
      <td>NaN</td>
      <td>30.54</td>
      <td>2.75</td>
      <td>0.5</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>34.09</td>
      <td>0.0</td>
    </tr>
    <tr>
      <td>6299353</td>
      <td>NaN</td>
      <td>2020-02-28 14:06:00</td>
      <td>2020-02-28 14:38:00</td>
      <td>NaN</td>
      <td>6.96</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>226</td>
      <td>135</td>
      <td>NaN</td>
      <td>30.30</td>
      <td>2.75</td>
      <td>0.5</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>0.3</td>
      <td>33.85</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
<p>6299349 rows × 18 columns</p>
</div>



```python
### load map
sf = shapefile.Reader("shape2/taxi_zones.shp")
fields_name = [field[0] for field in sf.fields[1:]]
shp_dic = dict(zip(fields_name, list(range(len(fields_name)))))
attributes = sf.records()
shp_attr = [dict(zip(fields_name, attr)) for attr in attributes]
df_map = pd.DataFrame(shp_attr).join(get_lat_lon(sf).set_index("LocationID"), on="LocationID")
display(df_map.head())

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
      <th>OBJECTID</th>
      <th>Shape_Leng</th>
      <th>Shape_Area</th>
      <th>zone</th>
      <th>LocationID</th>
      <th>borough</th>
      <th>longitude</th>
      <th>latitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>1</td>
      <td>0.116357</td>
      <td>0.000782</td>
      <td>Newark Airport</td>
      <td>1</td>
      <td>EWR</td>
      <td>-74.171533</td>
      <td>40.689483</td>
    </tr>
    <tr>
      <td>1</td>
      <td>2</td>
      <td>0.433470</td>
      <td>0.004866</td>
      <td>Jamaica Bay</td>
      <td>2</td>
      <td>Queens</td>
      <td>-73.822478</td>
      <td>40.610824</td>
    </tr>
    <tr>
      <td>2</td>
      <td>3</td>
      <td>0.084341</td>
      <td>0.000314</td>
      <td>Allerton/Pelham Gardens</td>
      <td>3</td>
      <td>Bronx</td>
      <td>-73.844953</td>
      <td>40.865747</td>
    </tr>
    <tr>
      <td>3</td>
      <td>4</td>
      <td>0.043567</td>
      <td>0.000112</td>
      <td>Alphabet City</td>
      <td>4</td>
      <td>Manhattan</td>
      <td>-73.977725</td>
      <td>40.724137</td>
    </tr>
    <tr>
      <td>4</td>
      <td>5</td>
      <td>0.092146</td>
      <td>0.000498</td>
      <td>Arden Heights</td>
      <td>5</td>
      <td>Staten Island</td>
      <td>-74.187558</td>
      <td>40.550664</td>
    </tr>
  </tbody>
</table>
</div>


## Q1: What are the most pickups and drop offs region?


```python
# pick-up and drop-off zone
df_PU_loc = df_taxi.loc[:,['PULocationID']].rename(columns={'PULocationID':'LocationID'})
df_DO_loc = df_taxi.loc[:,['DOLocationID']].rename(columns={'DOLocationID':'LocationID'})
loc_info = df_map[["LocationID", "zone", "borough"]]
df_PU_zone = zone_table(df_PU_loc, loc_info)
df_DO_zone = zone_table(df_DO_loc, loc_info)

# output
format_dict = {'percentage':'{0:,.4f}%'}
print("\nPick-up :")
display_percent(df_PU_zone.head(10))
print("\nDrop-off :")
display_percent(df_DO_zone.head(10))

```

    
    Pick-up :



<style  type="text/css" >
</style><table id="T_1d10c06a_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >LocationID</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>        <th class="col_heading level0 col3" >zone</th>        <th class="col_heading level0 col4" >borough</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >161</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >277369</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >4.4031%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow0_col3" class="data row0 col3" >Midtown Center</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow0_col4" class="data row0 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >237</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >276289</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >4.3860%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow1_col3" class="data row1 col3" >Upper East Side South</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow1_col4" class="data row1 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >236</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >258658</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >4.1061%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow2_col3" class="data row2 col3" >Upper East Side North</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow2_col4" class="data row2 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >162</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >232541</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >3.6915%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow3_col3" class="data row3 col3" >Midtown East</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow3_col4" class="data row3 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >230</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >226278</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >3.5921%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow4_col3" class="data row4 col3" >Times Sq/Theatre District</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow4_col4" class="data row4 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >186</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >225565</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >3.5808%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow5_col3" class="data row5 col3" >Penn Station/Madison Sq West</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow5_col4" class="data row5 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow6_col0" class="data row6 col0" >234</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow6_col1" class="data row6 col1" >193031</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow6_col2" class="data row6 col2" >3.0643%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow6_col3" class="data row6 col3" >Union Sq</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow6_col4" class="data row6 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow7_col0" class="data row7 col0" >170</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow7_col1" class="data row7 col1" >191817</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow7_col2" class="data row7 col2" >3.0450%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow7_col3" class="data row7 col3" >Murray Hill</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow7_col4" class="data row7 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow8_col0" class="data row8 col0" >48</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow8_col1" class="data row8 col1" >190547</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow8_col2" class="data row8 col2" >3.0249%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow8_col3" class="data row8 col3" >Clinton East</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow8_col4" class="data row8 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow9_col0" class="data row9 col0" >142</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow9_col1" class="data row9 col1" >188855</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow9_col2" class="data row9 col2" >2.9980%</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow9_col3" class="data row9 col3" >Lincoln Square East</td>
                        <td id="T_1d10c06a_127b_11eb_a8cb_874901be998arow9_col4" class="data row9 col4" >Manhattan</td>
            </tr>
    </tbody></table>


    
    Drop-off :



<style  type="text/css" >
</style><table id="T_1d11c9c4_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >LocationID</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>        <th class="col_heading level0 col3" >zone</th>        <th class="col_heading level0 col4" >borough</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >236</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >272802</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >4.3306%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow0_col3" class="data row0 col3" >Upper East Side North</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow0_col4" class="data row0 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >237</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >250885</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >3.9827%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow1_col3" class="data row1 col3" >Upper East Side South</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow1_col4" class="data row1 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >161</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >244454</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >3.8806%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow2_col3" class="data row2 col3" >Midtown Center</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow2_col4" class="data row2 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >170</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >194726</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >3.0912%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow3_col3" class="data row3 col3" >Murray Hill</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow3_col4" class="data row3 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >230</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >188214</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >2.9878%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow4_col3" class="data row4 col3" >Times Sq/Theatre District</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow4_col4" class="data row4 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >162</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >188016</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >2.9847%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow5_col3" class="data row5 col3" >Midtown East</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow5_col4" class="data row5 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow6_col0" class="data row6 col0" >142</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow6_col1" class="data row6 col1" >171781</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow6_col2" class="data row6 col2" >2.7270%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow6_col3" class="data row6 col3" >Lincoln Square East</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow6_col4" class="data row6 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow7_col0" class="data row7 col0" >234</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow7_col1" class="data row7 col1" >168084</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow7_col2" class="data row7 col2" >2.6683%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow7_col3" class="data row7 col3" >Union Sq</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow7_col4" class="data row7 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow8_col0" class="data row8 col0" >48</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow8_col1" class="data row8 col1" >168048</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow8_col2" class="data row8 col2" >2.6677%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow8_col3" class="data row8 col3" >Clinton East</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow8_col4" class="data row8 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow9_col0" class="data row9 col0" >239</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow9_col1" class="data row9 col1" >164119</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow9_col2" class="data row9 col2" >2.6053%</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow9_col3" class="data row9 col3" >Upper West Side South</td>
                        <td id="T_1d11c9c4_127b_11eb_a8cb_874901be998arow9_col4" class="data row9 col4" >Manhattan</td>
            </tr>
    </tbody></table>


- We can see that both of the most pick-up and drop-off zone is **East Harlem North**.
- However, the percentage of the top-10 zones are similar, we consider the most pick-up and drop-off borough next to see if there is any significant result.


```python
# pick-up and drop-off borough
df_PU_bor = borough_table(df_PU_zone)
df_DO_bor = borough_table(df_DO_zone)
print("\nPick-up :")
display_percent(df_PU_bor)
print("\nDrop-off :")
display_percent(df_DO_bor)
```

    
    Pick-up :



<style  type="text/css" >
</style><table id="T_1d153956_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >borough</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_1d153956_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >Manhattan</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >5797814</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >92.6372%</td>
            </tr>
            <tr>
                                <td id="T_1d153956_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >Queens</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >380675</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >6.0824%</td>
            </tr>
            <tr>
                                <td id="T_1d153956_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >Brooklyn</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >67971</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >1.0860%</td>
            </tr>
            <tr>
                                <td id="T_1d153956_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >Bronx</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >11283</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >0.1803%</td>
            </tr>
            <tr>
                                <td id="T_1d153956_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >EWR</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >621</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >0.0099%</td>
            </tr>
            <tr>
                                <td id="T_1d153956_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >Staten Island</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >262</td>
                        <td id="T_1d153956_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >0.0042%</td>
            </tr>
    </tbody></table>


    
    Drop-off :



<style  type="text/css" >
</style><table id="T_1d15d1f4_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >borough</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >Manhattan</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >5635947</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >90.0534%</td>
            </tr>
            <tr>
                                <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >Queens</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >317321</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >5.0703%</td>
            </tr>
            <tr>
                                <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >Brooklyn</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >251153</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >4.0130%</td>
            </tr>
            <tr>
                                <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >Bronx</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >41115</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >0.6570%</td>
            </tr>
            <tr>
                                <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >EWR</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >11461</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >0.1831%</td>
            </tr>
            <tr>
                                <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >Staten Island</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >1452</td>
                        <td id="T_1d15d1f4_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >0.0232%</td>
            </tr>
    </tbody></table>


- We can see that both of the most pick-up and drop-off borough is **Manhattan**.

## Q2: When are the peak hours and off-peak hours for taking taxi?
- We consider both pick-up and drop-off data.


```python
# draw graph
draw_hours(df_taxi, "Number of pick-up and drop-off passengers per hours")
```


    tpep_pickup_datetime
    0     172625
    1     119860
    2      87011
    3      60420
    4      45949
    5      52823
    6     120074
    7     230285
    8     296370
    9     295255
    10    289345
    11    303608
    12    328220
    13    334051
    14    353606
    15    362292
    16    346819
    17    397643
    18    438395
    19    396794
    20    350411
    21    347485
    22    322169
    23    247839
    Name: tpep_pickup_datetime, dtype: int64



    tpep_dropoff_datetime
    0     190086
    1     130200
    2      93406
    3      63441
    4      48507
    5      46865
    6     104867
    7     202559
    8     281071
    9     300016
    10    291413
    11    297905
    12    325803
    13    330441
    14    343241
    15    361495
    16    345751
    17    383579
    18    440175
    19    416998
    20    355380
    21    348737
    22    331576
    23    265837
    Name: tpep_dropoff_datetime, dtype: int64



![png](output_13_2.png)


<style>
    RR {
        color: "red";
    }
</style>

- By the graph above, both <font color="red">9 a.m. \~ 10 a.m. and 4 p.m. \~ 7 p.m.</font> are the <font color="red">peak</font> hours.
- The <font color="blue">off-peak</font> hours are <font color="blue">2 a.m. \~ 5 a.m.</font>


## Q3: What are the differences between short and long distance trips of taking taxi?
### Step:
    1. Define short and long distance trips
    2. Observe Spatial Difference
    3. Observe Temporal Difference

### 3-1 Define short and long distance trips
- Let's draw the graphs first and observe them.


```python
'''
3-1
'''
### type transform
df_taxi_dis = df_taxi.loc[:,['trip_distance']]
df_taxi_dis['trip_distance'] = pd.to_numeric(df_taxi_dis['trip_distance'], errors='coerce')
#print(df_taxi_dis.dtypes)
#print(df_taxi_dis.describe().loc[['max']])

### set the class interval of the graph
class_interval = 10
max_dis = df_taxi_dis.describe().loc[['max']].iloc[0]
bins_list = range(0, math.ceil(float(max_dis + class_interval)), class_interval)
#print(max_dis)
#print(*bins_list)

### show the graph
display(df_taxi_dis.describe())

#fig, ax = plt.subplots(1, 2)
fig, ax = plt.subplots(1, 2, figsize=(15, 7.5))
df_taxi_dis['trip_distance'].plot(ax=ax[0], kind='hist', bins=bins_list, label='Number of passengers')
ax[0].set_yscale('linear')
ax[0].set_xlabel("Distance (miles)")
ax[0].set_ylabel("Number of Passengers")
ax[0].legend(bbox_to_anchor=(0, 1.05), loc='lower left')
ax[0].axis('auto')
#ax[0].set_xlim(0, 1000)

df_taxi_dis['trip_distance'].plot(ax=ax[1], kind='hist', bins=bins_list, label='Number of passengers')
ax[1].set_yscale('log')
ax[1].set_xlabel("Distance (miles)")
ax[1].set_ylabel("Number of Passengers")
ax[1].legend(bbox_to_anchor=(0, 1.05), loc='lower left')
ax[1].axis('auto')
#ax[1].set_xlim(0, 1000)

fig.tight_layout()
plt.show()

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
      <th>trip_distance</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>count</td>
      <td>6.299349e+06</td>
    </tr>
    <tr>
      <td>mean</td>
      <td>2.830492e+00</td>
    </tr>
    <tr>
      <td>std</td>
      <td>3.690133e+00</td>
    </tr>
    <tr>
      <td>min</td>
      <td>0.000000e+00</td>
    </tr>
    <tr>
      <td>25%</td>
      <td>9.600000e-01</td>
    </tr>
    <tr>
      <td>50%</td>
      <td>1.600000e+00</td>
    </tr>
    <tr>
      <td>75%</td>
      <td>2.900000e+00</td>
    </tr>
    <tr>
      <td>max</td>
      <td>3.699400e+02</td>
    </tr>
  </tbody>
</table>
</div>



![png](output_17_1.png)


- By observing the graph above, we split short and long distance trips by <font color="red">**30(mils)**</font>.
- Then, we split the dataset next.


```python
split_gap = 30
df_taxi_short = df_taxi[(df_taxi.loc[:,['trip_distance']] < split_gap).all(1)]
df_taxi_long = df_taxi[(df_taxi.loc[:,['trip_distance']] >= split_gap).all(1)]

print("Number / percentage of short-distance trips :", df_taxi_short.shape[0], "/", round(100 * df_taxi_short.shape[0] / df_taxi.shape[0], 2), "%")
print("Number / percentage of long-distance trips :", df_taxi_long.shape[0], "/", round(100 * df_taxi_long.shape[0] / df_taxi.shape[0], 2), "%")

```

    Number / percentage of short-distance trips : 6296496 / 99.95 %
    Number / percentage of long-distance trips : 2853 / 0.05 %


### 3-2 Observe spatial difference

- zone and borough of the short-distance trips


```python
# pick-up and drop-off zone for short distance trips
df_PU_loc_S = df_taxi_short.loc[:,['PULocationID']].rename(columns={'PULocationID':'LocationID'})
df_DO_loc_S = df_taxi_short.loc[:,['DOLocationID']].rename(columns={'DOLocationID':'LocationID'})
loc_info = df_map[["LocationID", "zone", "borough"]]
df_PU_zone_S = zone_table(df_PU_loc_S, loc_info)
df_DO_zone_S = zone_table(df_DO_loc_S, loc_info)

# output
format_dict = {'percentage':'{0:,.4f}%'}
print("\nPick-up :")
display_percent(df_PU_zone_S.head(10))
print("\nDrop-off :")
display_percent(df_DO_zone_S.head(10))
```

    
    Pick-up :



<style  type="text/css" >
</style><table id="T_35031dbc_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >LocationID</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>        <th class="col_heading level0 col3" >zone</th>        <th class="col_heading level0 col4" >borough</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >161</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >277339</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >4.4047%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow0_col3" class="data row0 col3" >Midtown Center</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow0_col4" class="data row0 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >237</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >276265</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >4.3876%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow1_col3" class="data row1 col3" >Upper East Side South</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow1_col4" class="data row1 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >236</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >258654</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >4.1079%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow2_col3" class="data row2 col3" >Upper East Side North</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow2_col4" class="data row2 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >162</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >232526</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >3.6929%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow3_col3" class="data row3 col3" >Midtown East</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow3_col4" class="data row3 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >230</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >226237</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >3.5931%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow4_col3" class="data row4 col3" >Times Sq/Theatre District</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow4_col4" class="data row4 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >186</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >225508</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >3.5815%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow5_col3" class="data row5 col3" >Penn Station/Madison Sq West</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow5_col4" class="data row5 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow6_col0" class="data row6 col0" >234</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow6_col1" class="data row6 col1" >193020</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow6_col2" class="data row6 col2" >3.0655%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow6_col3" class="data row6 col3" >Union Sq</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow6_col4" class="data row6 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow7_col0" class="data row7 col0" >170</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow7_col1" class="data row7 col1" >191798</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow7_col2" class="data row7 col2" >3.0461%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow7_col3" class="data row7 col3" >Murray Hill</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow7_col4" class="data row7 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow8_col0" class="data row8 col0" >48</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow8_col1" class="data row8 col1" >190512</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow8_col2" class="data row8 col2" >3.0257%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow8_col3" class="data row8 col3" >Clinton East</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow8_col4" class="data row8 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow9_col0" class="data row9 col0" >142</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow9_col1" class="data row9 col1" >188853</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow9_col2" class="data row9 col2" >2.9993%</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow9_col3" class="data row9 col3" >Lincoln Square East</td>
                        <td id="T_35031dbc_127b_11eb_a8cb_874901be998arow9_col4" class="data row9 col4" >Manhattan</td>
            </tr>
    </tbody></table>


    
    Drop-off :



<style  type="text/css" >
</style><table id="T_3504248c_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >LocationID</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>        <th class="col_heading level0 col3" >zone</th>        <th class="col_heading level0 col4" >borough</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >236</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >272801</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >4.3326%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow0_col3" class="data row0 col3" >Upper East Side North</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow0_col4" class="data row0 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >237</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >250862</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >3.9842%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow1_col3" class="data row1 col3" >Upper East Side South</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow1_col4" class="data row1 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >161</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >244451</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >3.8823%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow2_col3" class="data row2 col3" >Midtown Center</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow2_col4" class="data row2 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >170</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >194723</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >3.0926%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow3_col3" class="data row3 col3" >Murray Hill</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow3_col4" class="data row3 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >230</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >188204</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >2.9890%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow4_col3" class="data row4 col3" >Times Sq/Theatre District</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow4_col4" class="data row4 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >162</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >188013</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >2.9860%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow5_col3" class="data row5 col3" >Midtown East</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow5_col4" class="data row5 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow6_col0" class="data row6 col0" >142</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow6_col1" class="data row6 col1" >171780</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow6_col2" class="data row6 col2" >2.7282%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow6_col3" class="data row6 col3" >Lincoln Square East</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow6_col4" class="data row6 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow7_col0" class="data row7 col0" >234</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow7_col1" class="data row7 col1" >168083</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow7_col2" class="data row7 col2" >2.6695%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow7_col3" class="data row7 col3" >Union Sq</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow7_col4" class="data row7 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow8_col0" class="data row8 col0" >48</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow8_col1" class="data row8 col1" >168040</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow8_col2" class="data row8 col2" >2.6688%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow8_col3" class="data row8 col3" >Clinton East</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow8_col4" class="data row8 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_3504248c_127b_11eb_a8cb_874901be998arow9_col0" class="data row9 col0" >239</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow9_col1" class="data row9 col1" >164117</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow9_col2" class="data row9 col2" >2.6065%</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow9_col3" class="data row9 col3" >Upper West Side South</td>
                        <td id="T_3504248c_127b_11eb_a8cb_874901be998arow9_col4" class="data row9 col4" >Manhattan</td>
            </tr>
    </tbody></table>



```python
# pick-up and drop-off borough for short distance trips
df_PU_bor_S = borough_table(df_PU_zone_S)
df_DO_bor_S = borough_table(df_DO_zone_S)
print("\nPick-up :")
display_percent(df_PU_bor_S)
print("\nDrop-off :")
display_percent(df_DO_bor_S)
```

    
    Pick-up :



<style  type="text/css" >
</style><table id="T_3507e694_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >borough</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_3507e694_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >Manhattan</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >5797065</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >92.6664%</td>
            </tr>
            <tr>
                                <td id="T_3507e694_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >Queens</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >378820</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >6.0555%</td>
            </tr>
            <tr>
                                <td id="T_3507e694_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >Brooklyn</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >67911</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >1.0856%</td>
            </tr>
            <tr>
                                <td id="T_3507e694_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >Bronx</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >11235</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >0.1796%</td>
            </tr>
            <tr>
                                <td id="T_3507e694_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >EWR</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >619</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >0.0099%</td>
            </tr>
            <tr>
                                <td id="T_3507e694_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >Staten Island</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >195</td>
                        <td id="T_3507e694_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >0.0031%</td>
            </tr>
    </tbody></table>


    
    Drop-off :



<style  type="text/css" >
</style><table id="T_3508dfd6_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >borough</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >Manhattan</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >5635712</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >90.0664%</td>
            </tr>
            <tr>
                                <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >Queens</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >316985</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >5.0659%</td>
            </tr>
            <tr>
                                <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >Brooklyn</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >251033</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >4.0119%</td>
            </tr>
            <tr>
                                <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >Bronx</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >41029</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >0.6557%</td>
            </tr>
            <tr>
                                <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >EWR</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >11181</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >0.1787%</td>
            </tr>
            <tr>
                                <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >Staten Island</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >1343</td>
                        <td id="T_3508dfd6_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >0.0215%</td>
            </tr>
    </tbody></table>


- zone and borough of the long-distance trips


```python
# pick-up and drop-off zone for long distance trips
df_PU_loc_L = df_taxi_long.loc[:,['PULocationID']].rename(columns={'PULocationID':'LocationID'})
df_DO_loc_L = df_taxi_long.loc[:,['DOLocationID']].rename(columns={'DOLocationID':'LocationID'})
loc_info = df_map[["LocationID", "zone", "borough"]]
df_PU_zone_L = zone_table(df_PU_loc_L, loc_info)
df_DO_zone_L = zone_table(df_DO_loc_L, loc_info)

# output
format_dict = {'percentage':'{0:,.4f}%'}
print("\nPick-up :")
display_percent(df_PU_zone_L.head(10))
print("\nDrop-off :")
display_percent(df_DO_zone_L.head(10))
```

    
    Pick-up :



<style  type="text/css" >
</style><table id="T_35102872_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >LocationID</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>        <th class="col_heading level0 col3" >zone</th>        <th class="col_heading level0 col4" >borough</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >132</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >1349</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >47.2836%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow0_col3" class="data row0 col3" >JFK Airport</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow0_col4" class="data row0 col4" >Queens</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >138</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >292</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >10.2348%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow1_col3" class="data row1 col3" >LaGuardia Airport</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow1_col4" class="data row1 col4" >Queens</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >186</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >57</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >1.9979%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow2_col3" class="data row2 col3" >Penn Station/Madison Sq West</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow2_col4" class="data row2 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >158</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >53</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >1.8577%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow3_col3" class="data row3 col3" >Meatpacking/West Village West</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow3_col4" class="data row3 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >265</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >48</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >1.6824%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow4_col3" class="data row4 col3" >nan</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow4_col4" class="data row4 col4" >nan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >68</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >45</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >1.5773%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow5_col3" class="data row5 col3" >East Chelsea</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow5_col4" class="data row5 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow6_col0" class="data row6 col0" >230</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow6_col1" class="data row6 col1" >41</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow6_col2" class="data row6 col2" >1.4371%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow6_col3" class="data row6 col3" >Times Sq/Theatre District</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow6_col4" class="data row6 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow7_col0" class="data row7 col0" >48</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow7_col1" class="data row7 col1" >35</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow7_col2" class="data row7 col2" >1.2268%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow7_col3" class="data row7 col3" >Clinton East</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow7_col4" class="data row7 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow8_col0" class="data row8 col0" >246</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow8_col1" class="data row8 col1" >32</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow8_col2" class="data row8 col2" >1.1216%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow8_col3" class="data row8 col3" >West Chelsea/Hudson Yards</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow8_col4" class="data row8 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35102872_127b_11eb_a8cb_874901be998arow9_col0" class="data row9 col0" >100</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow9_col1" class="data row9 col1" >32</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow9_col2" class="data row9 col2" >1.1216%</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow9_col3" class="data row9 col3" >Garment District</td>
                        <td id="T_35102872_127b_11eb_a8cb_874901be998arow9_col4" class="data row9 col4" >Manhattan</td>
            </tr>
    </tbody></table>


    
    Drop-off :



<style  type="text/css" >
</style><table id="T_35114be4_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >LocationID</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>        <th class="col_heading level0 col3" >zone</th>        <th class="col_heading level0 col4" >borough</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >265</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >1669</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >58.4998%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow0_col3" class="data row0 col3" >nan</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow0_col4" class="data row0 col4" >nan</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >1</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >280</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >9.8142%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow1_col3" class="data row1 col3" >Newark Airport</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow1_col4" class="data row1 col4" >EWR</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >132</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >163</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >5.7133%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow2_col3" class="data row2 col3" >JFK Airport</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow2_col4" class="data row2 col4" >Queens</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >44</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >36</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >1.2618%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow3_col3" class="data row3 col3" >Charleston/Tottenville</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow3_col4" class="data row3 col4" >Staten Island</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >237</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >23</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >0.8062%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow4_col3" class="data row4 col3" >Upper East Side South</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow4_col4" class="data row4 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >135</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >22</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >0.7711%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow5_col3" class="data row5 col3" >Kew Gardens Hills</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow5_col4" class="data row5 col4" >Queens</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow6_col0" class="data row6 col0" >117</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow6_col1" class="data row6 col1" >19</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow6_col2" class="data row6 col2" >0.6660%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow6_col3" class="data row6 col3" >Hammels/Arverne</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow6_col4" class="data row6 col4" >Queens</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow7_col0" class="data row7 col0" >158</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow7_col1" class="data row7 col1" >19</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow7_col2" class="data row7 col2" >0.6660%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow7_col3" class="data row7 col3" >Meatpacking/West Village West</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow7_col4" class="data row7 col4" >Manhattan</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow8_col0" class="data row8 col0" >264</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow8_col1" class="data row8 col1" >18</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow8_col2" class="data row8 col2" >0.6309%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow8_col3" class="data row8 col3" >nan</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow8_col4" class="data row8 col4" >nan</td>
            </tr>
            <tr>
                                <td id="T_35114be4_127b_11eb_a8cb_874901be998arow9_col0" class="data row9 col0" >200</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow9_col1" class="data row9 col1" >17</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow9_col2" class="data row9 col2" >0.5959%</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow9_col3" class="data row9 col3" >Riverdale/North Riverdale/Fieldston</td>
                        <td id="T_35114be4_127b_11eb_a8cb_874901be998arow9_col4" class="data row9 col4" >Bronx</td>
            </tr>
    </tbody></table>



```python
# pick-up and drop-off borough for long distance trips
df_PU_bor_L = borough_table(df_PU_zone_L)
df_DO_bor_L = borough_table(df_DO_zone_L)
print("\nPick-up :")
display_percent(df_PU_bor_L)
print("\nDrop-off :")
display_percent(df_DO_bor_L)
```

    
    Pick-up :



<style  type="text/css" >
</style><table id="T_3515b7a6_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >borough</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >Queens</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >1855</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >66.7026%</td>
            </tr>
            <tr>
                                <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >Manhattan</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >749</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >26.9328%</td>
            </tr>
            <tr>
                                <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >Staten Island</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >67</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >2.4092%</td>
            </tr>
            <tr>
                                <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >Brooklyn</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >60</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >2.1575%</td>
            </tr>
            <tr>
                                <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >Bronx</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >48</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >1.7260%</td>
            </tr>
            <tr>
                                <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >EWR</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >2</td>
                        <td id="T_3515b7a6_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >0.0719%</td>
            </tr>
    </tbody></table>


    
    Drop-off :



<style  type="text/css" >
</style><table id="T_35167876_127b_11eb_a8cb_874901be998a" ><thead>    <tr>        <th class="col_heading level0 col0" >borough</th>        <th class="col_heading level0 col1" >count</th>        <th class="col_heading level0 col2" >percentage</th>    </tr></thead><tbody>
                <tr>
                                <td id="T_35167876_127b_11eb_a8cb_874901be998arow0_col0" class="data row0 col0" >Queens</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow0_col1" class="data row0 col1" >336</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow0_col2" class="data row0 col2" >28.8165%</td>
            </tr>
            <tr>
                                <td id="T_35167876_127b_11eb_a8cb_874901be998arow1_col0" class="data row1 col0" >EWR</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow1_col1" class="data row1 col1" >280</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow1_col2" class="data row1 col2" >24.0137%</td>
            </tr>
            <tr>
                                <td id="T_35167876_127b_11eb_a8cb_874901be998arow2_col0" class="data row2 col0" >Manhattan</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow2_col1" class="data row2 col1" >235</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow2_col2" class="data row2 col2" >20.1544%</td>
            </tr>
            <tr>
                                <td id="T_35167876_127b_11eb_a8cb_874901be998arow3_col0" class="data row3 col0" >Brooklyn</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow3_col1" class="data row3 col1" >120</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow3_col2" class="data row3 col2" >10.2916%</td>
            </tr>
            <tr>
                                <td id="T_35167876_127b_11eb_a8cb_874901be998arow4_col0" class="data row4 col0" >Staten Island</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow4_col1" class="data row4 col1" >109</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow4_col2" class="data row4 col2" >9.3482%</td>
            </tr>
            <tr>
                                <td id="T_35167876_127b_11eb_a8cb_874901be998arow5_col0" class="data row5 col0" >Bronx</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow5_col1" class="data row5 col1" >86</td>
                        <td id="T_35167876_127b_11eb_a8cb_874901be998arow5_col2" class="data row5 col2" >7.3756%</td>
            </tr>
    </tbody></table>


### 3-3 Observe temporal difference



```python
draw_hours(df_taxi_short, "Short-distance Trips")
draw_hours(df_taxi_long, "Long-distance Trips")
```


![png](output_28_0.png)



![png](output_28_1.png)


- By the graphs above, we can see that tendency of pick-up and drop-off time are similar in both two graph.
- For **long distance trips**, the tendency of drop-off time is **1-hour later** than the pick-up time.
- Both peak and off-peak hours of the long-distance trips are about 1 \~ 2 hours earlier than the short-distance trips.

- The peak hours and the off-peak hours for short and long trips:

|                    | Peak Hours | Off-peak Hours |
| ------------------ | -------------- | ------------- |
| **Short-distance** | 9 a.m. \~ 10 a.m.<br>4 p.m. \~ 7 p.m. | 1 a.m. \~ 5 a.m. |
| **Long-distance**  | 11 a.m. \~ 12 a.m.<br>3 p.m. \~ 5 p.m. | 0 a.m. \~ 4 a.m. |


```python
def draw_map(shpFilePath, ax):
    sf = shapefile.Reader(shpFilePath)
    # plt.figure(figsize=(7.5, 7.5))
    for shape in sf.shapeRecords():
        x = [i[0] for i in shape.shape.points[:]]
        y = [i[1] for i in shape.shape.points[:]]
        ax.plot(x, y)
    #ax.axis('equal')
    #ax.axis('square')
    #plt.show()

fig, ax = plt.subplots(nrows=1, ncols=2, figsize=(15, 7.5))
ax = plt.subplot(1, 2, 1)
draw_map("shape/taxi_zones.shp", ax)
ax = plt.subplot(1, 2, 2)
draw_map("shape2/taxi_zones.shp", ax)
```


![png](output_30_0.png)

