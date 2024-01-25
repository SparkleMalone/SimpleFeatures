# Introduction to Simple Features in R

### Install libraries for this workshop
```{r, include=T}
install.packages('sf')
install.packages('devtools')
devtools::install_github("mikejohnson51/AOI", force = TRUE)
devtools::install_github("valentinitnelav/plotbiomes")
```

### Load the required libraries for this workshop
```{r, include=T}
library(sf)
library(AOI)
library(tidyverse)
library(ggplot2)
```
# Goals

The goals of this workshop are to:
1. Become familiar with simple features
2. Master simple feature manipulation
3. Visualize simple features

# Data

Import FluxNet_Sites_2024.csv. This table was created from the FLUXNET site list found at  https://fluxnet.org/sites/site-list-and-pages/?view=table. 

```{r, include=T}
FluxNet <- read.csv('Data/FluxNet_Sites_2024.csv')
```
This dataset includes:

 Column Name | Description |
|------:|-----------|
|SITE_ID| Unique site id|
|SITE_NAME|Site name|
|FLUXNET2015|License information for the data for the two FLUXNET Products|
|FLUXNET-CH4|License information for the data for the two FLUXNET Products|
|LOCATION_LAT|Location information|
|LOCATION_LONG|Location information|
|LOCATION_ELEV|Elevation in meters|
|IGBP|Vegetation type|
|MAT| Mean annual temperature in Celsius|
|MAT| Mean annual precipitation in mm|

Take a look at the file:
```{r, include=T}

View(FluxNet) 

```

We are interested in exploring the sites with methane data. Lets subset by FLUXNET-CH4.
```{r, include=T}

FLUXNET.CH4 <- FluxNet %>% filter( FLUXNET.CH4 != "")

```

This object is currently a dataframe. 
```{r, include=T}
class(FLUXNET.CH4)
```

Lets make it a simple feature using st_as sf().

```{r, include=T}
FLUXNET.CH4.shp <- st_as_sf(x = FLUXNET.CH4,                         
           coords = c("LOCATION_LONG",  "LOCATION_LAT"),
           crs = "+init=epsg:4326")

ggplot(data=FLUXNET.CH4.shp ) + geom_sf()

```
check the class:
```{r, include=T}
class(FLUXNET.CH4.shp)
```
Simple features describe how objects in the real world can be represented in computers. They have a geometry describing where on earth the feature is located, and they have attributes, which describe other properties about the feature. 

Look at the information about the geometry:

```{r, include=T}

FLUXNET.CH4.shp$geometry
```
If we print the first three features, we see their attribute values and an abridged version of the geometry.

```{r, include=T}
print(FLUXNET.CH4.shp, n = 3)
```
It is possible to create data.frame objects with geometry list-columns that are not of class sf by:

```{r, include=T}
Fluxnet.ch4.df <- as.data.frame(FLUXNET.CH4.shp)
```
Check the class:
```{r, include=T}
class(Fluxnet.ch4.df)
```
Such objects:
no longer register which column is the geometry list-column
no longer have a plot method, and
lack all of the other dedicated methods listed above for class sf

# Geometrical Operations

There are many geometrical operations that can be used to achieve simple feature manipulation.

```{r, include=T}
methods(class = "sf")
```
Below we will explore a few methods:

st_is_valid and st_is_simple return a boolean indicating whether a geometry is valid or simple.

```{r, include=T}

st_is_valid(FLUXNET.CH4.shp)

```
### Change the CRS to a projected EPGS with st_transform.
<a href= "https://www.nceas.ucsb.edu/sites/default/files/2020-04/OverviewCoordinateReferenceSystems.pdf"> 
Coordinate reference systems (CRS) </a> are like measurement units for coordinates: they specify which location on Earth a particular coordinate pair refers to. Simple feature have two attributes to store a CRS: epsg and proj4string. This implies that all geometries in a geometry list-column must have the same CRS. Both may be NA, e.g. in case the CRS is unknown, or when we work with local coordinate systems (e.g. inside a building, a body, or an abstract space).

proj4string is a generic, string-based description of a CRS, understood by the PROJ library. It defines projection types and (often) defines parameter values for particular projections, and hence can cover an infinite amount of different projections. This library (also used by GDAL) provides functions to convert or transform between different CRS. epsg is the integer ID for a particular, known CRS that can be resolved into a proj4string. Some proj4string values can be resolved back into their corresponding epsg ID, but this does not always work.

The importance of having epsg values stored with data besides proj4string values is that the epsg refers to particular, well-known CRS, whose parameters may change (improve) over time; fixing only the proj4string may remove the possibility to benefit from such improvements, and limit some of the provenance of datasets, but may help reproducibility.
Coordinate reference system transformations can be carried out using st_transform

```{r, include=T}
FLUXNET.CH4.shp = st_transform(FLUXNET.CH4.shp, '+init=epsg:4087')
```
Check the CRS:
```{r, include=T}
st_crs(FLUXNET.CH4.shp)
```
st_distance returns a dense numeric matrix with distances between geometries:

```{r, include=T}
st_distance(FLUXNET.CH4.shp[1,], FLUXNET.CH4.shp)
```
Use the package AOI to generate a polygon for South America and Brazil:

```{r, include=T}
# Creates a simple feature of South America:
s.america <- aoi_get(country= "South America", union=T)

# Re-project the s.america to match Fluxnet.ch4: 
s.america <- st_transform( s.america , '+init=epsg:4087') 

# Creates a simple feature of Brazil:
Brazil <- aoi_get(country= "Brazil", union=T)

# Re-project the Brazil to match Fluxnet.ch4:
Brazil <- st_transform( Brazil , '+init=epsg:4087') # Re-project the polygon to match Fluxnet.ch4
```
The commands st_intersects, st_disjoint, st_touches, st_crosses, st_within, st_contains, st_overlaps, st_equals, st_covers, st_covered_by, st_equals_exact and st_is_within_distance all return a sparse matrix with matching (TRUE) indexes, or a full logical matrix:

# How many towers are in South America?

```{r, include=T}

st_intersects(s.america, Fluxnet.ch4)
st_intersects(s.america, Fluxnet.ch4, sparse = FALSE)

```

# How many towers are in Brazil?
```{r, include=T}

st_intersects(Brazil, Fluxnet.ch4)

```
Where possible geometric operations such as st_distance(), st_length() and st_area() report results with a units attribute appropriate for the CRS. 

Calculate the area of Brazil:

```{r, include=T}
Brazil$Area <- st_area( Brazil)

```
#### Visualize the global distribution of towers:

First create a simple feature for all large terrestrial regions in Europe, Asia, the Americas, Africa, Australia and New Zealand:

```{r, include=T}

# Create a shapefile:
world <- aoi_get(country= c("Europe","Asia" ,"North America", "South America", "Australia","Africa", "New Zealand"))

```
 Look at the CRS:
```{r, include=T}
st_crs(world)
``` 
Re-project the polygon to match FLUXNET.CH4.shp:
```{r, include=T}
world <- st_transform( world , '+init=epsg:4087') 
```
Visualize the shapefile you created:
```{r, include=T}
ggplot(data=world) + geom_sf()
```
Use ggplot to visualize the global distribution of Fluxnet CH4 sites:

```{r, include=T}
ggplot() + geom_sf(data = world) + geom_sf(data = FLUXNET.CH4.shp) 
```

Extract the country from the world simple feature into FLUXNET.CH4.shp:
```{r, include=T}
FLUXNET.CH4.shp$Country <- st_intersection( world, FLUXNET.CH4.shp)$name
```
Explore the Fluxnet CH4 sites:

```{r, include=T}
names(FLUXNET.CH4.shp)

ggplot(data= FLUXNET.CH4.shp) + geom_point( aes(x=MAT, y=MAP))

ggplot(data= FLUXNET.CH4.shp) + geom_point( aes(x=MAT, y=MAP, col=IGBP))

FLUXNET.CH4.shp$IGBP <- as.factor( FLUXNET.CH4.shp$IGBP)
summary(FLUXNET.CH4.shp$IGBP)

```

### Writing files using st_write:

```{r, include=T}
write_sf(FLUXNET.CH4.shp, "FLUXNET.CH4.shp") 
```
When writing, you can use the following arguments to control update and delete: update=TRUE causes an existing data source to be updated, if it exists; this option is by default TRUE for all database drivers, where the database is updated by adding a table.

delete_layer=TRUE causes st_write try to open the data source and delete the layer; no errors are given if the data source is not present, or the layer does not exist in the data source.

delete_dsn=TRUE causes st_write to delete the data source when present, before writing the layer in a newly created data source. No error is given when the data source does not exist. This option should be handled with care, as it may wipe complete directories or databases.

### Additional Reading:
S. Scheider, B. Gräler, E. Pebesma, C. Stasch, 2016. Modelling spatio-temporal information generation. Int J of Geographic Information Science, 30 (10), 1980-2008. (open access)
Stasch, C., S. Scheider, E. Pebesma, W. Kuhn, 2014. Meaningful Spatial Prediction and Aggregation. Environmental Modelling & Software, 51, (149–165, open access).

# Post Workshop Assessment:

Explore the distribution of tower sites and create 2 visualizations of this dataset that may be helpful to understand in the design and development of models. You are welcome to use any additional data or just new plot types.
