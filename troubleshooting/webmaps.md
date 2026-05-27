# Web Maps

[Back to Main Guide](../README.md)

Web Maps are the hub of the ESRI online ecosystem. Every application built downstream, StoryMaps, Experience Builder, and Dashboards, inherits its layers, symbology, pop-up configuration, field aliases, and visibility settings directly from the web map. Configuration decisions made here propagate into every app. Fixing something in the web map fixes it everywhere. Fixing it app by app is expensive and inconsistent.

---

## Configure the Web Map Before Building Any App

This is not a suggestion. It is the lesson earned by doing it the other way.

Building an Experience Builder app or a StoryMap on top of an incompletely configured web map means that every pop-up field label, every layer visibility setting, every missing field alias surfaces as a problem inside the app that has to be traced back to the web map to fix, then re-verified in the app after the fix propagates. The propagation is not always immediate and is not always reliable without a manual refresh.

The correct sequence is: configure the web map completely, verify it in the web map viewer, then open Experience Builder or StoryMaps. Not the other way around.

### Pre-App Configuration Checklist

Before opening any downstream application builder, verify each of the following in the web map:

- [ ] Field aliases set for every layer, no raw geodatabase field names visible in pop-ups
- [ ] Pop-ups configured completely: correct fields, correct order, correct labels, hyperlinks set to link element type
- [ ] Symbology finalized: colors, size, transparency match the intended presentation
- [ ] Layer visibility ranges set intentionally, not left at default show-at-all-scales
- [ ] Layer draw order correct: labels and points above polygons, polygons above basemap
- [ ] All layers shared publicly if the downstream app will be public-facing
- [ ] Web map itself shared publicly if the downstream app will be public-facing
- [ ] Basemap selected and confirmed appropriate for the study area and application audience

---

## Field Aliases

Raw geodatabase field names are internal identifiers. They are not labels. Names like OBJECTID, Shape_Area, flood_acc_1, and FLDZONE_ID surfacing in a public-facing pop-up communicate nothing useful to a general audience and project an unfinished impression.

Set aliases in the web map layer properties under the Fields tab. Aliases defined here propagate into every app that consumes this web map. An alias set once in the web map does not need to be set again in Experience Builder, StoryMaps, or Dashboards.

**Convention used in this workflow:**
- Functional names, not technical identifiers: "Flood Zone" not "FLDZONE"
- Title case: "Structure Type" not "structure_type"
- Units where relevant: "Elevation (ft)" not "elev_ft"
- No underscores in displayed aliases

---

## Pop-Up Configuration

Pop-ups configured in the web map are the foundation for pop-up behavior in every downstream app. The configuration options available in the web map are more complete than what is available inside Experience Builder or StoryMaps individually.

### Field Visibility and Order

In the pop-up configuration panel, control which fields appear and in what order. Remove fields that have no value to the end user (OBJECTID, Shape_Length, internal processing fields). Order the remaining fields from most important to least important, top to bottom.

### The Hyperlink Element Type

This is the non-obvious configuration that costs time to discover without documentation.

A text element in an ArcGIS Online or Experience Builder pop-up cannot be made into a functional hyperlink by typing a URL into a text field. To create a working hyperlink in a pop-up, the element type must be set to link specifically, not text.

**Steps to add a working hyperlink to a pop-up:**
1. In the web map pop-up configuration, select Add content
2. Choose the link element type, not the text element type
3. In the link element configuration, set the display text (what the user sees) and the URL (where the link goes)
4. For links that use attribute values dynamically, use the field token syntax: `{field_name}` within the URL string

The Hunt VFD YouTube link in the Guadalupe Experience Builder app was configured this way. The text "Hunt VFD Flood Response Video" displays in the pop-up. The link element type makes it a functional clickable link. A text element with the same URL would display as plain unclickable text.

---

## Symbology

Symbology set in the web map carries into downstream apps. Finalizing symbology before building apps prevents having to chase symbol changes across multiple app configurations.

### Transparency

Layer transparency can be set in the web map layer properties. Note the known issue with transparency in Dashboard map panels: transparency set in the web map does not always render correctly inside Dashboard widgets. If a layer that appears correctly transparent in the web map renders as fully opaque in a Dashboard panel, set the transparency directly on the layer within the Dashboard map widget. See [Live Data Pipeline](live-data-pipeline.md) for Dashboard-specific notes.

### Visibility Ranges

The default layer visibility in ArcGIS Online shows layers at all zoom levels. This is rarely the right behavior for detailed point or line layers. A structure point layer showing at continent zoom level renders as an undifferentiated mass and provides no value.

Set visibility ranges to the zoom levels where the layer actually communicates something useful. Structure points and flood zone detail layers in this workflow were configured to become visible at county scale and below.

---

## Sharing and Permissions

Every layer the web map contains must be shared at a permission level that matches the web map's intended audience. The web map itself must also be shared at that level.

**Public apps require public sharing at every level:**
- Each individual hosted feature layer: shared to Everyone
- The web map: shared to Everyone
- Any downstream app: shared to Everyone

A partial sharing configuration produces blank map panels or missing layers for unauthenticated users with no clear error message. The app appears to load but the map content does not appear. The first diagnostic step is always to verify sharing permissions at every level.

**Verification method:** Open a private or incognito browser window and navigate to the app URL. An incognito window has no ArcGIS Online session. If the app loads correctly in incognito, sharing is configured correctly at every level.

---

## Layer Updates and Downstream Sync

When a hosted feature layer is updated, either through a republish from Pro or a direct edit in Online, downstream apps do not always reflect the change immediately.

### Republish Propagation

After republishing a web layer from ArcGIS Pro, the web map may continue to show the previous version until the map is refreshed. Open the web map in the Map Viewer and do a hard refresh (Ctrl+Shift+R in most browsers) to force the map to pull the latest layer version.

### StoryMap Sync Requirement

StoryMap content that references web map layers does not always update automatically when the web map is edited. Specific StoryMap edits, particularly changes to map extent, visible layers, or pop-up behavior within an embedded map panel, may only take effect after opening the StoryMap in the builder and re-saving, even if no edits are made in the StoryMap builder itself.

This was encountered in the Guadalupe workflow: a symbology change made in the web map required opening the StoryMap builder and saving the StoryMap without changes before the updated symbology appeared in the published StoryMap.

See also: [StoryMaps](storymaps.md)

---

## Multiple Web Maps for Different Audiences

A single set of published layers can support multiple web maps configured for different purposes or audiences. In this workflow, the same Guadalupe flood zone and structure layers supported:

- A detailed analysis web map in the publishing project for internal verification
- A public-facing web map configured for the StoryMap and Experience Builder apps
- A monitoring web map used in the Dashboard

Maintaining separate web maps for these purposes means that configuration changes for one audience do not inadvertently affect another. The underlying hosted feature layers are shared. The configuration of what to show, how to show it, and at what zoom levels is specific to each web map.

---

[Back to Main Guide](../README.md)

See also: [Publishing Errors](publishing-errors.md) | [StoryMaps](storymaps.md) | [Experience Builder](experience-builder.md) | [Live Data Pipeline](live-data-pipeline.md)
