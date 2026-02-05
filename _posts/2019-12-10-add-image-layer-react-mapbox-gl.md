---

title: Add Image Layer to React Mapbox GL

category: Gist
tags: [Geek]
date: 2019-12-10
---

Images can be added to Mapbox using a `Layer` and `Source`.

## Layer Configuration
Prepare a source configuration by referencing [this document](https://docs.mapbox.com/mapbox-gl-js/style-spec/#sources-image)

```json
const Image_SOURCE_OPTIONS = {
    "type": "image",
    "url": "https://docs.mapbox.com/mapbox-gl-js/assets/radar.gif",
    "coordinates": [
        [-80.425, 46.437],
        [-71.516, 46.437],
        [-71.516, 37.936],
        [-80.425, 37.936]
    ] 
};
```
For base64 images, the URL could be `data:image/png;base64,` + base64 image data:
```json
const Image_SOURCE_OPTIONS = {
    "type": "image",
    "url": "data:image/png;base64,...",
    "coordinates": [
        [-80.425, 46.437],
        [-71.516, 46.437],
        [-71.516, 37.936],
        [-80.425, 37.936]
    ] 
};
```

# Create a Source

```html
<Source id="source_id" tileJsonSource={RASTER_SOURCE_OPTIONS} />
```

## Create a Layer
Referencing the [react-mapbox-gl document](https://github.com/alex3165/react-mapbox-gl/blob/master/docs/API.md#layer), the type can only be `symbol`, `line`, `raster`, ... 
```html
 <Layer type="raster" id="layer_id" sourceId="source_id" />
```