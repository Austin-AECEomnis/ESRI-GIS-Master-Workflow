# ArcPy Scripting

[Back to Main Guide](../README.md)

ArcPy scripting discipline, the terrain distillation pipeline, ExtractMultiValuesToPoints equivalent workflow, live data pipeline integration, and every scripting gotcha encountered across both watershed analyses.

---

## Core Scripting Rules

These rules apply to every script in this workflow without exception. Each one exists because violating it cost real time in the original workflow.

**Always call `arcpy.ListFeatureClasses()` before referencing any layer by name.** Display names in the Contents pane are not geodatabase names. A script built against a display name fails silently or produces incorrect output with no obvious error message. Run the list, verify the exact internal names, use those names.

**Set `arcpy.env.workspace` explicitly at the top of every script.** Never assume the workspace is set correctly from a previous session or from the Pro environment. State it explicitly every time.

**Keep raw data, processed intermediates, and final outputs in separate directories.** A script operating on the Outputs/ folder should never be able to reach into Processed/ and grab a stale intermediate layer. Directory separation is the only reliable structural guarantee that the right version of a layer is used.

**Clear active selections before any export.** An active selection on a feature class exports only selected features and generates error 000733. Verify no selection is active before running Export Features or any tool that writes output from a feature class.

**Verify output field names before chaining tools.** ArcPy tools that add fields sometimes generate names that differ from what you specified, particularly when a collision exists with an existing field name. Check the output attribute table after each tool run before using that output as input to the next tool.

---

## Terrain Distillation Pipeline

The terrain distillation pipeline automates the full sequence from raw DEM to enriched structure point layer. Manual execution of this workflow took over an hour per run due to full county-wide DEM processing. The automated pipeline with clip-first design runs in under 10 minutes.

### Pipeline Sequence

```
1. Buffer flood zone polygon (3,000 feet)
2. Extract by Mask: clip DEM to buffered flood zone
3. Slope tool: run on clipped DEM
4. Aspect tool: run on clipped DEM
5. Flow Direction tool: run on clipped DEM
6. Flow Accumulation tool: run on Flow Direction output
7. Extract Multi Values to Points: sample all rasters at structure point locations
8. Export enriched structure layer to Outputs/
```

### Why 3,000 Feet

The 3,000-foot buffer is not arbitrary. A tighter buffer on the Guadalupe workflow produced 48 null values on raster extraction because structure points near the flood zone edge fell outside the clipped raster extent. The 3,000-foot buffer captures all structure points within the raster extent without processing unnecessary terrain beyond the study area. Do not reduce this buffer without verifying null value counts in the output.

### Annotated Pipeline Script

```python
import arcpy
import os

# Set environment
arcpy.env.workspace = r"C:\YourProject\YourProject.gdb"
arcpy.env.overwriteOutput = True

# Verify layer names before scripting
print("Feature classes in workspace:")
for fc in arcpy.ListFeatureClasses():
    print(f"  {fc}")

# Paths
flood_zone = "FloodZone_StudyArea"          # adjust to your actual FC name
structures = "Structures_StudyArea"          # adjust to your actual FC name
dem_raw = r"C:\YourProject\RawData\DEM_raw.tif"
output_folder = r"C:\YourProject\Outputs"

# Step 1: Buffer flood zone
buffer_output = os.path.join(arcpy.env.workspace, "FloodZone_Buffer3000ft")
arcpy.analysis.Buffer(flood_zone, buffer_output, "3000 Feet")
print("Buffer complete")

# Step 2: Clip DEM to buffer
dem_clipped = os.path.join(output_folder, "DEM_clipped.tif")
arcpy.sa.ExtractByMask(dem_raw, buffer_output).save(dem_clipped)
print("DEM clip complete")

# Step 3: Slope
slope_output = os.path.join(output_folder, "Slope.tif")
arcpy.sa.Slope(dem_clipped, "DEGREE").save(slope_output)
print("Slope complete")

# Step 4: Aspect
aspect_output = os.path.join(output_folder, "Aspect.tif")
arcpy.sa.Aspect(dem_clipped).save(aspect_output)
print("Aspect complete")

# Step 5: Flow Direction
flow_dir_output = os.path.join(output_folder, "FlowDir.tif")
arcpy.sa.FlowDirection(dem_clipped).save(flow_dir_output)
print("Flow direction complete")

# Step 6: Flow Accumulation
flow_acc_output = os.path.join(output_folder, "FlowAcc.tif")
arcpy.sa.FlowAccumulation(flow_dir_output).save(flow_acc_output)
print("Flow accumulation complete")

# Step 7: Extract multi-values to structure points
# Creates new fields on the structures layer for each raster
in_rasters = [
    [slope_output, "slope"],
    [aspect_output, "aspect"],
    [flow_acc_output, "flow_acc"]
]
arcpy.sa.ExtractMultiValuesToPoints(structures, in_rasters, "NONE")
print("Raster extraction complete")

# Step 8: Export enriched layer
enriched_output = os.path.join(arcpy.env.workspace, "Structures_Enriched")
arcpy.management.CopyFeatures(structures, enriched_output)
print("Export complete")
print("Pipeline finished")
```

### Field Name Verification After Extraction

After running ExtractMultiValuesToPoints, inspect the output attribute table before using the enriched layer downstream. ArcPy may append a numeric suffix to field names if a collision exists. The actual field name in the table is what downstream scripts and Online pop-ups must reference.

```python
# Verify field names on enriched output
fields = arcpy.ListFields("Structures_Enriched")
for field in fields:
    print(f"{field.name} ({field.type})")
```

---

## Live Data Pipeline Integration

The live USGS stream gauge pipeline uses GitHub Actions to fetch current gauge readings from the USGS Water Services API and write them to a hosted feature layer in ArcGIS Online. ArcPy is involved in the local data preparation and validation steps that feed into this pipeline.

For the full GitHub Actions YAML, USGS API endpoint details, the array index fix, and the iframe CSP resolution, see [Live Data Pipeline](live-data-pipeline.md).

### Local ArcPy Role in the Live Pipeline

ArcPy prepares the gauge location feature class that the pipeline updates. The gauge point feature class must exist as a hosted feature layer in Online before the GitHub Actions pipeline can write to it. The preparation steps are:

1. Create a point feature class in the analysis geodatabase at the gauge coordinates
2. Add fields for stage reading, timestamp, and any derived attributes
3. Export to shapefile, ZIP, and upload to Online as a hosted feature layer (see [Publishing Errors](publishing-errors.md) for the shapefile ZIP procedure)
4. Note the hosted feature layer's Item ID from the Online URL for use in the pipeline script

Once the hosted feature layer exists in Online, the GitHub Actions script updates it via the ArcGIS REST API on a scheduled interval. ArcPy is not involved in the automated update cycle, only in the initial feature layer creation.

---

## Known Scripting Gotchas

### arcpy.env.overwriteOutput

Set `arcpy.env.overwriteOutput = True` at the top of every script that writes output to existing paths. Without it, a script that has been run before will fail when it tries to write to a path that already exists, with a generic error that does not clearly indicate overwrite is the issue.

### Spatial Analyst License Check

Tools from the Spatial Analyst extension (Slope, Aspect, ExtractByMask, FlowDirection, FlowAccumulation, ExtractMultiValuesToPoints) require the Spatial Analyst extension to be active. If a script fails on one of these tools, verify the extension is checked out.

```python
# Check out Spatial Analyst
arcpy.CheckOutExtension("Spatial")
# Run Spatial Analyst tools here
arcpy.CheckInExtension("Spatial")
```

### Output Path Handling

Always construct output paths using `os.path.join()` rather than string concatenation. Hardcoded path separators cause failures when scripts are moved between machines or when folder structures change.

```python
import os
output = os.path.join(arcpy.env.workspace, "OutputLayerName")
# Not: output = arcpy.env.workspace + "\\" + "OutputLayerName"
```

### Verify Outputs Exist Before Chaining

After each tool run in a pipeline, verify the output exists and is not empty before passing it to the next tool. A tool that completes without throwing an error can still produce an empty or malformed output under certain conditions.

```python
if arcpy.Exists(dem_clipped):
    result = arcpy.management.GetCount(dem_clipped)
    print(f"Clipped DEM exists, cell count: {result}")
else:
    print("ERROR: Clipped DEM was not created")
    raise FileNotFoundError("DEM clip failed, stopping pipeline")
```

### Session Boundaries and Parameter Loss

ArcPy script parameters set during one Pro session are not automatically available in the next. At the top of every script, explicitly define all paths, workspace settings, and environment flags. Never assume a variable from a previous interactive session is still set.

---

[Back to Main Guide](../README.md)

See also: [ArcGIS Pro Project Management](arcgis-pro-project-management.md) | [DEM and Terrain Analysis](dem-and-terrain-analysis.md) | [Live Data Pipeline](live-data-pipeline.md) | [Publishing Errors](publishing-errors.md)
