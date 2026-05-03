## Cambridge Cherry Blossom Bike Map

![Cherry Blossoms](snapshot-map.jpg) 

This project builds on the assignment I completed as part of the Python course taught by the folks at Spatial Thoughts where we learned how to access data via APIs and analyze and visualise it using pandas. 

Check out their online resources and upcoming courses: https://spatialthoughts.com/

#### Data sources:¶
- City of Cambridge Street Trees: https://data.cambridgema.gov/Public-Works/Street-Trees/82zb-7qc9/about_data
- City of Cambrdige bike facilities: https://data.cambridgema.gov/Public-Works/Bike-Facilities/9aey-9g9p/about_data

#### Tools: 
- pandas - data cleaning, filtering and analysis
- folium - creating interactive web map
- requests - querying data from a public API
- Jupyter lab - development environment
- Github Pages - deployment

### Getting tree inventory data via the City of Cambridge's public API

The size of the dataset is over 44,000 rows and the API limit is currently 1,000 rows.

To get around the 1,000 row limit, I'm following the steps outlined in their "Learn more" section to get around the default API limit: https://support.socrata.com/hc/en-us/articles/202949268-How-to-query-more-than-1000-rows-of-a-dataset


```python
import requests
import pandas as pd
```


```python
url = 'https://data.cambridgema.gov/resource/82zb-7qc9.json' 

params = {"$limit": 50000}

response = requests.get(url, params=params)
response.raise_for_status()

df = pd.DataFrame(response.json())
print(f"Total rows: {len(df)}")
```

As of March 2026, the dataset said 42,711 rows and our dataframe returns 42,711 rows!

#### Exploring the dataset

To help determine what data columns could be useful to keep, I looked through the full metadata: https://www.cambridgema.gov/GIS/gisdatadictionary/Environmental/ENVIRONMENTAL_StreetTrees


```python
#I get the list of columns so I can make sure to spell each correctly
df.columns
```




    Index(['the_geom', 'treeid', 'treewellid', 'trunks', 'sitetype', 'species',
           'speciessho', 'overheadwi', 'streetnumb', 'streetname', 'memtree',
           'location', 'ownership', 'treegratea', 'diameter', 'abutsopena',
           'exposedroo', 'commonname', 'genus', 'scientific', 'creator',
           'inspectr', 'order_', 'locationre', 'sitereplan', 'solarratin',
           'treewellle', 'treewellwi', 'treewellde', 'scheduledr', 'sttreeprun',
           ':@computed_region_guic_hr4a', ':@computed_region_v7jj_366k',
           ':@computed_region_rffn_qbt6', ':@computed_region_swkg_bavi',
           ':@computed_region_e4yd_rwk4', 'removaldat', 'cartegraph', 'siteretire',
           'notes', 'offsttreep', 'plantdate', 'cartegra_1', 'wateringre',
           'cultivar', 'treewellco', 'adacomplia', 'bareroot', 'structural',
           'plantingco', 'biochar_ad', 'plantingse', 'pb', 'removalrea'],
          dtype='object')



#### Filtering for the columns we are interested in keeping using pandas
Based on the metadata, these are the columns I'm choosing to keep:
- the_geom
- sitetype
- species
- location
- commonname
- cultivar
- speciessho
- removaldat
- plantdate


```python
df_filtered = df[["the_geom", "sitetype", "species", "location", "commonname", "cultivar",
                 "speciessho", "removaldat", "plantdate"]]
df_filtered.head()
```

#### Let's look through the "sitetype" variables to see which rows to remove


```python
df_filtered["sitetype"].unique()
```




    array(['Retired', 'Stump', 'Tree', 'Not Installed Issue',
           'Refused Not Planted', 'Proposed Tree', nan, 'Planting Site',
           'Spar', 'Unknown', 'Damaged Tree Well'], dtype=object)



Let's keep only the rows with "Tree"


```python
df_filtered = df_filtered[df_filtered["sitetype"] == "Tree"]
```


```python
#Check to make sure only rows with "Tree" appear

df_filtered["sitetype"].unique()
```




    array(['Tree'], dtype=object)



Next we explore the "commonname" column so we can then filter for the species we are interested in.


```python
print(df_filtered["commonname"].value_counts().to_string())
```

    commonname
    Japanese cherry     356
    Cherry genus        320
    Sargent's cherry    187
    Autumn cherry       101
    Yoshino cherry       87
    Okame cherry         50
    Okame Cherry         47
    Cherry Genus         19
    Cherry plum          13
    

Based on the above results, let's filter the "commonname" column for instances of "Cherry" and "cherry"


```python
df_filtered = df_filtered[df_filtered["commonname"].str.contains("cherry", case=False, na=False)]

```

#### See what kind of cherry trees we have
We can remove the cherry trees that don't have the pink petals we're interested in:
- Pin cherry
- Black cherry tree
- Cornelian cherry
- Choke cherry


```python

df_filtered["commonname"].value_counts()
```




    commonname
    Japanese cherry     356
    Cherry genus        320
    Sargent's cherry    187
    Autumn cherry       101
    Yoshino cherry       87
    Okame cherry         50
    Okame Cherry         47
    Cherry Genus         19
    Cherry plum          13
    Name: count, dtype: int64



This part took some "manual" work. Since I don't know all of these species, I googled many of these to verify they were in line with what I was looking for, pink cherry flower trees. 



```python
df_filtered = df_filtered[~df_filtered["commonname"].str.contains("Pin cherry|Black cherry|Cornelian cherry|Chokecherry", case=False, na=False)]

```


```python
#Let's check to make sure "Pin cherry", "Black cherry", "Cornelian cherry", and "Chokecherry" were removed
df_filtered["commonname"].value_counts()
```




    commonname
    Japanese cherry     356
    Cherry genus        320
    Sargent's cherry    187
    Autumn cherry       101
    Yoshino cherry       87
    Okame cherry         50
    Okame Cherry         47
    Cherry Genus         19
    Cherry plum          13
    Name: count, dtype: int64




```python
df_filtered.head()
```

Since we're interested in trees we can easily see while biking, let's filter "location" so only Street Trees appear


```python
df_filtered = df_filtered[df_filtered["location"] == "Street Tree"]
```

#### Getting the coordinates
The coordinates are stored in the "the_geom" column in a dictionary. We need to separate them into usable lat lon values. 


```python
#The coordinates are stored in "the_geom" column, but are not in a format we can readily use
print(df_filtered["the_geom"].iloc[0])
```

    {'type': 'Point', 'coordinates': [-71.11136505349704, 42.367429286196874]}
    


```python
#seperating our lat lons and storing them in new latitude and longitude columns we created
df_filtered["longitude"] = df_filtered["the_geom"].str["coordinates"].str[0]
df_filtered["latitude"] = df_filtered["the_geom"].str["coordinates"].str[1]
```


```python
df_filtered.head()
```


```python
#Clean up the dataframe and keep only the columns we are intested in
df_filtered = df_filtered[["species", "commonname", "speciessho", "plantdate", "longitude", "latitude"]]
```


```python
df_filtered.head()
```

#### Using folium to map the tree locations


```python
import folium
```


```python
#centering our map in Cambridge, MA
m_c = folium.Map(location=[42.3736, -71.1097], zoom_start=12.5, tiles="CartoDB positron")
```


```python
for i, row in df_filtered.iterrows():
    folium.Marker(
        location=[row["latitude"], row["longitude"]],
        icon=folium.DivIcon(html="""
    <img src="cherry.png" style="
        width:18px;
        height:18px;
        opacity:0.5;
        display:block;
    ">
"""),
        popup=f"{row['commonname']}"
    ).add_to(m_c)

```

### Using the City of Cambridge's public API to access bike lane data

Bike lane data is available on the City of Cambridge Open Data website: 
https://data.cambridgema.gov/Geographic-Information-GIS-/Bike-Facilities-Map/rpti-wwqr 

We again can get the data using the city's API. Since the dataset contains less than 1,000 rows, we don't need to go through the additional steps we took for the tree data. 


```python
url_bikes = "https://data.cambridgema.gov/resource/9aey-9g9p.json"

response_bikes = requests.get(url_bikes, params=params)
response_bikes.raise_for_status()

df_bikes = pd.DataFrame(response_bikes.json())
print(f"Total rows: {len(df)}")
print(df_bikes.head())
```

    Total rows: 42711
                                                the_geom                street  \
    0  {'type': 'MultiLineString', 'coordinates': [[[...            Waverly St   
    1  {'type': 'MultiLineString', 'coordinates': [[[...           Danehy Park   
    2  {'type': 'MultiLineString', 'coordinates': [[[...  Monsignor OBrien Hwy   
    3  {'type': 'MultiLineString', 'coordinates': [[[...             Dudley St   
    4  {'type': 'MultiLineString', 'coordinates': [[[...            Peabody St   
    
      bike_fac pbike_fac                  facilitytype  \
    0    100.0       0.0                     Bike Lane   
    1    500.0       0.0      Bike Path/Multi-Use Path   
    2    400.0       0.0     Grade-Separated Bike Lane   
    3    620.0       0.0  Shared Lane Pavement Marking   
    4    820.0       0.0           Separated Bike Lane   
    
                                         facilitydetails lanemultiplier  \
    0        Bike Lane on both sides of a two-way street            2.0   
    1                           Bike Path/Multi-Use Path            2.0   
    2  Grade-Separated Bike Lane on both sides of a t...            2.0   
    3   Shared Lane Pavement Marking on a one-way street            1.0   
    4            Separated Bike Lane on a one-way street            1.0   
    
               last_edited created_date  
    0  2026-02-23T00:00:00          NaN  
    1  2026-02-23T00:00:00          NaN  
    2  2026-02-23T00:00:00          NaN  
    3  2026-02-23T00:00:00          NaN  
    4  2026-02-23T00:00:00          NaN  
    


```python
#Seeing the columns we have
df_bikes.columns
```




    Index(['the_geom', 'street', 'bike_fac', 'pbike_fac', 'facilitytype',
           'facilitydetails', 'lanemultiplier', 'last_edited', 'created_date'],
          dtype='object')




```python
#Seeing the kinds of bike facilities we have
df_bikes["facilitytype"].value_counts()
```




    facilitytype
    Separated Bike Lane                     198
    Bike Lane                               161
    Bike Path/Multi-Use Path                111
    Shared Lane Pavement Marking             91
    Grade-Separated Bike Lane                52
    Buffered Bike Lane                       29
    Shared Street                            12
    Contra-flow                               8
    Separated Bike Lane with Contra-flow      5
    Bus/Bike Lane                             4
    Name: count, dtype: int64



##### Let's remove the following facilities:
- Shared Lane Pavement Marking
- Shared Street



```python
df_bikes = df_bikes[~df_bikes["facilitytype"].str.contains("Shared Lane Pavement Marking|Shared Street", case=False, na=False)]

```


```python
df_bikes["facilitytype"].value_counts()
```




    facilitytype
    Separated Bike Lane                     198
    Bike Lane                               161
    Bike Path/Multi-Use Path                111
    Grade-Separated Bike Lane                52
    Buffered Bike Lane                       29
    Contra-flow                               8
    Separated Bike Lane with Contra-flow      5
    Bus/Bike Lane                             4
    Name: count, dtype: int64



#### Adding the bike facilities to our cherry tree map 



```python

for _, row in df_bikes.iterrows():
    geom = row["the_geom"]
    for line in geom["coordinates"]:
        # flip [lon, lat] to [lat, lon] for folium
        points = [[coord[1], coord[0]] for coord in line]
        folium.PolyLine(
            locations=points,
            color="darkgreen",
            weight=.2,
            opacity=1.5,
            popup=f"{row['street']}"
        ).add_to(m_c)


```

#### Adding a title to the map 


```python


title_html = '''
<div style="
    position: fixed; 
    top: 10px; left: 50%; transform: translateX(-50%);
    z-index:9999;
    background-color: white;
    padding: 10px 20px;
    border-radius: 8px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
    font-size:18px;
    font-weight: bold;
">
🌸 Cambridge Cherry Blossom Bike Map 🚲
</div>
'''
m_c.get_root().html.add_child(folium.Element(title_html))
```




    <branca.element.Element at 0x26064bd9090>



#### Adding a legend


```python
legend_html = '''
<div style="
    position: fixed; 
    bottom: 30px; left: 30px; width: 200px;
    z-index:9999; 
    background-color: white;
    padding: 10px;
    border-radius: 8px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
    font-size:14px;
">
<b>Legend</b><br><br>

<span style="font-size:16px;">🌸</span> Cherry Trees<br><br>

<div style="
    width: 30px;
    height: 4px;
    background-color: green;
    display: inline-block;
    vertical-align: middle;
    margin-right: 6px;
"></div>
Bike Lanes

</div>
'''
m_c.get_root().html.add_child(folium.Element(legend_html))
```




    <branca.element.Element at 0x26064bd9090>



#### Save the map as an html file


```python
m_c.save("cambridge_cherry_bike_map6.html")
```

### All done! 

#### Next steps
Because the tree dataset is so comprehensive, I'd like to keep exploring it and work toward the following future iterations:

- Overlay neighborhood boundaries and calculate cherry tree density based on the area of each neighborhood. 

- Expand the number of tree species to include 1-5 species so that people can choose from a dropdown menu the kinds of trees they are interested in seeing. The locations of the selected trees will populate on the map and the user can curate their bike route based on the tree locations.

- Allow users to enter two addresses and be able to see the number and species of the trees along their route

#### Hope you enjoy the pink blooms this year or next!



```python
#Ignore this cell!
!jupyter nbconvert --to markdown Cherry-Trees-Biking-Copy2.ipynb
```

    [NbConvertApp] Converting notebook Cherry-Trees-Biking-Copy2.ipynb to markdown
    [NbConvertApp] Writing 25650 bytes to Cherry-Trees-Biking-Copy2.md
    


```python

```
