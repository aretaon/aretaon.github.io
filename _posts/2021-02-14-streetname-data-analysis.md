---
layout:     post
title:      Streetname Data Analysis Part I
date:       2021-02-14 15:02:00
summary:    Obtaining a dataset of German street names
categories: Python
---

Inspired by the ["Deutschlandkarte" series of Die Zeit newspaper](https://www.zeit.de/serie/deutschlandkarte) I set out to a similar endavour.
Therefore I aimed at plotting the relative amount of different vocals in German street names onto a map of Germany.
The project was split into two parts: In part I I retrieve street names of the German Landkreise and count the vocals in these names
Part II than consists of plotting these values onto a map of Germany using [plotly's Chloropleth Maps](https://plotly.com/python/mapbox-county-choropleth/) and integrating those maps into the webpage.

## The geometry of German counties ("Landkreise")

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

It also directly contains the coordinates for a polygon object of the district that we need to extract the streets from that subset of the Germany map.
Using a short loop we can generate polygon-files for each of the districts.
There are multiple kinds of geometry objects defined in the [geoJSON specification}(https://tools.ietf.org/html/rfc7946#section-3.1.7), so here we check if it is a simple polygon (i.e. only one array of coordinates) or a MultiPolygon that is a list of polygon object as might be expected e.g. for counties with discontinuous borders (such as counties with islands).
Each polygon should be a closed circle (the first coordinate is identical to the last) and should be defined in a clockwise orientation.

## Geting the polygons

{% highlight python lineanchors %}
osmosis_call= ['/home/aretaon/progs/osmosis-0.48.3/bin/osmosis',
               '--read-xml',
               os.path.abspath('germany-latest.osm.bz2')]

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
            
    osmosis_call.extend(['--bounding-polygon', 'file="{}/{}.poly"'.format(os.path.abspath('polygons'), lkname),
                         '--way-key', 'keyList=highway', '--way-key', 'keyList=name',
                         '--tag-filter', 'reject-nodes', '--tag-filter', 'reject-relations',
                         '--write-xml', '"{}/{}-names.xml"'.format(os.path.abspath('xml'), lkname)])
{% endhighlight %}

## Using osmosis to select the streets

In the above code example I used [osmosis](https://wiki.openstreetmap.org/wiki/Osmosis/Polygon_Filter_File_Format) together with the polygon to extract the streetnames.
For a single polygon this is achieved with the following call

{% highlight bash lineanchors %}
~/progs/osmosis-0.48.3/bin/osmosis --read-xml file=~/20210209_streetnames/germany-latest.osm.bz2 --bounding-polygon file=~/20210209_streetnames/polygons/Aachen.poly --way-key keyList=highway --way-key keyList=name --tag-filter reject-nodes --tag-filter reject-relations --write-xml aachen-names.xml
{% endhighlight %}

This command reads in the full map of Germany (*--read-xml*) and selects a susbet given by the polygon coordinates that we extracted previously form the geoJSON (*--bounding-polygon*).
The elements within the polygon are then filtered and only way-elements (*--way-key*)that contain one of the two tags: *highway* or *name*, i.e. only named roads and highways (that might also have names).
Next, tag-filters are used to remove all nodes (as they are only single points in space and therefore cannot be streets) and all relations (that is all groups of elements that match the pattern so far).
Finally, the remaining objects are written in xml-format.
For a detailed description of all arguments available to osmosis see [the osmosis docs](https://wiki.openstreetmap.org/wiki/Osmosis/Detailed_Usage_0.48#--way-key_.28--wk.29).

I originally intended to call osmosis using Python [subprocess)[https://docs.python.org/3/library/subprocess.html) library but in my case the long argument list caused errors.
Therefore I wrote an sh-file containing all arguments and called it separately from the command line.

{% highlight python lineanchors %}

with open('osmosis.sh', 'w') as of:
    for idx, el in enumerate(osmosis_call):
        if idx % 2 == 0:
            of.write(el + '\\\\\n')
        else:
            of.write(el + ' ')

# call osmosis using the shell like bash ./osmosis.sh
{% endhighlight %}

## Getting plain street names from XML

The last part of data gathering is relatively straightforward as we have all streetnames in XMl and only want a simple plain txt-file with names.
So we open each xml-file using Python's [minidom XML implementation](https://docs.python.org/3/library/xml.dom.minidom.html) and select the streetnames based on their tags.
In the xml-files, streetnames are encoded as XML-tags with the attribute *k=name* and the streetname written e.g. in the following form:
{% highlight html lineanchors %}
<tag k="name" v="Schaapkamp"/>
{% endhighlight %}

In the following loop we only iterate through the xml-elements and for every streetname found, we add it to a list of streetnames.
After removing duplicate entries using a set operation, the streetnames are written to file, one file for each county.

{% highlight python lineanchors %}
for file in os.listdir('xml'):
    streets =  []
    xml = minidom.parse('xml/'+file)
    for element in xml.getElementsByTagName("tag"):
        if element.getAttribute('k') == "name":
            streets.append(element.getAttribute('v'))

    streets = sorted(list(set(streets)))
    with open('streets/{}.txt'.format(os.path.splitext(file)[0]), 'w') as of:
        of.write('\n'.join(streets) + '\n')
{% endhighlight %}

