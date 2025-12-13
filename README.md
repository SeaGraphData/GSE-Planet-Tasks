# Dekadal NDVI Composite Pipeline from Sentinel-2 L2A

This Python-based pipeline generates seamless, 10-day (dekadal) NDVI composite products from Sentinel-2 L2A satellite imagery. Designed for environmental and agricultural monitoring, it solves a key challenge: satellite revisit cycles are irregular (every 5-7 days), but standard reporting periods like the 1st, 11th, and 21st of each month require data on exact dates.

The core of the solution is a **temporal interpolation method**. The pipeline accesses data via the [Copernicus Data Space Ecosystem (CDSE)](https://dataspace.copernicus.eu/), applies rigorous cloud and shadow masking using the Scene Classification Layer (SCL), and then uses an Inverse Distance Weighting (IDW) algorithm. This algorithm intelligently estimates the NDVI for the exact target dekadal date by blending the nearest available clear-sky observations from before and after that date, ensuring a continuous, gap-free time series.

## Methodology and Implementation

The process begins by configuring authentication with the CDSE using OAuth 2.0 credentials, as outlined in the official [Authentication guide](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Overview/Authentication.html). A search is performed for all available Sentinel-2 L2A scenes over the Area of Interest (AOI) within a user-defined period.

The heavy lifting is done by a custom **Evalscript**—a JavaScript code that runs directly on the CDSE infrastructure. This script was developed by adapting logic and best practices from the community [Sentinel Hub Scripts](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel/) repository and multi-temporal examples found in the [Sentinel Hub Template Scripts](https://github.com/eu-cdse/notebook-samples/blob/main/sentinelhub/custom_scripts/2_multi_temporal_evalscripts.ipynb) notebook. For each pixel, the script processes all scenes in a flexible window around a target dekadal date (e.g., ±5 days). It calculates NDVI, filters out poor-quality pixels (clouds, shadows, cirrus), and collects valid values.

If multiple clear observations are available, a **temporal interpolation** is performed. The final NDVI value for the target date is calculated as a weighted average of the nearby values, where closer observations have a stronger influence. This yields a best-estimate, cloud-free composite for the exact 1st, 11th, or 21st of the month. A final, optional spatial interpolation step fills any remaining small gaps.

## Using the Pipeline

The main workflow is contained within a Jupyter Notebook. To use it:
1.  Ensure your CDSE OAuth credentials (`SH_CLIENT_ID`, `SH_CLIENT_SECRET`) are correctly set in the code.
2.  Define your Area of Interest (AOI) as a geometry polygon.
3.  Specify the target month and year. The code will automatically target the three dekadal dates within that period.
4.  Execute the cells sequentially. The pipeline will handle data discovery, processing, interpolation, and visualization.

The final output is a set of three analysis-ready GeoTIFF files—one for each dekadal period—representing the interpolated NDVI. These can be directly loaded into GIS software for further analysis.

## References
This project leverages the following key resources and services:
*   **CDSE** - [Copernicus Data Space Ecosystem](https://dataspace.copernicus.eu/)
*   **OAuth** - [CDSE Authentication Documentation](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Overview/Authentication.html)
*   **Sentinel Hub Template Scripts** - [Multi-temporal Evalscript Examples](https://github.com/eu-cdse/notebook-samples/blob/main/sentinelhub/custom_scripts/2_multi_temporal_evalscripts.ipynb)
*   **Sentinel Hub Scripts** - [Repository of Custom Evalscripts](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel/)
