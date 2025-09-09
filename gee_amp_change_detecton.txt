// example poi and aoi - I usually just draw on the map and import the geometries
// poi is the specific point you care about - important for not dealing with overlapping S1 frames
// aoi is the region you want to download data of (bounding box for downloads)

var aoi = ee.Geometry.Polygon(
  [[[-149.89456453298632, 59.936892432142464],
    [-149.89456453298632, 59.84906210109854],
    [-149.67483797048632, 59.84906210109854],
    [-149.67483797048632, 59.936892432142464]]]);

// Define the Point of Interest (POI) as a point
var poi = ee.Geometry.Point([-149.82313088460864, 59.902465333665035]);

// Define the time period of interest - give about a two week range for before the event and after
var startDate1 = '2024-07-20';  
var endDate1 = '2024-08-06';
var startDate2 = '2024-08-08';  
var endDate2 = '2024-09-01';    

var look_dir = 'DESCENDING' // look direction can be 'DESCENDING' or 'ASCENDING' and it can be helpful to look at both

// Load the Sentinel-1 ImageCollection - for both VV and VH polarizations
var sentinel1VVstart = ee.ImageCollection('COPERNICUS/S1_GRD')
                     .filterBounds(poi) // Filter by POI
                     .filterDate(startDate1, endDate1) // Filter by date
                     .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
                     .filter(ee.Filter.eq('instrumentMode', 'IW'))
                    .filter(ee.Filter.eq('orbitProperties_pass', look_dir));
var sentinel1VVend = ee.ImageCollection('COPERNICUS/S1_GRD')
                     .filterBounds(poi) // Filter by POI
                     .filterDate(startDate2, endDate2) // Filter by date
                     .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
                     .filter(ee.Filter.eq('instrumentMode', 'IW'))
                     .filter(ee.Filter.eq('orbitProperties_pass', look_dir));
var sentinel1VHstart = ee.ImageCollection('COPERNICUS/S1_GRD')
                     .filterBounds(poi) // Filter by POI
                     .filterDate(startDate1, endDate1) // Filter by date
                     .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
                     .filter(ee.Filter.eq('instrumentMode', 'IW'))
                    .filter(ee.Filter.eq('orbitProperties_pass', look_dir));
var sentinel1VHend = ee.ImageCollection('COPERNICUS/S1_GRD')
                     .filterBounds(poi) // Filter by POI
                     .filterDate(startDate2, endDate2) // Filter by date
                     .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
                     .filter(ee.Filter.eq('instrumentMode', 'IW'))
                     .filter(ee.Filter.eq('orbitProperties_pass', look_dir));


// Print the filtered collection to the console - dates for VV and VH will be the same
print('Filtered Sentinel-1 Scenes Before Event:', sentinel1VVstart);
print('Filtered Sentinel-1 Scenes After Event:', sentinel1VVend);


//////////////////////////////////////////////////////////////////

// Get all of the variables I want - intensity, dB, amplitude (all different ways to represent backscatter)

// do some variable manipulation since I don't know how to do this more cleanly
var VVListstart = sentinel1VVstart.toList(sentinel1VVstart.size());
var VVListend = sentinel1VVend.toList(sentinel1VVend.size());
var VHListstart = sentinel1VHstart.toList(sentinel1VHstart.size());
var VHListend = sentinel1VHend.toList(sentinel1VHend.size());

// GEE provides backscatter in dB
var VVbeforedB = ee.Image(VVListstart.get(0)).select('VV');
var VVafterdB = ee.Image(VVListend.get(0)).select('VV');
var VHbeforedB = ee.Image(VHListstart.get(0)).select('VH');
var VHafterdB = ee.Image(VHListend.get(0)).select('VH');

// intensity (also called power; value that ASF provides) I = 10^(dB/10)
var VVbeforeI = ee.Image(10).pow(VVbeforedB.divide(10));
var VVafterI = ee.Image(10).pow(VVafterdB.divide(10));
var VHbeforeI = ee.Image(10).pow(VHbeforedB.divide(10));
var VHafterI = ee.Image(10).pow(VHafterdB.divide(10));

// amplitude = sqrt(I) - not currently used in any plotting
var VVbeforeA = VVbeforeI.sqrt();
var VVafterA = VVafterI.sqrt();
var VHbeforeA = VHbeforeI.sqrt();
var VHafterA = VHafterI.sqrt();
//////////////////////////////////////////////////////////////////



//////////////////////////////////////////////////////////////////
// Now calculate the metrics we'll use to look at to detect change

// log difference of intensity - negative values mean amplitude decreased; positive mean amplitude increased
// log_diff = log10(avg_after/avg_before)
// mean of VV and VH before and after
var before = VVbeforeI.add(VHbeforeI).divide(2); 
var after = VVafterI.add(VHafterI).divide(2);
var log_diff_ASF = after.divide(before).log10();

// Log ratio intensity as defined by Jung & Yun (2020)
var VVlog_ratio_I= VVbeforeI.divide(VVafterI).log10().multiply(10);
var VHlog_ratio_I= VHbeforeI.divide(VHafterI).log10().multiply(10);

// Ratio dB
var VVratio_dB = VVbeforedB.divide(VVafterdB).abs();
var VHratio_dB = VHbeforedB.divide(VHafterdB).abs();

// difference
var VVdiff_dB = VVafterdB.subtract(VVbeforedB);
var VHdiff_dB = VHafterdB.subtract(VHbeforedB);

// RGB image where R = afterdB, G = beforedB, B = ratio_dB as developed by Gosling et al. (2025)
var VVrgbComposite = ee.Image.rgb(
  VVafterdB.toFloat(),
  VVbeforedB.toFloat(),
  VVratio_dB.toFloat()
);
var VHrgbComposite = ee.Image.rgb(
  VHafterdB.toFloat(),
  VHbeforedB.toFloat(),
  VHratio_dB.toFloat()
);

// Intensity correlation //
var VVcomposite = VVbeforedB.addBands(VVafterdB);
var VHcomposite = VHbeforedB.addBands(VHafterdB);

// Define the neighborhood (kernel)
var kernelSize = 50; // Size of the moving window size in meters
var kernel = ee.Kernel.circle(kernelSize, 'meters');

// Calculate the local correlation coefficients using reduceNeighborhood
var VVcorrelationMap = VVcomposite.reduceNeighborhood({
  reducer: ee.Reducer.pearsonsCorrelation(), // Pearson correlation
  kernel: kernel,
  skipMasked: true
});
var VHcorrelationMap = VHcomposite.reduceNeighborhood({
  reducer: ee.Reducer.pearsonsCorrelation(), // Pearson correlation
  kernel: kernel,
  skipMasked: true
});
//////////////////////////////////////////////////////////////////


//////////////////////////////////////////////////////////////////
// Put everything on the map - comment out what you don't want
// A lot of the color maps are not defined and you'll probably want to adjust them manually on the map to look better
var colorPalette = ['blue', 'white', 'red']; // A simple diverging palette

Map.addLayer(VHbeforedB, '', 'VH Before dB')
Map.addLayer(VHafterdB, '', 'VH After dB')
Map.addLayer(VVbeforedB, '', 'VV Before dB')
Map.addLayer(VVafterdB, '', 'VV After dB')
Map.addLayer(VVcorrelationMap.select('correlation'), {min: -1, max: 1, palette: ['blue', 'white', 'red']}, 'VV Local Correlation Coefficient');
Map.addLayer(VHcorrelationMap.select('correlation'), {min: -1, max: 1, palette: ['blue', 'white', 'red']}, 'VH Local Correlation Coefficient');
Map.addLayer(log_diff_ASF, '', 'Log Difference of intensity (VV+VH)'); // Adjust visualization parameters as needed
Map.addLayer(VVlog_ratio_I, '', 'VV Log Ratio intensity')
Map.addLayer(VHlog_ratio_I, '', 'VH Log Ratio intensity')
Map.addLayer(VVratio_dB, '', 'RatioVV dB');
Map.addLayer(VHratio_dB, '', 'RatioVH dB');
Map.addLayer(VVdiff_dB, '', 'DiffVV dB');
Map.addLayer(VHdiff_dB, '', 'DiffVH dB');
Map.addLayer(VVrgbComposite, {}, 'VV RGB Composite dB'); // Adding RGB composite layer
Map.addLayer(VHrgbComposite, {}, 'VH RGB Composite dB'); // Adding RGB composite layer

Map.centerObject(poi, 13); // Center the map on the POI

//////////////////////////////////////////////////////////////////


//////////////////////////////////////////////////////////////////
// save files that we want to save

// need to have the right projection for the area
var projection = VVratio_dB.select('VV').projection().getInfo();
print(projection);

// If you want to export the RGB image, you need to play with the min/max/gamma values
var VVrgbExport = VVrgbComposite.visualize({bands: ['vis-red', 'vis-green', 'vis-blue'], min: -27, max: -2, gamma: 1.2});
var VHrgbExport = VHrgbComposite.visualize({bands: ['vis-red', 'vis-green', 'vis-blue'], min: -27, max: -2, gamma: 1.2});

// Export the image, specifying the CRS, transform, and region.
Export.image.toDrive({
  image: log_diff_ASF,
  description: 'log_diff_ASF',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VVlog_ratio_I,
  description: 'VVlog_ratio_I',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VVratio_dB,
  description: 'VVratio_dB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VVdiff_dB,
  description: 'VVdiff_dB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VVrgbExport,
  description: 'VVrgbComposite',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VVbeforedB,
  description: 'VVbeforedB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VVafterdB,
  description: 'VVafterdB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});


Export.image.toDrive({
  image: VHlog_ratio_I,
  description: 'VHlog_ratio_I',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VHratio_dB,
  description: 'VHratio_dB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VHdiff_dB,
  description: 'VHdiff_dB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VHrgbExport,
  description: 'VHrgbComposite',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VHbeforedB,
  description: 'VHbeforedB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});

Export.image.toDrive({
  image: VHafterdB,
  description: 'VHafterdB',
  crs: projection.crs,
  crsTransform: projection.transform,
  region: aoi
});
