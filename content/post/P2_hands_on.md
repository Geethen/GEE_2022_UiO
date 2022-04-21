---
title: Part 2 hands on
date: 2022-04-13
hero: "/images/p2f0.png"
excerpt: Perform Land cover change detection and summarise data.
timeToRead: 4
authors:
- Geethen Singh
---
Access the complete script for this session [here](https://code.earthengine.google.com/985fda552f2834b935d14393077474b5?accept_repo=users%2Fzandersamuel%2Fee101_UiO)

![](/images/p2f0.png)
**Figure 1:** The final expected output for this session showing the transitions and persistence of land cover categories across South Africa.

---

## Learning Objectives

By the end of this practical you should be able to:

1. Understand the concept of Land Cover Change (LCC)
2. Implement a multi-class LCC
3. Relate changes in LC to a variable (in this case, relative wealth index (rwi))
4. Download the final results to a csv

Identifying and computing changes in LCC are one of the best proxies for ecosystem pressures which ultimately influence human-wellbeing. LCC involves the quantification of transitions between land cover classes or their persistence. In this hands on session, we wil go through the implementation of identifying and quantifying the area of land cover transitions and persistence. We will thereafter look at the relationship of unique transitions/persistence and mean (sd) of Relative Wealth Index (RWI) in South Africa (unfortunately, RWI is not available for Norway).

## Searching and Importing data

To access the GEE code editor, go to https://earthengine.google.com/ >Platform> Code Editor.

Within the code editor, go to the search bar and search for land cover 100. This is a Global Land cover dataset at a spatial resolution of 100 metres. Refer to the Table Schema for the definitions of the mapped LC categories.

We will also search for LSIB - a dataset containing country boundaries.

![](/images/p2f2.png)

**Figure 2:** The first entry corresponds to the land cover dataset of interest for this session.

---

![](/images/p2f3.png)

**Figure 3:** The Table Schema (classification legend showing the landcover classes available and their descriptions).

---

Once the two datasets (the landcover and the country boundaries datasets) are imported change their names in the import section to lc and countries, respectively.

Next, add a marker on South Africa.

In the code chunk below, we use the added marker to select the country boundary for South Africa (SA). We assign this boundary to 'SA'. Next, we center our map to the extent of SA using Map.centerObject and then visualise the boundary.

The number 5 specififed as an argument to Map.centerObject corresponds to the zoom level. Where, a higher number corresponds to higher zoom and a lower number corresponds to a lower zoom or greater map extent.

During the visualisation step, we specify false. this indicates that the SA layer will not automatically show when we click run. This behaviour can be changed by using true or simply remove the boolean(true/false) which will then resort to default behaviour- to show the layer on Run.

```js
var SA = countries.filterBounds(geometry);
Map.centerObject(SA, 5)
Map.addLayer(SA,{}, 'South African boundary',false);
```

## Filtering

In this step we will create two individual landcover images that will contain the LC for the year 2015 and 2019. These years correspond to the earliest and latest years of data available in the land cover product.

We access a specific years landcover data from the LC100 dataset by using filterDate. Thereafter, we use filterBounds to get the landcover data for our desired AOI (South Africa). These two filtering steps correspond to the spatio-temporal filtering. Next, we select the band of data 'discrete classification'- This band of data corresponds to the most probable/dominant landcover type per 100 metre pixel for the entire extent of South Africa.

We also use the .mosaic function, this allows us to to merge individual tiles into a single image.

The clip function allows us to remove any pixels outide our polygon of interest. This is in contrast to filterBounds which retruens any images/tiles that intersect with our polygon of interest.

Lastly, we use the .aside function. This prints the details of the output to the console - A useful sanity check.

```js
var lc15 = lc.filterDate('2015').filterBounds(SA).select('discrete_classification').mosaic().clip(SA).aside(print);
var lc19 = lc.filterDate('2019').filterBounds(SA).select('discrete_classification').mosaic().clip(SA).aside(print);

Map.addLayer(lc15,{},'lc15', false);
Map.addLayer(lc19,{},'lc19', false);
```

## Land cover change detection

Once we have the land cover for the two epochs (2015 and 2019), we need to identify the areas that have changed or remained the same.

To achieve this, we need to recognise that the maximum number of digits that are used to identify any unique landcover category is three. Therefore, we first multiply the 2015 landcover by a factor of 10000 (essentially append four zeros to the landcover) and then add the 2019 landcover data (This changes the last three zeros). Overall, this allows us to capture unique transitions between each of the different landcover categories.

When we multiply the before image with 10000 and add the after image, the resulting pixel values will be unique for each type of transition i.e. 1120020 represents a change from 112 (Closed forest, evergreen broad leaf) to 020 (Shrubs). The transition between the classes are seperated by a value zero.

```js
var merged = lc15.multiply(10000).add(lc19).rename('transitions');
```

## Quantify the area of each 'transition' category

It may be useful to compute the area of each unique landcover transition/persistence.

To do this, we first create an area image. This is an image that contains the area of each pixel stored as the pixel values. For example each pixel, will be a pixel with a value of 10 000 (square kilometres - 100 metres x 100 metres). To be able to aggregate the pixels that belong to a unique category, we add a second band- the landcover band. Thereafter, we apply the reduceRegion function to compute the sum of area covered by each unique category.

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
```

To improve the readability of the results we extract each group name and coreesponding area and visualise the results as a dictionary. The dictionary has the format of
landcover transition/persistence : area (squared kilometres).

```js
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

![](/images/p2f4.png)

**Figure 4:** The global distribution of the availability of Relative Wealth Index (RWI) information.

---

In a similar manner, we can determine what is the mean RWI per unique landcover transition. However, instead of aggregating an area image we aggregate a RWI image.

```js
var rwi = ee.FeatureCollection("projects/sat-io/open-datasets/facebook/relative_wealth_index");
var rwi_sa = rwi.filterBounds(SA).reduceToImage(['rwi'], ee.Reducer.first()).unmask();
Map.addLayer(rwi_sa,{},'South African RWI distribution',false);

// Mean RWI for each transition
var rwiImage = ee.Image(rwi_sa).addBands(merged);
var means = rwiImage.reduceRegion({
reducer: ee.Reducer.mean().group({
groupField: 1,
groupName: 'transitions',
}),
geometry: SA.geometry(),
scale: 1000,
maxPixels: 1e10
});
var classMeans = ee.List(means.get('groups'));

var classMeanLists = classMeans.map(function(item) {
var meanDict = ee.Dictionary(item);
var classNumber = ee.Number(meanDict.get('transitions')).format();
var mean = ee.Number(meanDict.get('mean'));
return ee.List([classNumber, mean]);
});

var mresult = ee.Dictionary(classMeanLists.flatten());
print(mresult);
```

## Export results as a csv to your Google Drive

At this point, you may want to use the results you obtained with other data you have locally through excel or R.

```js
var combined = classMeanLists.zip(classAreaLists);
var output = ee.FeatureCollection(combined.map(function(object){
  var dict1 = ee.Dictionary(ee.List(object).get(0));//mean RWI
  var dict2 = ee.Dictionary(ee.List(object).get(1));//area sum
  return ee.Feature(null, {'class':ee.Number(dict1.keys().get(0)), 'mean_rwi': dict1.values().get(0),
  'area':dict2.values().get(0)});
}));
print(output.limit(5));

Export.table.toDrive(output);
```
