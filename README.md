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

Now you should have a vector tile directory that you can host yourself. So how can you use these in a web map?

You can take a look at the "vector-tiles-map" subdirectory in this repository to see a sample folder structure for self-hosting pbf vector tiles. The mbtiles file is unnecessary, but is there to benchmark that step of the conversion process described above. To create a basic visualization of your vector tiles, you will need to set up an index file as follows. Notice that, while you are using mapbox-gl.js and mapbox-gl.css, you do not need a mapbox api key.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset='utf-8' />
    <title>MapBox GL JS Offline Example</title>
    <meta name='viewport' content='initial-scale=1,maximum-scale=1,user-scalable=no' />
    <script src='mapbox-gl.js'></script>
    <link href='mapbox-gl.css' rel='stylesheet' />
    <style>
        body { margin:0; padding:0; }
        #map { position:absolute; top:0; bottom:0; width:100%; }
    </style>
</head>
<body>

<div id='map'></div>
<script>

mapboxgl.accessToken = 'NOT-REQUIRED-WITH-YOUR-VECTOR-TILES-DATA';

var style = {
  "version": 8,
  "sources": {
    "composite": {
      "type": "vector",
      "tiles": [location.origin+location.pathname+"counties/{z}/{x}/{y}.pbf"]
    }
  },
  "glyphs": location.origin+location.pathname+"font/{fontstack}/{range}.pbf",
  "layers": [
    {
      "id": "background",
      "type": "background",
      "layout": {"visibility": "visible"},
      "paint": {"background-color": "rgb(239,239,239)"}
    },
    {
        "id": "tl-2010-17-county10",
        "type": "fill",
        "source": "composite",
        "source-layer": "tl_2010_17_county10",
        "layout": {},
        "paint": {
            "fill-color": "hsl(137, 44%, 45%)",
            "fill-opacity": 0.65,
            "fill-outline-color": "#000000"
        }
    },
    {
        "id": "tl-2010-18-county10",
        "type": "fill",
        "source": "composite",
        "source-layer": "tl_2010_18_county10",
        "layout": {},
        "paint": {
            "fill-color": "hsl(0, 85%, 69%)",
            "fill-outline-color": "#000000",
            "fill-opacity": 0.65
        }
    }
  ]
};

var map = new mapboxgl.Map({
  container: 'map',
  center: [-87.842, 39.8],
  zoom: 6,
  style: style
});

map.addControl(new mapboxgl.Navigation());
</script>

</body>
</html>
```

For the purposes of providing additional spatial context, I have included some vector basemap tiles from [maptiler](https://www.maptiler.com/) and customized in [maputnik](https://maputnik.github.io/). A demo can be seen [here](https://jebowe3.github.io/Vector-Tiles-From-Geojson/vector-tiles-map/).
