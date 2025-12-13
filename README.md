# GSE Planet Tasks

## Task 1: Dekadal NDVI Composite Pipeline from Sentinel-2 L2A

This Python-based pipeline generates 10-day (dekadal) NDVI composite products from Sentinel-2 L2A satellite imagery. Designed for environmental and agricultural monitoring, it solves a challenge: satellite revisit cycles are irregular (every 5-7 days), but standard reporting periods like the 1st, 11th, and 21st of each month require data on exact dates.

The core of the solution is a **temporal interpolation method**. The pipeline accesses data via the [Copernicus Data Space Ecosystem (CDSE)](https://dataspace.copernicus.eu/), applies cloud and shadow masking using the Scene Classification Layer (SCL), and then uses an Inverse Distance Weighting (IDW) algorithm. This algorithm intelligently estimates the NDVI for the exact target dekadal date by blending the nearest available clear-sky observations from before and after that date, ensuring a continuous, gap-free time series.

## Methodology and Implementation

The process begins by configuring authentication with the CDSE using OAuth 2.0 credentials, as outlined in the official [Authentication guide](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Overview/Authentication.html). A search is performed for all available Sentinel-2 L2A scenes over the Area of Interest (AOI) located in Graz (Austria) within a user-defined period (August 2025).

The code is done by an **Evalscript**—a JavaScript code that runs directly on the CDSE infrastructure. This script was developed by adapting logic and best practices from the community [Sentinel Hub Scripts](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel/) repository and multi-temporal examples found in the [Sentinel Hub Template Scripts](https://github.com/eu-cdse/notebook-samples/blob/main/sentinelhub/custom_scripts/2_multi_temporal_evalscripts.ipynb) notebook. For each pixel, the script processes all scenes in a flexible window around a target dekadal date (e.g., ±5 days). It calculates NDVI, filters out poor-quality pixels (clouds, shadows, cirrus), and collects valid values.

If multiple clear observations are available, a **temporal interpolation** is performed. The final NDVI value for the target date is calculated as a weighted average of the nearby values, where closer observations have a stronger influence. This yields a best-estimate, cloud-free composite for the exact 1st, 11th, or 21st of the month. A final, optional spatial interpolation step fills any remaining small gaps.

## Pipeline

The main workflow is contained within a Jupyter Notebook. To use it:
1.  Ensure your CDSE OAuth credentials (`SH_CLIENT_ID`, `SH_CLIENT_SECRET`) are correctly set in the code.
2.  Define your Area of Interest (AOI) as a geometry polygon.
3.  Specify the target month and year. The code will automatically target the three dekadal dates within that period.
4.  Execute the cells sequentially. The pipeline will handle data discovery, processing, interpolation, and visualization.

The final output is a set of three analysis-ready GeoTIFF files—one for each dekadal period—representing the interpolated NDVI. These can be directly loaded into GIS (QGIS, ArcGIS Pro) software for further analysis.



## Task 2:



## Task 1 References
This project leverages the following resources and services:
*   **CDSE** - [Copernicus Data Space Ecosystem](https://dataspace.copernicus.eu/)
*   **OAuth** - [CDSE Authentication Documentation](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Overview/Authentication.html)
*   **Sentinel Hub Template Scripts** - [Multi-temporal Evalscript Examples](https://github.com/eu-cdse/notebook-samples/blob/main/sentinelhub/custom_scripts/2_multi_temporal_evalscripts.ipynb)
*   **Sentinel Hub Scripts** - [Repository of Custom Evalscripts](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel/)








## Authors

*   [**Juan Fernandez**](mailto:juan.fernandez.sea@gmail.com) - [LinkedIn](https://www.linkedin.com/in/juan-fernandez-martinez/)

## Task 1 Prerequisites

The main pipeline is designed to run in a **Jupyter Notebook**. The development and testing of this project were done using **Anaconda Distribution 2.7.0**, which provides Jupyter Notebook and a robust foundation for scientific computing in Python.

For ease of environment setup and dependency management, we highly recommend using the Anaconda distribution or Miniconda for this project.

## Task 1 Dependencies

The core functionality of this Dekadal NDVI pipeline relies on the following key Python libraries. The recommended way to install them is via `conda` from the `conda-forge` channel.

| Category | Package | Typical Version | License | Purpose / Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Core & Computation** | [Python](https://www.python.org/) | 3.10+ | PSF | Base programming language. |
| | [NumPy](https://numpy.org/) | >=1.22.4 | BSD-3 | Fundamental package for array computations. |
| | [SciPy](https://scipy.org/) | >=1.10.0 | BSD-3 | Advanced mathematics and interpolation routines (`scipy.ndimage`). |
| **Geo-Spatial & CDSE Access** | [SentinelHub](https://pypi.org/project/sentinelhub/) | >=3.10 | MIT | **Essential** for accessing Sentinel-2 data from the Copernicus Data Space Ecosystem (CDSE). |
| | [Rasterio](https://rasterio.readthedocs.io/) | >=1.3.0 | BSD-3 | Reading and writing geospatial raster data (GeoTIFF). |
| **Data Handling & Utilities** | [Pandas](https://pandas.pydata.org/) | >=2.0.0 | BSD-3 | Data structure and analysis for metadata or time series. |
| | [PyYAML](https://pypi.org/project/PyYAML/) or [python-dotenv](https://pypi.org/project/python-dotenv/) | latest | MIT | For managing configuration files and environment variables (e.g., CDSE credentials). |
| **Visualization** | [Matplotlib](https://matplotlib.org/) | >=3.7.0 | Matplotlib | Creating static, animated, and interactive visualizations of NDVI results. |
| **Jupyter** | [Jupyter Notebook](https://jupyter.org/) | latest | BSD-3 | Interactive notebook environment to run the pipeline. |

### Recommended Installation via Conda

You can create a new Conda environment with all the necessary dependencies using the following command:

```bash
conda create -n ndvi_env -c conda-forge python=3.10 sentinelhub rasterio scipy pandas matplotlib jupyter
conda activate ndvi_env
