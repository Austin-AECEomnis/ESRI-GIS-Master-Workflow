# Experience Builder

[Back to Main Guide](../README.md)

Widget behavior, Filter widget isolation, the hyperlink element type configuration, publishing prerequisites, and every gotcha encountered during the Guadalupe Experience Builder build.

---

## What Experience Builder Is and Is Not

Experience Builder is a configurable web application builder that sits on top of published ArcGIS Online web maps and layers. It offers more interaction and filtering capability than a StoryMap and more configuration flexibility than the standard web map viewer. It is the right tool when the end user needs to query, filter, and explore data rather than follow a guided narrative.

Experience Builder is not a data editor. It is not a replacement for ArcGIS Pro analysis. It does not modify hosted feature layers. It is a presentation and interaction layer on top of data that has already been published and configured.

---

## The Web Map Prerequisite

Experience Builder apps are built on web maps. The web map must be completely configured before opening Experience Builder. Pop-up field aliases, layer symbology, visibility ranges, and sharing permissions set in the web map carry directly into the Experience Builder app.

Do not attempt to compensate for an incomplete web map by reconfiguring things inside Experience Builder. Configure the web map correctly and let Experience Builder inherit that configuration.

See also: [Web Maps](webmaps.md)

---

## The Hunt VFD Link: Hyperlink Element Type

This is the single most non-obvious configuration in the Experience Builder pop-up setup and the one most likely to produce a working result in the builder that fails in the published app.

**The problem:** A text element in an Experience Builder pop-up displays text. A URL typed into a text element displays as plain unclickable text. There is no way to make a text element into a hyperlink.

**The fix:** The element type must be set to link specifically.

**Steps to add a working hyperlink in an Experience Builder pop-up:**
1. In the Experience Builder widget configuration for the map widget, open the pop-up configuration
2. Select Add element or the equivalent content addition option
3. Choose link as the element type, not text
4. In the link element settings, enter the display text (what the user sees and clicks) in the label field
5. Enter the destination URL in the URL field
6. Save the widget configuration

**The Hunt VFD application:** In the Guadalupe Experience Builder app, the pop-up for the Hunt VFD structure point displays "Hunt VFD Flood Response Video" as a clickable link that opens the YouTube documentation of the VFD flooding. This link is a link element type. If it had been configured as a text element, it would have displayed the URL as unclickable text and provided no value to the user.

---

## Filter Widget Isolation

The Filter widget in Experience Builder applies to map content globally by default. A single Filter widget filters all data sources connected to the app simultaneously. This is rarely the intended behavior when an app contains multiple map panels or when specific filters should affect only specific layers.

**Isolating a Filter widget to a specific panel or data source:**
1. In the Filter widget configuration, locate the data source binding options
2. Set the data source to the specific layer or web map that the filter should affect
3. If multiple panels exist, configure a separate Filter widget for each panel that requires independent filtering

**Why this matters:** In the Guadalupe app, a filter intended to show only structures within a specific flood zone classification was initially configured as a single shared Filter widget. The filter applied to all layers simultaneously, including layers that had no flood zone field to filter against, producing unexpected behavior in multiple panels. Separate Filter widgets per targeted data source resolved this.

---

## Publishing Prerequisites

An Experience Builder app built on privately shared layers will appear to function correctly in the builder preview while displaying blank panels and missing content to unauthenticated users after publishing.

**The check that surfaces this before publishing:**

In the Experience Builder builder, use the Preview button to open the app in a preview mode. Then open the same preview URL in a private browser window without logging in. If content is missing in the private window, a sharing permission issue exists on one or more layers or on the web map itself.

**Resolution:** Identify every hosted feature layer the app references, verify each is shared to Everyone, verify the web map is shared to Everyone, republish the app, and re-test in a private window.

This check should be the final step before calling an Experience Builder app complete.

---

## Widget Reference

### Map Widget

The primary widget for displaying web map content. One map widget per map panel. Multiple map widgets can coexist in a single app for side-by-side comparison layouts.

**Critical first-use warning:** When Experience Builder opens a new app, a default basemap canvas is already present on the layout. Do not drag a Map widget onto this default basemap and attempt to build from there. This causes the widget and the default canvas to conflict visually and behaviorally, producing a map display that does not respond correctly to data source configuration and is difficult to untangle. The correct approach is to add the Map widget to a blank panel area, then immediately set its data source to your web map before configuring anything else. The default basemap canvas is a placeholder, not a starting point for widget placement.

**Key settings:**
- Data source: the web map this widget displays. Set this first, before any other widget configuration.
- Initial extent: the zoom level and center point the map opens to
- Pop-up configuration: inherits from the web map by default, can be modified per widget

### Filter Widget

Provides user-facing filter controls that update the map display based on attribute values. See Filter Widget Isolation section above for critical configuration guidance.

### List Widget

Displays feature attributes as a scrollable list that syncs with the map display. Selecting an item in the list highlights the corresponding feature on the map and vice versa. Useful for structured exploration of a specific feature class.

### Legend Widget

Displays the symbology legend for layers visible in the map widget. Automatically updates as layer visibility changes. Link the Legend widget to the specific Map widget it should describe when multiple Map widgets are present in the app.

---

## Known Issues

### Widget Configuration Loss on Re-link

If a map widget's data source is changed after other widgets have been linked to it, the dependent widgets (Filter, List, Legend) may lose their data source link and require reconfiguration. Change the map widget data source before configuring dependent widgets, not after.

### Pop-up Not Showing in Published App

If a pop-up that works correctly in the Experience Builder preview does not appear in the published app, verify:
1. The layer's pop-up is enabled in the web map (not just in the Experience Builder widget configuration)
2. The layer is shared publicly
3. The map widget in the app is pointing to the correct web map

### App URL After Publishing

The Experience Builder app URL is assigned at first publish and does not change on subsequent republishes. Bookmark or document the URL immediately after the first publish. The URL format is `https://experience.arcgis.com/experience/{item-id}/`.

The Guadalupe Experience Builder app URL: `https://experience.arcgis.com/experience/09c67703781c49ddbc0830655aba9473/`

---

[Back to Main Guide](../README.md)

See also: [Web Maps](webmaps.md) | [StoryMaps](storymaps.md) | [Publishing Errors](publishing-errors.md)
