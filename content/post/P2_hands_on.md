---
title: Part 2 hands on
date: 2022-04-13
hero: /images/p2f1.png
excerpt: Load, filter and visualise GEE data.
timeToRead: 4
authors:
  - Geethen Singh

---
Access the complete script for this session [here](https://code.earthengine.google.com/3d6ec3bd6c79711d142ad4c305d9571f)

![](/images/p2f1.png)
**Figure 1:** The final expected output for this session showing the transitions and persistence of land cover categories across South Africa.
***

## Learning Objectives
By the end of this practical you should be able to:
1. Understand the concept of Land Cover Change (LCC)
2. Implement a multi-class LCC 
3. Relate changes in LC to a variable (in this case, relative wealth index (rwi))
4. Download the final results to a csv

Identifying and computing changes in LCC are one of the best proxies for ecosystem pressures which ultimately influence human-wellbeing. LCC involves the quantification of transitions between land cover classes or their persistence. In this hands on session, we wil go through the implementation of identifying and quantifying the area of land cover transitions and persistence. We will thereafter look at the relationship of unique transitions/persistence and mean (sd) of Relative Wealth Index (RWI) in South Africa (unfortunately, RWI is not available for Norway).

## Searching and Importing data
To access the GEE code editor, go to https://earthengine.google.com/ >Platform> Code Editor.

Within the code editor, go to the search bar and search for LC100. This is a Global Land cover dataset at a spatial resolution of 100 metres. Refer to the Table Schema for the definitions of the mapped LC categories.

We will also search for LSIB - a dataset containing country boundaries.

![](/images/p2f2.png)
**Figure 1:** Process to upload a shapefile into GEE as a new assest imported into the script as a FeatureColection
***

Once the two datasets are imported change their names in the import section to lc and countries, respectively.

Next, add a marker on South Africa.

```js
var SA = countries.filterBounds(geometry);
Map.addLayer(SA,{}, 'South African boundary',false);
```

## Filtering
In this step we will create a two individual images that contain the LC for 2014 and 2019. These years correspond to the earliest and latest years of data available in this product. 

You will notice that we limit the data extent to the boundary of South Africa.

```js
var lc15 = lc.filterDate('2015').filterBounds(SA).select('discrete_classification').mosaic().clip(SA).aside(print);
var lc19 = lc.filterDate('2019').filterBounds(SA).select('discrete_classification').mosaic().clip(SA).aside(print);

Map.addLayer(lc15,{},'lc15', false);
Map.addLayer(lc19,{},'lc19', false);
```
## Land cover change detection
Once we have the land cover for the two epochs, we need to identify the areas that have changed or remained the same.

Since the maximum number of digits that are used to identify any unique landcover category is three. We first multiply the 104 landcover by a 10000 and then add the 2019 landcover data. This allows us to capture unique transitions between each of the different landcover categories.

We multiply the before image with 10000 and add the after image. The resulting pixel values will be unique for each type of transition i.e. 1120020 represents a change from 112 (Closed forest, evergreen broad leaf) to 020 (Shrubs).

```js
var merged = lc15.multiply(10000).add(lc19).rename('transitions');
```

## Quantify the area of each 'transition' category
To do this, we first create a area image. This is an image that contains the area of eachpixel as its value. To be able to aggregate this area image by the unique categories, we add this as a second band. Thereafter, we apply the reduceRegion function to compute the sum of area covered by each unique category.

```js
// Total area for each transition
var areaImage = ee.Image.pixelArea().addBands(merged);
var areas = areaImage.reduceRegion({
reducer: ee.Reducer.sum().group({
groupField: 1,
groupName: 'transitions',
}),
geometry: SA.geometry(),
scale: 1000,
maxPixels: 1e10
});
var classAreas = ee.List(areas.get('groups'));
var classAreaLists = classAreas.map(function(item) {
var areaDict = ee.Dictionary(item);
var classNumber = ee.Number(areaDict.get('transitions')).format();
var area = ee.Number(
areaDict.get('sum')).divide(1e6).round();
return ee.List([classNumber, area]);
});
var result = ee.Dictionary(classAreaLists.flatten());
print(result);
```

## Determine the mean RWI value for a particular LC 'transition'.
In a similar manner, we can determine what is the mean RWI per unique landcover transition. However, instead of aggregating a area image we aggregate a RWI image.

```js
var rwi = ee.FeatureCollection("projects/sat-io/open-datasets/facebook/relative_wealth_index");
var rwi_sa = rwi.filterBounds(SA).reduceToImage(['rwi'], ee.Reducer.first()).unmask();
Map.addLayer(rwi_sa,{},'South African RWI distribution',false);

// Mean RWI for each transition
var rwiImage = ee.Image(rwi_sa).addBands(merged);
var areas = rwiImage.reduceRegion({
reducer: ee.Reducer.mean().group({
groupField: 1,
groupName: 'transitions',
}),
geometry: SA.geometry(),
scale: 1000,
maxPixels: 1e10
});
var classAreas = ee.List(areas.get('groups'));

var classAreaLists = classAreas.map(function(item) {
var areaDict = ee.Dictionary(item);
var classNumber = ee.Number(areaDict.get('transitions')).format();
var mean = ee.Number(areaDict.get('mean'));
return ee.List([classNumber, mean]);
});

var result = ee.Dictionary(classAreaLists.flatten());
print(result);
```
## Export results as a csv to your Google Drive
At this point, you may want to use the results you obtained with other data you have locally through excel or R. 
```js
```