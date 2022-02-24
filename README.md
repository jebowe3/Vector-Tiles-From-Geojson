# Vector-Tiles-From-Geojson
The following is a step-by-step process for creating pbf vector tiles from geojson files using [tippecanoe](https://github.com/mapbox/tippecanoe) and [mbutil](https://github.com/mapbox/mbutil).

First, tippecanoe is not supported on Windows, so you will need a unix environment like MacOS or Linux.

Download this repository to your desktop. Notice that there is a data folder in this repository that includes two geojson files, one of Illinois counties and one for Indiana counties. Using tippecanoe, you can preserve these separate layers in your vector mbtiles output file. On MacOS, open Terminal, cd to the data folder, and input the following command (provided you have installed tippecanoe):

```bash
tippecanoe -zg -o counties-separate.mbtiles --coalesce-densest-as-needed --extend-zooms-if-still-dropping tl_2010_17_county10.geojson tl_2010_18_county10.geojson
```

You should now see a file named "counties-separate.mbtiles" in your data folder. From this file, you can create a directory of vector tiles in pbf format with mbutil (you will need to install this too). Return to Terminal and input the following:

```bash
mb-util --image_format=pbf counties-separate.mbtiles counties
```

With this, you should see a new directory called "counties" in your data folder. Inside this, you should notice subdirectories filled with pbf tiles for each zoom level. Next, you need to run gzip on the pbf files to unzip them for serving. Return to Terminal and type and enter the following:

```bash
cd counties
gzip -d -r -S .pbf *
find . -type f -exec mv '{}' '{}'.pbf \;
```
