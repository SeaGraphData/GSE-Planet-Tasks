# GSE Planet Tasks

## Task 1: Dekadal NDVI Composite Pipeline from Sentinel-2 L2A

This Python-based pipeline generates 10-day (dekadal) NDVI composite products from Sentinel-2 L2A satellite imagery. Designed for environmental and agricultural monitoring, it solves a challenge: satellite revisit cycles are irregular (every 5-7 days), but standard reporting periods like the 1st, 11th, and 21st of each month require data on exact dates.

The core of the solution is a **temporal interpolation method**. The pipeline accesses data via the [Copernicus Data Space Ecosystem (CDSE)](https://dataspace.copernicus.eu/), applies cloud and shadow masking using the Scene Classification Layer (SCL), and then uses an Inverse Distance Weighting (IDW) algorithm. This algorithm intelligently estimates the NDVI for the exact target dekadal date by blending the nearest available clear-sky observations from before and after that date, ensuring a continuous, gap-free time series.

### Methodology and Implementation

The process begins by configuring authentication with the CDSE using OAuth 2.0 credentials, as outlined in the official [Authentication guide](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Overview/Authentication.html). A search is performed for all available Sentinel-2 L2A scenes over the Area of Interest ([AOI](https://github.com/SeaGraphData/GSE-Planet-Tasks/blob/main/AOI_for_test.geojson)) located in Graz (Austria) within a user-defined period (August 2025).

The code is done by an **Evalscript**, a JavaScript code that runs directly on the CDSE infrastructure. This script was developed by adapting logic and best practices from the community [Sentinel Hub Scripts](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel/) repository and multi-temporal examples found in the [Sentinel Hub Template Scripts](https://github.com/eu-cdse/notebook-samples/blob/main/sentinelhub/custom_scripts/2_multi_temporal_evalscripts.ipynb) notebook. For each pixel, the script processes all scenes in a flexible window around a target dekadal date (e.g., ±5 days). It calculates NDVI, filters out poor-quality pixels (clouds, shadows, cirrus), and collects valid values.

If multiple clear observations are available, a **temporal interpolation** is performed. The final NDVI value for the target date is calculated as a weighted average of the nearby values, where closer observations have a stronger influence. This yields a best-estimate, cloud-free composite for the exact 1st, 11th, or 21st of the month. A final, optional spatial interpolation step fills any remaining small gaps.

### Pipeline

The main workflow is contained within a Jupyter Notebook. To use it:
1.  Ensure your CDSE OAuth credentials (`SH_CLIENT_ID`, `SH_CLIENT_SECRET`) are correctly set in the code.
2.  Define your Area of Interest (AOI) as a geometry polygon.
3.  Specify the target month and year. The code will automatically target the three dekadal dates within that period.
4.  Execute the cells sequentially. The pipeline will handle data discovery, processing, interpolation, and visualization.

The final output is a set of three analysis-ready GeoTIFF files—one for each dekadal period—representing the interpolated NDVI. These can be directly loaded into GIS (QGIS, ArcGIS Pro) software for further analysis.



### References
This project leverages the following key resources and services:

1. *Inverse Distance Weighting ([IDW Wikipedia](https://en.wikipedia.org/wiki/Inverse_distance_weighting))* — general theory and definition, including weighted averaging and decay with distance.

2. Susanto, P., De Souza, P., & He, J. (2016). *Spatiotemporal Interpolation for Environmental Modelling*. Sensors. This article reviews IDW and its application to time series interpolation contexts. DOI: https://doi.org/10.3390/s16081245

3. Applied Geomatics (2020). *Inverse distance weighting method optimization in the process of digital terrain model creation*. Vol. 12, 397–407 (2020). DOI : https://doi.org/10.1007/s12518-020-00307-6

4.  **CDSE** - [Copernicus Data Space Ecosystem](https://dataspace.copernicus.eu/)
5.   **OAuth** - [CDSE Authentication Documentation](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Overview/Authentication.html)
6.   **Sentinel Hub Template Scripts** - [Multi-temporal Evalscript Examples](https://github.com/eu-cdse/notebook-samples/blob/main/sentinelhub/custom_scripts/2_multi_temporal_evalscripts.ipynb)
7.   **Sentinel Hub Scripts** - [Repository of Custom Evalscripts](https://custom-scripts.sentinel-hub.com/custom-scripts/sentinel/)



## Task 2: Sentinel Hub BYOC Onboarding


This document describes the conceptual workflow used to onboard a large archive (approximately 50 TB) of high-resolution aerial imagery into the Sentinel Hub *Bring Your Own COG* (BYOC) service, with the objective of making the data accessible through the Copernicus Browser within the Copernicus Data Space Ecosystem (CDSE), which operates on CreoDIAS infrastructure.
 
A Mermaid code as well as an html Workflow has been added in this project in order to have a better understsandinfg of the process.



### Phase 1 – Data Assessment and Initial Planning

The onboarding process starts with a thorough understanding of the source dataset, which is currently hosted in a private AWS S3 bucket in the us-west-2 region. This phase is critical to avoid rework later in the process.

The main goal is to characterise the data and identify any constraints that may affect BYOC compatibility. Typical aspects reviewed during this phase include:

- Overall data volume and number of images or tiles  
- Band composition (e.g. RGB, RGB + NIR)  
- Coordinate reference systems in use  
- Bit depth, compression, and NoData handling  
- Presence or absence of overviews  



### Phase 2 – COG Readiness and Standardisation

Once the dataset is understood, the next step is to ensure that all imagery complies with Sentinel Hub BYOC requirements, which are based on Cloud Optimized GeoTIFFs (COGs).

At a conceptual level, this phase focuses on making the data *cloud-native*. This means that each file must support efficient partial reads over HTTP and behave predictably under high-concurrency access patterns.

The key principles applied during this phase are:

- Use of internally tiled GeoTIFFs with consistent block sizes  
- Generation of appropriate overview levels  
- Selection of compression methods optimised for cloud access  
- Harmonisation of band order, naming, and projections  

Rather than validating every file individually, a representative subset is typically tested first. Once the approach is confirmed, the same process is applied consistently across the full archive.



### Phase 3 – Data Transfer into the CDSE Ecosystem

With the imagery prepared in a BYOC-compatible format, the workflow moves on to data transfer. Given the dataset size (50 TB) and the target environment, data locality becomes a key consideration.

For production use, the preferred approach is to move the data into CreoDIAS object storage. This ensures that the imagery is physically close to Sentinel Hub processing services and avoids long-term cross-cloud latency.

In practical terms, this phase involves:

- Selecting an S3-compatible transfer method capable of handling large volumes  
- Executing the transfer as a bulk operation with parallelisation  
- Monitoring transfer integrity and completeness  

The transfer phase is treated as a purely logistical operation. No data transformation is expected at this stage, as all standardisation has already been completed earlier.



### Phase 4 – Object Storage Configuration

Before Sentinel Hub can access the data, the CreoDIAS object storage environment must be configured correctly.

This phase includes:

- Creating a dedicated object storage bucket in an appropriate CreoDIAS region  
- Organising the imagery in a clear and predictable directory structure  
- Applying a bucket policy that grants Sentinel Hub read-only permissions  

The permissions are intentionally minimal and typically limited to listing objects and reading files. This configuration step is small in scope but critical, as misconfigured access rights are a common cause of ingestion issues.


### Phase 5 – BYOC Collection Definition

After storage is ready, a BYOC collection is created within Sentinel Hub. Conceptually, this collection acts as a logical bridge between Sentinel Hub services and externally stored imagery.

During collection creation, the following elements are defined:

- Reference to the CreoDIAS bucket and storage region  
- Available bands and their data types  
- NoData values and basic metadata  

At this stage, the imagery itself is not yet ingested. The collection only defines how Sentinel Hub should interpret and access the data once tiles are registered.



### Phase 6 – Tile Registration and Ingestion

Once the collection exists, individual tiles are registered with it. Each tile corresponds to a specific path in the object storage bucket and represents a spatial unit that Sentinel Hub can index and serve.

Tile ingestion is usually performed in batches and automated to handle large volumes efficiently. For each tile, the ingestion process typically associates:

- The storage path  
- Optional sensing time information  
- A spatial cover geometry (recommended for performance and accuracy)  

Ingestion progress is monitored continuously, allowing failed tiles to be retried without impacting the rest of the dataset.



### Phase 7 – Validation and Functional Testing

After ingestion, the collection is validated to ensure it behaves correctly from a user and API perspective.

Validation typically includes:

- Verifying that tiles are correctly indexed  
- Testing spatial and temporal queries  
- Performing simple rendering or extraction requests via Sentinel Hub APIs  
- Visual inspection in the Copernicus Browser  

Only limited test cases are needed at this stage, as the goal is to confirm correct end-to-end integration rather than exhaustive data quality analysis.



### Phase 8 – Operational Optimisation

Once the collection is functional, attention shifts to performance and operational efficiency. This phase is iterative and may continue throughout the lifetime of the dataset.

Common optimisation measures include:

- Ensuring overview levels are properly generated  
- Using precise cover geometries to minimise unnecessary reads  
- Keeping evaluation scripts lightweight  
- Enabling cached services for frequently accessed layers  

These optimisations help control processing costs and improve responsiveness for end users.



### Phase 9 – Production Enablement

In the final phase, the collection is fully enabled for operational use. This includes making it visible in the Copernicus Browser, adding descriptive metadata, and defining default visualisations.

At this point, the dataset is considered production-ready and can be accessed through Sentinel Hub APIs and tools like any other supported data collection.



### References & Official Documentation
This project leverages the following resources and services:
1. [BYOC API Documentation](https://documentation.dataspace.copernicus.eu/APIs/SentinelHub/Byoc.html)
2. [sentinelhub-py Documentation](https://sentinelhub-py.readthedocs.io/)
3. [CreoDIAS Documentation](https://creodias.docs.cloudferro.com/)




















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
