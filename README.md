GDAL - Geospatial Data Abstraction Library
====

![Build Status](https://github.com/OSGeo/gdal/workflows/Ubuntu%2020.04%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/Ubuntu%2018.04%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/Ubuntu%2018.04%2032bit%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/MacOS%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/Windows%20builds/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/Android%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/ASAN%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/mingw_w64%20build/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/CLang%20Static%20Analyzer/badge.svg)
![Build Status](https://github.com/OSGeo/gdal/workflows/Code%20Checks/badge.svg)
[![Build Status](https://travis-ci.com/OSGeo/gdal.svg?branch=master)](https://travis-ci.com/OSGeo/gdal)
[![Build status](https://ci.appveyor.com/api/projects/status/jtwx0pcr0y01i17p/branch/master?svg=true)](https://ci.appveyor.com/project/OSGeo/gdal)
[![Build Status](https://scan.coverity.com/projects/749/badge.svg?flat=1)](https://scan.coverity.com/projects/gdal)
[![Documentation build Status](https://dev.azure.com/osgeo/gdal/_apis/build/status/OSGeo.gdal.doc?branchName=master&jobName=Documentation)](https://dev.azure.com/osgeo/gdal/_build/latest?definitionId=2&branchName=master&jobName=Documentation)
[![Fuzzing Status](https://oss-fuzz-build-logs.storage.googleapis.com/badges/gdal.svg)](https://bugs.chromium.org/p/oss-fuzz/issues/list?sort=-opened&can=1&q=proj:gdal)

GDAL is an open source library and set of tools for converting and manipulating spatial data. While GDAL itself deals with raster data – GeoTIFFs, ECW, DEMs and the like – its sister project, OGR, deals with vector data formats such as ESRI Shapefiles, MapInfo, KML, or SQL Server geography/geometry data. The OGR2OGR utility is the specific GDAL/OGR component for transforming data between these different vector formats. The scope of these recipes is to work with MSSQL Server, but other databases like Postgres, MySQL and Oracle Spatial are supported. To avoiding running to issues it's strongly recommended to wrap parameter values with double quotes. 

For clarity the recipes have been placed on multiple lines, but need to be passed in as one liners. 
The easiest way to install GDAL is with Anaconda by running: 
```
$conda install -c conda-forge gdal
```
ogr2ogr.exe default install location: C:\OSGeo4W64\bin and GDAL spatial reference data housed in: C:\OSGeo4W64\share\gdal

## Reproject Syntax:
```
ogr2ogr
-f "MSSQLSpatial" **what format are we writing the data to?
"MSSQL:server=MyServer;database=spatial;trusted_connection=yes;"  **where is the destination database? To/From Params Are Positional
"MSSQL:server=MyServer2;database=Staging;trusted_connection=yes;"  **where are we getting the data from?
-sql "SELECT TOP 100 ID, LocationGeometry.STAsBinary()   FROM WRK_AddressPoint" **What SQL do we want to run to pull the data?
-s_srs "EPSG:27700" **what is the source co-ordinate system?
-t_srs "EPSG:4326" **what is the destination co-ordinate system?
-overwrite **empties the destination table and reload
-lco "GEOM_TYPE=geography" ** destination field is geography, DEFAULT is geography if this layer output option is not provided
-lco "GEOM_NAME=LocationGeography" destination field is called 'LocationGeography'
-nln "DEST" ** destination table is called 'DEST'
```

## Reprojecting Ex:
```
ogr2ogr -f "MSSQLSpatial"
"MSSQL:server=MyMSSQLServer; database=DEV_Warehouse; trusted_connection=yes"
"MSSQL:server=MyMSSQLServer; database=DEV_Warehouse; trusted_connection=yes"
-sql "SELECT [Address], shape as shape FROM DEV_Warehouse.dbo.temp_Addresses"
-overwrite **your options are overwrite, update and append
-s_srs "http://spatialreference.org/ref/epsg/4326/"
-t_srs "http://spatialreference.org/ref/epsg/2285/"
**layer creation options
-lco "GEOM_NAME=reproj" 
-lco "OVERWRITE=YES"
-lco "SPATIAL_INDEX=NO" **the default behavior is to create an spatial index
-lco "GEOM_TYPE=geometry"
-nln "Reprojection" 
--config GDAL_HTTP_UNSAFESSL YES **optional config options rather than pulling EPSG definitions that
may or may not live on the system, ogr2ogr has curl installed, and will pull the definitions
from spatialreference.org
```

## Importing ESRI Shapefiles Into SQL Server:
```
ogr2ogr
-f "MSSQLSpatial"
"MSSQL:server=localhost\SQL2012Express;database=Test_DB;trusted_connection=yes;"
"tl_2010_06_zcta510.shp"
-a_srs "ESPG:4269"
-lco "GEOM_TYPE=geography"
-lco "GEOM_NAME=geog4269"
-nln "CaliforniaZCTA"
-progress
--config MSSQLSPATIAL_USE_BCP  **tells GDAL to use native SQL SERVER Native Client driver for Bulk Copy, useful for loading large data sets.  
```


## Exporting from SQL Server to ESRI Shapefile:
```
ogr2ogr 
-f "ESRI Shapefile" **export as ESRI Shapefile
"C:\temp\sqlexport.shp" **name of new shapefile
"MSSQL:server=localhost\sqlexpress;database=tempdb;tables=OGRExportTestTable;trusted_connection=yes;" **table source
```

## Import Shapefile into SQL Server:
```
ogr2ogr -f "MSSQLSpatial"
"MSSQL:server=(your server name here);database=(database name on server to publish shapefile to);trusted_connection=yes" 
"(directory and actual name of shapefile with .shp extension)"
-a_srs "(the ESPG projection code of the shapefile)"
-lco "PRECISION=NO"
-progress **optional paramer, shows progress on import
```

SQL Server will allow you to mix different types of geometry (Points, LineStrings, Polygons) within a single column of geometry or geography data. An ESRI shapefile, in contrast, can only contain a single homogenous type of geometry. To get around this, you'll need to filter by spatial type and export individually. For example:
```
ogr2ogr
-f "ESRI Shapefile"
"C:\temp\sqlexport_linestring.shp" 
"MSSQL:server=localhost\sqlexpress;database=tempdb;trusted_connection=yes;" 
-sql "SELECT shapeid, shapename, shapegeom.STAsBinary(), bufferedshape.ToString(), bufferedshapearea AS area FROM OGRExportTestTable WHERE shapegeom.STGeometryType() = 'LINESTRING'" 
-overwrite 
-lco "SHPT=ARC" 
-a_srs "EPSG:2199"
```

## Import Non-spatial Data: Although OGR was designed for working with spatial data, it can just as easily work with Dbase files and CSV files.
```
ogr2ogr
-f "MSSQLSpatial"
"MSSQL:server=localhost\SQL2012Express;database=Test_DB;trusted_connection=yes;"
somedata.csv
-nln "sometable"
```
