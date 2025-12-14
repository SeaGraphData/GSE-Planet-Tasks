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

### Summary

This guide outlines the complete workflow for onboarding a 50TB archive of high-resolution aerial imagery from AWS S3 (us-west-2) to the Sentinel Hub BYOC API for access via Copernicus Browser on the CDSE ecosystem (CreoDIAS). A Mermaid code as well as an html Workflow has been added in this project in order to have a better understsandinfg of the process.

**Key Challenges:**
- Large dataset volume (50TB)
- Cross-cloud provider migration (AWS → CreoDIAS)
- Regional difference (us-west-2 → WAW regions)
- COG format requirements
- Performance optimization for large-scale access



## Phase 1: Data Assessment & Planning

### 1.1 Initial Data Inventory

**Objective:** Understand your current data structure and requirements.

**Actions:**
```bash
# Create inventory of existing data
aws s3 ls s3://your-bucket/ --recursive > inventory.txt

# Analyze file types and sizes
aws s3 ls s3://your-bucket/ --recursive --human-readable --summarize

# Sample file analysis
aws s3 cp s3://your-bucket/sample_image.tif ./sample.tif
gdalinfo sample.tif
```

**Key Questions to Answer:**
- Current format (GeoTIFF, JPEG2000, other)?
- Number of tiles/images?
- Bands per image?
- Bit depth per band?
- Projection system(s)?
- Presence of overviews?
- Current compression?
- NoData values defined?

### 1.2 COG Validation/Conversion Strategy

**COG Requirements:**
- Internal tile size: 256×256 to 2048×2048
- Header size: < 1 MB
- Supported projections: WGS84, WebMercator, UTM zones, Europe LAEA
- Photometric interpretation: 1 (black) or 2 (RGB)
- No polar crossing
- Compression: DEFLATE, ZLIB, ZSTD (recommended), LZW, PACKBITS
- Max 100 bands per file
- Chunky format: max 10 bands per file

**Recommended GDAL Conversion Command:**
```bash
gdal_translate -of COG \
  -co COMPRESS=ZSTD \
  -co BLOCKSIZE=1024 \
  -co RESAMPLING=AVERAGE \
  -co OVERVIEWS=IGNORE_EXISTING \
  -co PREDICTOR=YES \
  -a_nodata 0 \
  input.tif output_cog.tif
```

**For Older GDAL Versions (<3.1):**
```bash
# Step 1: Convert to GeoTIFF
gdal_translate -of GTIFF -a_nodata 0 input.tif intermediate.tif

# Step 2: Build overviews
gdaladdo -r average --config GDAL_TIFF_OVR_BLOCKSIZE 1024 \
  intermediate.tif 2 4 8 16 32

# Step 3: Create COG
gdal_translate \
  -co TILED=YES \
  -co COPY_SRC_OVERVIEWS=YES \
  --config GDAL_TIFF_OVR_BLOCKSIZE 1024 \
  -co BLOCKXSIZE=1024 \
  -co BLOCKYSIZE=1024 \
  -co COMPRESS=DEFLATE \
  -co PREDICTOR=2 \
  intermediate.tif output_cog.tif
```

**Validation:**
```bash
# Validate COG compliance
rio cogeo validate output_cog.tif

# Or use GDAL
gdalinfo output_cog.tif | grep -A 10 "Overviews"
```

---

## Phase 2: Data Transfer Strategy

### 2.1 Transfer Options Analysis

#### Option A: Direct S3-to-S3 Transfer (RECOMMENDED for 50TB)

**Advantages:**
- Fastest for large datasets
- No intermediate storage needed
- Automated and scriptable
- Reduced egress costs with proper planning

**Tools:**
1. **AWS DataSync** (Preferred for large volumes)
   - Set up DataSync agent
   - Configure source: AWS S3 us-west-2
   - Configure destination: CreoDIAS S3
   - Schedule transfer during off-peak hours

2. **Rclone** (Flexible open-source option)
```bash
# Configure rclone
rclone config

# Transfer with multiple threads
rclone sync aws-s3:source-bucket creodias-s3:dest-bucket \
  --transfers 32 \
  --checkers 16 \
  --s3-chunk-size 128M \
  --s3-upload-concurrency 8 \
  --progress
```

#### Option B: CreoDIAS VM as Transfer Hub

**Use Case:** When you need data transformation during transfer

**Setup:**
```bash
# Launch high-bandwidth VM on CreoDIAS
# Choose: Large instance with high network throughput

# Install tools
sudo apt-get update
sudo apt-get install -y awscli rclone gdal-bin python3-pip

# Configure parallel downloads
aws configure set default.s3.max_concurrent_requests 100
aws configure set default.s3.max_queue_size 10000
```

**Transfer Script:**
```bash
#!/bin/bash
# parallel_transfer.sh

SOURCE_BUCKET="s3://aws-source-bucket"
DEST_BUCKET="creodias-destination-bucket"
TEMP_DIR="/mnt/large-volume"

# Download, convert, upload in parallel
for file in $(aws s3 ls $SOURCE_BUCKET --recursive | awk '{print $4}'); do
  {
    aws s3 cp "$SOURCE_BUCKET/$file" "$TEMP_DIR/" && \
    gdal_translate -of COG ... "$TEMP_DIR/$file" "$TEMP_DIR/cog_$file" && \
    aws s3 cp "$TEMP_DIR/cog_$file" s3://$DEST_BUCKET/ && \
    rm "$TEMP_DIR/$file" "$TEMP_DIR/cog_$file"
  } &
  
  # Limit concurrent processes
  if [[ $(jobs -r -p | wc -l) -ge 8 ]]; then wait -n; fi
done

wait
```

#### Option C: Hybrid Approach (Keep Data in AWS)

**Consideration:** May result in higher latency and cross-region data transfer costs

**Only viable if:**
- Testing/proof-of-concept phase
- Cost-benefit analysis favors cross-cloud access
- Data update frequency is high

---

## Phase 3: CreoDIAS Bucket Configuration

### 3.1 Bucket Creation

**Region Selection:**
- **WAW3-2**: Recommended for EO workloads (most computing resources)
- **WAW3-1**: Alternative in same data center
- **WAW4-1**: Third option

**Creation via CLI:**
```bash
# Using AWS CLI with CreoDIAS credentials
aws s3 mb s3://your-byoc-bucket --endpoint-url=https://s3.waw3-2.cloudferro.com

# Or via CreoDIAS dashboard
# Navigate to: Object Storage → Create Bucket
```

### 3.2 Bucket Policy Configuration

**Required Policy Structure:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Sentinel Hub BYOC Access",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::ddf4c98b5e6647f0a246f0624c8341d9:root"
            },
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::your-byoc-bucket",
                "arn:aws:s3:::your-byoc-bucket/*"
            ]
        }
    ]
}
```

**Apply Policy:**
```bash
# Save policy to file: bucket_policy.json
aws s3api put-bucket-policy \
  --bucket your-byoc-bucket \
  --policy file://bucket_policy.json \
  --endpoint-url=https://s3.waw3-2.cloudferro.com
```

**Python Script Alternative:**
```python
import boto3

s3_client = boto3.client(
    's3',
    endpoint_url='https://s3.waw3-2.cloudferro.com',
    aws_access_key_id='YOUR_ACCESS_KEY',
    aws_secret_access_key='YOUR_SECRET_KEY'
)

bucket_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "Sentinel Hub BYOC Access",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::ddf4c98b5e6647f0a246f0624c8341d9:root"},
        "Action": ["s3:GetBucketLocation", "s3:ListBucket", "s3:GetObject"],
        "Resource": [
            f"arn:aws:s3:::your-byoc-bucket",
            f"arn:aws:s3:::your-byoc-bucket/*"
        ]
    }]
}

s3_client.put_bucket_policy(
    Bucket='your-byoc-bucket',
    Policy=json.dumps(bucket_policy)
)
```

---

## Phase 4: BYOC Collection Configuration

### 4.1 Collection Creation via API

**Prerequisites:**
- Sentinel Hub OAuth2 client credentials
- Python with `sentinelhub` library

**Installation:**
```bash
pip install sentinelhub
```

**Configuration:**
```python
from sentinelhub import SHConfig

config = SHConfig()
config.sh_client_id = 'YOUR_CLIENT_ID'
config.sh_client_secret = 'YOUR_CLIENT_SECRET'
config.sh_base_url = 'https://sh.dataspace.copernicus.eu'
config.save()
```

**Create Collection:**
```python
import requests
from sentinelhub import SHConfig

config = SHConfig()
token = config.sh_auth_token

headers = {
    'Authorization': f'Bearer {token}',
    'Content-Type': 'application/json'
}

collection_data = {
    "name": "Aerial Imagery 50TB Collection",
    "s3Bucket": "PROJECT_ID:your-byoc-bucket",  # For CreoDIAS
    "storageId": "waw3-2",
    "additionalData": {
        "bands": {
            "Red": {
                "source": "RGB",
                "bandIndex": 1,
                "bitDepth": 16,
                "sampleFormat": "UINT"
            },
            "Green": {
                "source": "RGB",
                "bandIndex": 2,
                "bitDepth": 16,
                "sampleFormat": "UINT"
            },
            "Blue": {
                "source": "RGB",
                "bandIndex": 3,
                "bitDepth": 16,
                "sampleFormat": "UINT"
            },
            "NIR": {
                "source": "NIR",
                "bandIndex": 1,
                "bitDepth": 16,
                "sampleFormat": "UINT"
            }
        },
        "noDataValue": 0
    }
}

response = requests.post(
    'https://sh.dataspace.copernicus.eu/api/v1/byoc/collections',
    headers=headers,
    json=collection_data
)

collection_id = response.json()['data']['id']
print(f"Collection created with ID: {collection_id}")
```

### 4.2 Band Configuration Best Practices

**Band Naming Conventions:**
- Use valid JavaScript identifiers
- Avoid reserved keywords
- Use descriptive names: Red, Green, Blue, NIR, SWIR1, etc.
- Maintain consistency across all tiles

**Sample Format Options:**
- `UINT`: Unsigned integers (most common for imagery)
- `INT`: Signed integers
- `FLOAT`: Floating point (for derived products)

**Multi-file Organization:**
```
tile_001/
  ├── RGB_(BAND).tif       # Contains Red, Green, Blue
  ├── NIR_(BAND).tif       # Contains NIR band
  └── mask_(BAND).tif      # Contains quality/cloud mask

Path in BYOC: "tile_001/(BAND)_(BAND).tif"
```

---

## Phase 5: Tile Organization & Cover Geometries

### 5.1 Directory Structure

**Recommended Structure:**
```
your-byoc-bucket/
├── tiles/
│   ├── tile_001/
│   │   ├── RGB.tif
│   │   ├── NIR.tif
│   │   └── metadata.json
│   ├── tile_002/
│   │   ├── RGB.tif
│   │   ├── NIR.tif
│   │   └── metadata.json
│   └── ...
└── cover_geometries/
    ├── tile_001.geojson
    ├── tile_002.geojson
    └── ...
```

### 5.2 Cover Geometry Generation

**Method 1: GDAL trace_outline**
```bash
#!/bin/bash
# generate_cover_geometry.sh

INPUT_FILE="$1"
OUTPUT_WKT="$2"

# Generate outline
gdal_trace_outline "$INPUT_FILE" \
  -out-cs en \
  -wkt-out "$OUTPUT_WKT"
```

**Method 2: Simplified with GDAL Python**
```python
from osgeo import gdal, ogr, osr
import json

def create_cover_geometry(raster_path):
    """Generate cover geometry from raster footprint"""
    ds = gdal.Open(raster_path)
    if not ds:
        raise ValueError(f"Cannot open {raster_path}")
    
    # Get geotransform and size
    gt = ds.GetGeoTransform()
    cols = ds.RasterXSize
    rows = ds.RasterYSize
    
    # Calculate corners
    corners = [
        (gt[0], gt[3]),
        (gt[0] + cols * gt[1], gt[3]),
        (gt[0] + cols * gt[1], gt[3] + rows * gt[5]),
        (gt[0], gt[3] + rows * gt[5]),
        (gt[0], gt[3])
    ]
    
    # Get projection
    srs = osr.SpatialReference()
    srs.ImportFromWkt(ds.GetProjection())
    epsg_code = srs.GetAttrValue('AUTHORITY', 1)
    
    # Create GeoJSON
    geometry = {
        "type": "Polygon",
        "coordinates": [corners],
        "crs": {
            "type": "name",
            "properties": {
                "name": f"urn:ogc:def:crs:EPSG::{epsg_code}"
            }
        }
    }
    
    ds = None
    return geometry

# Usage
cover_geom = create_cover_geometry("tile_001/RGB.tif")
print(json.dumps(cover_geom, indent=2))
```

**Simplification (if needed):**
```python
from shapely.geometry import shape
from shapely.wkt import dumps

def simplify_geometry(geojson_geom, tolerance=10):
    """Simplify geometry to reduce point count"""
    geom = shape(geojson_geom)
    simplified = geom.simplify(tolerance, preserve_topology=True)
    
    # Convert back to GeoJSON format
    return {
        "type": "Polygon",
        "coordinates": [list(simplified.exterior.coords)],
        "crs": geojson_geom.get("crs")
    }
```

---

## Phase 6: Automated Tile Ingestion

### 6.1 Batch Ingestion Script

**Complete Python Implementation:**
```python
import requests
import json
from pathlib import Path
from sentinelhub import SHConfig
from datetime import datetime
import time

class BYOCIngester:
    def __init__(self, collection_id):
        self.config = SHConfig()
        self.collection_id = collection_id
        self.base_url = 'https://sh.dataspace.copernicus.eu/api/v1/byoc'
        self.headers = {
            'Authorization': f'Bearer {self.config.sh_auth_token}',
            'Content-Type': 'application/json'
        }
    
    def ingest_tile(self, tile_data):
        """Ingest a single tile"""
        url = f'{self.base_url}/collections/{self.collection_id}/tiles'
        
        payload = {
            "path": tile_data['path'],
            "sensingTime": tile_data['sensing_time'],
            "coverGeometry": tile_data.get('cover_geometry')
        }
        
        try:
            response = requests.post(url, headers=self.headers, json=payload)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            print(f"Error ingesting {tile_data['path']}: {e}")
            print(f"Response: {e.response.text}")
            return None
    
    def check_tile_status(self, tile_id):
        """Check ingestion status"""
        url = f'{self.base_url}/collections/{self.collection_id}/tiles/{tile_id}'
        response = requests.get(url, headers=self.headers)
        return response.json()['data']['status']
    
    def batch_ingest(self, tiles_list, batch_size=100):
        """Ingest tiles in batches"""
        total = len(tiles_list)
        successful = 0
        failed = []
        
        for i in range(0, total, batch_size):
            batch = tiles_list[i:i+batch_size]
            print(f"Processing batch {i//batch_size + 1} ({i+1}-{min(i+batch_size, total)} of {total})")
            
            for tile_data in batch:
                result = self.ingest_tile(tile_data)
                if result:
                    successful += 1
                    print(f"✓ Ingested: {tile_data['path']}")
                else:
                    failed.append(tile_data['path'])
                
                # Rate limiting - be respectful
                time.sleep(0.5)
            
            print(f"Batch complete. Success: {successful}, Failed: {len(failed)}")
        
        return successful, failed

# Usage Example
if __name__ == "__main__":
    collection_id = "YOUR_COLLECTION_ID"
    
    # Prepare tile list
    tiles = []
    for i in range(1, 1001):  # Example: 1000 tiles
        tile_data = {
            'path': f'tiles/tile_{i:04d}/(BAND).tif',
            'sensing_time': f'2024-{i%12+1:02d}-15T10:00:00Z',
            'cover_geometry': None  # Or load from file
        }
        tiles.append(tile_data)
    
    # Ingest
    ingester = BYOCIngester(collection_id)
    successful, failed = ingester.batch_ingest(tiles, batch_size=50)
    
    print(f"\n=== Ingestion Summary ===")
    print(f"Total tiles: {len(tiles)}")
    print(f"Successful: {successful}")
    print(f"Failed: {len(failed)}")
    
    if failed:
        print("\nFailed tiles:")
        for path in failed:
            print(f"  - {path}")
```



## Phase 9: Production Deployment

### 9.1 Copernicus Browser Integration

1. Enable collection in Dashboard
2. Configure visualization parameters
3. Set default rendering
4. Add collection description




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
