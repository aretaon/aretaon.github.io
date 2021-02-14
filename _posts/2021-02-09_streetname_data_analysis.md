---
layout: post
title: Geographic distribution of vocals in German street names
date: 2021-02-09 20:00
summary: First steps in OpenStreetMaps data analysis
categories: Python "Data analysis" OpenStreetMaps
---

Inspired by the "Deutschlandkarte" series of Die Zeit newspaper [see](https://www.zeit.de/serie/deutschlandkarte) I set out to a similar endavour.
Therefore I aimed at plotting the relative amount of different vocals in German street names onto a map of Germany.
The project was split into two parts: In part I I retrieve street names of the German Landkreise and count the vocals in these names
Part II than consists of plotting these values onto a map of Germany using [plotly's Chloropleth Maps](https://plotly.com/python/mapbox-county-choropleth/) and integrating those maps into the webpage.

## Part I: Obtaining the data

Openstreetmap data sliced for different regions can conveniently be downloaded from [the geofrabrik webserver](https://download.geofabrik.de/europe.html). So I obtained germany-latest.osm.bz2, uncompressed the file (using [pbzip2](https://github.com/ruanhuabin/pbzip2) helped a lot to speed things up) and saved in locally.
To get the streets for different Landkreise I first have to get bounding polygons for these. Luckily these are available as [geoJSON](https://geojson.org/) on github and can be downloaded and inspected within Python:

{% highlight python lineanchors %}
from urllib.request import urlopen
import json
with urlopen('https://raw.githubusercontent.com/isellsoap/deutschlandGeoJSON/master/4_kreise/3_mittel.geo.json') as response:
    counties = json.load(response)

lkname = counties['features'][0]['properties']['NAME_3']
poly = counties['features'][0]['geometry']['coordinates']
{% endhighlight %}

The dictionary counties contains two keys, 'types' and 'features', of which only 'features' contains information. One such entry for the Landkreis Oldenburg looks as follows:

{% highlight python lineanchors %}
{'type': 'Feature',
 'id': 0,
 'properties': {'ID_0': 86,
  'ISO': 'DEU',
  'NAME_0': 'Germany',
  'ID_1': 9,
  'NAME_1': 'Niedersachsen',
  'ID_2': 23,
  'NAME_2': 'Weser-Ems',
  'ID_3': 244,
  'NAME_3': 'Oldenburg',
  'NL_NAME_3': None,
  'VARNAME_3': None,
  'TYPE_3': 'Landkreise',
  'ENGTYPE_3': 'Rural district'},
 'geometry': {'type': 'Polygon',
  'coordinates': [[[8.65347957611084, 53.11003112792969],
# several lines removed here
    [8.578689575195426, 53.12683868408209],
    [8.635419845581112, 53.10214996337885],
    [8.65347957611084, 53.11003112792969]]]}}
{% endhighlight %}

It also directly contains the coordinates for the polygon object we need to extract the streets from a subset of the Germany map.
Using a short loop we can generate polygon-files for each of the counties. However, there are multiple kinds of geometry objects defined in geoJSON, so here we check if it is a simple polygon (i.e. only one array of coordinates) or a MultiPolygon that is a list of polygon object as might be expected e.g. for counties with discontinuous borders (such as counties with islands).
From the [geoJSON specification}(https://tools.ietf.org/html/rfc7946#section-3.1.7) I also learned that each polygon should be a closed circle (the first coordinate is identical to the last) and should be defined in a clockwise orientation.

{% highlight python lineanchors %}
cnt = 0
for l in counties['features']:
    cnt += 1
    lkname = l['properties']['NAME_3']
    with open('polygons/{}.poly'.format(lkname), 'w') as lk:
        poly_cnt=1
        lk.write(lkname+"\n")
        if l['geometry']['type'] == 'MultiPolygon':
            for geo_lines in l['geometry']['coordinates']:
                for elem in geo_lines:
                    lk.write(str(poly_cnt)+'\n')
                    for line in elem:
                        lk.write('\t{}\t\t{}\n'.format(line[0], line[1]))
                    lk.write('END\n')
                    poly_cnt += 1
        elif l['geometry']['type'] == 'Polygon':
            geo_lines = l['geometry']['coordinates']
            for elem in geo_lines:
                lk.write(str(poly_cnt)+'\n')
                for line in elem:
                    lk.write('\t{}\t\t{}\n'.format(line[0], line[1]))
                lk.write('END\n')
                poly_cnt += 1
        else:
            raise Exception('Unknown geometry type')
            
        lk.write('END\n')
{% endhighlight %}

To extract the streetnames I used [osmosis](https://wiki.openstreetmap.org/wiki/Osmosis/Polygon_Filter_File_Format) together with the polygons. For a single polygon this resulted in the following call

{% highlight bash lineanchors %}
~/progs/osmosis-0.48.3/bin/osmosis --read-xml file=~/20210209_streetnames/germany-latest.osm.bz2 --bounding-polygon file=~/20210209_streetnames/polygons/Aachen.poly --way-key keyList=highway --way-key keyList=name --tag-filter reject-nodes --tag-filter reject-relations --write-xml aachen-names.xml
{% endhighlight %}



