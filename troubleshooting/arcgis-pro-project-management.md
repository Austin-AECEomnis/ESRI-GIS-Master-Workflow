# ArcGIS Pro Project Management

[Back to Main Guide](../README.md)

Project file discipline, the two-.aprx CRS strategy, geodatabase survival, and everything learned about managing ArcGIS Pro projects across a multi-application ESRI workflow.

---

## The Two-.aprx Strategy

The single most impactful project management decision in this workflow was maintaining two separate .aprx project files from the start. This is not a workaround. It is the cleanest architectural solution to a genuine CRS conflict between local terrain analysis and ArcGIS Online publishing requirements.

**The conflict:** Terrain analysis tools produce the most accurate results when run in a local projected CRS appropriate to the study area, UTM or State Plane. ArcGIS Online requires WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857) for publishing. These two requirements cannot be satisfied simultaneously in a single project without on-the-fly reprojection masking mismatches that surface at publish time as error cascades.

**The solution:**

**Project 1, the Analysis project:** CRS set to the local projected system for the study area. For the Guadalupe workflow this was NAD83 UTM Zone 14N. For Hill Country studies generally, NAD83 Texas State Plane Central is also appropriate depending on the precision required. All terrain analysis, DEM processing, slope and aspect tools, and ArcPy scripting runs from this project. This is where the work happens.

**Project 2, the Publishing project:** CRS set to WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857) from the moment the project is created. Final processed layers are imported here after all analysis is complete. No analysis happens in this project. Its only job is to present clean, correctly projected layers to the Online publish analyzer.

**Setup sequence:** Create both projects before downloading any data. Set the CRS on each project explicitly before adding any layers. A project that inherits its CRS from the first layer added is a project waiting to cause problems.

See also: [CRS and Projection](crs-and-projection.md)

---

## The .aprx Deletion Risk

ArcGIS Pro will permanently delete a map from your project with a single click of the X button on a map tab. There is no confirmation dialog. There is no undo. The map is gone.

This happened during the Guadalupe workflow. The recovery procedure below works, but the time and stress it costs are entirely avoidable with disciplined save habits.

**What survives:** All data in the geodatabase survives intact. Feature classes, rasters, tables, and relationships stored in the .gdb are not affected by project file loss. The data is safe.

**What does not survive:** The project file itself. Map configurations, layer display names, symbology settings, layer order, group layer organization, and any map properties set in the .aprx are lost.

**Recovery procedure if the .aprx is lost:**
1. Open ArcGIS Pro and create a new project in the same root folder as the existing geodatabase
2. In the Catalog pane, navigate to the existing .gdb
3. Add feature classes and rasters back to the map by dragging from the Catalog pane
4. Reset the map frame CRS to the appropriate system (analysis CRS for Project 1, Web Mercator for Project 2)
5. Reapply symbology, aliases, and layer order
6. Save the project immediately and frequently from this point forward

**Prevention:** Save the project explicitly after every significant configuration change. The keyboard shortcut is Ctrl+S. Do not rely on autosave intervals. After adding layers, after setting symbology, after configuring pop-ups, after any change worth keeping: save.

---

## Display Names vs. Geodatabase Names

Layer names shown in the ArcGIS Pro Contents pane are display names. They are not the internal geodatabase names that ArcPy uses to reference feature classes.

A feature class created as FloodZone_KendallCounty_v3_final may display in the Contents pane as Flood Zone Kendall County. A script written to reference the display name will fail or produce incorrect output. A script written to reference the internal name works correctly regardless of what the display name is.

**Rule:** Always call `arcpy.ListFeatureClasses()` before writing any script that references layers by name. Inspect the output, confirm the exact internal names, and use those names in the script.

```python
import arcpy

arcpy.env.workspace = r"C:\YourProject\YourProject.gdb"
feature_classes = arcpy.ListFeatureClasses()
for fc in feature_classes:
    print(fc)
```

Run this once at the start of any scripting session. The output is the ground truth for what names to use.

See also: [ArcPy Scripting](arcpy-scripting.md)

---

## Geodatabase vs. Project File: Understanding What Lives Where

A common source of anxiety when something goes wrong in ArcGIS Pro is not knowing what is actually at risk. Understanding the separation between the project file and the geodatabase prevents panic and allows clear-headed recovery.

**The .aprx project file contains:**
- Map configurations and tab layout
- Layer references (pointers to data, not the data itself)
- Symbology and rendering settings
- Layer display names and aliases
- Group layer organization
- Map frame CRS settings
- Bookmarks and layout views

**The .gdb geodatabase contains:**
- Feature classes (the actual geometry and attribute data)
- Rasters
- Tables
- Relationship classes
- Domains

If the .aprx is lost, all data is safe. The project file is rebuilt by reconnecting to the existing geodatabase. If the geodatabase is corrupted or deleted, the data is gone regardless of what the project file says. Back up the geodatabase.

---

## Project Folder Organization

A well-organized project folder is the difference between a recoverable problem and an unrecoverable one. Consistent folder structure also makes ArcPy scripting more reliable because absolute paths remain stable.

Recommended structure for an ESRI workflow project:

```
ProjectName/
  ProjectName.gdb/        # primary geodatabase, all processed data
  RawData/                # downloaded source files, never modified after download
  Processed/              # intermediate outputs not yet ready for publishing
  Outputs/                # final layers, what scripts read from for final operations
  Scripts/                # ArcPy .py files with version comments in header
  Publishing/             # publishing .aprx and any shapefile ZIPs prepared for Online
  ProjectName.aprx        # analysis project file
```

The Publishing/ folder contains its own .aprx (the Web Mercator publishing project) and any shapefile ZIPs prepared for the Online upload workaround. Keeping publishing-specific files separate from analytical outputs prevents confusion about which version of a layer is the current final.

---

## Saving Discipline: A Practical Checklist

Save the project after each of the following:

- Adding new layers to the map
- Setting or changing the map frame CRS
- Applying symbology to any layer
- Configuring pop-ups
- Setting layer aliases
- Completing any geoprocessing operation
- Making any layout change
- Before running any long geoprocessing operation
- Before closing Pro for any reason

The cost of saving is two seconds. The cost of not saving is measured in hours.

---

[Back to Main Guide](../README.md)

See also: [CRS and Projection](crs-and-projection.md) | [ArcPy Scripting](arcpy-scripting.md) | [Publishing Errors](publishing-errors.md)
