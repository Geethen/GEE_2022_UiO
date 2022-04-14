---
title: Part 1 hands on
date: 2022-04-13
hero: /images/p1f0.png
excerpt: Load, filter and visualise GEE data.
timeToRead: 4
authors:
  - Geethen Singh

---
Access the complete script for this session [here](https://code.earthengine.google.com/3d6ec3bd6c79711d142ad4c305d9571f)

![](/images/p1f1.png)
**Figure 1:** The final expected output for this practical showing Sentinel-2, level 2A RGB imagery clipped to a boundary.
***

## Learning Objectives
By the end of this practical you should be able to:
1. Access and navigate the GEE User Interface
2. Search and import EO data
3. Filter imported data to location and by date
4. Visualise EO data
5. Customise visualisation parameters
6. Clip data to a boundary or region of interest

## Searching and Importing
To access the GEE code editor, go to https://earthengine.google.com/ >Platform> Code Editor.

Within the code editor, go to the search bar and search for Sentinel-2.
![](/images/p1f2.png)
**Figure 1:** Process to upload a shapefile into GEE as a new assest imported into the script as a FeatureColection
***

Click on the second Sentinel-2 MSI: Multispectral Instrument, level-2A option. This brings up the metadata of the Sentinel-2 satellite mission. The metadata includes information regarding the details of the algorithms used to pre-process a dataset. It will also provide information regarding the dataset availability, spatio-temporal and spectral resolution (as covered in the earlier lecture), scaling factors that are unique to Google’s ingestion of the data and if you are dealing with a derived EO product (such as tree cover), a reference to the journal article associated with the published dataset.

To import the dataset that allows for its processing or analysis within GEE, click on the import button at the bottom of the metadata pop up block in the code editor. This will add an entry within the Imports section of your script called imagecollection – a generic name. This imagecollection contains all Sentinel-2, level-2A correctly captured images, globally for the entire period this satellite has been in operation. Change the generic imagecollection name to s22a – a more meaningful name.

Note, when changing the names of imports, try to choose short and meaningful names as you will likely be typing this name repeatedly.

## Filtering
Whilst we could process data for the entire globe by using the entire image collection, it is usually unnecessary, computationally inefficient and abusive of the free GEE resources. Therefore, you should always attempt to only select those images of interest that are relevant to answer your question. There are numerous ways to filter data to the required inputs. In this practical we will cover, how to filter data spatially (for an area of interest) and temporally (for a duration of interest).

```js
var filtered = s22a.filterBounds(geometry)
                    .filterDate('2020-06-20','2020-06-30')
                    .first();
```
Geometry, in this case refers to a point of interest. This can be added by clicking on the add a marker button followed by clicking in the map area. This will add an entry within the imports section of your script called ‘geometry’.
![](/images/p1f3.png)

We may go on to clip this selected ‘first’ image to a boundary of interest. To do this, we require a second geometry object. This can be created by clicking on the draw a rectangle button and then drawing a rectangle within the map area. Thereafter, rename this to ‘boundary’.

```js
var clipped = filtered.clip(boundary);
```

## Visualisation
At this point, although we have written code, if we click run, the code will not seem to do anything. To be able to visualise the output of the filtering and clip pre-processing steps we have carried out above, we will need to visualise the final output (‘clipped’).

```js
Map.centerObject(geometry, 12);
Map.addLayer(clipped,rgb_vp,'filtered first scene');
```
Here, we centre our map area on ‘geometry’, the point we have added earlier. Thereafter, we add the clipped layer, specifying visualisation parameters (second argument) and a layer name (third argument). At this point, if the code is executed, you will receive an error since the visualisation parameters, ‘rgb_vp’ has not yet been defined.

An easy way to set the visualisation parameters is to run the script without any visualisation parameters (replace rgb_vp with a set of empty {}). This will display a black image since the bands for visualisation and the range of band values has not yet been set. To set the visualisation parameters, hover over the Layers button and click the cog wheel beside the added layer name (‘filtered first scene’) and set the parameters according to Figure 3 (below). Lastly, click apply and then the import button. This will add an entry within the imports section called imageVisParam. Rename this to rgb_vp. The RGB bands are necessary to create a True Colour Image (TCI), that when viewed will have a natural appearance. At the same time if instead you select the NIR, green and red band in place of the RGB channels respectively, changes in health vegetation become more pronounced. This is referred to as a False Colour Image (FCI). To explore and better understand the value of FCI’s refer to this [link](http://gsp.humboldt.edu/olm/Courses/GSP_216/lessons/composites.html).

![](/images/p1f4.png)

**Figure 3:** B2-4 correspond to the Blue, Green and Red bands, respectively. The stretch: 100% is used to automatically set the range values. The range values correspond to the minimum and maximum pixel values within the image displayed i.e., the filtered first scene that was
8
clipped to the boundary. Keep in mind that the RGB band numbers may vary for different satellite platforms.
To incorporate our new specified visualisation parameters, you can then add rgb_vp into you Map.addLayer code: 

```js
Map.addLayer(clipped,rgb_vp,'filtered first scene');
```

## Practical 1 Exercise
• Repeat this practical for the Landsat-8 surface reflectance, tier-1 product.
• Repeat this practical for the SRTM elevation dataset. Hint, this is a single band dataset, adjust the visualisation parameters accordingly.