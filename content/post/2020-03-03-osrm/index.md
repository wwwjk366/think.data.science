---
title: Python Tutorial of OSRM(Open Sourced Routing Machine) and Applications
author: ''
date: '2020-03-03'
slug: osrm
categories:
  - Python
  - Tutorial
  - Visualization
tags:
  - Map
  - Routing
  - Package
subtitle: ''
summary: ''
authors: []
lastmod: '2020-03-03T22:04:53Z'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---



I came across a wonderful open source project recently --- Project OSRM ([link](http://project-osrm.org/)) --- A modern C++ routing engine for shortest paths in road networks. You can imagine it as a free version of Google Maps API, without live traffic of course. It is very valuable for my work because my current company has large shipping and logistic services. Being able to calculate the distance and directions between locations in a timely fashion will enable us to research and modeling on route optimization, leads generation, etc.

The solution itself is quite straightforward and I am able to setup an API sandbox running in a couple of hours.

First you need to get the OSRM back-end running as a container on your machine. The process is very easy to follow on the project's github page [here](https://github.com/Project-OSRM/osrm-backend). After that you can easily interact with it in python, let's take a look:

## Package Needed

We need `folium` package to draw the routes on the map and `polyline` to decode the routes from the API output. We will talk about that more later.


```python
import requests
import folium
import polyline
```

## Single Request

You can request the driving route by supply the latitude and longitude of your start and end points, separate by  , and ;


```python
url = "http://10.22.168.65:9080/route/v1/driving/-117.851364,33.698206;-117.838925,33.672260"
r = requests.get(url)
res = r.json()
res
```




    {'code': 'Ok',
     'routes': [{'geometry': 'gttlEfyhnUpBtC|k@e`Aro@zo@tf@jXhFaMxSe]lDgKlAqIGwTcIyB',
       'legs': [{'steps': [],
         'distance': 4995.3,
         'duration': 409.1,
         'summary': '',
         'weight': 422.5}],
       'distance': 4995.3,
       'duration': 409.1,
       'weight_name': 'routability',
       'weight': 422.5}],
     'waypoints': [{'hint': 'O14sgD1eLIAMAQAASQEAAAAAAAAAAAAAWs1fQprEiEIAAAAAAAAAAIYAAAClAAAAAAAAAAAAAACQBgAAnbv5-EQxAgIcu_n4njECAgAAfwLrq6bJ',
       'distance': 15.580755,
       'name': '',
       'location': [-117.851235, 33.698116]},
      {'hint': 'S64XgFKuF4BLAAAAKgAAAAAAAAA_AAAA2iJQQqE95kEAAAAAb1YuQksAAAAqAAAAAAAAAD8AAACQBgAA5-z5-MXLAQKz6_n4RMwBAgAArwHrq6bJ',
       'distance': 31.847501,
       'name': 'Carlson Avenue',
       'location': [-117.838617, 33.672133]}]}



The output is easy to follow. This trip has a distance of 4995 meters and travel time of 409 seconds, with the routes encoded using google's [Polyline Algorithm](https://developers.google.com/maps/documentation/utilities/polylinealgorithm). We can use python package `polyline` to decode it into coordinates:


```python
polyline.decode('gttlEfyhnUpBtC|k@e`Aro@zo@tf@jXhFaMxSe]lDgKlAqIGwTcIyB')
```




    [(33.69812, -117.85124),
     (33.69755, -117.85199),
     (33.69036, -117.84156),
     (33.68258, -117.84938),
     (33.67623, -117.85344),
     (33.67506, -117.85119),
     (33.67173, -117.84636),
     (33.67086, -117.8444),
     (33.67047, -117.84271),
     (33.67051, -117.83923),
     (33.67213, -117.83862)]



Now this is something we can work with! Lets wrap it into a function:


```python
def get_route(pickup_lon, pickup_lat, dropoff_lon, dropoff_lat):
    
    loc = "{},{};{},{}".format(pickup_lon, pickup_lat, dropoff_lon, dropoff_lat)
    url = "http://10.22.168.65:9080/route/v1/driving/"
    r = requests.get(url + loc) 
    if r.status_code!= 200:
        return {}
  
    res = r.json()   
    routes = polyline.decode(res['routes'][0]['geometry'])
    start_point = [res['waypoints'][0]['location'][1], res['waypoints'][0]['location'][0]]
    end_point = [res['waypoints'][1]['location'][1], res['waypoints'][1]['location'][0]]
    distance = res['routes'][0]['distance']
    
    out = {'route':routes,
           'start_point':start_point,
           'end_point':end_point,
           'distance':distance
          }

    return out
```


```python
pickup_lon, pickup_lat, dropoff_lon, dropoff_lat = -117.851364,33.698206,-117.838925,33.672260
test_route = get_route(pickup_lon, pickup_lat, dropoff_lon, dropoff_lat)
test_route
```




    {'route': [(33.69812, -117.85124),
      (33.69755, -117.85199),
      (33.69036, -117.84156),
      (33.68258, -117.84938),
      (33.67623, -117.85344),
      (33.67506, -117.85119),
      (33.67173, -117.84636),
      (33.67086, -117.8444),
      (33.67047, -117.84271),
      (33.67051, -117.83923),
      (33.67213, -117.83862)],
     'start_point': [33.698116, -117.851235],
     'end_point': [33.672133, -117.838617],
     'distance': 4995.3}



## Draw the route on map 

Now we have the output nicely organized in coordinates format, let's use `folium` package to chart the routes and see if it makes sense or not.


```python
def get_map(route):
    
    m = folium.Map(location=[(route['start_point'][0] + route['end_point'][0])/2, 
                             (route['start_point'][1] + route['end_point'][1])/2], 
                   zoom_start=13)

    folium.PolyLine(
        route['route'],
        weight=8,
        color='blue',
        opacity=0.6
    ).add_to(m)

    folium.Marker(
        location=route['start_point'],
        icon=folium.Icon(icon='play', color='green')
    ).add_to(m)

    folium.Marker(
        location=route['end_point'],
        icon=folium.Icon(icon='stop', color='red')
    ).add_to(m)

    return m
```


```python
get_map(test_route)
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVM9ZmFsc2U7IExfTk9fVE9VQ0g9ZmFsc2U7IExfRElTQUJMRV8zRD1mYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS40LjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NvZGUuanF1ZXJ5LmNvbS9qcXVlcnktMS4xMi40Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS40LjAvZGlzdC9sZWFmbGV0LmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLm1pbi5jc3MiLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC10aGVtZS5taW4uY3NzIi8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9MZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy8yLjAuMi9sZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy5jc3MiLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9yYXdjZG4uZ2l0aGFjay5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIi8+CiAgICA8c3R5bGU+aHRtbCwgYm9keSB7d2lkdGg6IDEwMCU7aGVpZ2h0OiAxMDAlO21hcmdpbjogMDtwYWRkaW5nOiAwO308L3N0eWxlPgogICAgPHN0eWxlPiNtYXAge3Bvc2l0aW9uOmFic29sdXRlO3RvcDowO2JvdHRvbTowO3JpZ2h0OjA7bGVmdDowO308L3N0eWxlPgogICAgCiAgICA8bWV0YSBuYW1lPSJ2aWV3cG9ydCIgY29udGVudD0id2lkdGg9ZGV2aWNlLXdpZHRoLAogICAgICAgIGluaXRpYWwtc2NhbGU9MS4wLCBtYXhpbXVtLXNjYWxlPTEuMCwgdXNlci1zY2FsYWJsZT1ubyIgLz4KICAgIDxzdHlsZT4jbWFwXzU3ZmY2YzVhOTk1YTQ3Nzc4ODFmNmNiNTc4NGJjY2VhIHsKICAgICAgICBwb3NpdGlvbjogcmVsYXRpdmU7CiAgICAgICAgd2lkdGg6IDEwMC4wJTsKICAgICAgICBoZWlnaHQ6IDEwMC4wJTsKICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgIHRvcDogMC4wJTsKICAgICAgICB9CiAgICA8L3N0eWxlPgo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgPGRpdiBjbGFzcz0iZm9saXVtLW1hcCIgaWQ9Im1hcF81N2ZmNmM1YTk5NWE0Nzc3ODgxZjZjYjU3ODRiY2NlYSIgPjwvZGl2Pgo8L2JvZHk+CjxzY3JpcHQ+ICAgIAogICAgCiAgICAKICAgICAgICB2YXIgYm91bmRzID0gbnVsbDsKICAgIAoKICAgIHZhciBtYXBfNTdmZjZjNWE5OTVhNDc3Nzg4MWY2Y2I1Nzg0YmNjZWEgPSBMLm1hcCgKICAgICAgICAnbWFwXzU3ZmY2YzVhOTk1YTQ3Nzc4ODFmNmNiNTc4NGJjY2VhJywgewogICAgICAgIGNlbnRlcjogWzMzLjY4NTEyNDUsIC0xMTcuODQ0OTI2XSwKICAgICAgICB6b29tOiAxMywKICAgICAgICBtYXhCb3VuZHM6IGJvdW5kcywKICAgICAgICBsYXllcnM6IFtdLAogICAgICAgIHdvcmxkQ29weUp1bXA6IGZhbHNlLAogICAgICAgIGNyczogTC5DUlMuRVBTRzM4NTcsCiAgICAgICAgem9vbUNvbnRyb2w6IHRydWUsCiAgICAgICAgfSk7CgoKICAgIAogICAgdmFyIHRpbGVfbGF5ZXJfYzdlM2Q4MjZkZDYzNDdjOGEyZjllYThmOWFlNjE2ZWEgPSBMLnRpbGVMYXllcigKICAgICAgICAnaHR0cHM6Ly97c30udGlsZS5vcGVuc3RyZWV0bWFwLm9yZy97en0ve3h9L3t5fS5wbmcnLAogICAgICAgIHsKICAgICAgICAiYXR0cmlidXRpb24iOiBudWxsLAogICAgICAgICJkZXRlY3RSZXRpbmEiOiBmYWxzZSwKICAgICAgICAibWF4TmF0aXZlWm9vbSI6IDE4LAogICAgICAgICJtYXhab29tIjogMTgsCiAgICAgICAgIm1pblpvb20iOiAwLAogICAgICAgICJub1dyYXAiOiBmYWxzZSwKICAgICAgICAib3BhY2l0eSI6IDEsCiAgICAgICAgInN1YmRvbWFpbnMiOiAiYWJjIiwKICAgICAgICAidG1zIjogZmFsc2UKfSkuYWRkVG8obWFwXzU3ZmY2YzVhOTk1YTQ3Nzc4ODFmNmNiNTc4NGJjY2VhKTsKICAgIAogICAgICAgICAgICAgICAgdmFyIHBvbHlfbGluZV9mNzM5OWU1MTdjOGE0MTMyOTNmMTBkMTc3Mzg1YzYxYyA9IEwucG9seWxpbmUoCiAgICAgICAgICAgICAgICAgICAgW1szMy42OTgxMiwgLTExNy44NTEyNF0sIFszMy42OTc1NSwgLTExNy44NTE5OV0sIFszMy42OTAzNiwgLTExNy44NDE1Nl0sIFszMy42ODI1OCwgLTExNy44NDkzOF0sIFszMy42NzYyMywgLTExNy44NTM0NF0sIFszMy42NzUwNiwgLTExNy44NTExOV0sIFszMy42NzE3MywgLTExNy44NDYzNl0sIFszMy42NzA4NiwgLTExNy44NDQ0XSwgWzMzLjY3MDQ3LCAtMTE3Ljg0MjcxXSwgWzMzLjY3MDUxLCAtMTE3LjgzOTIzXSwgWzMzLjY3MjEzLCAtMTE3LjgzODYyXV0sCiAgICAgICAgICAgICAgICAgICAgewogICJidWJibGluZ01vdXNlRXZlbnRzIjogdHJ1ZSwKICAiY29sb3IiOiAiYmx1ZSIsCiAgImRhc2hBcnJheSI6IG51bGwsCiAgImRhc2hPZmZzZXQiOiBudWxsLAogICJmaWxsIjogZmFsc2UsCiAgImZpbGxDb2xvciI6ICJibHVlIiwKICAiZmlsbE9wYWNpdHkiOiAwLjIsCiAgImZpbGxSdWxlIjogImV2ZW5vZGQiLAogICJsaW5lQ2FwIjogInJvdW5kIiwKICAibGluZUpvaW4iOiAicm91bmQiLAogICJub0NsaXAiOiBmYWxzZSwKICAib3BhY2l0eSI6IDAuNiwKICAic21vb3RoRmFjdG9yIjogMS4wLAogICJzdHJva2UiOiB0cnVlLAogICJ3ZWlnaHQiOiA4Cn0KICAgICAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAgICAgLmFkZFRvKG1hcF81N2ZmNmM1YTk5NWE0Nzc3ODgxZjZjYjU3ODRiY2NlYSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgdmFyIG1hcmtlcl9jMWRhYjFkZjE4MTE0MDYxYTg0YTBjYmViMjJkYWE4ZCA9IEwubWFya2VyKAogICAgICAgICAgICBbMzMuNjk4MTE2LCAtMTE3Ljg1MTIzNV0sCiAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpLAogICAgICAgICAgICAgICAgfQogICAgICAgICAgICApLmFkZFRvKG1hcF81N2ZmNmM1YTk5NWE0Nzc3ODgxZjZjYjU3ODRiY2NlYSk7CiAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl84YjJhNzMzY2YxN2I0ZTlkODVlODI5MDk3MTRhMTNiNCA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ3BsYXknLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyX2MxZGFiMWRmMTgxMTQwNjFhODRhMGNiZWIyMmRhYThkLnNldEljb24oaWNvbl84YjJhNzMzY2YxN2I0ZTlkODVlODI5MDk3MTRhMTNiNCk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgdmFyIG1hcmtlcl81MTc5ZjNmYmVhYzg0NmE0ODg1N2ZjYjVjYTA5M2U5MSA9IEwubWFya2VyKAogICAgICAgICAgICBbMzMuNjcyMTMzLCAtMTE3LjgzODYxN10sCiAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpLAogICAgICAgICAgICAgICAgfQogICAgICAgICAgICApLmFkZFRvKG1hcF81N2ZmNmM1YTk5NWE0Nzc3ODgxZjZjYjU3ODRiY2NlYSk7CiAgICAgICAgCiAgICAKCiAgICAgICAgICAgICAgICB2YXIgaWNvbl82M2IyMjgyNjg0OTE0MmViOGRhMzliZTIyYzBkYTExYSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbih7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogJ3N0b3AnLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ3JlZCcsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl81MTc5ZjNmYmVhYzg0NmE0ODg1N2ZjYjVjYTA5M2U5MS5zZXRJY29uKGljb25fNjNiMjI4MjY4NDkxNDJlYjhkYTM5YmUyMmMwZGExMWEpOwogICAgICAgICAgICAKPC9zY3JpcHQ+" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>



I just randomly pick two points in Irvine, CA and the route looks pretty good!

## Benchmarking 

If I want to use this API to processing data for me, I would like to know how fast it can handle my requests. Here I randomly generated another 1000 coordinates and request the routes from our docker backend as a mini stress test:


```python
import numpy as np
import pandas as pd
lon1 = np.random.uniform(-117.4,-118, 1000).round(6)
lon2 = np.random.uniform(-117.4,-118, 1000).round(6)
lat1 = np.random.uniform(33.6,33.8, 1000).round(6)
lat2 = np.random.uniform(33.6,33.8, 1000).round(6)
df = pd.DataFrame({'pickup_lon': lon1,
              'pickup_lat': lat1,
              'dropoff_lon': lon2,
              'dropoff_lat': lat2,
             })
```


```python
df.head()
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
      <th>pickup_lon</th>
      <th>pickup_lat</th>
      <th>dropoff_lon</th>
      <th>dropoff_lat</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-117.650723</td>
      <td>33.696095</td>
      <td>-117.400615</td>
      <td>33.675578</td>
    </tr>
    <tr>
      <th>1</th>
      <td>-117.653614</td>
      <td>33.713656</td>
      <td>-117.920080</td>
      <td>33.679549</td>
    </tr>
    <tr>
      <th>2</th>
      <td>-117.484076</td>
      <td>33.671013</td>
      <td>-117.960667</td>
      <td>33.741495</td>
    </tr>
    <tr>
      <th>3</th>
      <td>-117.599436</td>
      <td>33.656727</td>
      <td>-117.481877</td>
      <td>33.643613</td>
    </tr>
    <tr>
      <th>4</th>
      <td>-117.968429</td>
      <td>33.776134</td>
      <td>-117.469914</td>
      <td>33.739116</td>
    </tr>
  </tbody>
</table>
</div>




```python
%%time
df['routes'] = df.apply(lambda x: get_route(x['pickup_lon'], 
                                            x['pickup_lat'],
                                            x['dropoff_lon'], 
                                            x['dropoff_lat']), axis=1)
```

    CPU times: user 1.55 s, sys: 118 ms, total: 1.67 s
    Wall time: 5.72 s


Not bad at all! With single container and it can finish the request async in 6 seconds. If we put it on a multiple node docker swarm cluster with proper load balancer, I believe the performance will be very staisfactory.

Check with a random data I requested:


```python
get_map(df.loc[900,'routes'])
```





<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVM9ZmFsc2U7IExfTk9fVE9VQ0g9ZmFsc2U7IExfRElTQUJMRV8zRD1mYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS40LjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NvZGUuanF1ZXJ5LmNvbS9qcXVlcnktMS4xMi40Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS40LjAvZGlzdC9sZWFmbGV0LmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLm1pbi5jc3MiLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC10aGVtZS5taW4uY3NzIi8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9MZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy8yLjAuMi9sZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy5jc3MiLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9yYXdjZG4uZ2l0aGFjay5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIi8+CiAgICA8c3R5bGU+aHRtbCwgYm9keSB7d2lkdGg6IDEwMCU7aGVpZ2h0OiAxMDAlO21hcmdpbjogMDtwYWRkaW5nOiAwO308L3N0eWxlPgogICAgPHN0eWxlPiNtYXAge3Bvc2l0aW9uOmFic29sdXRlO3RvcDowO2JvdHRvbTowO3JpZ2h0OjA7bGVmdDowO308L3N0eWxlPgogICAgCiAgICA8bWV0YSBuYW1lPSJ2aWV3cG9ydCIgY29udGVudD0id2lkdGg9ZGV2aWNlLXdpZHRoLAogICAgICAgIGluaXRpYWwtc2NhbGU9MS4wLCBtYXhpbXVtLXNjYWxlPTEuMCwgdXNlci1zY2FsYWJsZT1ubyIgLz4KICAgIDxzdHlsZT4jbWFwX2JmNDQ5OWQ3ODIwODQ2YWFhMjQwODJjMjQ5ODAyOWVlIHsKICAgICAgICBwb3NpdGlvbjogcmVsYXRpdmU7CiAgICAgICAgd2lkdGg6IDEwMC4wJTsKICAgICAgICBoZWlnaHQ6IDEwMC4wJTsKICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgIHRvcDogMC4wJTsKICAgICAgICB9CiAgICA8L3N0eWxlPgo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgPGRpdiBjbGFzcz0iZm9saXVtLW1hcCIgaWQ9Im1hcF9iZjQ0OTlkNzgyMDg0NmFhYTI0MDgyYzI0OTgwMjllZSIgPjwvZGl2Pgo8L2JvZHk+CjxzY3JpcHQ+ICAgIAogICAgCiAgICAKICAgICAgICB2YXIgYm91bmRzID0gbnVsbDsKICAgIAoKICAgIHZhciBtYXBfYmY0NDk5ZDc4MjA4NDZhYWEyNDA4MmMyNDk4MDI5ZWUgPSBMLm1hcCgKICAgICAgICAnbWFwX2JmNDQ5OWQ3ODIwODQ2YWFhMjQwODJjMjQ5ODAyOWVlJywgewogICAgICAgIGNlbnRlcjogWzMzLjY2NjAyMTUsIC0xMTcuNzExMTQxNV0sCiAgICAgICAgem9vbTogMTMsCiAgICAgICAgbWF4Qm91bmRzOiBib3VuZHMsCiAgICAgICAgbGF5ZXJzOiBbXSwKICAgICAgICB3b3JsZENvcHlKdW1wOiBmYWxzZSwKICAgICAgICBjcnM6IEwuQ1JTLkVQU0czODU3LAogICAgICAgIHpvb21Db250cm9sOiB0cnVlLAogICAgICAgIH0pOwoKCiAgICAKICAgIHZhciB0aWxlX2xheWVyXzQ4YWM2YjM0YTU0OTRjMjY5ODg0MmUxYzQwMDQwZTgzID0gTC50aWxlTGF5ZXIoCiAgICAgICAgJ2h0dHBzOi8ve3N9LnRpbGUub3BlbnN0cmVldG1hcC5vcmcve3p9L3t4fS97eX0ucG5nJywKICAgICAgICB7CiAgICAgICAgImF0dHJpYnV0aW9uIjogbnVsbCwKICAgICAgICAiZGV0ZWN0UmV0aW5hIjogZmFsc2UsCiAgICAgICAgIm1heE5hdGl2ZVpvb20iOiAxOCwKICAgICAgICAibWF4Wm9vbSI6IDE4LAogICAgICAgICJtaW5ab29tIjogMCwKICAgICAgICAibm9XcmFwIjogZmFsc2UsCiAgICAgICAgIm9wYWNpdHkiOiAxLAogICAgICAgICJzdWJkb21haW5zIjogImFiYyIsCiAgICAgICAgInRtcyI6IGZhbHNlCn0pLmFkZFRvKG1hcF9iZjQ0OTlkNzgyMDg0NmFhYTI0MDgyYzI0OTgwMjllZSk7CiAgICAKICAgICAgICAgICAgICAgIHZhciBwb2x5X2xpbmVfY2VhZTk4MzU2YWVlNDU5ZjhmZGNmZDhlNWM3MGU0MzggPSBMLnBvbHlsaW5lKAogICAgICAgICAgICAgICAgICAgIFtbMzMuNjYwMTgsIC0xMTcuNjY4NDldLCBbMzMuNjYzNDYsIC0xMTcuNjY2NjddLCBbMzMuNjY3NzEsIC0xMTcuNjYyMDFdLCBbMzMuNjcyNDYsIC0xMTcuNjYwMzZdLCBbMzMuNjc1NjgsIC0xMTcuNjY5MThdLCBbMzMuNjc4MDksIC0xMTcuNjcxODRdLCBbMzMuNjg0ODksIC0xMTcuNjc2ODldLCBbMzMuNjk0ODgsIC0xMTcuNjkxMDddLCBbMzMuNzA2MjUsIC0xMTcuNzE0Nl0sIFszMy43MTA1OSwgLTExNy43MTkwNF0sIFszMy43MTE0MywgLTExNy43MjA5NV0sIFszMy43MTEzLCAtMTE3LjcyMjc2XSwgWzMzLjcxMDI4LCAtMTE3LjcyNDQzXSwgWzMzLjcwNDM3LCAtMTE3LjcyODE4XSwgWzMzLjY5ODA0LCAtMTE3LjczNDk0XSwgWzMzLjY3ODI3LCAtMTE3Ljc1MTYxXSwgWzMzLjY3NTIxLCAtMTE3Ljc1MzQ0XSwgWzMzLjY3MTg2LCAtMTE3Ljc1MzhdXSwKICAgICAgICAgICAgICAgICAgICB7CiAgImJ1YmJsaW5nTW91c2VFdmVudHMiOiB0cnVlLAogICJjb2xvciI6ICJibHVlIiwKICAiZGFzaEFycmF5IjogbnVsbCwKICAiZGFzaE9mZnNldCI6IG51bGwsCiAgImZpbGwiOiBmYWxzZSwKICAiZmlsbENvbG9yIjogImJsdWUiLAogICJmaWxsT3BhY2l0eSI6IDAuMiwKICAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsCiAgImxpbmVDYXAiOiAicm91bmQiLAogICJsaW5lSm9pbiI6ICJyb3VuZCIsCiAgIm5vQ2xpcCI6IGZhbHNlLAogICJvcGFjaXR5IjogMC42LAogICJzbW9vdGhGYWN0b3IiOiAxLjAsCiAgInN0cm9rZSI6IHRydWUsCiAgIndlaWdodCI6IDgKfQogICAgICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgICAgICAuYWRkVG8obWFwX2JmNDQ5OWQ3ODIwODQ2YWFhMjQwODJjMjQ5ODAyOWVlKTsKICAgICAgICAgICAgCiAgICAKICAgICAgICB2YXIgbWFya2VyXzI2ZGNkODVmY2U3ZDRkZjBhYTdlZjM0NGM4MGEzYmNhID0gTC5tYXJrZXIoCiAgICAgICAgICAgIFszMy42NjAxOCwgLTExNy42Njg0ODVdLAogICAgICAgICAgICB7CiAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKSwKICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgKS5hZGRUbyhtYXBfYmY0NDk5ZDc4MjA4NDZhYWEyNDA4MmMyNDk4MDI5ZWUpOwogICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fMDY0ZmMyNjJhZWQxNGE2Yjg4NTM2NTk3ZDk0ZGJjZmMgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdwbGF5JywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdncmVlbicsCiAgICAgICAgICAgICAgICAgICAgcHJlZml4OiAnZ2x5cGhpY29uJywKICAgICAgICAgICAgICAgICAgICBleHRyYUNsYXNzZXM6ICdmYS1yb3RhdGUtMCcKICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgICAgIG1hcmtlcl8yNmRjZDg1ZmNlN2Q0ZGYwYWE3ZWYzNDRjODBhM2JjYS5zZXRJY29uKGljb25fMDY0ZmMyNjJhZWQxNGE2Yjg4NTM2NTk3ZDk0ZGJjZmMpOwogICAgICAgICAgICAKICAgIAogICAgICAgIHZhciBtYXJrZXJfNTgyYThmYzhmOTk0NGY5OGFmOGE3MTEzN2NhZjIzMDIgPSBMLm1hcmtlcigKICAgICAgICAgICAgWzMzLjY3MTg2MywgLTExNy43NTM3OThdLAogICAgICAgICAgICB7CiAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKSwKICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgKS5hZGRUbyhtYXBfYmY0NDk5ZDc4MjA4NDZhYWEyNDA4MmMyNDk4MDI5ZWUpOwogICAgICAgIAogICAgCgogICAgICAgICAgICAgICAgdmFyIGljb25fNTNhNmY5ZjUzY2M2NDg4NGFjZTY5YzE2OGY0YTNmNWEgPSBMLkF3ZXNvbWVNYXJrZXJzLmljb24oewogICAgICAgICAgICAgICAgICAgIGljb246ICdzdG9wJywKICAgICAgICAgICAgICAgICAgICBpY29uQ29sb3I6ICd3aGl0ZScsCiAgICAgICAgICAgICAgICAgICAgbWFya2VyQ29sb3I6ICdyZWQnLAogICAgICAgICAgICAgICAgICAgIHByZWZpeDogJ2dseXBoaWNvbicsCiAgICAgICAgICAgICAgICAgICAgZXh0cmFDbGFzc2VzOiAnZmEtcm90YXRlLTAnCiAgICAgICAgICAgICAgICAgICAgfSk7CiAgICAgICAgICAgICAgICBtYXJrZXJfNTgyYThmYzhmOTk0NGY5OGFmOGE3MTEzN2NhZjIzMDIuc2V0SWNvbihpY29uXzUzYTZmOWY1M2NjNjQ4ODRhY2U2OWMxNjhmNGEzZjVhKTsKICAgICAgICAgICAgCjwvc2NyaXB0Pg==" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>



## Potential applications

Now we have seen the beauty of the OSRM. You can imagine how many use cases it could potentially has. I actually used it to generate features in one Kaggle competition --- NYC taxi fare prediction [(link)](https://www.kaggle.com/c/new-york-city-taxi-fare-prediction/overview). In this competition, you were asked to predict the taxi fares given some basic features including the pickup and dropoff coordinates. As we all know that Haversine distance is different than the actually driving distance, especially in NYC. My intuition is that using the predicted driving distance will increase the model accuracy. Because that is how the taxi fares are calculated anyway. I was absolutely right. By adding this trip distance to the data, I am able to achieve the score of 3.09 which is about 300/1500 on the leaderboard, using just 10% of the data! (full dataset is too big to work with on my laptop) I will publish my detailed approach in the next posts if you are interested.

So what are you waiting for? Starting routing now!

![](https://media.giphy.com/media/U4YEoSRgz5JtebnYqH/giphy.gif)
