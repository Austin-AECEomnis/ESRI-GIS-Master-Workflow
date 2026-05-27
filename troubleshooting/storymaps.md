# StoryMaps

[Back to Main Guide](../README.md)

Configuration, layout decisions, live layer embedding, the web map dependency, and sync behavior encountered during the Guadalupe StoryMap build.

---

## StoryMaps and the Web Map Dependency

A StoryMap does not contain its own layers. It references layers through the web map. Every map panel in a StoryMap is a view into a configured web map. This means the web map must be completely configured before building the StoryMap, and changes to a StoryMap's map content must be made in the web map, not in the StoryMap builder.

This is the most important thing to understand about StoryMaps before starting one.

If a pop-up field label is wrong in a StoryMap, fix the field alias in the web map. If a layer's color is wrong, fix the symbology in the web map. If a layer is not showing at the right zoom level, fix the visibility range in the web map. The StoryMap builder controls narrative layout, text, and media. The web map controls everything about how the map data looks and behaves.

See also: [Web Maps](webmaps.md)

---

## Cascade Layout

The Cascade layout is the StoryMap format used in this workflow. Cascade displays narrative content and map panels as the reader scrolls vertically, with the map updating as the reader moves through sections. It is the most effective format for a story that follows a geographic progression or a sequential analytical narrative.

### Section Structure

Each section in the Cascade layout contains a combination of narrative text, media, and optionally a map panel. The map panel can be configured to display a specific extent, a specific set of visible layers, and a specific zoom level for that section.

**Key decision per section:** What should the map be showing while the reader is reading this section? Set the map extent and visible layers for each section specifically. Leaving the map at the default full extent for every section produces a disjointed reading experience where the map never relates to what the text is describing.

### Setting Map Extent Per Section

In the StoryMap builder, each map panel has an Edit button that opens the map configuration for that panel. From there, navigate the map to the exact extent you want displayed when a reader reaches that section, set the visible layers, and save. The saved extent and layer visibility are locked to that section and will not change unless you edit them.

---

## Embedding Live Layers

The Guadalupe StoryMap embeds live hosted feature layers from ArcGIS Online, including the flood zone polygons, structure points, and the USGS gauge layer. These layers update in the StoryMap automatically as the underlying hosted feature layers are updated.

**Requirement:** Live layers embedded in a StoryMap must be shared publicly. A layer that is shared only to your organization will not display for unauthenticated readers of the published StoryMap.

**Verification:** Open the published StoryMap in a private browser window to confirm all layers are visible to unauthenticated users.

---

## Breadcrumb Navigation to Other Apps

The published ESRI applications in this workflow are connected as a deliberate relay chain, not a hub. Each application contains exactly one outbound link directing the user to the next application in the sequence. The StoryMap links to Experience Builder. Experience Builder links to the Dashboard. The Dashboard is the terminus.

This was an intentional design decision. Embedding multiple outbound links from a single app fragments the user's journey and dilutes the narrative progression. One link per app keeps the reader moving through the story in a purposeful direction, with each application building on what came before it.

The StoryMap's outbound link is embedded as a text hyperlink in the closing narrative section. In the Cascade layout, adding a hyperlink is straightforward: select the text, use the link option in the text formatting toolbar, and paste the destination URL.

**StoryMap outbound link target:**
- Experience Builder app: `https://experience.arcgis.com/experience/09c67703781c49ddbc0830655aba9473/`

The Experience Builder app then carries its own single outbound link to the Dashboard. The Dashboard URL is documented in [Experience Builder](experience-builder.md) and [Live Data Pipeline](live-data-pipeline.md).

---

## Sync Behavior: When Web Map Changes Do Not Propagate Immediately

Changes made to a web map do not always appear immediately in a StoryMap that references that web map. Two sync behaviors were encountered in this workflow:

### Symbology Changes

A symbology change made in the web map layer properties required the following steps before it appeared in the published StoryMap:

1. Make the symbology change in the web map and save the web map
2. Open the StoryMap in the builder
3. Save the StoryMap without making any other changes
4. Publish the StoryMap
5. Hard refresh the published StoryMap in the browser (Ctrl+Shift+R)

Simply saving the web map was not sufficient. The StoryMap builder re-save step was required to force the StoryMap to pull the updated web map configuration.

### Layer Visibility Changes

Changing which layers are visible in a web map section within the StoryMap builder sometimes required a full builder save and republish before the change was reflected in the live published StoryMap. If a layer visibility change is not appearing in the published version after a standard save, republish the StoryMap explicitly from the builder's Publish button rather than relying on autosave.

---

## Known Issues

### Map Panel Extent Reset

When editing an existing StoryMap section's map configuration, navigating the map to set a new extent and then saving occasionally resets the extent to the previous saved state rather than the new one. If this occurs, open the map panel editor again, navigate to the desired extent, and save a second time. The second save typically sticks.

### Layer Loading in Published StoryMap

If a layer appears correctly in the StoryMap builder but does not load in the published version, the most common cause is a sharing permission issue on the layer or the web map. See [Web Maps](webmaps.md) for the verification procedure using a private browser window.

### Media Upload Size

StoryMaps has a file size limit for directly uploaded images and video. For large media files, host the media externally (YouTube for video, an ArcGIS Online item for images) and embed via URL rather than direct upload. The Hunt VFD flood response video in the Guadalupe workflow is embedded as a YouTube link rather than a direct upload for this reason.

---

[Back to Main Guide](../README.md)

See also: [Web Maps](webmaps.md) | [Experience Builder](experience-builder.md) | [Live Data Pipeline](live-data-pipeline.md)
