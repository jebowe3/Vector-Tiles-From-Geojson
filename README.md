# Vector-Tiles-From-Geojson
The following is a step-by-step process for creating pbf vector tiles from geojson files using [tippecanoe](https://github.com/mapbox/tippecanoe) and mbutil[https://github.com/mapbox/mbutil].

First, tippecanoe is not supported on Windows, so you will need a unix environment like MacOS or Linux.

Download this repository to your desktop. Notice that there is a data folder in this repository that includes two geojson files, one of Illinois counties and one for Indiana counties. Using tippecanoe, you can preserve these separate layers in your vector mbtiles output file. On MacOS, open Terminal, cd to the data folder, and input the following command (provided you have downloaded tippecanoe):

```bash
tippecanoe -zg -o counties-separate.mbtiles --coalesce-densest-as-needed --extend-zooms-if-still-dropping tl_2010_17_county10.geojson tl_2010_18_county10.geojson
```
