# Introduction to rasters with the terra package.

### Install libraries for this workshop:
```{r, include=T}
install.packages('sf')
install.packages('terra')
install.packages("remotes")
install.packages("tidyverse")
install.packages('tidyterra')
remotes::install_github("mikejohnson51/AOI")
remotes::install_github("mikejohnson51/climateR")
```
### Load libraries:

```{r, include=T}
library(terra)
library(remotes)
library(tidyverse)
library(AOI)
library(climateR)
library(sf)
library(tidyterra)
```
# Rasters
A raster is a spatial data structure that subdivides an extent into rectangles known as "cells" (or "pixels"). Each cell has the capacity to store one or more values. This type of data structure is commonly referred to as a "grid" and is frequently juxtaposed with simple features.

The terra package offers functions designed for the creation, reading, manipulation, and writing of raster data. The terra package is built around a number of “classes” of which the SpatRaster and SpatVector are the most important.

#### SpatRaster
A SpatRaster object stores a number of fundamental parameters that describe it. These include the number of columns and rows, the coordinates of its spatial extent, and the coordinate reference system. In addition, a SpatRaster can store information about the file(s) in which the raster cell values are stored.

#### SpatVector
A SpatVector represents “vector” data, that is, points, lines or polygon geometries and their tabular attributes.

### Working with climate data:

To become familiar with working with rasters, we will download climate data for an area of interest (AOI). 
```{r, include=T}

# First create an AOI
global <- aoi_get(country= c("Europe","Asia" ,"North America", "South America", "Australia","Africa", "New Zealand"))

# Visualize your AOI
ggplot(data = global) +geom_sf()
```
We will use TerraClimate, a dataset of high-spatial resolution (1/24°, ~4-km) monthly climate and climatic water balance for global terrestrial surfaces from 1958–2015.

Abatzoglou, J., Dobrowski, S., Parks, S. et al. TerraClimate, a high-resolution global dataset of monthly climate and climatic water balance from 1958–2015. Sci Data 5, 170191 (2018). https://doi.org/10.1038/sdata.2017.191

Download climate data using the library climateR for the AOI. For this exercise we will use climate normals, multi-decadal averages for climate variables like temperature and precipitation. They provide a baseline that allows us to understand the location’s average condition.

Downloaded monthly precipitation ("ppt"), monthly temperature minimum ("tmin") and monthly temperature maximum ("tmax")climate normals.

```{r, include=T}

global.normals <- global %>% getTerraClimNormals(varname = c("ppt", "tmin", "tmax"))

global.normals
```
What is this object:
```{r, include=T}
class(global.normals)
```

A RasterStack is a collection of RasterLayer objects with the same spatial extent and resolution. In essence it is a list of RasterLayer objects.

To access the raster stacks:
```{r, include=T}
global.normals$ppt
global.normals$tmin
global.normals$tmax
```
### Raster algebra
Many generic functions that allow for simple and elegant raster algebra have been implemented for SpatRaster objects, including the normal algebraic operators such as +, -, *, /, logical operators such as >, >=, <, ==, !} and functions such as abs, round, ceiling, floor, trunc, sqrt, log, log10, exp, cos, sin, max, min, range, prod, sum, any, all. In these functions you can mix terra objects with numbers, as long as the first argument is a terra object. If you use multiple SpatRaster objects, all objects must have the same resolution and origin. 

```{r, include=T}
# Separate raster stacks: 

global.ppt <- global.normals$ppt %>% sum(na.rm = TRUE)
global.tmin <- global.normals$tmin %>% mean(na.rm = TRUE)
global.tmax <- global.normals$tmax %>% mean(na.rm = TRUE)

# look at the object
plot(global.ppt)
plot(global.tmin)
plot(global.tmax)

# Check the name of the layers:
names(global.ppt)
names(global.tmin)
names(global.tmax)

# re-name the layers:
names(global.ppt) <- "ppt"
names(global.tmin) <- "tmin"
names(global.tmax) <- "tmax"

```
Summary functions (min, max, mean, prod, sum, median, cv, range, any, all) always return a SpatRaster object. Perhaps this is not obvious when using functions like min, sum or mean.

Use global if instead of a SpatRaster you want a single number summarizing the cell values of each layer.

```{r, include=T}
global( global.ppt, na.rm=T, mean)
global( global.tmin, na.rm=T, mean)
global( global.tmax, na.rm=T, mean)
```

### Spatial Summaries!

You might also find it useful to create zonal summaries for each polygon within the simple feature. To do this we can use the function zonal, which takes a SpatRast and a SpatVect.

```{r, include=T}
global.ppt.country <- zonal(x = global.ppt, 
z= vect(global) , fun = "mean", as.polygons=TRUE,  na.rm=TRUE)

global.tmin.country <- zonal(x = global.tmin, 
z= vect(global) ,fun = "mean", as.polygons=TRUE,  na.rm=TRUE)

global.tmax.country <- zonal(x = global.tmax, 
z= vect(global) ,fun = "mean", as.polygons=TRUE,  na.rm=TRUE)

```
Convert the SpatVect back to a simple feature and plot it.

```{r, include=T}

global.ppt.country.sf <- st_as_sf(global.ppt.country)   
global.tmin.country.sf <- st_as_sf(global.tmin.country)  
global.tmax.country.sf <- st_as_sf(global.tmax.country)  

ggplot( data=global.ppt.country.sf ) + geom_sf(aes(fill= ppt))
ggplot( data=global.tmin.country.sf ) + geom_sf(aes(fill= tmin))
ggplot( data=global.tmax.country.sf ) + geom_sf(aes(fill= tmax))
```
Combine SpatRast into a raster stack:
```{r, include=T}
global( global.ppt, na.rm=T, mean)
global( global.tmin, na.rm=T, mean)
global( global.tmax, na.rm=T, mean)

global.climate <- c( global.ppt, global.tmin, global.tmax)
```
### Extracting information to a point file:

Import your point file FLUXNET.ch4:

```{r, include=T}
FLUXNET.ch4 <- st_read(dsn="Data", layer="FLUXNET_CH4")
```
Ensure both files have the same coordinate reference system (CRS):
```{r, include=T}
FLUXNET.ch4 
global.climate 

FLUXNET.ch4  <- st_transform(FLUXNET.ch4, crs= crs(global.climate ))

ggplot() + geom_spatraster( data=global.climate, aes(fill = ppt)) +geom_sf( data =FLUXNET.ch4 )
```
Extract information from your raster stack using terra::extract()

```{r, include=T}
FLUXNET.ch4$ppt <-terra::extract( global.climate,FLUXNET.ch4)$ppt

FLUXNET.ch4$tmax <-terra::extract( global.climate,FLUXNET.ch4)$tmax

FLUXNET.ch4$tmin <-terra::extract( global.climate,FLUXNET.ch4)$tmin 
```
FLUXNET data can be used to understand patterns in natural methane fluxes. Evaluating the conditions where measurements are taken is essential to designing a useful model. 

We will use data from FLUXNET CH4 to explore patterns in natural methane emissions. Explore the distribution of tower sites and create 2 visualizations of this dataset that may be helpful to understand in the design and development of models. You are welcome to use any additional data or just new plot types.
