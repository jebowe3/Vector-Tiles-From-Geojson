# How to Create Vector Tiles from GeoJSON Files
The following is a step-by-step process for creating pbf vector tiles you can serve all by yourself from geojson files using [tippecanoe](https://github.com/mapbox/tippecanoe) and [mbutil](https://github.com/mapbox/mbutil).

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

You can take a look at the "vector-tiles-map" subdirectory in this repository to see a sample folder structure for self-hosting pbf vector tiles. The mbtiles file is unnecessary, but is there to benchmark that step of the conversion process described above. To create a basic visualization of your vector tiles, you will need to set up an index file as follows. Notice that, while you are using mapbox-gl.js and mapbox-gl.css, you do not need a mapbox api key. This is because you are using v1.13.0, and Mapbox did not require an api key up to version 1.13 of mapbox-gl.js.

First, set up the basic framework for the tile viewer.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset='utf-8' />
    <title>MapBox GL JS Self-Served Vector Tiles</title>
    <meta name='viewport' content='initial-scale=1,maximum-scale=1,user-scalable=no' />
    <!-- link to mapbox-gl.js and mapbox-gl.css libraries here -->
    <script src='mapbox-gl.js'></script>
    <link href='mapbox-gl.css' rel='stylesheet' />
    <!-- basic css to style the viewer goes below -->
    <style>
        body {
          margin:0;
          padding:0;
        }
        #map {
          position:absolute;
          top:0;
          bottom:0;
          width:100%;
        }
    </style>  
</head>
<body>
<!-- id the map div -->
<div id='map'></div>
<script>
// javascript will go here
</script>
</body>
</html>
```

Notice that there is nothing between the html script tags. This is where you will call and customize the pbf vector tiles. The code for this is as follows.

```javascript
mapboxgl.accessToken = 'NOT-REQUIRED-WITH-YOUR-VECTOR-TILES-DATA'; // notice that this is not required

// the following style object points to the vector tiles in your "counties" directory in "sources"
const style = {
  "version": 8,
  "sources": { // the sources for your tile sets
    "composite": {
      "type": "vector",
      "tiles": [location.origin+location.pathname+"counties/{z}/{x}/{y}.pbf"]
    }
  },
  "glyphs": location.origin+location.pathname+"font/{fontstack}/{range}.pbf", // this points to the "font" directory
  // notice how you identify and style the vector tile layers in the following "layers" array
  "layers": [
    {
      "id": "background",
      "type": "background",
      "layout": {"visibility": "visible"},
      "paint": {"background-color": "rgb(239,239,239)"}
    },
    {
        "id": "tl-2010-17-county10", // this is the id you can use later to add popup content and conditional styles
        "type": "fill", // "fill" means this is a polygon -- other types exist for lines, points, etc -- see mapbox-gl.js documentation
        "source": "composite",
        "source-layer": "tl_2010_17_county10", // this is the name of the geojson layer
        "layout": {},
        "paint": {  // this is where you style the vector tile layer
            "fill-color": "hsl(137, 44%, 45%)",
            "fill-opacity": 0.65,
            "fill-outline-color": "#000000"
        }
    },
    {
        "id": "tl-2010-18-county10", // this is the id you can use later to add popup content and conditional styles
        "type": "fill",
        "source": "composite",
        "source-layer": "tl_2010_18_county10", // this is the name of the geojson layer
        "layout": {},
        "paint": { // this is where you style the vector tile layer
            "fill-color": "hsl(0, 85%, 69%)",
            "fill-outline-color": "#000000",
            "fill-opacity": 0.65
        }
    }
  ]
};

// define the map, set the center coordinates and the initial zoom, and feed it the above defined custom style
const map = new mapboxgl.Map({
  container: 'map',
  center: [-87.842, 39.8],
  zoom: 6,
  style: style
});

// Add zoom and rotation controls to the map.
map.addControl(new mapboxgl.NavigationControl());
```

Now, when you load this map, you should see the counties of Illinois and Indiana color-coded by state as vector tiles on your map. This is fine, but you may want to interact with these and retrieve the attribute data that was included with the initial geojson files. As an example, you can add popup interactivity to this map as follows. After the javascript that adds the zoom and rotation controls, include the following.

```javascript
// create a popup without adding to map
let popup = new mapboxgl.Popup({
  closeButton: false,
  closeOnClick: false
});

// when the map loads, call a function
map.on('load', function() {
  // when you move your mouse over the layer identified below, do something...
  // the formula for this is 'map.on({event}, {layer id}, function(x){})'
  map.on('mousemove', "tl-2010-17-county10", function(e) {

    // change the cursor style as a UI indicator.
    map.getCanvas().style.cursor = 'pointer';

    // define the county names
    // with "e.features[0].properties," you have access to all of the properties from the initial geojson layer
    let ILCountyNames = e.features[0].properties.NAMELSAD10 + ", Illinois";

    // get the popup defined above
    popup
      .setLngLat(e.lngLat) // the location of the cursor
      .setHTML(ILCountyNames) // feed the popup the county name and state
      .addTo(map) // add the popup to the map

  });
  // do the same thing for the other county layer
  map.on('mousemove', "tl-2010-18-county10", function(e) {

    // change the cursor style as a UI indicator.
    map.getCanvas().style.cursor = 'pointer';

    // define the county names
    // with "e.features[0].properties," you have access to all of the properties from the initial geojson layer
    let INCountyNames = e.features[0].properties.NAMELSAD10 + ", Indiana";

    // get the popup defined above
    popup
      .setLngLat(e.lngLat) // the location of the cursor
      .setHTML(INCountyNames) // feed the popup the county name and state
      .addTo(map) // add the popup to the map

  });

  // when the mouse leaves the vector...
  map.on('mouseleave', 'tl-2010-17-county10', function(e) {
    // change the cursor style back to initial setting
    map.getCanvas().style.cursor = '';
    // and remove the popup
    popup.remove();
  });

  // when the mouse leaves the vector...
  map.on('mouseleave', 'tl-2010-18-county10', function(e) {
    // change the cursor style back to initial setting
    map.getCanvas().style.cursor = '';
    // and remove the popup
    popup.remove();
  });

});
```

After saving these changes, you should be able to see a popup identifying each county as you move your cursor over the county tiles. Also notice how the cursor changes to a pointer as you move over the counties. When all is finished, your code should appear as follows.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset='utf-8' />
    <title>MapBox GL JS Self-Served Vector Tiles</title>
    <meta name='viewport' content='initial-scale=1,maximum-scale=1,user-scalable=no' />
    <!-- link to mapbox-gl.js and mapbox-gl.css libraries here -->
    <script src='mapbox-gl.js'></script>
    <link href='mapbox-gl.css' rel='stylesheet' />
    <!-- basic css to style the viewer goes below -->
    <style>
        body {
          margin:0;
          padding:0;
        }
        #map {
          position:absolute;
          top:0;
          bottom:0;
          width:100%;
        }
    </style>  
</head>
<body>
<!-- id the map div -->
<div id='map'></div>
<script>
mapboxgl.accessToken = 'NOT-REQUIRED-WITH-YOUR-VECTOR-TILES-DATA'; // notice that this is not required

// the following style object points to the vector tiles in your "counties" directory in "sources"
const style = {
  "version": 8,
  "sources": { // the sources for your tile sets
    "composite": {
      "type": "vector",
      "tiles": [location.origin+location.pathname+"counties/{z}/{x}/{y}.pbf"]
    }
  },
  "glyphs": location.origin+location.pathname+"font/{fontstack}/{range}.pbf", // this points to the "font" directory
  // notice how you identify and style the vector tile layers in the following "layers" array
  "layers": [
    {
      "id": "background",
      "type": "background",
      "layout": {"visibility": "visible"},
      "paint": {"background-color": "rgb(239,239,239)"}
    },
    {
        "id": "tl-2010-17-county10", // this is the id you can use later to add popup content and conditional styles
        "type": "fill", // "fill" means this is a polygon -- other types exist for lines, points, etc -- see mapbox-gl.js documentation
        "source": "composite",
        "source-layer": "tl_2010_17_county10", // this is the name of the geojson layer
        "layout": {},
        "paint": {  // this is where you style the vector tile layer
            "fill-color": "hsl(137, 44%, 45%)",
            "fill-opacity": 0.65,
            "fill-outline-color": "#000000"
        }
    },
    {
        "id": "tl-2010-18-county10", // this is the id you can use later to add popup content and conditional styles
        "type": "fill",
        "source": "composite",
        "source-layer": "tl_2010_18_county10", // this is the name of the geojson layer
        "layout": {},
        "paint": { // this is where you style the vector tile layer
            "fill-color": "hsl(0, 85%, 69%)",
            "fill-outline-color": "#000000",
            "fill-opacity": 0.65
        }
    }
  ]
};

// define the map, set the center coordinates and the initial zoom, and feed it the above defined custom style
const map = new mapboxgl.Map({
  container: 'map',
  center: [-87.842, 39.8],
  zoom: 6,
  style: style
});

// Add zoom and rotation controls to the map.
map.addControl(new mapboxgl.NavigationControl());

// create a popup without adding to map
let popup = new mapboxgl.Popup({
  closeButton: false,
  closeOnClick: false
});

// when the map loads, call a function
map.on('load', function() {
  // when you move your mouse over the layer identified below, do something...
  // the formula for this is 'map.on({event}, {layer id}, function(x){})'
  map.on('mousemove', "tl-2010-17-county10", function(e) {

    // change the cursor style as a UI indicator.
    map.getCanvas().style.cursor = 'pointer';

    // define the county names
    // with "e.features[0].properties," you have access to all of the properties from the initial geojson layer
    let ILCountyNames = e.features[0].properties.NAMELSAD10 + ", Illinois";

    // get the popup defined above
    popup
      .setLngLat(e.lngLat) // the location of the cursor
      .setHTML(ILCountyNames) // feed the popup the county name and state
      .addTo(map) // add the popup to the map

  });
  // do the same thing for the other county layer
  map.on('mousemove', "tl-2010-18-county10", function(e) {

    // change the cursor style as a UI indicator.
    map.getCanvas().style.cursor = 'pointer';

    // define the county names
    // with "e.features[0].properties," you have access to all of the properties from the initial geojson layer
    let INCountyNames = e.features[0].properties.NAMELSAD10 + ", Indiana";

    // get the popup defined above
    popup
      .setLngLat(e.lngLat) // the location of the cursor
      .setHTML(INCountyNames) // feed the popup the county name and state
      .addTo(map) // add the popup to the map

  });

  // when the mouse leaves the vector...
  map.on('mouseleave', 'tl-2010-17-county10', function(e) {
    // change the cursor style back to initial setting
    map.getCanvas().style.cursor = '';
    // and remove the popup
    popup.remove();
  });

  // when the mouse leaves the vector...
  map.on('mouseleave', 'tl-2010-18-county10', function(e) {
    // change the cursor style back to initial setting
    map.getCanvas().style.cursor = '';
    // and remove the popup
    popup.remove();
  });

});
</script>
</body>
</html>
```

From the instructions above, you can see how multiple vector geojson files can be combined into one pbf tile set that you can style and serve yourself. For the purposes of providing additional spatial context, I have included some vector basemap tiles from [maptiler](https://www.maptiler.com/) and customized in [maputnik](https://maputnik.github.io/) in my html file. A demo can be seen [here](https://jebowe3.github.io/Vector-Tiles-From-Geojson/vector-tiles-map/).
