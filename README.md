# 3D Visualization of Gradient Surface Metrics in R


## BACKGROUND AND PURPOSE
Landscape ecologists have long understood that ecological processes can 
being influenced by landscape pattern. The prevailing paradigm for capturing
and analyzing spatial patterns of landscape features has been the patch-mosaic model (PMM).
This model is based largely on a categorical representation of spatial heterogeneity. 
The patch-mosaic model is well suited for landscapes with distinct edges, such as those
that have experienced anthropogenic disturbance (e.g., clear cuts, agricultural, etc.).
However, the PMM is less effective in landscapes characterized by continuous spatial
heterogeneity (McGarigal and Cushman 2005). In the PMM approach, once a patch is 
defined (i.e., categorized), the heterogeneity within the patch is lost. Many
ecological features vary continuously, as gradients across space. And increasingly
we're able to create continuous representations of landscape features using 
remote sensing techniques (e.g., percent land cover). In recent years, there has been
some slow progress toward developing models of landscape structure based on continuous
rather than categorical data (McGarigal and Cushman 2005; McGarigal et al. 2009). Much
of the advancement in this domain has centered around quantifying surface complexity
using tools developed in the field of surface metrology, a field that spun out of 
physics and mechanical engineering. The ecologist  or biologist may recognize some 
of these tools as they've been used to derived variables that represent topographic variability
(e.g., topographic roughness) from digital elevation models. Recently, more work
has been done to understand how these tools may be applied to continuous representations
of landscape heterogeneity (McGarigal et al. 2009) and specifically how they compare
to the more familiar metrics based on the patch-mosaic model (Frazier 2016, Kedron et al. 2018). 
In this "gradient surface" paradigm, ecological attributes are conceptualized
as 3D surfaces. 

My goal here was to review some of the most recent literature in gradient surface
modeling in landscape ecology to understand what variables are emerging as useful,
and to calculate and visualize these metrics in a way that helps to understand if
they may be meaningful for our analyses. Of particular interest were variables
that capture some aspect of landscape edges, as well as patch connectivity or
isolation. I initially selected variables based on these interests and on the findings
by McGarigal et al. (2009) and Kedron et al. (2018) on how surface metrics relate
to the more familiar patch-based metrics. I narrowed the list to two variables
of possible relevance, which I present here. Other metrics could certainly be
relevant, but were excluded for various reasons. The two metrics presented here
were among the easier to comprehend. For example, the "core fluid retention index"
was very similar to one of the chosen variables in terms of how it relates to 
patch-based metrics, but was a heavier lift in terms of comprehension, so "surface
skewness" was used instead.


```r
#Load libraries
library(raster)
library(rgdal)
library(rasterVis)
library(geodiv)
library(rgeos)
library(rgl)
library(RColorBrewer)
```

## Load and Prepare % Live Forest Data
Let's load the % live forest data, reproject it to an equal area projection, 
crop it to reduce the amount of data we're working with, and visualize it. 
```r
#Set working directory
setwd("C:/Users/ericm/Documents/ForGithub/GradientSurfaceMetrics")

#Load 2018 % Live Forest Cover Raster Data
fcov <- raster("RasterData/LiveForestFractionalCover.tif")

#Reproject to equal area projection (a requirement of the metric calculation function)
ref <- "+proj=aea +lat_1=34 +lat_2=40.5 +lat_0=0 +lon_0=-120 +x_0=0 +y_0=-4000000 +ellps=GRS80 +datum=NAD83 +units=m +no_defs "
fcov <- projectRaster(fcov, res = 30, crs = ref, method = "bilinear")

#Load extent shapefile to crop layer. Just to reduce the area we're working with.
ext <- readOGR("TestExtent/TestExtent.shp")
ext <- spTransform(ext, ref)
#crop the raster
fcov <- crop(fcov, ext)
#visualize
forestCols <- colorRampPalette(c('lightyellow1', 'darkgreen'))(100)
forestTheme <- rasterVis::rasterTheme(region = forestCols)
rasterVis::levelplot(fcov, margin = F, par.settings = forestTheme,
                     ylab = NULL, xlab = NULL,
                     main = '% Live Forest')
```
![ForestCover](https://github.com/ericlmcgregor/GradientSurfaceMetrics/blob/main/thumbnails/ForestCover.JPG)

## A note on visualizations: 
Because surface metrics use a 3-Dimensional model of the variable of interest 
(live forest in this case), visualizations will be in 3D to help us understand how 
the landscape is being captured. 

Let's make an interactive visualization of the entire area of interest in 3D. You
should be able to click and hold down to manipulate the angle. You'll notice the z axis (height) 
represents % forest cover, x and y are spatial coordinates. Red to Green represents
low to high % forest cover values.

```r
open3d()
breaks <- c(0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100)
plot3D(fcov, lit = FALSE, specular = "black", shininess = 5, at = breaks,
       col=colorRampPalette(brewer.pal(7, "RdYlGn")))
decorate3d(main = "%Live Forest", axes = T, box = T, xlab = "", ylab = "", zlab="")
#To make interactive plots in RMarkdown
rglwidget()
```
![ForestCover3D](https://github.com/ericlmcgregor/GradientSurfaceMetrics/blob/main/thumbnails/ForestCover3D.JPG)

## Define a function to calculate and plot surface metrics
The function will:
Calculate a given surface metric within a given moving window size. Then, place
the values of the new metric layer into bins (defined by nclasses) and take a 
stratified sample (nsamples). We will extract the surface metric values and visualize
with a 3D model of live forest. The purpose is to be able to see how surface metric
values across strata compare with the specific landscapes for which they are calculated. 

```r
calcMet <- function(raster, metric, window, nclasses, nsamples){
  #remove trend in data
  rt <- remove_plane(fcov)
  #wrangle window into awkward format for texture_image
  size <- (window-1)/2
  mrast <- texture_image(rt, window_type = 'square', size = size, in_meters = F,
                          metric = metric, parallel = T, ncores = 6)
  #epsg_proj = 3310
  #calculate % forest cover mean within window
  avg <- focal(raster, w = matrix(1, window, window), fun=mean)
  #mask metric values with mean moving window output to get rid of edge issues.
  mrast <- mask(mrast, avg)
  #plot metric raster
  # plot(mrast)
  #write raster output
  # writeRaster(mrast, paste("MetricOutputRasters/", metric, "_", window, ".tif", sep = ""))
  
  #Now, let's select some random pixels in different value ranges
  #First, we'll stratify the new layer into equal bins. Or if "sdr" use bins 
  #defined in GIS after initial run. 
  if (metric == "sdr"){
    b = c(0.7, 2.25, 3.8, 5.5, 10)
    c <- raster::cut(mrast, breaks = b)
  } else {
    c <- raster::cut(mrast, nclasses)
  }
  #Let's take samples from each stratum
  set.seed(17)
  samp <- sampleStratified(c, size = nsamples, na.rm = T, xy = T, sp = T)
  #Add the stratum info
  samp <- extract(c, samp, sp = T)
  samp <- extract(mrast, samp, sp = T)
  samp <- extract(avg, samp, sp = T)
  #The extraction is consistently adding an extra column for some reason. So, 
  #remove that. And rename fields.
  samp <- samp[,c(1:4, 6:7)]
  names(samp) <- c("cell", "x", "y", "strata", "metric", "meanForest")
  samp$metric <- round(samp$metric, digits = 2)
  samp$meanForest <- round(samp$meanForest, digits = 2)
  
  ##Buffer the sample locations to create an area the same as moving window size.
  #Get window size
  w <- (window*res(raster)[1])/2
  #Apply buffer
  buffs <- gBuffer(samp, byid = TRUE, width = w, quadsegs = 1, capStyle = "SQUARE" )

  ##Now, let's crop our original forest cover raster with the buffer.
  #Get strata identifier
  strata <- unique(buffs$strata)
  #The proj4string items are in different order, so let's make them match
  crs(raster) <- crs(buffs)
  #create an empty list to fill with cropped rasters
  rList <- list()
  #crop % forest cover raster with each sample location buffer and add to list.
  for (i in 1:nrow(buffs)){
    rList[[i]] <- crop(raster, buffs[i,])
  }
  
  ###Make 3D plots###
  #Define breaks for plotting % forest cover
  breaks <- c(0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100)
  #Initiate 3D plot
  open3d()
  #Define a matrix to layout multipanel plot
  n <- nclasses*nsamples
  m <- matrix(1:n, nrow=nclasses, ncol = nsamples, byrow = T)
  #Layout plot
  layout3d(mat = m, widths = rep.int(15, ncol(m)),
           heights = rep.int(15, nrow(m)), sharedMouse = T)
  #Loop through the list of rasters and plot each
  #Each row in the plot panel represents a bin of metric values
  for (i in 1:length(rList)){
    next3d()   
    plot3D(rList[[i]], lit = FALSE, specular = "black", shininess = 5, at = breaks,
           col=colorRampPalette(brewer.pal(7, "RdYlGn")))
    decorate3d(main = "%Live Forest", sub = paste(metric, "=", buffs[i,]$metric, " ; ", 
                                                 "%For = ", buffs[i,]$meanForest, sep=" "),
               axes = F, box = F, xlab = "", ylab = "", zlab="")
  }
}


```

## Calculate Metrics

A note on interpreting these visualizations. 
Each row on a plot represents a bin. In this case, I am dividing the data
into 5 equal sized bins, and extracting 3 random samples from each bin, so there are 5 rows 
and 3 columns on the plot graphic. This is to get an idea of how the range of metric
values capture % forest cover characteristics.
Moving from the top row to the bottom row you move from low to high surface metric values. 
Note that plots on each row are not necessarily in order based on their metric value, they're just
randomly chosen from within the bin.
Each image is a 3D representation of % Live Forest with its corresponding surface
metric value displayed below, as well as the mean % forest cover for that plot. 
These plots are interactive, so you can click and manipulate the angle. Red represents 
no/low forest cover values and green represents higher forest cover values. Height represents higher forest cover values. 
Axes and legends were removed from plots because they obscured the images at this scale. 


## Surface Skewness
The first metric we'll look at is surface skewness (ssk).
Ssk quantifies the asymmetry of the canopy cover distribution,
which provides insight into the percentages of live forest most prevalent in the landscape.
Sensitive to variability in the overall height distribution, but not spatial arrangement.
Measures aspects of composition, but not configuration. Measures the 'shape' of the surface
height distribution and is not affected by the surface roughness per se.
A surface with an equal number of peaks and valleys will have a surface skewness of zero.
If the bulk of the surface is above the mean surface height, values will be negatively skewed.
If the bulk of the surface is below the mean surface height, values will be positively skewed.
This is a measure of landscape dominance similar to patch-based evenness metrics, and may
also indicate patch isolation.
Kedron et al. (2018) make correlations between common patch-based metrics and surface metrics.
Strong negative correlation (-0.81) with AREA_MN (Mean Patch Area) patch-based metric.
Strong positive relationship (0.76) with ENN_AM (Area-weighted mean Euclidean nearest neighbor).
Strong positive correlation (0.76) with PARA_AM (Area-weighted mean perimeter-area ratio).
See Kedron et al. (2018) for correlations with other patch-based metrics like PLAND,
DIVISION, PLADJ, CONTIG_AM. Also, FRAGSTATS guide 4.2 suggests ssk is closely related
to Simpson's evenness index and largest patch index. 

"Increases in the number of pixels in a landscape with low percent canopy cover
would correspond to increasing Ssk... At the same time, landscapes with more
low percent canopy cover pixels will have less forested landscape area, and as those
areas split apart greater distances between neighboring forest patches. The splitting
of the landscape would increase the perimeter of forest patches while reducing their
area, an observation supported by the positive correlation [with]the shape metric
PARA_AM (Area-weighted perimeter area ratio)." (Kedron & Frazier 2019)

### Ssk - 330 meter scale
```r
#Let's start with an 11x11 moving window (330m on a side).
#Each plot image is 330 X 330 meters.
ssk330 <- calcMet(raster = fcov, metric = "ssk", window = 11, nclasses = 5,
               nsamples = 3)
rglwidget()

```
![SSK](https://github.com/ericlmcgregor/GradientSurfaceMetrics/blob/main/thumbnails/SSK.JPG)

### The remainder of the metrics were not calculated and visualized for the purposes of this markdown document. 
### Ssk - 1050 meter scale
```r
#Let's also look at 1050 meter scale, a 35 x 35 pixel window
#Each plot is 1050 x 1050 meters. 
ssk1050 <- calcMet(raster = fcov, metric = "ssk", window = 35, nclasses = 5,
                  nsamples = 3)
rglwidget()
```


## Root Mean Square Slope
The RMS value of the surface slope within the sampling area. Variance in the local slope
across the surface. According to McGarigal et al. (2009) this is analogous to edge density
and contrast metrics (e.g., contrast-weighted edge density, total edge contrast index) in
the patch-based paradigm, whereby greater local slope variation equates to greater
density and contrast of edges. Appears to have "the greatest overall analogy to the patch-based
measures of spatial heterogeneity and overall patchiness" (FRAGSTATS guide 4.2)
Sensitive to the overall height distribution as well as the spatial arrangement, 
location or distribution of surface peaks and valleys (configuration) (McGarigal et al. 2009).
Values increase as local slope variability increases. Kedron et al. (2018) found
that Sdq correlated with edge density (0.58) and landscape shape index (0.41).

### Sdq - 330 meter scale
```r
#330 meter (11 x 11 pixel) moving window
sdq <- calcMet(raster = fcov, metric = "sdq", window = 11, nclasses = 5,
               nsamples = 3)
rglwidget()
```

### Sdq - 1050 meter scale
```r
#1050 meter (35 x 35 pixel) moving window
sdq <- calcMet(raster = fcov, metric = "sdq", window = 35, nclasses = 5,
               nsamples = 3)
rglwidget()
```

## General Thoughts and Comments
As expected, ssk appears to be closely related to mean % live forest which is 
similar to the patch metric PLAND (percentage of landscape), a basic measure of
composition. In general, at both scales, lower ssk values equate to landscapes that
are more blanketed with live forest. And as ssk values rise we see more open areas 
on the landscape. This is particularly true at the 330m scale, since areas that are
nearly fully open exist on the landscape at that scale, but not as much at the 1050m scale. 
The relation to PARA_AM makes sense when looking at the visualizations. 
As shape complexity increases (more variability in peaks and valleys) 
we see an increase in ssk values, this is particularly noticeable at the 1050m scale. I also
see the relation to ENN_AM, as there is more distance between forested peaks as ssk
values rise (at the 1050m scale). The relationship to PARA_AM and ENN_AM seems more
complicated at the 330m scale, because there are areas on the landscape at that scale
where forest cover is almost completely absent. 

It does appear that sdq is related to edge density and contrast. In general, it appears
that samples with more peaks and valleys have higher sdq values. It also seems that
sdq increase with the magnitude of difference between peaks and valleys (steeper slopes). 
For example, at the 1050m scale, compare the top left sdq image with any of the images
on the bottom row. The top left image has peaks, but they have lower live forest values, and 
thus the slopes from peaks to valleys are gentler. Whereas the images on the bottom row
have mostly high forest cover values with deeper valleys (i.e., more contrast). 
Also note how the interplay of slope steepness and and different % forest values can 
influence the sdq value. For example, at the 330m scale, look at the second row 
from the top. The first two images from the left have very similar sdq values (13.2 & 13.09),
but very different mean % forest cover values (66.15% & 18.7%). The left-most image
has a low sdq value because it has mostly high forest cover with some moderate cover
values (i.e., mostly forested with low contrast edges). The image to it's right
has similar sdq values but is characterized by low forest cover values to even lower
forest cover values. In both cases the slope gradients are gentle. By contrast, the 
samples with high sdq values should have some combination of an increased number 
of peaks and steeper slopes between peaks. That is, high density of high forest cover 
values with really low forest cover values in between. 





#### Literature Cited

Frazier, A.E. Surface metrics: scaling relationships and downscaling behavior. Landscape Ecol 31, 351???363 (2016). https://doi.org/10.1007/s10980-015-0248-7

Kedron P.J., Frazier A.E. (2019) Gradient Analysis and Surface Metrics for Landscape Ecology. In: Mueller L., Eulenstein F. (eds) Current Trends in Landscape Research. Innovations in Landscape Research. Springer, Cham. https://doi.org/10.1007/978-3-030-30069-2_22

Kedron, P.J., Frazier, A.E., Ovando-Montejo, G.A. et al. Surface metrics for landscape ecology: a comparison of landscape models across ecoregions and scales. Landscape Ecol 33, 1489???1504 (2018). https://doi.org/10.1007/s10980-018-0685-1

McGarigal, K., Tagil, S. & Cushman, S.A. Surface metrics: an alternative to patch metrics for the quantification of landscape structure. Landscape Ecol 24, 433???450 (2009). https://doi.org/10.1007/s10980-009-9327-y

McGarigal, Kevin; Cushman, Samuel A. 2005. The gradient concept of landscape structure [Chapter 12]. In: Wiens, John A.; Moss, Michael R., eds. Issues and Perspectives in Landscape Ecology. Cambridge University Press. p. 112-119.

McGarigal, K., SA Cushman, and E Ene. 2012. FRAGSTATS v4: Spatial Pattern Analysis Program for Categorical and Continuous Maps. Computer software program produced by the authors at the University of Massachusetts, Amherst. Available at the following web site: http://www.umass.edu/landeco/research/fragstats/fragstats.html



