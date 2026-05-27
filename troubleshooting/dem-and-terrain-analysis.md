# DEM and Terrain Analysis

[Back to Main Guide](../README.md)

Everything learned about DEM preparation, terrain tool performance, and raster analysis across both watershed studies. The clip-first design documented here reduced pipeline runtime from over an hour to under ten minutes.

---

## The Clip-First Design

Running slope, aspect, or flow accumulation on a full county-wide DEM is the most common performance mistake in this workflow. A full county DEM contains millions of cells. ArcGIS Pro's terrain tools process every cell regardless of how many are relevant to your study area. On the hardware used in this workflow, a full county DEM run took over an hour and in some cases did not complete.

The solution is to clip the DEM to the study area before running any terrain tools. The clip operation itself is fast. Every terrain tool that runs on the clipped DEM is working on a fraction of the original cell count.

**Clip-first runtime comparison for the Guadalupe workflow:**
- Full county DEM terrain pipeline: 60-plus minutes, sometimes incomplete
- Clip-first terrain pipeline: under 10 minutes, consistent completion

This is not a marginal improvement. It is the difference between a workflow that is practical and one that is not.

---

## The 3,000-Foot Buffer

The clip boundary is not the flood zone polygon itself. It is a 3,000-foot buffer around the flood zone polygon.

**Why the buffer is necessary:** Structure points near the edge of the flood zone polygon need terrain data from a slightly larger area to produce valid raster extractions. In the initial Guadalupe workflow, a tighter clip produced 48 null values in the enriched structure layer because those structure points fell just outside the clipped raster extent. Null values in a terrain-enriched layer contaminate analysis results and require identifying, diagnosing, and rerunning the affected subset.

**Why 3,000 feet specifically:** This value was validated against the structure point distribution in the Guadalupe study area. It captures all structure points within the raster extent without extending the processing area unnecessarily into terrain that has no analytical value for the flood zone analysis.

**For a different study area:** If the structure point distribution extends closer to or farther from the flood zone edge than in the Guadalupe corridor, adjust this buffer accordingly. The diagnostic is straightforward: after running ExtractMultiValuesToPoints, query the output for null values in the extracted terrain fields. If nulls exist, increase the buffer and rerun.

```python
# Check for null values in extracted terrain fields after pipeline run
import arcpy
arcpy.env.workspace = r"C:\YourProject\YourProject.gdb"

null_count = 0
with arcpy.da.SearchCursor("Structures_Enriched", ["slope", "aspect", "flow_acc"]) as cursor:
    for row in cursor:
        if any(v is None for v in row):
            null_count += 1

print(f"Null terrain values found: {null_count}")
if null_count > 0:
    print("Increase buffer distance and rerun pipeline")
```

---

## DEM Source and Preparation

### USGS 1/3 Arc-Second DEM

The DEMs used in this workflow were downloaded from the USGS National Map at 1/3 arc-second resolution. This resolution is appropriate for the scale of analysis in both watershed studies.

**Delivered CRS:** NAD83 Geographic (EPSG: 4269), decimal degrees. This CRS is not suitable for terrain analysis. Reproject before running any terrain tools.

**Reprojection target:** NAD83 UTM Zone 14N (EPSG: 26914) for the Hill Country study areas. This puts the DEM in meters, which is the correct unit for slope calculations and flow accumulation thresholds.

**Reprojection tool:** Project Raster (Data Management Tools, Projections and Transformations, Raster). Set the output coordinate system to EPSG: 26914 and use Bilinear resampling for continuous raster data like elevation.

### File Naming

Name the reprojected DEM clearly and immediately. A DEM file with no indication of its CRS or processing state causes confusion when multiple versions accumulate in the project folder.

Recommended convention: `DEM_StudyArea_UTM14N.tif`

Avoid: `DEM_final.tif`, `DEM_copy.tif`, `DEM2.tif`

---

## Terrain Tool Reference

All terrain tools in this workflow require the Spatial Analyst extension. Verify the extension is checked out before running.

### Slope

**Tool:** Spatial Analyst, Surface, Slope

**Input:** Clipped, reprojected DEM in projected CRS

**Output type:** DEGREE for most analytical purposes. PERCENT_RISE for hydraulic applications.

**Key setting:** Ensure the Z unit matches the horizontal unit. If the DEM is in meters (UTM), the Z unit should also be meters. A Z factor of 1.0 is correct when both units match.

### Aspect

**Tool:** Spatial Analyst, Surface, Aspect

**Input:** Clipped, reprojected DEM

**Output:** Degrees from north, 0 to 360. Flat areas return -1.

**Note:** Aspect values of -1 (flat) in structure point extraction produce -1 in the enriched layer attribute, not null. This is expected behavior. Do not treat -1 aspect values as errors.

### Flow Direction

**Tool:** Spatial Analyst, Hydrology, Flow Direction

**Input:** Clipped, reprojected DEM

**Method:** D8 (default in ArcGIS Pro). For the purposes of this workflow, D8 produces consistent results and is appropriate for the scale of analysis.

**Output:** Integer raster encoding flow direction as powers of 2 (1, 2, 4, 8, 16, 32, 64, 128).

### Flow Accumulation

**Tool:** Spatial Analyst, Hydrology, Flow Accumulation

**Input:** Flow Direction output

**Output:** Integer raster where each cell value represents the number of upstream cells draining into it. High values indicate channels and drainage pathways.

**Threshold for channel identification:** In the Guadalupe workflow, flow accumulation values above a tested threshold were used to identify cells representing active drainage channels for structure proximity analysis. The specific threshold depends on raster resolution, study area size, and the drainage density appropriate to your watershed. Test multiple thresholds visually before committing to one for analysis.

---

## ExtractMultiValuesToPoints

This tool samples raster values at each point location in a structure point feature class and writes those values as new fields on the point layer. It is the ArcGIS Pro equivalent of QGIS's chained Sample Raster Values workflow.

**Tool location:** Spatial Analyst, Extraction, Extract Multi Values to Points

**Input point feature class:** The structure point layer in the same CRS as the rasters being sampled.

**Input rasters:** A list of rasters with associated output field name prefixes. Each raster adds one field to the point layer.

**Field name collision:** If a specified field name conflicts with an existing field on the point layer, ArcGIS Pro appends a numeric suffix automatically. Verify actual output field names after the tool runs before using them in any downstream script or Online pop-up configuration.

**Interpolation:** Set to NONE for categorical or discrete rasters. Bilinear interpolation is appropriate for continuous rasters like elevation, slope, and flow accumulation when point locations do not fall exactly on cell centers.

---

## Known Issues

### Null Values at Raster Edges

Described above under the 3,000-foot buffer section. Null values in extracted terrain fields indicate structure points that fell outside the clipped raster extent. Fix: increase the buffer and rerun.

### Slow Tool Progress on Large Rasters

If a terrain tool appears to hang with no progress bar movement, check Task Manager for CPU and disk activity before assuming a failure. Large raster operations can appear frozen for several minutes while processing. For the full county DEM runs in this workflow, apparent freezes of 10 to 15 minutes were followed by eventual completion or genuine failure. The clip-first design eliminates this ambiguity because smaller rasters process predictably.

### CRS Mismatch Between Points and Rasters

If the structure point feature class and the raster being sampled are in different CRS values, ExtractMultiValuesToPoints may complete without error but populate all extracted fields with null. Verify that both the point layer and all rasters are in the same CRS before running the tool. Right-click each layer, select Properties, and check the Source tab for the CRS.

---

[Back to Main Guide](../README.md)

See also: [ArcPy Scripting](arcpy-scripting.md) | [CRS and Projection](crs-and-projection.md) | [ArcGIS Pro Project Management](arcgis-pro-project-management.md)
