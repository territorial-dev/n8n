# n8n-geospatial

n8n workflow automation platform enhanced with comprehensive geospatial capabilities including QGIS, GDAL, PDAL, and Tippecanoe.

## Features

- **n8n 1.123.0** - Workflow automation platform
- **QGIS 3.44.2** - Geographic Information System with Python bindings
- **GDAL** - Geospatial Data Abstraction Library for raster/vector data
- **PDAL** - Point Data Abstraction Library for LiDAR/point cloud processing
- **Tippecanoe** - Build vector tilesets from large collections of GeoJSON features
- **MinIO Client** - High-performance object storage client
- **GraphicsMagick** - Image processing capabilities
- **Node.js 20.x** - JavaScript runtime

## Quick Start

### Using Docker Compose (Recommended)

```bash
# Clone the repository
git clone https://github.com/your-username/n8n-geospatial.git
cd n8n-geospatial

# Start the service
docker-compose up -d

# Access n8n at http://localhost:5678
```

### Using Docker directly

```bash
# Build the image
docker build -t n8n-geospatial .

# Run the container
docker run -d \
  --name n8n-geospatial \
  -p 5678:5678 \
  -v ./n8n-data:/home/node/.n8n \
  n8n-geospatial
```

## Geospatial Capabilities

### QGIS Integration
- Full QGIS 3.44.2 installation with Python bindings
- Access via Python scripts in n8n workflows
- Support for vector and raster processing

### GDAL Tools
- Command-line tools: `gdalinfo`, `gdal_translate`, `ogr2ogr`
- Python bindings for programmatic access
- Support for 200+ geospatial formats

### PDAL Tools
- Point cloud processing and analysis
- LiDAR data manipulation
- Python bindings for workflow integration

### Tippecanoe
- Build vector tilesets from large collections of GeoJSON features
- Command-line tools: `tippecanoe`, `tile-join`, `tippecanoe-enumerate`, `tippecanoe-decode`
- Optimize GeoJSON data for web mapping applications
- Create efficient MBTiles for Mapbox, MapLibre, and other tile servers

## Other utilities

### MinIO Client
- High-performance object storage client (`mc`)
- S3-compatible API for object storage operations
- Manage buckets, objects, and policies
- Useful for storing and retrieving geospatial data files

## Example Workflows

### 1. Geospatial Data Processing
```python
# In n8n Python node
import qgis.core
from osgeo import gdal

# Load and process vector data
layer = qgis.core.QgsVectorLayer("path/to/data.shp", "layer", "ogr")
# Process data...
```

### 2. Raster Analysis
```python
# GDAL raster operations
dataset = gdal.Open("input.tif")
# Perform analysis...
```

### 3. Point Cloud Processing
```python
# PDAL point cloud operations
import pdal
pipeline = pdal.Pipeline("pipeline.json")
pipeline.execute()
```

### 4. Vector Tile Generation
```bash
# In n8n Execute Command node
# Convert GeoJSON to vector tiles
tippecanoe -o output.mbtiles input.geojson

# Join attributes from CSV to existing tiles
tile-join -o joined.mbtiles -c attributes.csv existing.mbtiles

# Decode tiles back to GeoJSON
tippecanoe-decode output.mbtiles
```

### 5. Object Storage Operations
```bash
# In n8n Execute Command node
# Configure MinIO client
mc alias set myminio https://minio.example.com ACCESS_KEY SECRET_KEY

# Upload geospatial data
mc cp data.geojson myminio/geospatial-bucket/

# Download data
mc cp myminio/geospatial-bucket/data.geojson ./

# List objects
mc ls myminio/geospatial-bucket/
```

## Development

### Building Locally
```bash
# Build for testing
docker build -t n8n-geospatial:test .

# Run tests
./build.sh
```

### Testing
The automated workflow tests:
- QGIS accessibility and Python bindings
- n8n web interface accessibility
- GDAL command-line tools and Python bindings
- PDAL command-line tools and Python bindings
- Tippecanoe command-line tools
- MinIO Client accessibility

### Logs
```bash
# View container logs
docker-compose logs n8n

# Access container shell
docker exec -it n8n-geospatial bash
```

## Upgrading

### n8n
```dockerfile
# In Dockerfile, line 61
RUN npm install -g n8n@1.123.0  # Change version here
```

### QGIS  
```dockerfile
# In Dockerfile, lines 1 and 37
FROM qgis/qgis:3.44.2-bookworm AS builder  # Change version here
FROM qgis/qgis:3.44.2-bookworm             # And here
```

### Rebuild
```bash
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

## n8n 2.x test image

`Dockerfile.n8n@2` is a parallel build for trying n8n 2.x before it replaces the pinned 1.x version above. Same geospatial toolchain (QGIS, GDAL, PDAL, Tippecanoe, LAStools, MinIO Client, pmtiles) as the production `Dockerfile`, pinned to **n8n 2.38.6** (latest 2.x release at time of writing).

### Building and running

```bash
docker build -f "Dockerfile.n8n@2" -t n8n-geospatial:n8n2-test .

docker run -d \
  --name n8n2-test \
  -p 5679:5678 \
  -v ./n8n-data:/home/node/.n8n \
  n8n-geospatial:n8n2-test
```

Use a different host port so it can run alongside the v1 container.

### Why it sets extra environment variables

n8n 2.0 changed several security defaults that would otherwise break this image's Execute-Command-based geospatial workflows. `Dockerfile.n8n@2` bakes in the following compatibility flags, mirrored from a known-working n8n 2.9.1 production deployment (skyforest-infra's `skyport2.yml`) that runs the same kind of QGIS/GDAL/Tippecanoe-via-Execute-Command setup:

| Variable | Value | Why |
| --- | --- | --- |
| `N8N_RUNNERS_ENABLED` | `false` | Disables the new sandboxed task runners so Code/Execute Command nodes keep running in-process against this container's own filesystem and Python env — the sandboxed Python runner doesn't inherit `/opt/conda`'s `PYTHONPATH`, so `qgis.core`/`osgeo` imports would break under it |
| `N8N_ENABLE_EXECUTE_COMMAND` | `true` | The Execute Command node is disabled by default in 2.0; this repo's workflows call `tippecanoe`/`mc`/`lastools`/`qgis` CLIs through it |
| `NODES_EXCLUDE` | `[]` | Also needed to re-enable Execute Command and Local File Trigger, which are excluded by default in 2.0 |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Restores Code node access to `process.env` (e.g. MinIO credentials) |
| `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES` / `N8N_RESTRICT_FILE_ACCESS_TO` | `false` / `/` | Together, stop 2.0's file-access lockdown from blocking reads/writes to mounted input/output volumes |
| `NODE_FUNCTION_ALLOW_EXTERNAL` | `node-fetch,fs,fs-extra` | Whitelists npm packages commonly needed inside Code nodes |
| `N8N_RUN_MIGRATIONS` | `true` | Runs database migrations automatically on startup |

### CI

`.github/workflows/build-and-test-n8n2.yml` builds and smoke-tests this image on push/PR touching the Dockerfile, and on manual dispatch. Images are pushed to the same `paschendale/n8n-geospatial` Docker Hub repo under `n8n2-`-prefixed tags (`n8n2-test`, `n8n2-sha-...`) so they don't collide with the production `latest`/`test` tags.

### Upgrading

```dockerfile
# In Dockerfile.n8n@2
RUN npm install -g n8n@2.38.6  # Change version here
```

## License

MIT License - see [LICENSE](LICENSE) file for details.

