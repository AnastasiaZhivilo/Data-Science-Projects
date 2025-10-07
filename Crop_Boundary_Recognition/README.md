# Workflow Summary: Preparing Sentinel-2 Data for Instance Segmentation
This workflow details the process of selecting a study area, generating multi-temporal Sentinel-2 imagery, creating labeled masks (ground truth), and formatting the data for training a U-Net model for field boundary detection.

## Phase 1: Data Acquisition and Study Area Definition

### Step 1: Define Bounding Box (AOI)
* **Action:** We initially defined a large geographic bounding box (Area of Interest or AOI) based on a center coordinate (e.g., 146.9561 
∘
  E, −34.7022 
∘
  S).

* **Output:** An ee.Geometry.Rectangle object (bounding_box_geom) that constrains all subsequent Earth Engine data filtering.

### Step 2: Acquire and Process Satellite Imagery

* **Action:** We accessed the Sentinel-2 Level-2A (Surface Reflectance) data collection (COPERNICUS/S2_SR_HARMONIZED).

* **Filtering:** The collection was filtered by date range (e.g., '2024-07-01' to '2024-09-15') and cloud percentage (e.g., <10%).

* **Cloud Masking:** A cloud-masking function (mask_s2_clouds) was applied to remove cloud and cirrus pixels.

* **Composite Generation:** A median composite (median_image) was created to ensure a clear, cloud-free representation of the AOI. The image was explicitly reprojected to the standard Sentinel-2 10-meter scale and projection (EPSG:4326, 10m) for alignment.

* **Output:** The multi-band median composite, which serves as the visual input (RGB) and the spectral input (all 30 bands) for the U-Net model.

## Phase 2: Ground Truth Label Generation

### Step 3: Create Field Polygons (Labeling)
* **Action:** Using the interactive map, field boundaries were drawn as polygons over the target area.

* **Data Storage:** The coordinates of these polygons were saved to a GeoJSON file (my_training_polygons.geojson) on Google Drive.

### Step 4: Export Aligned GeoTIFF Masks
* **Action:** We iterated through each saved polygon and executed an Earth Engine export task.

* **Rasterization:** Each vector polygon was rasterized onto a single-band image where the field interior has a value of 1 (mask_value) and the outside is 0 (unmasked).

* **Alignment (Critical):** The rasterized mask was explicitly reprojected and clipped to ensure its pixels were perfectly aligned with the grid of the Sentinel-2 median composite image.

* **Export Region Fix:** The export region was set to the full AOI bounding box to guarantee stable GeoTIFF metadata, resolving previous issues where local reading pipelines failed to correctly interpret clipped files.

* **Output:** One GeoTIFF mask file (e.g., Polygon_1_Export.tif) per field, where the mask is perfectly aligned to the satellite image grid.

## Phase 3: Data Preparation for U-Net

### Step 5: Local Stack Creation
* **Action:** The Sentinel-2 input images (30 bands) and all exported GeoTIFF masks were loaded locally from Google Drive.

* **Stacking:** The multi-band Sentinel-2 images were stacked into a single 3D input volume (e.g., H×W×30).

### Step 6: Generating Instance Labels
* **Action:** The multiple single-field mask GeoTIFFs were processed to create the target label maps required for the U-Net.

* **Boundary Map:** This involves applying morphological operations (like dilation/erosion) to the mask to generate a thin line representing the exact field boundary.

* **Distance Map:** A Euclidean distance transform was applied to the mask to create a distance map.

* **U-Net Target:** The final ground truth consists of a 3D volume (H×W×2) containing the Boundary Map (Channel 1) and the Distance Map (Channel 2).

### Step 7: Tiling and Training
* **Action:** The large input stack and the label maps are sliced into smaller, fixed-size tiles (e.g., 256x256) to fit into GPU memory for training.

* **Model:** A U-Net architecture is initialized, taking the H×W×30 Sentinel-2 tile as input and aiming to predict the H×W×2 label map (Boundary + Distance).

* **Result:** The trained model is used to predict field boundaries on unseen satellite imagery.
