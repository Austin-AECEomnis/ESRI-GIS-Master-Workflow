# ESRI GIS Master Workflow

A field manual documenting a complete ESRI platform workflow from ArcGIS Pro project setup through web publishing, data collection, application development, and live data monitoring. Built from real project work across two Texas Hill Country watershed flood analyses, every lesson here was earned in the field, not the classroom.

**Environment:** ArcGIS Pro 3.x | ArcGIS Online | Survey123 | Experience Builder | ArcGIS Dashboards | ArcPy | GitHub Actions

**Study Areas:** Guadalupe River / Kerr County (Repository 4) | Llano River / Llano and Burnet Counties (Repository 5)

---

## Quick Reference Index

Jump directly to any section or supporting reference file from here.

### README Sections

- [Before You Start](#before-you-start)
- [ArcGIS Pro Project Setup](#arcgis-pro-project-setup)
- [ArcPy Scripting](#arcpy-scripting)
- [Publishing to ArcGIS Online](#publishing-to-arcgis-online)
- [Web Maps](#web-maps)
- [Survey123](#survey123)
- [Experience Builder](#experience-builder)
- [Dashboards and Live Data](#dashboards-and-live-data)
- [Order of Operations](#order-of-operations)
- [Related Repositories](#related-repositories)
- [Supporting Reference Files](#supporting-reference-files)

### Supporting Reference Files

- [Publishing Errors](troubleshooting/publishing-errors.md): error codes 00079, 00230, 00216, 00374, 24195, 000733
- [ArcGIS Pro Project Management](troubleshooting/arcgis-pro-project-management.md): .aprx strategy, project file loss, geodatabase survival, two-project CRS solution
- [ArcPy Scripting](troubleshooting/arcpy-scripting.md): layer distillation, terrain automation, live pipeline integration, scripting discipline
- [CRS and Projection](troubleshooting/crs-and-projection.md): Web Mercator requirement, on-the-fly reprojection, two-.aprx strategy, datum transformations
- [DEM and Terrain Analysis](troubleshooting/dem-and-terrain-analysis.md): clip-first design, buffer distance, runtime performance, null value prevention
- [Web Maps](troubleshooting/webmaps.md): layer configuration, pop-up setup, symbology, downstream app dependencies
- [StoryMaps](troubleshooting/storymaps.md): cascade layout, live layer embedding, breadcrumb navigation, sync behavior
- [Experience Builder](troubleshooting/experience-builder.md): widget rules, Filter widget isolation, pop-up hyperlinks, publishing prerequisites
- [Live Data Pipeline](troubleshooting/live-data-pipeline.md): GitHub Actions YAML, USGS API, Dashboard sequencing, iframe CSP behavior

---

## Before You Start

This section covers everything you should decide, configure, and verify before opening ArcGIS Pro. Every item here represents time lost in the original workflow by discovering it mid-project. Five minutes of planning at this stage prevents hours of untangling later.

### Know Your End Destination Before Touching Any Data

The single most important decision you will make is where this project ends up. If the answer is ArcGIS Online, your entire project must be designed around Web Mercator (EPSG: 3857) from the beginning. This is not something you retrofit at the end. Plan backward from your publishing destination and map every reprojection step in advance.

ArcGIS Pro reprojects layers on the fly for display purposes. The map looks correct. The symbology lines up. Everything appears fine. It is not fine. On-the-fly reprojection masks CRS mismatches that will surface the moment you attempt to publish, producing a cascade of errors that appear to be independent problems but are almost always the same root cause. See [CRS and Projection](troubleshooting/crs-and-projection.md) for the full treatment.

### The Two-.aprx Strategy

Because local terrain analysis and ArcGIS Online publishing have conflicting CRS requirements, the cleanest solution is to maintain two separate .aprx project files from the start.

**Project 1 (Analysis):** Uses your local projected CRS, UTM or State Plane appropriate to your study area. All terrain analysis, DEM processing, and ArcPy scripting runs from this project. This is where the analytical work lives.

**Project 2 (Publishing):** Uses WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857). Final layers are imported here for publishing to Online. No analysis happens in this project. Its only job is to satisfy Online's CRS requirements cleanly.

This approach costs one extra project file and saves hours of projection troubleshooting. See [ArcGIS Pro Project Management](troubleshooting/arcgis-pro-project-management.md) and [CRS and Projection](troubleshooting/crs-and-projection.md) for full detail.

### Organize Your Folder Structure Before Creating Any File

Raw data, processed intermediates, and final outputs belong in separate directories. A script cannot grab a stale intermediate file if it cannot reach it. Name your folders clearly and consistently before the first dataset lands on disk.

Suggested structure:

```
ProjectName/
  RawData/          # downloaded source files, never modified
  Processed/        # reprojected layers, clipped DEMs, intermediate outputs
  Outputs/          # final layers only, what scripts and tools should read from
  Scripts/          # ArcPy .py files
  Publishing/       # the second .aprx and any shapefile ZIPs prepared for Online
```

### Verify Your Source Data CRS Before Downloading

FEMA NFHL layers arrive in geographic NAD83. USGS DEMs arrive in geographic NAD83. ArcGIS Living Atlas layers often arrive in Web Mercator. None of these match each other or your analysis CRS out of the box. Know this in advance so reprojection is a planned step, not an emergency fix.

### Field Name Verification Before Scripting

Display names in the ArcGIS Pro Contents pane are not the same as internal geodatabase field names. Run `arcpy.ListFeatureClasses()` and inspect field names programmatically before writing any script that references them. A script built against a display name will fail silently or produce incorrect output. See [ArcPy Scripting](troubleshooting/arcpy-scripting.md).

---

## ArcGIS Pro Project Setup

ArcGIS Pro is the foundation of the entire ESRI workflow. Every published layer, every web application, every dashboard originates here. Configuration decisions made in Pro propagate downstream into every application you build on top of it. Getting Pro right is not optional.

### Project File Discipline

The .aprx project file is the only component of your project that does not survive deletion gracefully. Your data lives in the geodatabase and is safe. The project file is not. ArcGIS Pro will delete a map tab with a single click of the X button with no confirmation dialog and no undo. This has happened. The recovery procedure works, but it costs time and stress that a disciplined save habit eliminates entirely.

Save frequently and explicitly. Do not rely on autosave. After any significant configuration change, save the project. See [ArcGIS Pro Project Management](troubleshooting/arcgis-pro-project-management.md) for the full recovery procedure if a project file is lost.

### Setting Up the Analysis Project

Open Pro and create a new project in your root project folder. Set the map frame CRS to your local projected coordinate system before adding any data. In the Contents pane, right-click the Map item and set the coordinate system explicitly. Adding data before setting the map CRS allows Pro to inherit a CRS from the first layer added, which may not be what you want.

Add your raw source layers and reproject each one to match the map CRS before running any analysis. Do not rely on on-the-fly reprojection for analytical accuracy.

### Setting Up the Publishing Project

Create the second .aprx in your Publishing/ folder. Set its map frame CRS to WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857) immediately. This project receives only final layers that have been fully processed and are ready to publish. Nothing else goes in here.

Add a basemap before attempting any publish. The web layer and web map publish analyzers require a basemap to be present. You can change or remove the basemap in ArcGIS Online after publishing. It just needs to exist to satisfy the analyzer. Omitting it generates error 00216.

### DEM and Terrain Analysis

Running slope, aspect, or flow accumulation on a full county-wide DEM will take over an hour and may not complete. Always clip the DEM to your study area before running any terrain tools.

The correct clip approach: buffer your flood zone polygon by 3,000 feet, then use that buffer as the Extract by Mask input. A tighter clip caused 48 null values on terrain extraction in the original Guadalupe workflow. The 3,000-foot buffer captures all structure points within the raster extent without processing unnecessary terrain.

This clip-first design, automated in ArcPy, reduced runtime from 60-plus minutes to under 10 minutes. See [DEM and Terrain Analysis](troubleshooting/dem-and-terrain-analysis.md) and [ArcPy Scripting](troubleshooting/arcpy-scripting.md).

---

## ArcPy Scripting

ArcPy is the automation layer that ties the Pro workflow together. It handles the terrain distillation pipeline, the layer preparation process, and feeds directly into the live data components downstream. Scripting discipline here pays dividends across the entire workflow.

For the full scripting reference including the terrain automation pipeline, ExtractMultiValuesToPoints equivalent, live pipeline integration, and all known scripting gotchas, see [ArcPy Scripting](troubleshooting/arcpy-scripting.md).

### Core Scripting Rules

Always call `arcpy.ListFeatureClasses()` before referencing any layer by name in a script. Display names in the Contents pane do not always match internal geodatabase names. One mismatch produces a silent failure or incorrect output with no obvious error message.

Keep raw data, processed intermediates, and final outputs in separate directories. A script operating on the Outputs/ folder should never be able to reach into Processed/ and grab a stale intermediate. Directory separation is the only reliable way to prevent a script from using the wrong version of a layer.

Clear active selections before running any export. An active selection on a feature class causes the 000733 naming conflict error and the export will fail or produce incomplete output.

### Terrain Distillation Pipeline

The ArcPy terrain pipeline automates the clip-first DEM workflow: buffer the flood zone, clip the DEM, run slope and aspect tools, extract multi-value raster attributes to structure point locations, and output a final enriched layer ready for publishing. The full annotated script with all validated parameters is in [ArcPy Scripting](troubleshooting/arcpy-scripting.md).

---

## Publishing to ArcGIS Online

Publishing is where the analytical work becomes a live product. It is also where the largest concentration of errors in this entire workflow occurred. The publishing error cascade (00079, 00230, 00216, 00374, 24195) appears to present multiple independent problems. It does not. Most errors in the cascade share the same root cause and clearing one often clears several simultaneously.

The full error reference with cause, symptom, fix, and sequencing guidance for every error code encountered in this workflow is in [Publishing Errors](troubleshooting/publishing-errors.md). If you are actively blocked by a publish error, go there first.

### The Analyzer

Before publishing, run the Analyze function in the Share As Web Layer or Share As Web Map dialog. The analyzer surfaces errors and warnings before the publish attempt. Errors block publishing. Warnings do not, but some warnings (24195 date field time zone) appear alarming and are safely ignored.

Work through errors in this order: CRS errors first (00079, 00230), then unique ID errors (00374), then basemap errors (00216). Clearing the CRS errors via the map frame fix frequently resolves 00079 and 00230 simultaneously.

### Shapefile ZIP Workaround

The Add Auto Increment ID tool that resolves error 00374 in some contexts is not licensed at ArcGIS Personal Use tier. The workaround is to export the feature class to a shapefile, ZIP all components (.shp, .shx, .dbf, .prj, and others) at the root level of the ZIP with no subfolder, and upload directly to ArcGIS Online as a hosted feature layer. All shapefile components must be at the root level. A subfolder inside the ZIP will cause the upload to fail.

Clear any active selections before exporting to shapefile. An active selection exports only selected features and generates the 000733 naming conflict error.

---

## Web Maps

Web Maps are the hub of the entire ESRI online ecosystem. Every application you build downstream, StoryMaps, Experience Builder, and Dashboards, inherits its layers, symbology, pop-up configuration, and layer visibility directly from the web map. Getting the web map right is not just about the web map. It is about every application that will ever consume it.

Time invested in configuring the web map correctly before building any application downstream is time saved many times over. A misconfigured pop-up field, a layer with incorrect symbology, a visibility range set wrong, these all propagate into every app built on top of it. Fixing them in the web map fixes them everywhere. Fixing them app by app is expensive.

For full configuration guidance, layer ordering, pop-up setup, symbology, and all known sync and dependency issues, see [Web Maps](troubleshooting/webmaps.md).

### Configuration Priorities Before Building Any App

Set field aliases for every layer before touching any downstream application. Experience Builder and StoryMaps inherit these aliases. Raw field names from the geodatabase (OBJECTID, Shape_Area, flood_acc_1) surfacing in a public application is avoidable and unprofessional.

Configure pop-ups completely in the web map. Decide which fields display, in what order, with what labels, and whether any fields link to external content. Pop-up hyperlinks require a specific element type configuration that is not obvious. See [Web Maps](troubleshooting/webmaps.md) for the exact setting.

Set layer visibility ranges intentionally. A layer that is visible at all zoom levels regardless of whether it renders usefully at that scale creates a cluttered experience in every downstream app.

### Sharing and Permissions

Every layer and the web map itself must be shared publicly before any downstream application can display it to an unauthenticated user. This is a common point of confusion. A StoryMap or Experience Builder app shared publicly will show blank map panels or missing layers if the underlying web map or any of its layers are not also shared publicly. Verify sharing at every level: each individual layer, the web map, and the app.

---

## Survey123

Survey123 fits into this workflow as a data collection layer that feeds the platform rather than as a downstream consumer of published data. Once your web maps and layers are established in ArcGIS Online, Survey123 allows field data collection that writes directly back into hosted feature layers, extending the analytical foundation with real-time observations.

### Where Survey123 Fits in the Sequence

Survey123 is presented here after Web Maps because it requires a functioning ArcGIS Online environment to publish and share forms, but it does not depend on Experience Builder or Dashboards being complete first. The order of Survey123, Experience Builder, and Dashboards after Online is flexible and should be driven by your team's priorities. If field data collection is the immediate need, build Survey123 first. If the public-facing application is the priority, build Experience Builder first. If operational monitoring is the goal, move directly to Dashboards. All three can coexist and are not sequentially dependent on each other once Online is established.

### Form Design Considerations

Design your Survey123 form fields to match the field schema of the hosted feature layer they will write to. A field name mismatch between the survey and the target layer produces records that appear to submit successfully but do not populate the intended fields. Verify the target layer's field names via the layer's data tab in ArcGIS Online before building the form.

Keep required fields to the minimum needed for analytical validity. Overly restrictive required field configurations reduce field compliance and produce incomplete records.

---

## Experience Builder

Experience Builder is the application layer where published layers become a configurable, interactive public-facing product. It is more flexible than StoryMaps and more operationally focused than a simple web map viewer, but that flexibility comes with a specific set of rules that are not fully documented in ESRI's official guidance.

For all widget-specific gotchas, Filter widget isolation behavior, hyperlink element configuration, and publishing prerequisites, see [Experience Builder](troubleshooting/experience-builder.md).

### The Hunt VFD Link

The Hunt VFD YouTube link embedded in the Experience Builder pop-up for the Guadalupe workflow is an example of the pop-up hyperlink element type configuration done correctly. A text element in a pop-up cannot be made into a functional hyperlink. It must be configured as a link element specifically. This is a non-obvious distinction in the Experience Builder widget editor that costs time to discover without documentation.

### Filter Widget Isolation

The Filter widget in Experience Builder applies globally to all map panels in the app by default. If your app contains multiple map panels intended to show different filtered views simultaneously, each panel requires its own dedicated Filter widget configured to target that panel only. A single shared Filter widget will override all panels at once, which is rarely the intended behavior.

### Publishing Prerequisites

An Experience Builder app cannot be published publicly if any layer it references is not also shared publicly. This check does not always surface as an obvious error in the Experience Builder interface. If a published app shows blank panels or missing content to an unauthenticated user, the first thing to verify is the sharing permissions of every layer and web map the app references.

---

## Dashboards and Live Data

Dashboards are the final layer of the ESRI workflow and the correct last step in the build sequence. A Dashboard consumes everything that came before it: published web layers, configured web maps, live data feeds. Building a Dashboard before those components are stable produces a monitoring surface that requires constant reconfiguration as upstream elements change.

The live USGS stream gauge pipeline that feeds the Guadalupe workflow Dashboard is documented in full, including the GitHub Actions YAML, the USGS API endpoint, the array index fix, the iframe CSP block resolution, and the new endpoint failure that required rerouting. See [Live Data Pipeline](troubleshooting/live-data-pipeline.md).

### The Dashboard Sequencing Rule

Configure and validate every data source the Dashboard will consume before opening Dashboard builder. A Dashboard built on top of unstable or misconfigured layers requires rebuilding widgets each time an upstream layer changes. Static layers first, live feeds last, Dashboard after both are confirmed stable.

### Attribute Transparency Bug

A known issue exists with attribute transparency in ArcGIS Dashboards where layer transparency set in the web map does not always render correctly inside Dashboard map panels. If a layer that appears correctly transparent in the web map renders as fully opaque inside the Dashboard, set the transparency directly on the layer within the Dashboard map widget rather than relying on the inherited web map setting.

---

## Order of Operations

The sequence below is the recommended build order for this workflow. It is not arbitrary. Each step creates dependencies that the next step requires. Deviating from this order is possible but increases the likelihood of having to undo and redo configuration work.

1. **Folder structure and CRS decisions**: before any data is downloaded
2. **Create both .aprx files**: analysis project in local CRS, publishing project in Web Mercator
3. **Data acquisition and reprojection**: all source layers brought to analysis CRS
4. **DEM clip and terrain analysis**: clip first, then slope, aspect, flow accumulation
5. **ArcPy pipeline**: automate layer distillation, verify outputs against manual results
6. **Prepare publishing layers**: import finals into publishing .aprx, verify Web Mercator CRS
7. **Run the analyzer, resolve all errors**: see [Publishing Errors](troubleshooting/publishing-errors.md)
8. **Publish to ArcGIS Online**
9. **Configure web map**: aliases, pop-ups, symbology, visibility ranges, sharing
10. **Survey123 / Experience Builder / Dashboards**: in the order your project priorities require, all are valid after step 9

The transition from step 9 to step 10 is the only flexible point in the sequence. Once the web map is dialed in, the downstream applications can be built in any order depending on whether field data collection, public application, or operational monitoring is the team's immediate priority. This repo presents Survey123, then Experience Builder, then Dashboards as one logical sequence. Your project may call for a different order. The web map is the prerequisite for all three. That constraint is fixed.

---

## Related Repositories

| Repository | Platform | Study Area | Key Finding |
|---|---|---|---|
| [ArcGIS-Pro-00374-WebPublishing-Workaround](https://github.com/Austin-AECEomnis/ArcGIS-Pro-00374-WebPublishing-Workaround) | ArcGIS Pro / Online | Kerr County | Error 00374 shapefile ZIP workaround for Personal Use tier |
| [ArcGIS-Pro-3.7-Sharing-Maps-to-ArcGIS-Online-Projection-Hack](https://github.com/Austin-AECEomnis/ArcGIS-Pro-3.7-Sharing-Maps-to-ArcGIS-Online-Projection-Hack) | ArcGIS Pro / Online | Guadalupe River | Multi-error cascade (00079, 00230, 00374, 00216, 24195) from map frame projection mismatch: verified single-step fix documented |
| [USGS-Guadalupe-LiveStage](https://github.com/Austin-AECEomnis/USGS-Guadalupe-LiveStage) | GitHub Actions / USGS API | Guadalupe River | Live gauge pipeline stable, USGS endpoint rerouted |
| [ArcPy-Guadalupe-Terrain-Analysis](https://github.com/Austin-AECEomnis/ArcPy-Guadalupe-Terrain-Analysis) | ArcGIS Pro / ArcPy | Guadalupe River / Kerr County | 255 structures in elevated terrain risk zones, 107 matching Camp Mystic signature |
| [QGIS-LlanoRiver-FloodAnalysis](https://github.com/Austin-AECEomnis/QGIS-LlanoRiver-FloodAnalysis) | QGIS / GitHub Pages | Llano River / Llano and Burnet Counties | 23 structures in FEMA X zone directly inundated, mirrors Hunt VFD finding |
| [QGIS-OpenSource-Master-Workflow](https://github.com/Austin-AECEomnis/QGIS-OpenSource-Master-Workflow) | QGIS / PyQGIS / GitHub Pages | Llano River | Companion master workflow repo for the open-source stack |

---

## Supporting Reference Files

| File | Contents |
|---|---|
| [publishing-errors.md](troubleshooting/publishing-errors.md) | All error codes: 00079, 00230, 00216, 00374, 24195, 000733. Cause, symptom, fix, sequence. |
| [arcgis-pro-project-management.md](troubleshooting/arcgis-pro-project-management.md) | .aprx deletion risk, recovery procedure, two-.aprx CRS strategy, geodatabase survival |
| [arcpy-scripting.md](troubleshooting/arcpy-scripting.md) | Terrain distillation pipeline, live pipeline integration, scripting discipline, known gotchas |
| [crs-and-projection.md](troubleshooting/crs-and-projection.md) | Web Mercator requirement, on-the-fly reprojection masking, map frame fix, datum transformations |
| [dem-and-terrain-analysis.md](troubleshooting/dem-and-terrain-analysis.md) | Clip-first design, 3,000-foot buffer, runtime performance, null value prevention |
| [webmaps.md](troubleshooting/webmaps.md) | Layer configuration, pop-up setup, symbology, sharing requirements, downstream app dependencies |
| [storymaps.md](troubleshooting/storymaps.md) | Cascade layout, live layer embedding, StoryMap-to-webmap sync, breadcrumb navigation |
| [experience-builder.md](troubleshooting/experience-builder.md) | Widget rules, Filter widget isolation, hyperlink element type, publishing prerequisites |
| [live-data-pipeline.md](troubleshooting/live-data-pipeline.md) | GitHub Actions YAML, USGS API endpoint, array index fix, iframe CSP block, Dashboard sequencing |

---

*Built by Austin Addington Berlin, AECE Omnis LLC. [LinkedIn](https://linkedin.com/in/austinberlin) | [GitHub](https://github.com/Austin-AECEomnis) | ArcGIS Online portfolio: [aeceomnis.maps.arcgis.com](https://aeceomnis.maps.arcgis.com) | [StoryMap](https://arcg.is/0bGXv02) | [Experience Builder](https://experience.arcgis.com/experience/09c67703781c49ddbc0830655aba9473/) | [Survey123](https://aeceomnis.maps.arcgis.com/home/item.html?id=2505498aa1ce4283ba21b824a49a81ea) | [Dashboard](https://www.arcgis.com/apps/dashboards/ac7607e8f4fa4185a97697b25cd6b181)*

*Companion repository: [QGIS-OpenSource-Master-Workflow](https://github.com/Austin-AECEomnis/QGIS-OpenSource-Master-Workflow)*
