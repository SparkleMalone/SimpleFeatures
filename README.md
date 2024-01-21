# Introduction to Simple Features in R

### Load the required libraries for this workshop
```{r, include=T}
library(sf)
library(AOI)
library(tidyverse)
library(ggplot2)
```
Simple features describe how objects in the real world can be represented in computers. They have a geometry describing where on Earth the feature is located, and they have attributes, which describe other properties about the feature. 

 The following command reads the site location for Fluxnet CH4 tower locations:
 
```{r, include=T}

Fluxnet.ch4 <- read_sf( dsn= "Data", layer= 'CH4_sites', crs="+init=epsg:4326")

```
Look at the class:

```{r, include=T}

class( Fluxnet.ch4)
Fluxnet.ch4$geometry
```
If we print the first three features, we see their attribute values and an abridged version of the geometry
```{r, include=T}
print(Fluxnet.ch4, n = 3)
```
Methods for sf objects are:

methods(class = "sf")

It is possible to create data.frame objects with geometry list-columns that are not of class sf, e.g. by:
```{r, include=T}
Fluxnet.ch4.df <- as.data.frame(Fluxnet.ch4)
```
Check the class:
```{r, include=T}
class(nc.no_sf)
```
Such objects:
no longer register which column is the geometry list-column
no longer have a plot method, and
lack all of the other dedicated methods listed above for class sf

### Geometrical Operations
The standard for simple feature access defines a number of geometrical operations. st_is_valid and st_is_simple return a boolean indicating whether a geometry is valid or simple.

```{r, include=T}
st_is_valid(Fluxnet.ch4)
```
# Change the CRS to a projected EPGS prior to measuring distance:
```{r, include=T}
Fluxnet.ch4 = st_transform(Fluxnet.ch4, '+init=epsg:4087')
st_crs(Fluxnet.ch4)
```
st_distance returns a dense numeric matrix with distances between geometries. st_relate returns a character matrix with the DE9-IM values for each pair of geometries:

```{r, include=T}
st_distance(Fluxnet.ch4[1,], Fluxnet.ch4)
```
Use the package AOI to generate a polygon for South America and Brazil:

```{r, include=T}
s.america <- aoi_get(country= "South America", union=T)
s.america <- st_transform( s.america , '+init=epsg:4087') # Re-project the polygon to match Fluxnet.ch4

Brazil <- aoi_get(country= "Brazil", union=T)
Brazil <- st_transform( Brazil , '+init=epsg:4087') # Re-project the polygon to match Fluxnet.ch4
```

The commands st_intersects, st_disjoint, st_touches, st_crosses, st_within, st_contains, st_overlaps, st_equals, st_covers, st_covered_by, st_equals_exact and st_is_within_distance return a sparse matrix with matching (TRUE) indexes, or a full logical matrix:


# How many towers are in South America?

```{r, include=T}
st_intersects(s.america, Fluxnet.ch4)
st_intersects(s.america, Fluxnet.ch4, sparse = FALSE)
```

# How many towers are in Brazil?
```{r, include=T}
st_intersects(Brazil, Fluxnet.ch4)
```

The commands st_buffer, st_boundary, st_convexhull, st_union_cascaded, st_simplify, st_triangulate, st_polygonize, st_centroid, st_segmentize, and st_union return new geometries, e.g.:

```{r, include=T}
buffer <- st_buffer(s.america, dist = 100000)
plot(buffer, border = 'red')
plot(s.america, add=T)
```

Commands st_intersection, st_union, st_difference, st_sym_difference return new geometries that are a function of pairs of geometries:

```{r, include=T}
u <- st_difference(s.america, Brazil)
plot(u)
```

Where possible geometric operations such as st_distance(), st_length() and st_area() report results with a units attribute appropriate for the CRS:
```{r, include=T}
Brazil$Area <- st_area( Brazil)
```

### Visualize the global distribution of towers:

First create a simple feature for all large terrestrial regions in Europe, Asia, the Americas, Africa, and Australia:
```{r, include=T}
world <- aoi_get(country= c("Europe","Asia" ,"North America", "South America", "Australia","Africa", "New Zealand"))

st_crs(world)

world <- st_transform( world , '+init=epsg:4087') # Re-project the polygon to match Fluxnet.ch4
```

Use ggplot to visualize the global distribution:

```{r, include=T}
ggplot() + geom_sf(data = world) + geom_sf(data = Fluxnet.ch4) 
```

Extract the country from the world simple feature into Fluxnet.ch4:
```{r, include=T}
Fluxnet.ch4$Country <- st_intersection( world,Fluxnet.ch4)$name
```

Explore tower dataset:

```{r, include=T}
names(Fluxnet.ch4)

ggplot(data=Fluxnet.ch4) + geom_point( aes(x=MAT, y=MAP))

ggplot(data=Fluxnet.ch4) + geom_point( aes(x=MAT, y=MAP, col=IGBP))

Fluxnet.ch4$IGBP <- as.factor(Fluxnet.ch4$IGBP)
summary(Fluxnet.ch4$IGBP)











# Writing files using st_write:

write_sf(nc, "nc.shp") # silently overwrites


Driver-specific options
The dsn and layer arguments to st_read and st_write denote a data source name and optionally a layer name. Their exact interpretation as well as the options they support vary per driver, the GDAL driver documentation is best consulted for this. When the layer and driver arguments are not specified, st_read tries to guess them from the datasource, or else simply reads the first layer, giving a warning in case there are more.
st_read typically reads the coordinate reference system as proj4string. GDAL cannot retrieve SRID (EPSG code) from proj4string strings, and, when needed, it has to be set by the user. 
st_drivers() returns a data.frame listing available drivers, and their metadata: names, whether a driver can write, and whether it is a raster and/or vector driver. All drivers can read. Reading of some common data formats is illustrated below:
st_layers(dsn) lists the layers present in data source dsn, and gives the number of fields, features and geometry type for each layer:
st_layers(system.file("osm/overpass.osm", package="sf"))
we see that in this case, the number of features is NA because for this xml file the whole file needs to be read, which may be costly for large files. We can force counting by:
Sys.setenv(OSM_USE_CUSTOM_INDEXING="NO")
st_layers(system.file("osm/overpass.osm", package="sf"), do_count = TRUE)


Create, read, update and delete
GDAL provides the crud (create, read, update, delete) functions to persistent storage. st_read (or read_sf) are used for reading. st_write (or write_sf) creates, and has the following arguments to control update and delete:
update=TRUE causes an existing data source to be updated, if it exists; this option is by default TRUE for all database drivers, where the database is updated by adding a table.
delete_layer=TRUE causes st_write try to open the data source and delete the layer; no errors are given if the data source is not present, or the layer does not exist in the data source.
delete_dsn=TRUE causes st_write to delete the data source when present, before writing the layer in a newly created data source. No error is given when the data source does not exist. This option should be handled with care, as it may wipe complete directories or databases.
Coordinate reference systems and transformations
Coordinate reference systems (CRS) are like measurement units for coordinates: they specify which location on Earth a particular coordinate pair refers to. We saw above that sfc objects (geometry list-columns) have two attributes to store a CRS: epsg and proj4string. This implies that all geometries in a geometry list-column must have the same CRS. Both may be NA, e.g. in case the CRS is unknown, or when we work with local coordinate systems (e.g. inside a building, a body, or an abstract space).
proj4string is a generic, string-based description of a CRS, understood by the PROJ library. It defines projection types and (often) defines parameter values for particular projections, and hence can cover an infinite amount of different projections. This library (also used by GDAL) provides functions to convert or transform between different CRS. epsg is the integer ID for a particular, known CRS that can be resolved into a proj4string. Some proj4string values can be resolved back into their corresponding epsg ID, but this does not always work.
The importance of having epsg values stored with data besides proj4string values is that the epsg refers to particular, well-known CRS, whose parameters may change (improve) over time; fixing only the proj4string may remove the possibility to benefit from such improvements, and limit some of the provenance of datasets, but may help reproducibility.
Coordinate reference system transformations can be carried out using st_transform, e.g. converting longitudes/latitudes in NAD27 to web mercator (EPSG:3857) can be done by:
nc.web_mercator <- st_transform(nc, 3857)
st_geometry(nc.web_mercator)[[4]][[2]][[1]][1:3,]


Conversion, including to and from sp
sf objects and objects deriving from Spatial (package sp) can be coerced both ways:
showMethods("coerce", classes = "sf")

methods(st_as_sf)

methods(st_as_sfc)





Further reading:
S. Scheider, B. Gräler, E. Pebesma, C. Stasch, 2016. Modelling spatio-temporal information generation. Int J of Geographic Information Science, 30 (10), 1980-2008. (open access)
Stasch, C., S. Scheider, E. Pebesma, W. Kuhn, 2014. Meaningful Spatial Prediction and Aggregation. Environmental Modelling & Software, 51, (149–165, open access).

