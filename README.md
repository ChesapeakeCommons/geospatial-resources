# Geospatial resources

Helpful resources to reference when preparing geospatial data for PostgreSQL. In most cases it can be easier to work with data directly on a remote server, just make sure to install `gdal` first.

## Install `gdal` (optional)

Ubuntu <20.04

```
add-apt-repository ppa:ubuntugis/ppa
apt-get update
apt-get install gdal-bin
apt-get install libgdal-dev
```

Ubuntu 20.04

```
apt update
apt install gdal-bin
apt install libpq-dev
apt install libgdal-dev
```

## Install PostGIS

Ubuntu 20.04

```
apt update
apt install postgis postgresql-12-postgis-3
```

```
-- Enable PostGIS (as of 3.0 contains just geometry/geography)
CREATE EXTENSION postgis;
-- enable raster support (for 3+)
CREATE EXTENSION postgis_raster;
-- Enable Topology
CREATE EXTENSION postgis_topology;
-- Enable PostGIS Advanced 3D
-- and other geoprocessing algorithms
-- sfcgal not available with all distributions
CREATE EXTENSION postgis_sfcgal;
-- fuzzy matching needed for Tiger
CREATE EXTENSION fuzzystrmatch;
```

## Acquire data

[United States Census Cartographic Boundary Files](https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html)

[USGS National Hydrography Products](https://www.usgs.gov/national-hydrography/access-national-hydrography-products)

[USGS Geographic Names Information System](https://www.usgs.gov/u.s.-board-on-geographic-names/download-gnis-data)

[USGS Gap Analysis Project](https://www.usgs.gov/programs/gap-analysis-project)

[USGS Data Catalog](https://data.usgs.gov/datacatalog/)

```
curl -O {remote_file}
```

Example: Fetch USGS Watershed Boundary Dataset (WBD) for the entire United States.

```
curl -O https://prd-tnm.s3.amazonaws.com/StagedProducts/Hydrography/WBD/National/GDB/WBD_National_GDB.zip
```

## Process data

[PostGIS: loading data with `ogr2ogr`](https://postgis.net/workshops/postgis-intro/loading_data.html)

**Re-project original shapefile**

```
ogr2ogr {output}.shp -t_srs "EPSG:4326" {input}.shp
```

**Upload source files to remote server**

Run this command from the local directory containing the new shapefile archive. Remove the `--dry-run` flag after confirming file capture.

```
rsync -Paz {source}* {user}@{remote_address}:{remote_directory} --dry-run
```

**Load data into database using `ogr2ogr`**

Before running this command, `cd` into the remote server directory referenced above.

```
ogr2ogr \
-f "PostgreSQL" \
PG:"host=localhost port=5432 user='{user}' password='{password}' dbname='{dbname}'" \
{filename}.shp
-nln {layer} \
-nlt PROMOTE_TO_MULTI \
-lco PRECISION=NO
```

Option definitions

`-nln`: The nln option stands for “new layer name”, and sets the table name that will be created in the target database.

`-nlt`: The nlt option stands for “new layer type”. For shapefile input in particular, the new layer type is often a multi-part geometry, so the system needs to be told in advance to use `MultiPolygon` instead of `Polygon` for the geometry type.

`-lco`: The lco option stands for “layer create option”. `PRECISION` controls how numeric fields are represented in the database. The default when loading a shapefile is to use the database `numeric` type, which is more precise but sometimes harder to work with than simple number types like `integer` and `double precision`. Use `NO` to turn off the `numeric` type.

**Load local CSV into remote database using `psql`**

```
psql -h {host} -d {database} -p {port} -U {user} -W -c "\copy table(columns) from '/path/to/file.csv' csv header"
```

Pass the `-W` flag to present a password prompt.

## Vector operations

### `ogrinfo`

See `ogrinfo` docs [here](https://gdal.org/programs/ogrinfo.html).

**Get vector information**

For all layers:

```
ogrinfo -al {input}.shp -geom=NO
```

The above command lists all features and their attributes. This is useful for identifying attribute names and types for subsequent queries (if needed).

```
OGRFeature(WBDHU6):403
  tnmid (String) = {98F077FE-7446-47CD-91D7-582D8ABE1BD1}
  metasource (String) = (null)
  sourcedata (String) = (null)
  sourceorig (String) = (null)
  sourcefeat (String) = (null)
  loaddate (Date) = 2020/06/25
  referenceg (String) = (null)
  areaacres (Real) = 13170082.500000000000000
  areasqkm (Real) = 53297.480000000003201
  states (String) = AK
  huc6 (String) = 190303
  name (String) = Nushagak River
  globalid (String) = {2FEDF99B-F00A-47F4-9831-545EE5D98E6C}
  shape_Leng (Real) = 33.025103154207081
  shape_Area (Real) = 8.486723012714339
```

For a specific layer:

```
ogrinfo {input}.shp {layer-name} -geom=NO
```

Filter features:

```
ogrinfo -al -where "{attribute} {operator} {pattern}" {input}.shp -geom=NO
```

**Print vector extent**

```
ogrinfo {input}.shp {layer-name} | grep Extent
```
```
Extent: (-179.229655, -14.610193) - (179.856675, 71.439573)
```

**Print count of features with attributes matching a given pattern**

```
ogrinfo {input}.shp {layer-name} | grep "{pattern}" | sort | uniq -c
```

### `ogr2ogr`

See `ogr2ogr` docs [here](https://gdal.org/programs/ogr2ogr.html).

**Reproject vector**

```
ogr2ogr {output}.shp -t_srs "EPSG:4326" {input}.shp
```

**Convert between vector formats**

ESRI shapefile to GeoJSON:

```
ogr2ogr -f "GeoJSON" {output}.json {input}.shp
```

ESRI geodatabase to shapefile (this can take a long time depending on the size of the input):

```
ogr2ogr -f "ESRI Shapefile" {output_directory} {input}.gdb
```

**Read from a zip file**

This assumes that archive.zip is in the current directory. This example just extracts the file, but any ogr2ogr operation should work. It's also possible to write to existing zip files.

```
ogr2ogr -f "GeoJSON" {output}.geojson /path/to/archive.zip/zipped_dir/{input}.geojson
```

**Clip vectors by bounding box**

```
ogr2ogr -f "ESRI Shapefile" {output}.shp {input}.shp -clipsrc {min_x} {min_y} {max_x} {max_y}
```

**Clip one vector by another**

```
ogr2ogr -clipsrc {clipping_polygon}.shp {output}.shp {input}.shp
```

**Query subset of features and extract results to file**

```
ogr2ogr -where "{attribute} {operator} {pattern}" {output}.shp {input}.shp
```

Examples:

```
ogr2ogr -where "STATES like '%VA%' OR STATES like '%WV%'" {output}.shp {input}.shp

ogr2ogr -where "HUC12 in ('020700040902','020700041108','020700040901')" {output}.shp {input}.shp
```

**Select feature attributes**

See: [`SELECT`](https://gdal.org/user/ogr_sql_dialect.html#select)

```
ogr2ogr -f "ESRI Shapefile" -sql "SELECT {column1}, {column2} FROM {layer}" {output}.shp {input}.shp
```
```
ogr2ogr -f "ESRI Shapefile" -sql "SELECT huc6 AS code,name,states FROM WBDHU6" tidy.shp WBDHU6.shp

ogrinfo -al tidy.shp -geom=NO

OGRFeature(layer):index
  code (String) = 031501
  name (String) = Coosa-Tallapoosa
  states (String) = AL,GA,TN
```

**Extract data from a REST API**

```
ogr2ogr -f GeoJSON output.geojson "{url}" OGRGeoJSON
```

## Mapshaper

Mapshaper requires Node.js.

With Node installed, you can install the latest release version of mapshaper using npm. Install with the "-g" flag to make the executable scripts available systemwide.

```
npm install -g mapshaper
```

Read [the wiki](https://github.com/mbloch/mapshaper/wiki/Introduction-to-the-Command-Line-Tool) for full documentation.

Examples:

```
mapshaper {input}.shp -dissolve type -o {output}.shp
```
