# Map matching on OpenStreetMap, a practical tutorial

This tutorial is based on the [fast map matching program](https://github.com/cyang-kth/fmm).
It also supports road network in ESRI Shapefile.

### Demonstration

<img src="img/demo1.gif" width="400"/> <img src="img/demo2.gif" width="400"/>

### Content

- [1. Download routable network from OSM](#1-download-routable-network-from-osm)
- [2. Run map matching with fmm](#2-run-map-matching-with-fmm)

### Tools used

- [OSMNX](https://github.com/gboeing/osmnx) for downloading OSM data
- [Fast map matching program (Python extension)](https://github.com/cyang-kth/fmm) for map matching

### 1. Download routable network from OSM

[OSMNX](https://github.com/gboeing/osmnx) provides a convenient way to download a routable shapefile directly in python.

The original osmnx saves network in an undirected way, therefore we use the script below to save the graph to a bidirectional network where all edges are directed and bidirectional edges (osm tag one-way is false) are represented by two reverse edges.

<div align="center">
  <img src="img/dual.png" width="410"/>
</div>

```python
import osmnx as ox
import time
from shapely.geometry import Polygon
import os
import numpy as np

def save_graph_shapefile_directional(G, filepath=None, encoding="utf-8"):
    # default filepath if none was provided
    if filepath is None:
        filepath = os.path.join(ox.settings.data_folder, "graph_shapefile")

    # if save folder does not already exist, create it (shapefiles
    # get saved as set of files)
    if not filepath == "" and not os.path.exists(filepath):
        os.makedirs(filepath)
    filepath_nodes = os.path.join(filepath, "nodes.shp")
    filepath_edges = os.path.join(filepath, "edges.shp")

    # convert undirected graph to gdfs and stringify non-numeric columns
    gdf_nodes, gdf_edges = ox.utils_graph.graph_to_gdfs(G)
    gdf_nodes = ox.io._stringify_nonnumeric_cols(gdf_nodes)
    gdf_edges = ox.io._stringify_nonnumeric_cols(gdf_edges)
    # We need an unique ID for each edge
    gdf_edges["fid"] = np.arange(0, gdf_edges.shape[0], dtype='int')
    # save the nodes and edges as separate ESRI shapefiles
    gdf_nodes.to_file(filepath_nodes, encoding=encoding)
    gdf_edges.to_file(filepath_edges, encoding=encoding)

print("osmnx version",ox.__version__)

# Download by a bounding box
bounds = (17.4110711999999985,18.4494298999999984,59.1412578999999994,59.8280297000000019)
x1,x2,y1,y2 = bounds
boundary_polygon = Polygon([(x1,y1),(x2,y1),(x2,y2),(x1,y2)])
G = ox.graph_from_polygon(boundary_polygon, network_type='drive')
start_time = time.time()
save_graph_shapefile_directional(G, filepath='./network-new')
print("--- %s seconds ---" % (time.time() - start_time))

# Download by place name
place ="Stockholm, Sweden"
G = ox.graph_from_place(place, network_type='drive', which_result=2)
save_graph_shapefile_directional(G, filepath='stockholm')

# Download by a boundary polygon in geojson
import osmnx as ox
from shapely.geometry import shape
json_file = open("stockholm_boundary.geojson")
import json
data = json.load(json_file)
boundary_polygon = shape(data["features"][0]['geometry'])
G = ox.graph_from_polygon(boundary_polygon, network_type='drive')
save_graph_shapefile_directional(G, filepath='stockholm')
```

In the third manner, here is a screenshot of the network in QGIS.

<div align="center">
  <img src="img/raw_data.png" width="410"/>
</div>

### 2. Run map matching with fmm

Install the [**fmm** program](https://github.com/cyang-kth/fmm) in **C++ and Python** extension following the [instructions](https://github.com/cyang-kth/fmm/wiki).

The network downloaded from OSMNX using the above script is compatible with fmm
as it contains
- fid: id of edge
- u: source node of an edge
- v: target node of an edge

Below we show running fmm in Python to match GPS trajectory to the network.

```python
from fmm import FastMapMatch,Network,NetworkGraph,UBODTGenAlgorithm,UBODT,FastMapMatchConfig

### Read network data

network = Network("network/edges.shp","fid","u","v")
print "Nodes {} edges {}".format(network.get_node_count(),network.get_edge_count())
graph = NetworkGraph(network)


### Precompute an UBODT table

# Can be skipped if you already generated an ubodt file
ubodt_gen = UBODTGenAlgorithm(network,graph)
status = ubodt_gen.generate_ubodt("network/ubodt.txt", 0.02, binary=False, use_omp=True)
print status

### Read UBODT

ubodt = UBODT.read_ubodt_csv("network/ubodt.txt")

### Create FMM model
model = FastMapMatch(network,graph,ubodt)

### Define map matching configurations

k = 8
radius = 0.003
gps_error = 0.0005
fmm_config = FastMapMatchConfig(k,radius,gps_error)


### Run map matching for wkt
wkt = "LineString(104.10348 30.71363,104.10348 30.71363,104.10348 30.71363,104.10348 30.71363,104.10348 30.71363)"
result = model.match_wkt(wkt,fmm_config)

### Print map matching result
print "Opath ",list(result.opath)
print "Cpath ",list(result.cpath)
print "WKT ",result.mgeom.export_wkt()
```

A more complete version of notebook (including matching GPS data stored in files)
can be found at https://github.com/cyang-kth/fmm/blob/master/example/notebook/fmm_example.ipynb.

STMATCH algorithm which does not need precomputation can also be used. https://github.com/cyang-kth/fmm/blob/master/example/notebook/stmatch_example.ipynb

### Citation information

Please cite fmm in your publications if it helps your research:

```
Can Yang & Gyozo Gidofalvi (2018) Fast map matching, an algorithm
integrating hidden Markov model with precomputation, International Journal of Geographical Information Science, 32:3, 547-570, DOI: 10.1080/13658816.2017.1400548
```

Bibtex format

```
@article{Yang2018fast,
author = {Can Yang and Gyozo Gidofalvi},
title = {Fast map matching, an algorithm integrating hidden Markov model with precomputation},
journal = {International Journal of Geographical Information Science},
volume = {32},
number = {3},
pages = {547-570},
year  = {2018},
publisher = {Taylor & Francis},
doi = {10.1080/13658816.2017.1400548},
}
```

### Contact information

Can Yang, Ph.D. student at KTH, Royal Institute of Technology in Sweden

Email: cyang(at)kth.se

Homepage: https://people.kth.se/~cyang/
