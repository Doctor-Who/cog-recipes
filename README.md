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

## Raster with 1 band : DSM or DEM in ASC format for example

1. Build en VRT

```bash
gdalbuildvrt my_dsm.vrt -addalpha -a_srs EPSG:2154 /dsm_directory/*.asc
```

2. Translate to COG

```bash
gdal_translate my_dsm.vrt my_dsm_output_cog.tif -of COG -co RESAMPLING=NEAREST -co OVERVIEW_RESAMPLING=NEAREST -co COMPRESS=DEFLATE -co PREDICTOR=2 -co NUM_THREADS=20 -co BIGTIFF=IF_NEEDED
```

RESAMPLING method can be adjust depending your usage.
Adjust NUM_THREAD to your hardware.

## Raster with 3 bands : Orthoimagery

1. Convert each JP2 to TIF

Create a **0_TIF** directory and then go to inside the directory that contains JP2 files

```bash
for f in *.jp2; do gdal_translate -of GTiff -co TILED=YES -co BIGTIFF=YES -co BLOCKXSIZE=512 -co BLOCKYSIZE=512 -co NUM_THREADS=20 -co COMPRESS=ZSTD -co PREDICTOR=2 ${f} ../0_TIF/${f%.*}.tif; done
```

**BLOCKXSIZE** and **BLOCKYSIZE** is very important for next step. If you change these values, do same at step 3.

2. Build VRT

```bash
gdalbuildvrt my_orthophotography.vrt 0_TIF/*.tif -addalpha -hidenodata -a_srs EPSG:2154
```

Combine **-addalpha -hidenodata** will set a transparency and avoid black or white no data pixel around your area of interest.

3. Translate to COG

```bash
gdal_translate my_orthophotography.vrt my_orthophotography_output_cog.tif -of COG -co BLOCKSIZE=512 -co OVERVIEW_RESAMPLING=BILINEAR -co COMPRESS=JPEG -co QUALITY=85 -co NUM_THREADS=ALL_CPUS -co BIGTIFF=YES
```

## Good practices

- JPG offer most weight to performance ratio.
- As JP2 is already compress, to avoid image degradation, compression is quite low 85~90.
- If you start from native TIF, then adjust around 75-80 compression QUALITY.
- RESAMPLING method depending of user choice but BILINEAR offer beautiful rendering.

:warning: For image processing (classification, segmentation, viewshed, etc.) use instead NEAREST to avoid pixels values alteration. :warning:
(:pray: thanks to @vincentsarago for this advice)

## 4 Band and up or 16 bits images

Cannot use JPG compression as is is limited to 3 Band (RGB) use DEFLATE (safer - most compatible) or ZSTD (more efficient but less compatible with other GIS application)

## Known issue

- Use **gdalbuildvrt** and then **gdal_translate** is much more efficient than using **gdalwarp**

- Without any proprietary JP2 driver (ERDAS, KAKADU) you must first decompress each file to TIF and then build the COG file.
If you do not, there may be artefacts or corrupted pixels on large dataset with the OpenJP2 driver.

- A COG is bigger file than JP2 or ECW format but completely readable without any proprietary and faster.

- There is more efficient compression type in GDAL than DEFLATE for exemple ZSTD or JXL but some GIS server or GIS desktop software could be not able to read it.