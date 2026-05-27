# Publishing Errors Reference

[Back to Main Guide](../README.md)

All error codes encountered across the ESRI GIS workflow, consolidated in one place. Each entry covers what triggered the error, what the symptom looks like, and the exact fix. Errors that share a root cause are grouped together because clearing one often clears several simultaneously.

---

## Quick Reference

| Error Code | Short Description | Fix Summary |
|---|---|---|
| 00079 | Layer has no spatial reference | Fix map frame CRS, not the layer |
| 00230 | Layer CRS does not match map | Fix map frame CRS, not the layer |
| 00216 | No basemap present | Add any basemap before publishing |
| 00374 | Unique IDs not assigned | Use Assign IDs Sequentially in web map dialog, or shapefile ZIP workaround |
| 24195 | Date field has no time zone | Warning only, does not block publishing |
| 000733 | Naming conflict on export | Clear active selection before exporting |

---

## The Error Cascade: Why Multiple Errors Share One Root Cause

Errors 00079, 00230, 00216, and 00374 frequently appear together in the Analyze panel and look like four separate problems. They are not. Most of them are symptoms of the same map frame projection issue. Attempting to fix each error individually wastes significant time. The correct approach is to address the root cause first and re-run the analyzer before touching anything else.

The root cause in this workflow was the map frame CRS. ArcGIS Pro reprojects layers on the fly for display, which means everything looks correct in the map canvas while the underlying CRS mismatch exists. The publish analyzer sees the actual CRS values, not the display reprojection, and flags every layer that does not match Online's requirements.

**Fix the map frame first. Then re-run the analyzer. Then address whatever remains.**

---

## Error 00079: Layer Has No Spatial Reference

**Trigger:** One or more layers in the publishing project do not have a recognized spatial reference that satisfies the web layer or web map analyzer.

**Symptom:** The Analyze panel flags specific layers with error 00079 alongside a message that the layer has no spatial reference or an unrecognized one.

**Common misdiagnosis:** Attempting to reproject individual layers one at a time. This treats the symptom, not the cause.

**Fix:** Right-click the Map item in the Contents pane, not a layer, and select Properties. Navigate to the Coordinate System tab and set the map frame CRS to WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857). Re-run the analyzer. In most cases this clears 00079 and 00230 simultaneously.

**Why this works:** Online's analyzer evaluates layers relative to the map frame CRS. When the map frame is set correctly, layers that are already in Web Mercator or a compatible system pass the check. The fix is at the map level, not the layer level.

See also: [CRS and Projection](crs-and-projection.md)

---

## Error 00230: Layer CRS Does Not Match Map

**Trigger:** A layer's CRS does not match the map frame CRS in the publishing project.

**Symptom:** The Analyze panel flags specific layers with error 00230. Often appears alongside 00079.

**Fix:** Same as 00079. Right-click the Map item in Contents, set the map frame to Web Mercator (EPSG: 3857), re-run the analyzer. Fixing the map frame resolves both 00079 and 00230 in the same operation.

**If 00230 persists after the map frame fix:** The layer itself may be stored in an incompatible CRS that on-the-fly reprojection cannot resolve for publishing purposes. Export the layer to a new feature class explicitly projected to Web Mercator using the Project tool, add the reprojected version to the publishing project, and remove the original.

Known instance: CampMystic_Boundary was created in NAD83 State Plane Texas Central and required explicit reprojection to Web Mercator even after the map frame was corrected.

See also: [CRS and Projection](crs-and-projection.md)

---

## Error 00216: No Basemap Present

**Trigger:** The publishing project has no basemap added to the map.

**Symptom:** The Analyze panel flags error 00216 with a message that no basemap is present. Straightforward to identify.

**Fix:** Add any basemap to the publishing project map. In the Map tab, select Basemap and choose any option. The specific basemap does not matter for publishing purposes. You can change or remove the basemap entirely in ArcGIS Online after the publish completes. The basemap just needs to exist to satisfy the analyzer.

**Note:** This is one of the simpler errors in the cascade but easy to overlook when focused on CRS issues. Always add a basemap to the publishing project as part of initial setup rather than waiting for the analyzer to flag it.

---

## Error 00374: Unique IDs Not Assigned

**Trigger:** One or more feature layers do not have unique IDs assigned, which Online requires to identify individual features for pop-ups, queries, and app interactions.

**Symptom:** The Analyze panel flags error 00374. The fix offered differs depending on which publish dialog you are in.

**Critical distinction:** Two different publish dialogs exist in ArcGIS Pro and they behave differently for this error.

**In the Share As Web Map dialog:** The Analyze panel offers "Assign IDs Sequentially" as a direct fix option. Click it. This resolves 00374 without leaving the dialog.

**In the Share As Web Layer dialog:** The "Assign IDs Sequentially" option is not present. The Add Auto Increment ID geoprocessing tool resolves this in some licensing tiers but is not available at ArcGIS Personal Use tier.

**Workaround for Personal Use tier:**
1. Export the feature class to a shapefile using Export Features
2. Clear any active selection before exporting (see error 000733 below)
3. ZIP all shapefile components at the root level of the ZIP with no subfolder: .shp, .shx, .dbf, .prj, .cpg, and any others present
4. Log in to ArcGIS Online, navigate to Content, select New Item, and upload the ZIP directly as a hosted feature layer
5. Online assigns unique IDs automatically during the hosted feature layer creation process

**Shapefile ZIP requirement:** All components must be at the ZIP root level. A subfolder inside the ZIP causes the upload to fail with no clear error message. Open the ZIP and verify the .shp file is visible immediately on open, not inside a folder.

---

## Error 24195: Date Field Has No Time Zone

**Trigger:** A feature layer contains a date field that does not have a time zone specified.

**Symptom:** The Analyze panel flags 24195 as a warning. It appears alongside other errors and can look alarming in a list of problems.

**Fix:** None required. Error 24195 is a yellow warning, not a red error. It does not block publishing. Leave it and proceed.

**Note:** Do not spend time trying to resolve this warning. It surfaced repeatedly in this workflow and had no effect on the publish outcome or the behavior of the published layer in Online, StoryMaps, Experience Builder, or Dashboards.

---

## Error 000733: Naming Conflict on Feature Class Export

**Trigger:** An active selection is present on a feature class when running Export Features or a similar export operation.

**Symptom:** The export fails with error 000733, a naming conflict message that does not immediately suggest a selection is the cause.

**Fix:** Before running any export, check the feature class for an active selection. In the Contents pane, a selected feature class shows a count next to the layer name. Clear the selection using the Selection tab or by right-clicking the layer and choosing Clear Selection. Re-run the export.

**Why this matters for publishing:** The shapefile ZIP workaround for error 00374 requires a clean export. An active selection on the source feature class will cause the export to include only selected features and generate error 000733. Always verify no selection is active before exporting any layer intended for publishing.

---

## Sequencing Errors Correctly

When multiple errors appear in the Analyze panel simultaneously, work in this order:

1. **Map frame CRS first.** Set the publishing project map frame to Web Mercator and re-run the analyzer. This frequently clears 00079 and 00230 together.
2. **Basemap second.** If 00216 is present, add a basemap and re-run.
3. **Unique IDs third.** Address 00374 via the web map dialog fix or the shapefile ZIP workaround.
4. **Ignore 24195.** It is a warning and does not require action.
5. **Check for active selections before any export.** Prevents 000733 before it occurs.

---

[Back to Main Guide](../README.md)

See also: [CRS and Projection](crs-and-projection.md) | [ArcGIS Pro Project Management](arcgis-pro-project-management.md) | [Web Maps](webmaps.md)
