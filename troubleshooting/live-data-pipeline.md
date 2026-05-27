# Live Data Pipeline

[Back to Main Guide](../README.md)

The GitHub Actions USGS stream gauge pipeline, Dashboard sequencing, iframe CSP behavior, and every issue encountered keeping live data flowing into the Guadalupe workflow dashboard.

---

## Pipeline Overview

The live data pipeline fetches current stream gauge readings from the USGS Water Services API on a scheduled interval and writes them to a hosted feature layer in ArcGIS Online. The Dashboard consumes that hosted feature layer to display current stage readings alongside the static flood analysis layers.

**Components:**
- USGS Water Services API: data source, gauge readings updated every 15 minutes by USGS on their end. Note that our GitHub Actions workflow fetches from the API and writes to Online every 60 minutes. The Dashboard reflects 60-minute granularity, not 15-minute granularity, regardless of how frequently USGS updates the source.
- GitHub Actions: scheduled workflow that fetches the API every 60 minutes and updates the hosted feature layer in Online
- ArcGIS Online hosted feature layer: the bridge between the API and the Dashboard
- ArcGIS Dashboard: the consumer, displays current gauge data alongside static analysis layers

**Repository:** [USGS-Guadalupe-LiveStage](https://github.com/Austin-AECEomnis/USGS-Guadalupe-LiveStage)

---

## GitHub Actions YAML

The pipeline runs on a scheduled GitHub Actions workflow. The YAML file defines the schedule, the Python environment, and the script execution.

### YAML Paste Corruption

The YAML file for this pipeline was corrupted twice during initial setup by paste operations that introduced invisible characters or incorrect indentation. YAML is whitespace-sensitive and GitHub Actions is strict about syntax. A corrupted YAML file fails silently in some cases and with a cryptic exit code error in others.

**Validation approach before committing:**
1. Use a YAML linter (yamllint.com or a VS Code YAML extension) to validate syntax before committing
2. Check indentation visually: GitHub Actions YAML uses 2-space indentation consistently
3. After committing, check the Actions tab in the GitHub repository immediately to verify the workflow was parsed correctly. A workflow that does not appear in the Actions tab was not parsed. A workflow with a red X on the first run has a syntax or configuration error.

### Python Script Paste Corruption

The Python script called by the GitHub Actions workflow was similarly corrupted during initial paste into the GitHub web editor. The GitHub web editor is the primary editing interface when VS Code is not available.

**Known paste artifact:** The string `html` appeared at the start of a code block after a paste operation through the GitHub web editor, corrupting the Python syntax. This artifact is not visible during paste and only surfaces when the script runs and fails with a syntax error.

**Prevention:** After pasting any code into the GitHub web editor, scroll through the entire file looking for unexpected characters, particularly at the start of code blocks and at line beginnings. The `html` artifact specifically appears as `htmlimport` or `html    variable_name` at the top of pasted Python files.

### Exit Code 128

Exit code 128 in GitHub Actions indicates a git operation failure, typically a permission issue or a reference to a branch or remote that does not exist.

**Encountered in this workflow:** Exit code 128 appeared when the Actions workflow attempted to push updated data back to the repository using an improperly configured git credential. Resolution: the pipeline was redesigned to write directly to the ArcGIS Online hosted feature layer via the REST API rather than committing back to the repository. No git push operation is required in the final pipeline design.

---

## USGS Water Services API

### Primary Endpoint

The USGS Water Services API provides real-time and historical gauge data. The endpoint format for the Guadalupe gauge used in this workflow:

```
https://waterservices.usgs.gov/nwis/iv/?format=json&sites={site_number}&parameterCd=00065&siteStatus=active
```

Where `{site_number}` is the USGS gauge site identifier. For the Guadalupe River at Hunt gauge used in this workflow, the site number is documented in the repository.

### New Endpoint Failure

The original USGS API endpoint used in this pipeline was deprecated by USGS and stopped returning data without a clear deprecation notice. The pipeline continued running without error but wrote stale or empty data to the hosted feature layer.

**Diagnosis:** The pipeline appeared healthy in GitHub Actions (green checkmark, no errors) but the Dashboard showed gauge readings that were not updating. The issue was confirmed by calling the original API endpoint directly in a browser and observing that the response was empty or in a changed format.

**Resolution:** USGS published a new endpoint URL. The pipeline script was updated to use the new endpoint. The repository documents the current working endpoint.

**Lesson:** A pipeline that runs without errors is not the same as a pipeline that is producing correct output. Periodic manual verification that the Dashboard gauge reading matches the USGS website reading confirms end-to-end pipeline health.

### Array Index Fix

The USGS API response is a JSON structure where gauge readings are nested inside arrays. The initial pipeline script assumed a specific array index for the current reading that was valid for the original endpoint but incorrect for the new endpoint structure.

**Symptom:** The pipeline ran without errors but wrote incorrect or null gauge values to the hosted feature layer.

**Fix:** The script was updated to navigate the JSON response structure explicitly rather than assuming a fixed array index. The corrected approach iterates the response structure to locate the current value field regardless of its position in the array.

```python
import requests
import json

url = "https://waterservices.usgs.gov/nwis/iv/?format=json&sites=SITE_NUMBER&parameterCd=00065&siteStatus=active"
response = requests.get(url)
data = response.json()

# Navigate the response structure explicitly
try:
    time_series = data["value"]["timeSeries"]
    if time_series:
        values = time_series[0]["values"][0]["value"]
        if values:
            current_stage = float(values[0]["value"])
            current_time = values[0]["dateTime"]
            print(f"Stage: {current_stage} ft at {current_time}")
        else:
            print("No values in response")
    else:
        print("No time series in response")
except (KeyError, IndexError) as e:
    print(f"Response structure error: {e}")
    print(json.dumps(data, indent=2))  # Print full response for diagnosis
```

---

## Dashboard Sequencing

The Dashboard is the final step in the build sequence. It consumes everything that came before it and requires all upstream components to be stable before Dashboard configuration begins.

**Required before opening Dashboard builder:**
- All hosted feature layers published and confirmed updating correctly
- Web map configured with correct layer symbology and pop-ups
- Live pipeline confirmed writing current data to the gauge hosted feature layer
- All layers shared publicly if the Dashboard will be public-facing

Building a Dashboard on top of an unstable or misconfigured data source means rebuilding widgets each time an upstream element changes. Stabilize first, build once.

### Dashboard Indicator Widgets

The stage reading from the USGS gauge pipeline displays in the Dashboard as an Indicator widget. The Indicator widget shows a single value from a specified field in the hosted feature layer.

**Configuration:** In the Indicator widget settings, select the gauge hosted feature layer as the data source, select the stage reading field as the statistic field, and set the statistic to Last (most recent value) rather than Sum, Count, or Average.

### Map Widgets in Dashboard

Dashboard map widgets reference web maps directly. The same web map configuration and layer behavior that applies in Experience Builder and StoryMaps applies here. Layer transparency set in the web map may not render correctly inside Dashboard map panels.

**Known issue:** Layer transparency set in the web map does not always carry into Dashboard map panel widgets. If a layer appears correctly transparent in the web map viewer but renders fully opaque in the Dashboard, set transparency directly on the layer within the Dashboard map widget configuration rather than relying on the inherited web map setting.

---

## iframe CSP Block

The initial attempt to embed the live gauge Dashboard directly inside the Experience Builder app as an iframe was blocked by Content Security Policy (CSP) headers.

**What happened:** ArcGIS Online and ArcGIS Dashboard pages include CSP headers that prevent their content from being loaded inside an iframe hosted on a different domain. The iframe appeared in the Experience Builder layout but displayed a blank panel in the published app.

**Resolution:** Rather than embedding the Dashboard as an iframe, the Experience Builder app links to the Dashboard via a standard hyperlink. The user is directed to the Dashboard as a separate page rather than seeing it embedded. This is the approach documented in the breadcrumb navigation strategy: each application links to the next, creating a connected portfolio without iframe embedding.

---

## GitHub Pages Propagation Timing

GitHub Actions pipeline changes and GitHub Pages updates do not propagate immediately. After committing a change to the pipeline script or the YAML file, allow 2 to 5 minutes before testing the result. A test run initiated immediately after a commit may execute against the previous version of the script.

The GitHub Actions tab in the repository shows the execution log for each run. Check the log to confirm the run used the updated script before diagnosing a failure.

---

[Back to Main Guide](../README.md)

See also: [ArcPy Scripting](arcpy-scripting.md) | [Web Maps](webmaps.md) | [Experience Builder](experience-builder.md)
