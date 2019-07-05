---
layout: post
title:  "Getting elevation data from the Dutch SDI with Python"
date:   2019-07-03 18:19:00 +0200
categories: [nods, geoanalysis]
tags: [ahn2, python, wcs]
---

I am working in a project in which I am using the weather observations produced by the [WOW-NL](https://wow.knmi.nl/) network of [citizen weather stations](https://en.wikipedia.org/wiki/Citizen_Weather_Observer_Program) (CWS). The goal is to assess whether these CWS produce high-resolution weather observations that can contribute to the official workflows for weather forecast. Only for the province of Utrecht, my first pocket-size study area, observations are produced by the millions. Literally. Three years of sampling conform a set of 11.6 million observations for a study area which is, roughly, 60x50 km. 

Naturally, since the WOW-NL is a volunteered initiative, the observations require filtering and correction before being suitable to enrich any official weather forecast workflow. After applying a series of mechanistic filters (e.g. remove observation if metadata is incorrect), now comes the time of correcting some of the measurements, starting with temperature. The standard atmospheric lapse rate is the rate at which temperature changes with altitude. Hence, temperature observations need correction following this formula [(Napoly et al., 2018)](https://www.frontiersin.org/articles/10.3389/feart.2018.00118/full).

$T_{WOW'} = T_{WOW} = 0.0065 (E - mean(E))$

Where T_{WOW} is the temperature measured by the CWS, E is the elevation at the CWS' site, and mean(E) is the mean elevation on a window around the CWS. Dutch government agencies have put a lot of effort in creating a solid SDI, so that the general public can have access to official spatial data collections paid with tax money (because taxes are good). This means that in [PDOK.nl](https://www.pdok.nl/) you can find a vast collection of spatial data, and access it for free. I decided to extract my elevation data from [AHN2](https://www.pdok.nl/introductie/-/article/actueel-hoogtebestand-nederland-ahn2-), a LIDAR-based layer available at 0.5 and 5m in multiple data formats. The challenge I had today is that this layer is too big to download, and I had no practical experience on how to use Python to query a [WCS](https://en.wikipedia.org/wiki/Web_Coverage_Service) service and retrieve the elevation data I needed for 300+ CWS. Below you can find my approach. 

I installed [OWSLib](https://geopython.github.io/OWSLib/) and started exploring what was pending from my [AHN2 WCS](https://www.pdok.nl/geo-services/-/article/actueel-hoogtebestand-nederland-ahn2-#bd5244588dee076015304c452b21aa46) entry point: name of the layer, supported CRS, output formats (e.g. TIFF, NC), and the available resolutions. After tinkering around with OWSLib, PIL, geopandas (!) and [checking](https://ouranosinc.github.io/pavics-sdi/tutorials/owslib_intro.html) [several](https://publicwiki.deltares.nl/display/OET/WCS+primer) [useful](https://buildmedia.readthedocs.org/media/pdf/gsky/latest/gsky.pdf) [links](https://www.mapserver.org/ogc/wcs_server.html# http://osgeo-org.1560.x6.nabble.com/WCS-2-0-1-not-returning-same-image-as-WCS-1-0-0-td5407294.html#a5407308) [from other people](https://raw.githubusercontent.com/geopython/OWSLib/master/owslib/coverage/wcs201.py) I came up with the following code:

```python
import csv
import numpy as np
import geopandas as gpd
from PIL import Image
from owslib.wcs import WebCoverageService
import matplotlib.pyplot as plt

def mean_elev_ahn2(img_wcs):
    return np.round(np.array(Image.open(img_wcs)).mean(), decimals=1)

def visualize_map(img_wcs, ext, pop=False):
    if pop == True:
        img = Image.open(img_wcs)
        plt.imshow(np.asarray(img), extent=ext, cmap=plt.cm.gist_earth)
        plt.show()

def get_elevation(wcs, path_in, path_ou):
    data = gpd.read_file(path_in)
    header = data.columns.tolist()
    header.insert(4, "elev_ahn2")
    with open(path_ou, "w", newline="") as w:
        writer = csv.writer(w, delimiter=";")
        for index, row in data.iterrows():
            xmin, ymin, xmax, ymax = [int(item) for item in row['geometry'].bounds]
            bbox = [xmin, ymin, xmax, ymax]
            subsets = [('x', xmin, xmax), ('y', ymin, ymax)]
            img_wcs = wcs.getCoverage(identifier=[ahn5m], name=ahn5m, subsets=subsets, crs=crs, format=format,
                                      version=version, resx=resx, resy=resy)
            print("Requesting URL: ", img_wcs.geturl())
            elev_ahn2 = mean_elev_ahn2(img_wcs)
            newrow = row.tolist()
            newrow.insert(4, elev_ahn2)
            writer.writerow(newrow)
            visualize_map(img_wcs, bbox, pop=False)
            
            
################
# Main program #
################

# Input/output files
# ------------------------------------------------------------------------------------------
path_in_shp = r"./WOW/WOW_Stations/WOW_NL_stations_RDNew_500m.shp"
path_ou_csv = r"./WOWQ/data/out/WOW_stations_elevation_AHN2.csv"

# WCS parameters
# -------------------------------------------------------------------------------------------
ahn5m = "ahn2_5m"
crs = 'EPSG:28992'
xmin0, ymin0, xmax0, ymax0 = [174441, 174941, 444925, 445425]
format = "image/tiff"
version = "2.0.1"
resx, resy = [5, 5]
wcs_url = "https://geodata.nationaalgeoregister.nl/ahn2/wcs"

# Connecting to AHN2 WCS and processing data
# -------------------------------------------------------------------------------------------
wcs = WebCoverageService(wcs_url, version=version)
get_elevation(wcs, path_in_shp, path_ou_csv)
```

Note that I had prepared beforehand with QGIS a shapefile with a squared buffer around each of my CWS.

What actually took me longer was the way of sub setting the chunk of coverage that I needed. Turns out that the API between WCS 1.0 and 2.0 has changed substantially, thus it took me a while to realize how to pass the parameters for the 2.0 version, since I was mixing both of them :-) . For the rest, is basically turning the returned coverage into a PIL Image(), from here to a numpy array to compute the average elevation and save each record in a CSV file. Very neat!