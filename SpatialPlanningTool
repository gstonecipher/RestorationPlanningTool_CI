///////////////////////////////////////////////////////////////////////////////////////////////////////////
// Author: Grace Stonecipher
// Date: 8/19/2020
// Purpose: This code creates a tool for prioritizing areas for restoration. It contains code for both front
// end(user interface) and backend (calculations). The left side of the tool allows the user to select a country
// and set criteria to calculate available area, and then to weight different factors to identify high and 
// low priority areas within their available area. The right side of the tool allows the user to enter project
// details, and then to draw a polygon around an area of interest in order to calculate project outcomes.
////////////////////////////////////////////////////////////////////////////////////////////////////////////

/////////////////////////////////// Create Initial Layout Objects /////////////////////////////////////////////////////////////////////
// Create left panel
var leftPanel = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'),
  style: {width: '325px'}
});

// Create right panel
var rightPanel = ui.Panel({
  layout: ui.Panel.Layout.flow('vertical'),
  style: {width: '340px'}
});

// Create map object
var map = ui.Map();

// set default to satellite w/ labels
map.setOptions("HYBRID");

// zoom to show whole world initially
map.setCenter(19.057812499999986,22.171837772154703, 2);

// Create the title label.
var title = ui.Label('Restoration Planning Tool', {color: "gray", fontWeight: "bold"});
title.style().set('position', 'top-center');
map.add(title);

// create the legend panel
var legend = ui.Panel({
  style: {
  position: 'bottom-left',
  }
});

// Clear area and add elements
ui.root.clear();
ui.root.insert(0, leftPanel);
ui.root.insert(1, map);
ui.root.insert(2, rightPanel);
///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


////////////////////////////////////// Country Selector Widget ////////////////////////////////////////////////////////////////////////
// This widget allows the user to select a country from a dropdown list.
// The map will then be zoomed to the country and the outline of the country is drawn.
// Canada doesn't work...

// pull in feature collection of all countries
var countries = ee.FeatureCollection("USDOS/LSIB/2017");

//pull country names from feature collection
var countryNames = countries.aggregate_array("COUNTRY_NA").sort();

// define selector widget
var countryDropdown = ui.Select({items: countryNames.getInfo(), onChange: redraw});

// add placeholder text to widget
countryDropdown.setPlaceholder("Choose a country...");

// set variables outside of function so that they can be used in multiple places
var selectedCountry = null;
var countryOfInterest;
var selectedCountry_min_dist;
var selectedCountry_max_dist;
var selectedCountry_min_cost;
var selectedCountry_max_cost;
var selectedCountry_min_rate;
var selectedCountry_max_rate;

// define redraw function; once country is selected, zoom to country & draw it
// pull min/max values for priority variables for selected country
function redraw(countryName){
  selectedCountry = ee.Feature(countries.filter(ee.Filter.eq('COUNTRY_NA', countryName)).first());
  map.centerObject(selectedCountry); // zoom to selected country
  var styling = {color: 'red', fillColor: '00000000'}; // styling for country; red outline, no fill
  map.addLayer(ee.FeatureCollection(selectedCountry).style(styling), {}, countryName); // add country to map
  countryOfInterest = countryName;
  
  // once country has been selected, pull min/max values for all variables
  // distance to forest
  var dist2forest_min_asset = ee.FeatureCollection("users/gracestonecipher/dist2forest_min");
  var dist2forest_max_asset = ee.FeatureCollection("users/gracestonecipher/dist2forest_95_perc_max");
  // pull min and max distances from the tables
  selectedCountry_min_dist = ee.Number(dist2forest_min_asset.filter(ee.Filter.eq("COUNTRY_NA", countryOfInterest)).first().get('min'));
  selectedCountry_max_dist = ee.Number(dist2forest_max_asset.filter(ee.Filter.eq("COUNTRY_NA", countryOfInterest)).first().get('p95'));
  
  // opportunity cost
  var opp_cost_minMax = ee.FeatureCollection("users/gracestonecipher/min_max_table_opp_cost");
  // pull min and max costs from the table
  selectedCountry_min_cost = opp_cost_minMax.filter(ee.Filter.eq("COUNTRY_NA", countryOfInterest)).first().get('min_cost');
  selectedCountry_max_cost = opp_cost_minMax.filter(ee.Filter.eq("COUNTRY_NA", countryOfInterest)).first().get('max_cost');
  
  // carbon sequestration - above ground
  var carbon_seq_minMax = ee.FeatureCollection("users/gracestonecipher/min_max_table_carbon_seq"); //AGB
  // pull min and max rates from the table
  selectedCountry_min_rate = carbon_seq_minMax.filter(ee.Filter.eq("COUNTRY_NA", countryOfInterest)).first().get('min_rate');
  selectedCountry_max_rate = carbon_seq_minMax.filter(ee.Filter.eq("COUNTRY_NA", countryOfInterest)).first().get('max_rate');
}

// Build country panel
var countryPanel = ui.Panel();
var countryLabel = ui.Label("Country Selection (required)", {color: "gray", fontWeight: "bold"});
countryPanel.add(countryLabel);
countryPanel.add(countryDropdown);
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


////////////////////////////// Landcover Selector Widget ////////////////////////////////////////////////////////////////////////////////
//pull in landcover asset
var lcover = ee.Image("users/geflanddegradation/toolbox_datasets/lcov_esacc_1992_2018");

// reclassify to 7 classes
var lc2018 = lcover.select('y2018').remap([10,11,12,20,30,40,50,60,61,62,70,71,72,80,81,82,90,100,110,120,121,122,130,140,150,151,152,153,160,170,180,190,200,201,202,210,220],
                                             [ 3, 3, 3, 3, 3, 3, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  1,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  4,  4,  4,  5,  6,  6,  6,  7,  6]);

// extract grassland & cropland and remap to binary
var lc2018_grassland = lc2018.updateMask(lc2018.eq(2)).remap([0,2],[0,1]);
var lc2018_cropland = lc2018.updateMask(lc2018.eq(3)).remap([0,3],[0,1]);

// add grassland and cropland to the map, start not shown
map.addLayer(lc2018_grassland, {palette:['d8d800']}, "Grassland", false);
map.addLayer(lc2018_cropland, {palette:['a50f15']}, "Cropland", false);

// determine scale of landcover layer
var lcScale = lc2018.projection().nominalScale();

// create empty images with same scale as landcover layer
var restorationLandcover = ee.Image(0).setDefaultProjection("EPSG:4326", null, lcScale);
var grasslandCriteria = ee.Image(0).setDefaultProjection("EPSG:4326", null, lcScale);
var croplandCriteria = ee.Image(0).setDefaultProjection("EPSG:4326", null, lcScale);

// create grassland checkbox, start unchecked
var checkbox_grassland = ui.Checkbox('Grassland', false);
checkbox_grassland.onChange(function(grassChecked) {
  // if box is checked, add layer to applicable area
  if (grassChecked){
    restorationLandcover = restorationLandcover.unmask(0).add(lc2018_grassland.unmask(0));
  }
  // if box is unchecked, remove layer from applicable area
  else {
    restorationLandcover = restorationLandcover.unmask(0).subtract(lc2018_grassland.unmask(0));
  }
});

// create cropland checkbox, start unchecked
var checkbox_cropland = ui.Checkbox('Cropland', false);

checkbox_cropland.onChange(function(cropChecked) {
  // if box is checked, add layer to applicable area
  if (cropChecked){
    restorationLandcover = restorationLandcover.unmask(0).add(lc2018_cropland.unmask(0));
  }
  // if box is unchecked, remove layer from applicable area
  else {
    restorationLandcover = restorationLandcover.unmask(0).subtract(lc2018_cropland.unmask(0));
  }
});

// Build landcover Panel
var landcoverPanel = ui.Panel();
var landcoverLabel = ui.Label("Areas suitable for restoration", {color: "gray", fontWeight: "bold"});
var landcoverSubtext = ui.Label("Check one or both boxes to select the type(s) of landcover that you wish to restore. Use the 'Layers' menu on the map to visualize the extent of each landcover type.", {fontSize: "10px"});
landcoverPanel.add(landcoverLabel);
landcoverPanel.add(landcoverSubtext);
landcoverPanel.add(checkbox_grassland);
landcoverPanel.add(checkbox_cropland);

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//////////////////////////////// Build Restoration Type Widget ////////////////////////////////////////////////////////////////////////
// pull landcover for 1992 and remap
var lc1992 = lcover.select('y1992').remap([10,11,12,20,30,40,50,60,61,62,70,71,72,80,81,82,90,100,110,120,121,122,130,140,150,151,152,153,160,170,180,190,200,201,202,210,220],
                                            [ 3, 3, 3, 3, 3, 3, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  1,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  4,  4,  4,  5,  6,  6,  6,  7,  6]);

// pull out historical forest layer
var historicalForest = lc1992.updateMask(lc1992.eq(1));

// remap to pull all landcover but forest
var notHistoricalForest = lc1992.remap([1,2,3,4,5,6,7],[0,1,1,1,1,1,1]);
notHistoricalForest = notHistoricalForest.updateMask(notHistoricalForest.eq(1));

// add historical forest layer
map.addLayer(historicalForest, {palette:['006d2c']}, "Historical Forest", false);

// create image of all 0s
var historicalLandcover = ee.Image(0).setDefaultProjection("EPSG:4326", null, lcScale);

// create reforestation checkbox
var checkbox_reforestation = ui.Checkbox("Reforestation", false);
checkbox_reforestation.onChange(function(reforestationChecked) {
  // if box is checked, add layer to historical landcover applicable area
  if (reforestationChecked){
    historicalLandcover = historicalLandcover.unmask(0).add(historicalForest.unmask(0));
  }
  // if box is unchecked, remove layer from historical applicable area
  else {
    historicalLandcover = historicalLandcover.unmask(0).subtract(historicalForest.unmask(0));
  }
});

// create afforestation checkbox
var checkbox_afforestation = ui.Checkbox("Afforestation", false);
checkbox_afforestation.onChange(function(afforestationChecked) {
  // if box is checked, add layer to historical landcover applicable area
  if (afforestationChecked){
    historicalLandcover = historicalLandcover.unmask(0).add(notHistoricalForest.unmask(0));
  }
  // if box is unchecked, remove layer from historical landcover applicable area
  else {
    historicalLandcover = historicalLandcover.unmask(0).subtract(notHistoricalForest.unmask(0));
  }
});

// Build Restoration Type Panel
var restorationTypePanel = ui.Panel();
var restorationTypeLabel = ui.Label("Type of restoration", {color: "gray", fontWeight: "bold"});
var restorationTypeSubtext = ui.Label("Check one or both boxes to select the type(s) of restoration you wish to do. Reforestation selects areas that were forested in 1992. Afforestation selects areas that were not forested in 1992. Use the 'Layers' menu to visualize the extent of forest in 1992.", {fontSize: "10px"});
restorationTypePanel.add(restorationTypeLabel);
restorationTypePanel.add(restorationTypeSubtext);
restorationTypePanel.add(checkbox_reforestation);
restorationTypePanel.add(checkbox_afforestation);

///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

///////////////////////////////// Build Available Area Widget //////////////////////////////////////////////////////////////////////////
// create button to show available area, based on checkbox selections
// create available area variable outside of function
var availableArea = null;

var availableAreaButton = ui.Button({
  label: 'Display Available Area',
  style: {stretch: "horizontal"},
  // build function for what will happen when the button is clicked
  onClick: function() {
    // update restoration landcover layer based on landcover checkboxes
    restorationLandcover = restorationLandcover.updateMask(restorationLandcover);
    
    // update historical landcover layers based on restoration type checkboxes
    historicalLandcover = historicalLandcover.updateMask(historicalLandcover);
    
    // create new variable to account for restoration type and landcover
    // takes area available given chosen restoration types (previously forested/previously not forested/both)
    // updates area to only include chosen current landcover (grassland/cropland/either)
    // where restoration landcover is null (0), also set historical landcover to 0
    availableArea = historicalLandcover.where(restorationLandcover.unmask(0).eq(0), 0);
    
    // update available area with mask to get rid of 0s and clip to country
    availableArea = availableArea.clip(selectedCountry).updateMask(availableArea);
    
    // set visualization parameters for available area layer
    var availableAreaViz = {min: 1, max: 1, palette: ["purple"]};
    
    // add available area to map, start shown
    map.addLayer(availableArea, availableAreaViz , "Available Area");
    
    // build color box for legend
    var availAreaColorBox = ui.Label({
      style: {
        backgroundColor: "purple",
          // Use padding to give the box height and width.
          padding: '7px',
          margin: '0 0 0px 0',
        }
    });
    
    // set available area label for legend
    var availAreaLegendLabel = ui.Label({
        value: "Available Area",
        style: {fontSize: '11px', margin: '0 0 0px 6px'}
      });
      
    // create available area legend panel, add color box and label
    var availAreaLegendPanel = ui.Panel({
        widgets: [availAreaColorBox, availAreaLegendLabel],
        layout: ui.Panel.Layout.Flow('horizontal')
      });
      
    // add avaialble area panel to legend  
    legend.add(availAreaLegendPanel);
    
    // add legend to the map
    map.add(legend);
  }
});
////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

///////////////////////////////// Build Criteria Panel //////////////////////////////////////////////////////////////////////////////////
var whiteSpace = ui.Label("");
var criteriaPanelLabel = ui.Label("1. Criteria for Restoration", {color: "black", fontWeight: "bold", fontSize: "16px"});
var criteriaPanel = ui.Panel();
criteriaPanel.add(criteriaPanelLabel);
criteriaPanel.add(countryPanel);
criteriaPanel.add(landcoverPanel);
criteriaPanel.add(restorationTypePanel);
criteriaPanel.add(availableAreaButton);
var warningMessage = ui.Label("This may take a few minutes. Make sure that available area is displayed before proceeding to the next step. More area will appear as you zoom in.", {fontSize: '10px'});
criteriaPanel.add(warningMessage);
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

// add criteria panel to left panel
leftPanel.add(criteriaPanel);

//////////////////////////////// Build Forest Distance Section ////////////////////////////////////////////////////////////////////////////////
//build slider widget
var forestDistanceSlider = ui.Slider({
  min: 0,
  max: 5,
  value: 0, // default value is 0
  step: 1,
  style: {stretch: 'horizontal', width:'300px' }
  });

// set default importance to 0
var forestDistanceImportance = 0;

// update forest distance importance whenever slider is moved
forestDistanceSlider.onChange(function(value) {
  forestDistanceImportance = value;
});

// Build labels for slider
var labelSpaces = "\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0\xa0";
var forestDistanceSliderLabel = ui.Label("Not important" + labelSpaces + labelSpaces + "Very Important", {fontSize: "10px"});

// define color palette - greens
var dist2forestPalette = ['#005a32','#238b45','#41ab5d','#74c476','#a1d99b','#c7e9c0','#edf8e9'];

// Build legend entry
var forestDistLegendLabel = ui.Label("Proximity to Forest", {fontSize: '12px', fontWeight: 'bold', padding: '5px 0px 0px 0px', margin: '0px 0px 0px 0px'});

var forestDistLegendLabela = ui.Label("close", {fontSize: '10px'});
var forestDistLegendLabelb = ui.Label("far", {fontSize: '10px'});

var gradient = ee.Image.pixelLonLat().select(0); // gradient image using longitude
var forestDistLegendImage = gradient.visualize({min: 0, max: 100, palette: dist2forestPalette}); // assign color palette to gradient

// create thumbnail from the colorized gradient image
var forestDistGradient = ui.Thumbnail({
  image: forestDistLegendImage,
  params: {bbox:'0,0,100,10', dimensions:'100x10'},
});

// build legend panel, add labels & thumbnail
// dont add overall label b/c this has horizontal flow
var forestDistLegendPanel = ui.Panel({
  widgets: [forestDistLegendLabela, forestDistGradient, forestDistLegendLabelb],
  layout: ui.Panel.Layout.Flow('horizontal'),
  style: {margin: '5 px 0px 5 px 0px'}
});

// build button to display layer
var dist2forestButton = ui.Button({
  label: 'Display Layer',
  style: {width: '85px', margin: '6px',textAlign: "center"},
  // on click, add layer to map & info to legend
  onClick: function() {
    map.addLayer(dist2forest.clip(selectedCountry), {min: selectedCountry_min_dist.getInfo(), max: selectedCountry_max_dist.getInfo(), palette: dist2forestPalette}, "Proximity to Forest");
    legend.add(forestDistLegendLabel);
    legend.add(forestDistLegendPanel);
  }
});

// Build forest Distance Panel
var forestDistancePanel = ui.Panel();
var forestDistanceTitlePanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var forestDistanceLabel = ui.Label("Proximity to existing forest", {color: "grey", fontWeight: "bold", padding: '5px 25px 5px 1px'});
forestDistanceTitlePanel.add(forestDistanceLabel);
forestDistanceTitlePanel.add(dist2forestButton);
var forestDistancesubLabel = ui.Label("How important is it that restoration occurs near existing forest?", {fontSize: "10px"});
forestDistancePanel.add(forestDistanceTitlePanel);
forestDistancePanel.add(forestDistancesubLabel);
forestDistancePanel.add(forestDistanceSliderLabel);
forestDistancePanel.add(forestDistanceSlider);
///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//////////////////////////////// Build Opportunity Cost Slider ////////////////////////////////////////////////////////////////////////////////////
var opportunityCostSlider = ui.Slider({
  min: 0,
  max: 5,
  value: 0, 
  step: 1,
  style: {stretch: 'horizontal', width:'300px' }
  });

// set default to 0
var opportunityCostImportance = 0;

// update opportunity cost important whenever slider is moved
opportunityCostSlider.onChange(function(value) {
  opportunityCostImportance = value;
});

// Build labels for slider
var opportunityCostLegend = ui.Label("Not important" + labelSpaces + labelSpaces + "Very Important", {fontSize: "10px"});

// define color palette
var oppCostPalette = ['#f2f0f7','#dadaeb','#bcbddc','#9e9ac8','#807dba','#6a51a3','#4a1486']; // purples

// Build legend entry
var oppCostLegendLabel = ui.Label("Opportunity Cost", {fontSize: '12px', fontWeight: 'bold', padding: '5px 0px 0px 0px', margin: '0px 0px 0px 0px'});

var oppCostLegendLabela = ui.Label("low", {fontSize: '10px', padding: '0px 7px 0px 2px'});
var oppCostLegendLabelb = ui.Label("high", {fontSize: '10px'});

var gradient = ee.Image.pixelLonLat().select(0); // gradient image using longitude
var oppCostLegendImage = gradient.visualize({min: 0, max: 100, palette: oppCostPalette}); // colorize gradient with palette

// create thumbnail from the colorized gradient image
var oppCostGradient = ui.Thumbnail({
  image: oppCostLegendImage,
  params: {bbox:'0,0,100,10', dimensions:'100x10'},
});

// build legend panel, add widgets
var oppCostLegendPanel = ui.Panel({
  widgets: [oppCostLegendLabela, oppCostGradient, oppCostLegendLabelb],
  layout: ui.Panel.Layout.Flow('horizontal'),
  style: {margin: '5 px 0px 5 px 0px'}
});

// Build show layer button
var oppCostButton = ui.Button({
  label: 'Display Layer',
  style: {width: '85px', margin: '6px',textAlign: "center"},
  // on click, add layer to map and entry to legend
  onClick: function() {
    map.addLayer(opp_cost.clip(selectedCountry), {min: selectedCountry_min_cost.getInfo(), max: selectedCountry_max_cost.getInfo(), palette: oppCostPalette}, "Opportunity Cost");
    legend.add(oppCostLegendLabel);
    legend.add(oppCostLegendPanel);
  }
});

// Build Opportunity Cost Panel
var opportunityCostPanel = ui.Panel();
var opportunityCostTitlePanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var opportunityCostLabel = ui.Label("Low opportunity cost", {color: "grey", fontWeight: "bold", padding: '5px 62.5px 5px 1px'});

opportunityCostTitlePanel.add(opportunityCostLabel);
opportunityCostTitlePanel.add(oppCostButton);
var opportunityCostsubLabel = ui.Label("How important is it that restoration occurs on land with low opportunity cost?", {fontSize: "10px"});
opportunityCostPanel.add(opportunityCostTitlePanel);
opportunityCostPanel.add(opportunityCostsubLabel);
opportunityCostPanel.add(opportunityCostLegend);
opportunityCostPanel.add(opportunityCostSlider);
///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//////////////////////////////// Build Carbon Sequestration Slider ////////////////////////////////////////////////////////////////////////////////
var carbonSeqSlider = ui.Slider({
  min: 0,
  max: 5,
  value: 0, 
  step: 1,
  style: {stretch: 'horizontal', width:'300px' }
  });

// set default to 0
var carbonSeqImportance = 0;

// on change, update carbon seq importance
carbonSeqSlider.onChange(function(value) {
  carbonSeqImportance = value;
});

// Build labels for slider
var carbonSeqLegend = ui.Label("Not important" + labelSpaces + labelSpaces + "Very Important", {fontSize: "10px"});

// define palette
var carbonSeqPalette = ['#feedde','#fdd0a2','#fdae6b','#fd8d3c','#f16913','#d94801','#8c2d04']; // oranges

// Build legend
var carbonSeqLegendLabel = ui.Label("Carbon Sequestration Rate", {fontSize: '12px', fontWeight: 'bold', padding: '5px 0px 0px 0px', margin: '0px 0px 0px 0px'});

var carbonSeqLegendLabela = ui.Label("low", {fontSize: '10px', padding: '0px 7px 0px 2px'});
var carbonSeqLegendLabelb = ui.Label("high", {fontSize: '10px'});

var gradient = ee.Image.pixelLonLat().select(0); // select longitude to define gradient
var carbonSeqLegendImage = gradient.visualize({min: 0, max: 100, palette: carbonSeqPalette}); // colorize gradient using palette

// create thumbnail from the colorized gradient image
var carbonSeqGradient = ui.Thumbnail({
  image: carbonSeqLegendImage,
  params: {bbox:'0,0,100,10', dimensions:'100x10'},
});

// build panel, add widgets
var carbonSeqLegendPanel = ui.Panel({
  widgets: [carbonSeqLegendLabela, carbonSeqGradient, carbonSeqLegendLabelb],
  layout: ui.Panel.Layout.Flow('horizontal'),
  style: {margin: '5 px 0px 5 px 0px'}
});

// define show layer button
var carbonSeqButton = ui.Button({
  label: 'Display Layer',
  style: {width: '85px', margin: '6px',textAlign: "center"},
  // on click, add layer to map and entry to legend
  onClick: function() {
    map.addLayer(carbon_seq_AGB.clip(selectedCountry), {min: selectedCountry_min_rate.getInfo(), max: selectedCountry_max_rate.getInfo(), palette: carbonSeqPalette}, "Carbon Sequestration Potential");
    legend.add(carbonSeqLegendLabel);
    legend.add(carbonSeqLegendPanel);
    
  }
});

// Build forest Distance Panel
var carbonSeqPanel = ui.Panel();
var carbonSeqTitlePanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var carbonSeqLabel = ui.Label("Carbon sequestration potential", {color: "grey", fontWeight: "bold", padding: '5px 2px 5px 1px'});
carbonSeqTitlePanel.add(carbonSeqLabel);
carbonSeqTitlePanel.add(carbonSeqButton);
var carbonSeqsubLabel = ui.Label("How important is it that restoration occurs on land with high carbon sequestration potential?", {fontSize: "10px"});
carbonSeqPanel.add(carbonSeqTitlePanel);
carbonSeqPanel.add(carbonSeqsubLabel);
carbonSeqPanel.add(carbonSeqLegend);
carbonSeqPanel.add(carbonSeqSlider);
///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

////////////////////////////////// Run weighted overlay////////////////////////////////////////////////////////////////////////////////////////////

var dist2forest = ee.Image("projects/ci_geospatial_assets/forest/dist_forest_2018_esa_meters");
// import opportunity cost layer & min/max table
var opp_cost = ee.Image("users/geflanddegradation/nbs/l4_b_co_opp_cost");


// import carbons sequestration layers, import min/max table for above ground
var carbon_seq_AGB = ee.Image("users/mgonzalez-roglich/GROA_ORIGINAL_AGB_CARBON");
var carbon_seq_BGB = ee.Image("users/mgonzalez-roglich/GROA_ORIGINAL_BGB_CARBON");


var runAnalysisButton = ui.Button({
  label: 'Run Analysis',
  style: {stretch: "horizontal"},
  onClick: function() {
    // Normalize and invert layers (where necessary)
    // FOREST PROXIMITY
    // use unit scale to normalize the pixel values
    var dist2forest_unitScale = dist2forest.clip(selectedCountry).unitScale(selectedCountry_min_dist, selectedCountry_max_dist);
    
    // create inverse of unit-scaled image to represent proximity to forest
    var dist2forest_unitScale_inverse = ee.Image(1).subtract(dist2forest_unitScale);
    
    // OPPORTUNITY COST
    // use unit scale to normalize the pixel values
    var opp_cost_unitScale = opp_cost.clip(selectedCountry).unitScale(selectedCountry_min_cost, selectedCountry_max_cost);
    
    // create inverse of unit-scaled image - highest opportunity cost is set to 0, lowest opportunity cost is set to 1
    var opp_cost_unitScale_inverse = ee.Image(1).subtract(opp_cost_unitScale);
    
    // CARBON SEQUESTRATION
    // use unit scale to normalize the pixel values
    var carbon_seq_unitScale = carbon_seq_AGB.clip(selectedCountry).unitScale(selectedCountry_min_rate, selectedCountry_max_rate);
    
    // WEIGHTED OVERLAY
    // weight each layer using user-input values
    var dist2forest_weighted = dist2forest_unitScale_inverse.multiply(forestDistanceImportance);
    
    var opp_cost_weighted = opp_cost_unitScale_inverse.multiply(opportunityCostImportance);
    
    var carbon_seq_weighted = carbon_seq_unitScale.multiply(carbonSeqImportance);
    
    // add weighted layers together
    var restorationPriority_unscaled = dist2forest_weighted.unmask().add(opp_cost_weighted.unmask()).add(carbon_seq_weighted.unmask());
    
    // find min and max within the available area (from part 1)
    var restorationPriority_minMax = restorationPriority_unscaled.updateMask(availableArea).reduceRegion({reducer: ee.Reducer.minMax().setOutputs(['min_value','max_value']), maxPixels: 10e10});
    
    // pull min and max values
    var restorationPriority_min = restorationPriority_minMax.get("constant_min_value");

    var restorationPriority_max = restorationPriority_minMax.get('constant_max_value');
    
    // use unit scale to normalize the weighted overlay - only look at available area from step 1
    var restorationPriority_scaled = restorationPriority_unscaled.updateMask(availableArea).unitScale(restorationPriority_min, restorationPriority_max);
    
    var restorationPriorityPalette = ['#f1eef6','#d4b9da','#c994c7','#df65b0','#e7298a','#ce1256','#91003f'];
    
    // Build legend
    var restorationPriorityLegendLabel = ui.Label("Restoration Priority", {fontSize: '12px', fontWeight: 'bold', padding: '5px 0px 0px 0px', margin: '0px 0px 0px 0px'});

    var restorationPriorityLegendLabela = ui.Label("low", {fontSize: '10px', padding: '0px 7px 0px 2px'});
    var restorationPriorityLegendLabelb = ui.Label("high", {fontSize: '10px'});

    var gradient = ee.Image.pixelLonLat().select(0);
    var restorationPriorityLegendImage = gradient.visualize({min: 0, max: 100, palette: restorationPriorityPalette});

    // create thumbnail from the image
    var restorationPriorityGradient = ui.Thumbnail({
      image: restorationPriorityLegendImage,
      params: {bbox:'0,0,100,10', dimensions:'100x10'},
    });
    

    var restorationPriorityLegendPanel = ui.Panel({
      widgets: [restorationPriorityLegendLabela, restorationPriorityGradient, restorationPriorityLegendLabelb],
      layout: ui.Panel.Layout.Flow('horizontal'),
      style: {margin: '5 px 0px 5 px 0px'}
    });
    
    legend.add(restorationPriorityLegendLabel);
    legend.add(restorationPriorityLegendPanel);

    map.addLayer(restorationPriority_scaled, {min: 0, max: 1, palette: restorationPriorityPalette}, "Restoration Priority Scaled");
  }
});

////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

///////////////////////////////// Build Reset Button ///////////////////////////////////////////////////////////////////////////////////////////////////

var resetButton = ui.Button ({
  label: 'Reset',
  style: {stretch: "horizontal"},
  onClick: function() {
    countryDropdown.setValue(null, false);
    countryDropdown.setPlaceholder("Choose a country...");
    checkbox_grassland.setValue(false);
    checkbox_cropland.setValue(false);
    checkbox_reforestation.setValue(false);
    checkbox_afforestation.setValue(false);
    forestDistanceSlider.setValue(0);
    opportunityCostSlider.setValue(0);
    carbonSeqSlider.setValue(0);
    legend.clear();
    map.clear();
    map.setOptions("HYBRID");
    map.addLayer(lc2018_grassland, {palette:['d8d800']}, "Grassland", false);
    map.addLayer(lc2018_cropland, {palette:['a50f15']}, "Cropland", false);
    map.addLayer(historicalForest, {palette:['006d2c']}, "Historical Forest", false);
  }
});
////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

///////////////////////////////// Build Priorities Panel ////////////////////////////////////////////////////////////////////////////////////////
var line = ui.Label("_______________________________________________");
var prioritiesPanelLabel = ui.Label("2. Priorities for Restoration", {color: "black", fontWeight: "bold", fontSize: "16px"});
var prioritiesPanelDetails = ui.Label("Use the sliders below to set the importance of each factor. Use the buttons to display the layers on the map. Use the 'Layers' menu on the map to turn the layers off.", {fontSize: '10px'});
var prioritiesPanel = ui.Panel();
prioritiesPanel.add(line);
prioritiesPanel.add(prioritiesPanelLabel);
prioritiesPanel.add(prioritiesPanelDetails);
prioritiesPanel.add(forestDistancePanel);
prioritiesPanel.add(opportunityCostPanel);
prioritiesPanel.add(carbonSeqPanel);

var analysisPanel = ui.Panel();
var analysisLabel = ui.Label("3. Restoration Priority Analysis", {color: "black", fontWeight: "bold", fontSize: "16px"})
var warningLabel = ui.Label("This may take a few minutes. You may need to zoom in to better see restoration priorities. Use the button below to reset the analysis.", {fontSize: '10px', stretch: 'horizontal'});
analysisPanel.add(analysisLabel)
analysisPanel.add(runAnalysisButton);
analysisPanel.add(warningLabel);

prioritiesPanel.add(analysisPanel);
prioritiesPanel.add(resetButton);
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

leftPanel.add(prioritiesPanel);

/////////////////////////////////// Build Project Area Drawing Panel /////////////////////////////////////////////////////////////////////////////////
// code for drawing tools taken directly from https://developers.google.com/earth-engine/tutorials/community/drawing-tools-region-reduction
var projectPanelLabel = ui.Label("4. Project Details", {fontSize: "16px", fontWeight: "bold"});
var projectDetailsLabel = ui.Label("Default numbers are given for project length, cost, and return based on average values from models in Peru, Indonesia, and Cambodia. These numbers can be changed to better suit your project.", {fontSize: '10px'})
var projectAreaLabel = ui.Label("Draw project area", {color: "gray", fontWeight: "bold"});
var projectAreaDetails = ui.Label("Zoom into an area of interest and use the buttons below to draw a project area. You will then be able to see estimated outcomes, costs, and benefits for your project.", {fontSize: '10px'});


/////////////////////////////////////// Drawing Tools //////////////////////////////////////////////////////
var drawingTools = map.drawingTools();
drawingTools.setShown(false); // don't show the drawing tools on the map - triggered by buttons instead

// remove any previously drawn shapes each time new shape is drawn
while (drawingTools.layers().length() > 0) {
  var layer = drawingTools.layers().get(0);
  drawingTools.layers().remove(layer);
}

// create dummy geometry layer to hold drawn geometries
var dummyGeometry = ui.Map.GeometryLayer({geometries: null, name: 'geometry', color: '23cba7'});

// add dummy geometry to map
drawingTools.layers().add(dummyGeometry);

// create funtions to be triggered by the buttons
// remove all drawn geometries and reset all outcome boxes
function clearGeometry() {
  while (drawingTools.layers().length() > 0) {
    var layer = drawingTools.layers().get(0);
    drawingTools.layers().remove(layer);
  }
  areaBox.setValue("");
  endSpecBox.setValue("");
  benefBox.setValue("");
  AGcarbonSeqBox.setValue("");
  BGcarbonSeqBox.setValue("");
  totalCostBox.setValue("");
  totalReturnBox.setValue("");
  costReturnRatioBox.setValue("");
  
}

// draw a rectangle
function drawRectangle() {
  clearGeometry();
  drawingTools.setShape('rectangle');
  drawingTools.draw();
}

// draw a polygon
function drawPolygon() {
  clearGeometry();
  drawingTools.setShape('polygon');
  drawingTools.draw();
}

// create lag time so that stats aren't being calculated constantly
drawingTools.onDraw(ui.util.debounce(calculateStats, 500));
drawingTools.onEdit(ui.util.debounce(calculateStats, 500));

// define aoi variable - this will be updated within the calculate stats function
var aoi;

// define symbols for the buttons
var symbol = {
  rectangle: '⬛',
  polygon: '🔺',
};

// create a panel to hold the drawing buttons
var drawingPanel = ui.Panel({
  widgets: [
    ui.Button({
      label: symbol.rectangle + ' Rectangle',
      onClick: drawRectangle,
      style: {stretch: 'horizontal'}
    }),
    ui.Button({
      label: symbol.polygon + ' Polygon',
      onClick: drawPolygon,
      style: {stretch: 'horizontal'}
    }),
    ui.Button({
      label: "Clear All Drawings",
      onClick: clearGeometry,
      style: {stretch: 'horizontal'}
    })
    ]
});
////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

////////////////////////////////// Project Details Panel ////////////////////////////////////////////////////////////


// define project length label & textbox
var projectLengthLabel = ui.Label("Enter project length (years)", {color: "gray", fontWeight: "bold", padding: '4px 10px 0px 0px'});
var projectLengthBox = ui.Textbox({value: 15, style: {stretch: "horizontal"}}); // default value is 15

// define project length based on number in textbox
var projectLength = ee.Number(projectLengthBox.getValue());

// update project length when textbox entry is changed - need to cast as number twice
projectLengthBox.onChange(function(value) {
  projectLength = ee.Number(Number(value));
});

// build project length panel, add label and textbox
var projectLengthPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
projectLengthPanel.add(projectLengthLabel);
projectLengthPanel.add(projectLengthBox);

// define average cost label and textbox
var averageCostLabel = ui.Label("Enter project cost ($/ha/yr)", {color: "gray", fontWeight: "bold", padding: '4px 11px 0px 0px'});
var averageCostBox = ui.Textbox({value: 1686, style: {stretch: "horizontal"}}); // default value is 1686

// define average cost based on number in textbox
var averageCost = ee.Number(averageCostBox.getValue());

// update average cost when textbox entry is changed - cast as number twice
averageCostBox.onChange(function(value) {
  averageCost = ee.Number(Number(value));
});

// build average cost panel, add label and textbox
var averageCostPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
averageCostPanel.add(averageCostLabel);
averageCostPanel.add(averageCostBox);

// define average return label and textbox
var averageReturnLabel = ui.Label("Enter project return ($/ha/yr)", {color: "gray", fontWeight: "bold", padding: '4px 0px 0px 0px'});
var averageReturnBox = ui.Textbox({value: 3788, style: {stretch: "horizontal"}}); // default value is 3788

// define average return based on number in textbox
var averageReturn = ee.Number(averageReturnBox.getValue());

// update average return when textbox entry is changed - cast as number twice
averageReturnBox.onChange(function(value) {
  averageReturn = ee.Number(Number(value));
});

// build average return panel, add label and textbox
var averageReturnPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
averageReturnPanel.add(averageReturnLabel);
averageReturnPanel.add(averageReturnBox);
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

/////////////////////////////////////// Build Project Details Panel ////////////////////////////////////////////////////////////////
var line2 = ui.Label("_________________________________________________");
var statsWarning = ui.Label("It may take a few minutes to calculate statistics. Please don't change anything below this line.", {fontSize: '10px'});

var projectAreaPanel = ui.Panel();
projectAreaPanel.add(projectPanelLabel);
projectAreaPanel.add(projectDetailsLabel);
projectAreaPanel.add(projectLengthPanel);
projectAreaPanel.add(averageCostPanel);
projectAreaPanel.add(averageReturnPanel);
projectAreaPanel.add(projectAreaLabel);
projectAreaPanel.add(projectAreaDetails);
projectAreaPanel.add(drawingPanel);
projectAreaPanel.add(statsWarning);
projectAreaPanel.add(line2);
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

////////////////////////////////////////// Build Project Statistics Panel ////////////////////////////////////////////////
var statisticsPanel = ui.Panel();
var statisticsLabel = ui.Label("Estimated project outcomes", {fontSize: "16px", fontWeight: "bold"});

// build area stats panel with label and textbox
var areaStatsPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var areaLabel = ui.Label("Area (ha)", {color: "gray", fontWeight: "bold", padding: '5px 148px 0px 0px'});
var areaBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right"}});
areaStatsPanel.add(areaLabel);
areaStatsPanel.add(areaBox);

// build endangered species stats panel with label and textbox
var endSpecStatsPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var endSpecLabel = ui.Label("Endangered Species (#)", {color: "gray", fontWeight: "bold", padding: '5px 56px 0px 0px'});
var endSpecBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right"}});
endSpecStatsPanel.add(endSpecLabel);
endSpecStatsPanel.add(endSpecBox);

// build beneficiaries panel with label and textbox
var benefStatsPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var benefLabel = ui.Label("Indirect Beneficiaries (# people)", {color: "gray", fontWeight: "bold", padding: '5px 5px 0px 0px'});
var benefBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right"}});
benefStatsPanel.add(benefLabel);
benefStatsPanel.add(benefBox);

// build above ground carbon sequestration with label and textbox
var AGcarbonSeqStatsPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var AGcarbonSeqLabel = ui.Label("Total Above-Ground \nCarbon Sequestration (tons C)", {color: "gray", fontWeight: "bold", padding: '5px 16px 0px 0px', 'whiteSpace': 'pre'});
var AGcarbonSeqBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right", padding: '8px 0px 0px 0px'}});
AGcarbonSeqStatsPanel.add(AGcarbonSeqLabel);
AGcarbonSeqStatsPanel.add(AGcarbonSeqBox);

// build below ground carbon sequestration with label and textbox
var BGcarbonSeqStatsPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var BGcarbonSeqLabel = ui.Label("Total Below-Ground \nCarbon Sequestration (tons C)", {color: "gray", fontWeight: "bold", padding: '5px 16px 0px 0px', 'whiteSpace': 'pre'});
var BGcarbonSeqBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right", padding: '8px 0px 0px 0px'}});
BGcarbonSeqStatsPanel.add(BGcarbonSeqLabel);
BGcarbonSeqStatsPanel.add(BGcarbonSeqBox);

// cost/return label
var costReturnLabel = ui.Label("Estimated project costs and returns", {fontSize: "16px", fontWeight: "bold"});

// build total cost panel with label and textbox
var totalCostPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var totalCostLabel = ui.Label("Total Project Cost ($)", {color: "gray", fontWeight: "bold", padding: '5px 72px 0px 0px'});
var totalCostBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right"}});
totalCostPanel.add(totalCostLabel);
totalCostPanel.add(totalCostBox);

// build total return panel with label and textbox
var totalReturnPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var totalReturnLabel = ui.Label("Total Project Return ($)", {color: "gray", fontWeight: "bold", padding: '5px 59px 0px 0px'});
var totalReturnBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right"}});
totalReturnPanel.add(totalReturnLabel);
totalReturnPanel.add(totalReturnBox);

// build cost/return ratio panel with label and textbox
var costReturnRatioPanel = ui.Panel({layout: ui.Panel.Layout.flow('horizontal')});
var costReturnRatioLabel = ui.Label("Return/Cost Ratio", {color: "gray", fontWeight: "bold", padding: '5px 94px 0px 0px'});
var costReturnRatioBox = ui.Textbox({value: "", style: {width: "85px", position: "middle-right"}});
costReturnRatioPanel.add(costReturnRatioLabel);
costReturnRatioPanel.add(costReturnRatioBox);

// add all panels to the main panel
statisticsPanel.add(statisticsLabel);
statisticsPanel.add(areaStatsPanel);
statisticsPanel.add(endSpecStatsPanel);
statisticsPanel.add(benefStatsPanel);
statisticsPanel.add(AGcarbonSeqStatsPanel);
statisticsPanel.add(BGcarbonSeqStatsPanel);
statisticsPanel.add(costReturnLabel);
statisticsPanel.add(totalCostPanel);
statisticsPanel.add(totalReturnPanel);
statisticsPanel.add(costReturnRatioPanel);
///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

///////////////////////////////// Add all data for Endangered Species Calculation /////////////////////////////////////////////////////////////
//Adds species polygons for all IUCN species
// amphibians (Tailless Amphibians, Tailed Amphibians, and Caecilian Amphibians)
var amphi = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_amphibians_20191210_s");
  
// birds from Birdlife (2019/03/20)
var birds = ee.FeatureCollection("users/geflanddegradation/biodiversity/botw_20190320_simple_status");
  
// Reef-forming Corals
var corals1 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_corals1_20191210_s");
var corals2 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_corals2_20191210_s");
var corals3 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_corals3_20191210_s");
var corals = corals1.merge(corals2).merge(corals3);
  
// Cone Snails
var csnail = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_conus_20191210_s");
  
// Fresh water (Fishes, Molluscs, Plants, Odonata (dragonflies and damselflies), Shrimps, Crabs, Crayfishes)
var fwater1 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_freshwater1_20191210_s");
var fwater2 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_freshwater2_20191210_s");
var fwater3 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_freshwater3_20191210_s");
var fwater4 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_freshwater4_20191210_s");
var fwater5 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_freshwater5_20191210_s");
var fwater6 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_freshwater6_20191210_s");
var fwater = fwater1.merge(fwater2).merge(fwater3).merge(fwater4).merge(fwater5).merge(fwater6);
  
// lobsters
var lobster = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_lobster_20191210_s");
  
// mammals (terrestrial and marine)
var mammals = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_mammals_20191210_s");
  
// mangroves
var mangrov = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_mangroves_20191210_s");
  
// Marine Fishes (Angelfishes, Bonefishes and Tarpons, Butterflyfishes, Blennies, Clupeiformes, Damselfishes,
// Groupers, Hagfishes, Pufferfishes, Seabreams, Porgies and Picarels, Surgeonfishes, Tangs and Unicornfishes,
// Syngathiform fishes (Seahorses & Pipefishes, Ghost Pipefishes, Seamoths, Shrimpfishes, Trumpetfishes, Cornetfishes),
// Wrasses and Parrotfishes, and Tunas and Billfishes)
var mfish1 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_marine_fish1_20191210_s");
var mfish2 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_marine_fish2_20191210_s");
var mfish3 = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_marine_fish3_20191210_s");
var mfish = mfish1.merge(mfish2).merge(mfish3);
  
// reptiles (Sea Snakes, Chameleons, and Crocodiles and Alligators)
var repti = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_reptiles_20191210_s");
  
// sea cucumbers
var seacucu = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_seacucumbers_20191210_s");
  
// seagrasses
var seagras = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_seagrasses_20191210_s");
  
// Chondrichthyes (sharks, rays and chimaeras)
var sharks = ee.FeatureCollection("users/geflanddegradation/biodiversity/iucn_sharks_rays_20191210_s"); 

// create species count table within given project Area
// NOTE: this takes in the entire drawn polygon, not just the available area
// so if a polygon was drawn over an area with no possibility for restoration, this number will NOT be 0
var speciesCountAnalysis = function(projectArea){
  var sp_count_table = projectArea.map(function(poly){
    var amphi_thr = ee.Dictionary(amphi.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var birds_thr = ee.Dictionary(birds.filterBounds(poly.geometry()).filter(ee.Filter.inList('RedList_28', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var corals_thr = ee.Dictionary(corals.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var csnail_thr = ee.Dictionary(csnail.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var fwater_thr = ee.Dictionary(fwater.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var lobster_thr = ee.Dictionary(lobster.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var mammals_thr = ee.Dictionary(mammals.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var mangrov_thr = ee.Dictionary(mangrov.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var mfish_thr = ee.Dictionary(mfish.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var repti_thr = ee.Dictionary(repti.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var seacucu_thr = ee.Dictionary(seacucu.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var seagras_thr = ee.Dictionary(seagras.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
    var sharks_thr = ee.Dictionary(sharks.filterBounds(poly.geometry()).filter(ee.Filter.inList('category', ["CR","EN","VU"])).aggregate_histogram('binomial')).size();
        return poly.set({'sp_tthreat': (amphi_thr.add(birds_thr).add(corals_thr).add(csnail_thr).add(fwater_thr).add(lobster_thr).add(mammals_thr).add(mangrov_thr).add(mfish_thr).add(repti_thr).add(seacucu_thr).add(seagras_thr).add(sharks_thr))});
  });
  return sp_count_table;
};
////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

/////////////////////////////////////// Create Calculate Stats Function //////////////////////////////////////////////////////////////////////////////////////////////////
function calculateStats() {
  // set area of interest equal to the drawn project area
  aoi = drawingTools.layers().get(0).getEeObject();
  
  // clip the available area to the aoi
  var projectAvailableArea = availableArea.clip(aoi);
  
  // create image overlapping available area where each pixel value equals its area
  var pixelArea = availableArea.multiply(ee.Image.pixelArea()).divide(10000);
  
  // use reducer to sum over the pixel area image to calculate total area; extract value
  var restorationArea = ee.Number(pixelArea.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: projectAvailableArea.geometry(),
    scale: availableArea.projection().nominalScale(),
    maxPixels: 10e10
  }).get('constant'));
  
  // set area box equal to calculated area; use getInfo to pull to client side
  areaBox.setValue(restorationArea.round().getInfo());
  
  // run species count analysis function over area of interest (entire drawn polygon, not just available area)
  // cast to feature collection to fit function
  // extract value
  var endSpec = speciesCountAnalysis(ee.FeatureCollection(aoi)).first().get('sp_tthreat');
  
  // set end spec box equal to # species, pull to client side
  endSpecBox.setValue(endSpec.getInfo());
  
  // create image of population data for 2020
  var worldpop = ee.ImageCollection("WorldPop/GP/100m/pop").filter(ee.Filter.eq('year', 2020)).mosaic();
  
  // sum over area of available area within aoi
  var beneficiaries = ee.Number(worldpop.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: projectAvailableArea.geometry(),
    scale: 100,
    maxPixels: 10e10
  }).get('population')).round().getInfo(); // extract value, pull to client side
  
  // set benef box equal to calculated value
  benefBox.setValue(beneficiaries);
  
  // calculate average above-ground carbon seq rate over available area within aoi
  var AG_CarbonSeq_mean = ee.Number(carbon_seq_AGB.unmask().reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: projectAvailableArea.geometry(),
    scale: carbon_seq_AGB.projection().nominalScale(),
    maxPixels: 10e10
  }).get('b1'));
  
  // multiply average rate by available project area by project length
  var AG_CarbonSeq_total = restorationArea.multiply(AG_CarbonSeq_mean).multiply(projectLength).round().getInfo();
  
  // set AG carbon box equal to total tons
  AGcarbonSeqBox.setValue(AG_CarbonSeq_total);
  
  // calculate average below-ground carbon seq rate over available area within aoi
  var BG_CarbonSeq_mean = ee.Number(carbon_seq_BGB.unmask().reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: projectAvailableArea.geometry(),
    scale: carbon_seq_BGB.projection().nominalScale(),
    maxPixels: 10e10
  }).get('b1'));
  
  // multiply average rate by available project area by project length
  var BG_CarbonSeq_total = restorationArea.multiply(BG_CarbonSeq_mean).multiply(ee.Number(projectLength)).round().getInfo();
  
  // set BG carbon box equal to total tons
  BGcarbonSeqBox.setValue(BG_CarbonSeq_total);
  
  // total project cost is average cost/ha/yr * project length * area
  var totalProjectCost = averageCost.multiply(projectLength).multiply(restorationArea).round().getInfo();
  
  // set total cost box equal to total cost
  totalCostBox.setValue(totalProjectCost);
  
  // total project return is average return/ha/yr * project length * area
  var totalProjectReturn = averageReturn.multiply(projectLength).multiply(restorationArea).round().getInfo();
  
  // set total return box to total return
  totalReturnBox.setValue(totalProjectReturn);
  
  // ratio is total return/total cost
  var costReturnRatio = ee.Number(totalProjectReturn).divide(ee.Number(totalProjectCost)).getInfo();
  
  // set ratio box to ratio
  costReturnRatioBox.setValue(costReturnRatio.toFixed(1));
}
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


// when drawing tools are used, call calculate stats function
drawingTools.onDraw(calculateStats);

// add project area panel and statistics panel to right panel
rightPanel.add(projectAreaPanel);
rightPanel.add(statisticsPanel);

