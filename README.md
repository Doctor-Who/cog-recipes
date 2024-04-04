# GDAL CheatSheet for COG (Cloud Optimized GeoTIFF)

:fr: french version >>> <https://github.com/geo2france/cog-tips>

## Requirements

GDAL version >= 3.1

COG driver documentation : https://gdal.org/drivers/raster/cog.html

## Use cases

Improve network performance to display as fast as possible large raster in GIS software. Mainly :
- DSM
- Orthoimagery

Try to streamline disk usage instead of raw raster + raster tiles.
Consider COG is generated from a mosaic from several tiles.

## Single-band Raster: DTM, DSM in ASC format for example

1. Build the VRT

**Linux**

```bash
gdalbuildvrt my_dsm.vrt -addalpha -a_srs EPSG:2154 /dsm_directory/*.asc
```

**Windows**

```batch
gdalbuildvrt.exe C:\dsm\my_dsm.vrt C:\dsm_directory\*.asc -addalpha -a_srs EPSG:2154
```

2. Convert to COG

**Linux**

```bash
gdal_translate input_dsm.vrt my_dsm_output_cog.tif -of COG -co RESAMPLING=NEAREST  -co OVERVIEW_RESAMPLING=NEAREST -co COMPRESS=DEFLATE -co PREDICTOR=2 -co NUM_THREADS=20 -co BIGTIFF=IF_NEEDED
```

**Windows**

```batch
ggdal_translate.exe C:\dsm\input_dsm.vrt C:\dsm\my_dsm_output_cog.tif -of COG -co BLOCKSIZE=512 -co OVERVIEW_RESAMPLING=NEAREST -co COMPRESS=DEFLATE -co PREDICTOR=2 -co NUM_THREADS=20 -co BIGTIFF=IF_NEEDED
```

**RESAMPLING** = resampling method that can be adjusted based on your needs.
Adjust the NUM_THREAD parameter according to your machine.

## 3-band Raster: Orthoimagery

1. Convert each JP2 tile to TIF

Create a directory **0_TIF** and navigate to it containing the JP2 files before running the following command:

**Linux**

```bash
for f in *.jp2; do gdal_translate -of GTiff -co TILED=YES -co BIGTIFF=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=20 -co COMPRESS=ZSTD -co PREDICTOR=2 ${f} ../0_TIF/${f%.*}.tif; done
```

**Windows**

```batch
FOR %%F IN (C:\ortho\jpg2\*.jp2) DO gdal_translate.exe -of GTiff -co TILED=YES -co BIGTIFF=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=ALL_CPUS -co COMPRESS=ZSTD -co PREDICTOR=2 -a_srs EPSG:2154 %%F C:\ortho\0_TIF\%%~nxF.tif
```

**BLOCKXSIZE** and **BLOCKYSIZE** are crucial for the following steps. If you change these values, do the same in step 3.

2. Build the VRT

**Linux**

```bash
gdalbuildvrt my_orthophotography.vrt 0_TIF/*.tif -addalpha -hidenodata -a_srs EPSG:2154
```

**Windows**

```batch
gdalbuildvrt.exe C:\ortho\my_orthophotography.vrt C:\ortho\0_TIF\*.tif -addalpha -hidenodata -a_srs EPSG:2154
```

Combine the options **-addalpha -hidenodata** to set nodata as transparent (avoids black or white borders around the mosaic)

3. Convert to COG

**Linux**

```bash
gdal_translate my_orthophotography.vrt my_orthophotography_output_cog.tif -of COG -co BLOCKSIZE=512 -co OVERVIEW_RESAMPLING=BILINEAR -co COMPRESS=JPEG -co QUALITY=85 -co NUM_THREADS=ALL_CPUS -co BIGTIFF=YES
```

**Windows**

```batch
gdal_translate.exe C:\ortho\my_orthophotography.vrt C:\ortho\my_orthophotography_output_cog.tif -of COG -co BLOCKSIZE=512 -co OVERVIEW_RESAMPLING=BILINEAR -co COMPRESS=JPEG -co QUALITY=85 -co NUM_THREADS=12 -co BIGTIFF=YES
```

## Special Cases

Thanks :pray: to @bchartier for these contributions as well as the commands for Windows.

### Cropping an Image Based on a Contour (before converting to COG)

If you have white or black pixels outside your image that are not designated as _nodata_, you can crop your images based on a contour layer.

**Linux**

```bash
gdalwarp -of GTiff -co TILED=YES -co BIGTIFF=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=12 -co COMPRESS=ZSTD -co PREDICTOR=2 -s_srs EPSG:2154 -t_srs EPSG:2154 -dstalpha -cutline area_of_interest.shp input_image.jp2 image_output.tif
```

**Windows**

```batch
gdalwarp.exe -of GTiff -co TILED=YES -co BIGTIFF=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=12 -co COMPRESS=ZSTD -co PREDICTOR=2 -s_srs EPSG:2154 -t_srs EPSG:2154 -dstalpha -cutline C:\data\area_of_interest.shp C:\ortho\input_image.jp2 C:\ortho\image_output.tif
```

### Converting a JP2 Image to RGBA Encoded TIFF (before converting to COG)

**Linux**

```bash
gdal_translate -of GTiff -co BIGTIFF=YES -co TILED=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=12 -co COMPRESS=ZSTD -co PREDICTOR=2 -b 1 -b 2 -b 3 -b mask -colorinterp red,green,blue,alpha -a_srs EPSG:2154 input_image.jp2 output_image.tif
```

**Windows**

```batch
gdal_translate.exe -of GTiff -co BIGTIFF=YES -co TILED=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=12 -co COMPRESS=ZSTD -co PREDICTOR=2 -b 1 -b 2 -b 3 -b mask -colorinterp red,green,blue,alpha -a_srs EPSG:2154 C:\ortho\input_image.jp2 C:\ortho\output_image.jp2
```

## Best Practices

- JPG compression offers the best weight/performance ratio (lossy compression).
- JP2 is already a compressed format (potentially lossy depending on the codec used), set a moderate compression between 85-90 to avoid degrading the image.
- If you have raw source files (TIF), you can set a stronger compression of 75-80 in the QUALITY parameter.
- The RESAMPLING parameter depends on your choices or users. From our experience, BILINEAR provides the best visual output.

:warning: If you are performing image processing tasks (classification, segmentation, visibility calculation, etc.), opt for NEAREST to prevent alteration of pixel values during resampling. (:pray: Thanks to @vincentsarago for the advice)

## Images with 4 bands or more / 16-bit images

JPG compression is limited to 3 bands (RGB), use DEFLATE (safer and more compatible) or ZSTD (more efficient but may have compatibility issues with other GIS components depending on GDAL compilation methods).

## Known Issues

- Using **gdalbuildvrt** followed by **gdal_translate** is faster than using **gdalwarp**.

- Without proprietary JP2 codecs (ERDAS, KAKADU), you must first decompress each JP2 into TIF and then build the VRT to convert it into COG. Failure to do so may result in artifacts or corrupted pixels on large datasets with the OpenJP2 driver.

- A COG file will be larger than a JP2 or ECW but faster to read and more interoperable without client or server-side proprietary components.

- There are more efficient compression methods than DEFLATE such as ZSTD or JPEG-XL, but not all GIS applications (desktop or server) will be able to read them.