Extract NBR index using GEE (SLC issue resolved)
---

In this code, we calculated the Normalized Burn Ratio (NBR) over a span of 20 years using Landsat-5, Landsat-7, and Landsat-8. Additionally, we addressed the Scan Line Corrector (SLC) issue and resolved it using the `focal_mean` algorithm.

---

### **Centering the Map**

```javascript
Map.centerObject(geometry);
```
Centers the map view in the Google Earth Engine (GEE) interface based on a specific `geometry` variable, which defines the area of interest. 
- **`geometry`**:  It represents the geographic region the code is applied to.

---

### **Function to Calculate Normalized Burn Ratio (NBR)**

```javascript
function calculateNBR(image) {
```
Defines a function called `calculateNBR`. It calculates the Normalized Burn Ratio (NBR) for a given satellite image. 
- NBR is often used for analyzing post-fire burn severity.

```javascript
  var nbr;
```
Declares a variable, `nbr`, to hold the computed NBR value.

```javascript
  if (image.bandNames().contains('SR_B5')) {
```
Checks if the image contains the band `SR_B5`. Landsat 8 images include `SR_B5`, so this condition is used to distinguish Landsat 8 images.

```javascript
    nbr = image.normalizedDifference(['SR_B5', 'SR_B7']).rename('NBR');
```
For Landsat 8:
- Computes the normalized difference index using the bands `SR_B5` (Near-Infrared) and `SR_B7` (Shortwave Infrared).
- Renames the result to `NBR`.

```javascript
  } else {
    nbr = image.normalizedDifference(['SR_B4', 'SR_B7']).rename('NBR');
  }
```
For Landsat 5 and 7:
- Computes NBR using `SR_B4` (Near-Infrared) and `SR_B7` (Shortwave Infrared). 
- Renames the output band to `NBR`.

```javascript
  return nbr;
```
Returns the calculated NBR.

---

### **Function to Fill Gaps in Landsat 7 Images**

```javascript
function fillGapsL7(image) {
```
Defines a function to fill image gaps in Landsat 7 caused by the **SLC-off error** (scan line corrector malfunction).

```javascript
  var filled = image.focal_mean(1, 'square', 'pixels', 4)
    .blend(image);
```
- Uses a **focal mean filter** (radius of 1 pixel, square kernel, sampling 4 pixels) to calculate the average values around missing pixels.
- Blends the result with the original image to fill the gaps.

```javascript
  return filled.copyProperties(image, image.propertyNames());
```
Returns the gap-filled image while retaining all the metadata/properties of the original.

---

### **Landsat 5 Image Collection**

```javascript
var landsat5 = ee.ImageCollection("LANDSAT/LT05/C02/T1_L2")
```
Accesses the Landsat 5 collection (`C02/T1_L2` refers to Collection 2 Level 2 images).

```javascript
  .filterBounds(geometry)
```
Filters the collection to include only images within the geographic boundary specified by `geometry`.

```javascript
  .filterDate('2003-04-01', '2011-09-22')
```
Filters the dates to select Landsat 5 images between April 1, 2003, and September 22, 2011 (before Landsat 7's SLC-off period).

```javascript
  .filter(ee.Filter.equals('WRS_PATH', 168))
  .filter(ee.Filter.equals('WRS_ROW', 35))
```
Filters for images located at a specific **WRS path/row** grid (Path 168, Row 35).

```javascript
  .filter(ee.Filter.calendarRange(3, 10, 'month'))
```
Filters images acquired between March and October, which limits the data to warmer months (growing season/vegetation analysis).

```javascript
  .filter(ee.Filter.lessThan('CLOUD_COVER', 10));
```
Filters out images where cloud cover exceeds 10%.

---

### **Reproject Landsat 5 Images**

```javascript
var reprojectedLandsat5 = landsat5.map(function(image) {
  return image.reproject({
    crs: 'EPSG:4326',
    scale: 30
  });
});
```
- Maps over the Landsat 5 collection to reproject each image to the **EPSG:4326** CRS (latitude-longitude projection).
- Sets the pixel scale to 30 meters.

---

### **Landsat 7 Image Collection**

```javascript
var landsat7 = ee.ImageCollection("LANDSAT/LE07/C02/T1_L2")
```
Accesses the Landsat 7 collection.

```javascript
  .filterBounds(geometry)
  .filterDate('2011-09-23', '2013-04-07')
```
Filters Landsat 7 images from September 23, 2011, to April 7, 2013 (overlapping with Landsat 5 and 8).

```javascript
  .filter(ee.Filter.equals('WRS_PATH', 168))
  .filter(ee.Filter.equals('WRS_ROW', 35))
  .filter(ee.Filter.calendarRange(3, 10, 'month'))
  .filter(ee.Filter.lessThan('CLOUD_COVER', 10))
  .map(fillGapsL7);
```
Filters apply the same logic as Landsat 5; however, `fillGapsL7` is applied to each image to address gaps.

---

### **Reproject Landsat 7 Images**

```javascript
var reprojectedLandsat7 = landsat7.map(function(image) {
  return image.reproject({
    crs: 'EPSG:4326',
    scale: 30
  });
});
```
Reprojects Landsat 7 images like Landsat 5.

---

### **Landsat 8 Image Collection**
Similar to the steps for Landsat 5 and 7:
```javascript
var landsat8 = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
  .filterBounds(geometry)
  .filterDate('2013-04-08', '2023-12-01')
  .filter(ee.Filter.equals('WRS_PATH', 168))
  .filter(ee.Filter.equals('WRS_ROW', 35))
  .filter(ee.Filter.calendarRange(3, 10, 'month'))
  .filter(ee.Filter.lessThan('CLOUD_COVER', 10));
```
Starts date filtering from after Landsat 7's overlap period and ends in December 2023.

---

### **Reproject Landsat 8 Images**

```javascript
var reprojectedLandsat8 = landsat8.map(function(image) {
  return image.reproject({
    crs: 'EPSG:4326',
    scale: 30
  });
});
```

---

### **Calculate NBR for Each Collection**

```javascript
var nbr5 = landsat5.map(calculateNBR);
var nbr7 = landsat7.map(calculateNBR);
var nbr8 = landsat8.map(calculateNBR);
```
Applies the `calculateNBR` function to calculate NBR for each image in the Landsat 5, Landsat 7, and Landsat 8 collections.

---

### **Stacking NBR Bands**

```javascript
var stackpro5 = nbr5.toBands();
var stackpro7 = nbr7.toBands();
var stackpro8 = nbr8.toBands();

var stack = ee.Image.cat([stackpro5, stackpro7, stackpro8]);
```
- Converts NBR images from each collection into multi-band images.
- Combines all three collections into a single stacked image.

---

### **Export Stacked NBR**

```javascript
Export.image.toDrive({
  image: stack,
  description: 'combined_nbr',
  scale: 60,
  region: geometry,
  maxPixels: 1e13
});
```
- Exports the stacked NBR image to Google Drive.
- Sets a pixel resolution of 60 meters, defines the region as `geometry`, and allows exporting up to `1e13` pixels.

---

### **Extract Acquisition Dates**

```javascript
function extractDates(imageCollection) {
  return imageCollection.map(function(image) {
    return ee.Feature(null, {date: ee.Date(image.get('system:time_start'))});
  });
}
```
Creates a function to extract the acquisition dates of each image from a collection.

```javascript
var datesLandsat5 = extractDates(landsat5);
var datesLandsat7 = extractDates(landsat7);
var datesLandsat8 = extractDates(landsat8);
```
Applies the function to Landsat 5, 7, and 8 collections.

```javascript
print('Dates Landsat 5:', datesLandsat5.aggregate_array('date'));
print('Dates Landsat 7:', datesLandsat7.aggregate_array('date'));
print('Dates Landsat 8:', datesLandsat8.aggregate_array('date'));
```
Prints the extracted dates for debugging or review.

---

### **Visualizing Images**

```javascript
var sampleL7Image = nbr7.first();
Map.addLayer(sampleL7Image.clip(geometry), {min: -1, max: 1, palette: ['blue', 'red', 'green']}, 'Landsat 7 NBR Sample');
```
- Visualizes the first NBR image from Landsat 7 clipped to `geometry`.
- Uses a color scale with minimum and maximum values for NBR.

```javascript
var originalL7Image = landsat7.first();
Map.addLayer(originalL7Image.clip(geometry), {bands: ['SR_B4', 'SR_B3', 'SR_B2'], min: 0, max: 3000}, 'Original L7 Image (Before NBR)');
```
- Compares the original Landsat 7 image using RGB bands (`SR_B4`, `SR_B3`, `SR_B2`).
