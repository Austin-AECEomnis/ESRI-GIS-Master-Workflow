# CRS and Projection

[Back to Main Guide](../README.md)

Every coordinate reference system issue encountered across the ESRI workflow, from the on-the-fly reprojection trap to the map frame fix that clears multiple publish errors simultaneously. CRS mismatches were the single largest source of lost time in this workflow.

---

## The Core Problem: On-the-Fly Reprojection Masks Mismatches

ArcGIS Pro reprojects layers on the fly for display purposes. When layers with mismatched CRS values are added to a map, Pro silently reconciles them visually. The map looks correct. Symbology aligns. Spatial relationships appear accurate.

This is a trap.

On-the-fly reprojection is a display convenience, not a data correction. The underlying CRS values stored in each layer remain unchanged. When you attempt to publish to ArcGIS Online, the analyzer evaluates those stored values, not the display, and flags every layer whose stored CRS does not satisfy Online's requirements. The result is a cascade of errors (00079, 00230) that appears to be multiple independent problems but almost always traces back to the same root cause: the map frame CRS was never explicitly set to Web Mercator.

**The rule:** Never trust the map display as confirmation that CRS is correct for publishing. Always verify the map frame CRS and each layer's stored CRS explicitly before attempting to publish.

See also: [Publishing Errors](publishing-errors.md)

---

## ArcGIS Online's CRS Requirement

ArcGIS Online's native CRS is **WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857)**.

This is not the same as WGS 1984 geographic (EPSG: 4326). The distinction matters and confuses people regularly.

| System | EPSG | Type | Use |
|---|---|---|---|
| WGS 1984 Web Mercator Auxiliary Sphere | 3857 | Projected | ArcGIS Online publishing requirement |
| WGS 1984 Geographic | 4326 | Geographic | GPS coordinates, many web services |
| NAD83 Geographic | 4269 | Geographic | FEMA NFHL, USGS DEMs as delivered |
| NAD83 UTM Zone 14N | 26914 | Projected | Local terrain analysis, Hill Country |

Setting the publishing project map frame to EPSG: 4326 (geographic WGS84) will not satisfy Online's requirements. It must be EPSG: 3857 specifically.

---

## The Map Frame Fix

When errors 00079 and 00230 appear in the publish analyzer, the fix is at the map frame level, not the individual layer level.

**Steps:**
1. In the Contents pane of the publishing project, right-click the Map item at the top of the layer list, not a layer
2. Select Properties
3. Navigate to the Coordinate System tab
4. In the search field, type 3857 or Web Mercator
5. Select WGS 1984 Web Mercator Auxiliary Sphere
6. Click OK
7. Re-run the analyzer

In the Guadalupe workflow, this single operation cleared errors 00079 and 00230 simultaneously. Re-running the analyzer after the map frame fix before touching any other setting is the correct next step. Additional errors may also resolve without further action.

---

## Source Data CRS Reference

Every data source used in this workflow arrives in a specific CRS. Knowing this in advance prevents surprises.

| Data Source | Delivered CRS | Notes |
|---|---|---|
| FEMA NFHL flood zones | NAD83 Geographic (EPSG: 4269) | Reproject before analysis |
| USGS DEM (1/3 arc-second) | NAD83 Geographic (EPSG: 4269) | Reproject to projected CRS before terrain tools |
| ArcGIS Living Atlas layers | Usually Web Mercator (EPSG: 3857) | Verify on each layer, not consistent |
| OSM via Living Atlas | Web Mercator (EPSG: 3857) | Generally consistent |
| Manually created features | Inherits from map frame at creation time | Verify immediately after creation |

Running terrain analysis tools (slope, aspect, flow accumulation) on a geographic CRS (decimal degrees) produces unreliable results because the cell size units are degrees, not meters or feet. Always reproject DEMs to a local projected CRS before running any terrain tools.

---

## The Two-.aprx CRS Strategy

Because local terrain analysis and Online publishing have conflicting CRS requirements, the cleanest solution is maintaining two separate .aprx project files: one in the local projected CRS for analysis, one in Web Mercator for publishing.

This is covered in full in [ArcGIS Pro Project Management](arcgis-pro-project-management.md). The CRS implications are:

**Analysis project CRS:** NAD83 UTM Zone 14N (EPSG: 26914) for the Hill Country study areas in this workflow. Set before adding any data. All terrain tools run in this project.

**Publishing project CRS:** WGS 1984 Web Mercator Auxiliary Sphere (EPSG: 3857). Set before adding any layers. Only final processed layers come into this project.

**Reprojection step between projects:** When a final layer is ready to move from the analysis project to the publishing project, use the Project tool (Data Management Tools, Projections and Transformations) to create an explicit Web Mercator copy. Add that copy to the publishing project. Do not drag a UTM layer into the Web Mercator project and rely on on-the-fly reprojection to handle it.

---

## Known CRS Issues from This Workflow

### CampMystic_Boundary Persistence

CampMystic_Boundary was originally created in NAD83 State Plane Texas Central. Even after the map frame was set to Web Mercator and errors 00079 and 00230 cleared for other layers, this layer continued to generate error 00230. The fix was to run the Project tool explicitly on this layer to produce a new feature class in Web Mercator, add the reprojected version to the publishing project, and remove the original.

**Lesson:** On-the-fly reprojection can satisfy the analyzer for most layers but fail on specific layers depending on how they were originally created. If a layer persists with 00230 after the map frame fix, run the Project tool on that layer specifically.

### Datum Transformation Warning

When reprojecting between NAD83 and WGS84-based systems, ArcGIS Pro may prompt for a datum transformation selection. For practical purposes in this workflow (continental US, flood analysis applications), the difference between NAD83 and WGS84 is approximately one meter and the default transformation is acceptable. Select the recommended transformation and proceed.

### Web Mercator in QGIS Transformation Dialog

During the QGIS Llano workflow, the QGIS CRS transformation dialog displayed a Web Mercator option in the transformation path for a layer being reprojected to UTM. This was a diagnostic artifact of QGIS reading the layer's embedded metadata, not an indication that Web Mercator was required or appropriate for the QGIS analysis. The UTM reprojection was correct and the Web Mercator reference in the dialog was safely ignored.

This is documented here because the two workflows (ESRI and QGIS) were developed in parallel and the QGIS observation caused temporary confusion about whether the ESRI Web Mercator requirement had somehow bled into the QGIS workflow. It had not. The ESRI Web Mercator requirement is specific to ArcGIS Online publishing and has no equivalent requirement in the QGIS stack.

---

## CRS Verification Checklist

Before publishing anything to ArcGIS Online, verify each of the following:

- [ ] Publishing project map frame is set to EPSG: 3857, not 4326, not 26914, not anything else
- [ ] Every layer in the publishing project has been explicitly reprojected to Web Mercator, not just visually aligned via on-the-fly reprojection
- [ ] No layer in the publishing project is in a geographic CRS
- [ ] The Analyze panel has been run and CRS-related errors (00079, 00230) are clear
- [ ] Layers created manually inside the publishing project were created after the map frame CRS was set to Web Mercator

---

[Back to Main Guide](../README.md)

See also: [Publishing Errors](publishing-errors.md) | [ArcGIS Pro Project Management](arcgis-pro-project-management.md) | [DEM and Terrain Analysis](dem-and-terrain-analysis.md)
