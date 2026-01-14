# Google Photos to Apple Photos Migration Guide

Complete guide for correctly migrating your photo library from Google Photos to Apple Photos on macOS, preserving all metadata (dates, times, GPS locations).

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step-by-Step Process](#step-by-step-process)
- [Troubleshooting](#troubleshooting)
- [Common Errors Explained](#common-errors-explained)
- [Credits](#credits)

## Overview

When you download your photos from Google Photos via Google Takeout, the metadata (date, time, GPS location) is separated from the photos themselves into `.supplemental-metadata.json` files. This guide helps you merge this metadata back into your photos before importing them into Apple Photos.

### What This Guide Does
- ✅ Merges JSON metadata back into photo files
- ✅ Preserves original dates and GPS locations
- ✅ Fixes photos with dates in filenames (e.g., `2013-06-11 16.19.16.jpg`)
- ✅ Checks for HEIC/JPEG format mismatches
- ✅ Prepares photos for clean import into Apple Photos

### What You'll Need
- macOS computer
- [ExifTool](https://exiftool.org/) installed via Homebrew
- Your Google Takeout download (unzipped)
- ~30-60 minutes (depending on library size)

## Prerequisites

### Install ExifTool

```bash
brew install exiftool
```

Verify installation:
```bash
exiftool -ver
```

### Prepare Your Files

1. Download your library from [Google Takeout](https://takeout.google.com)
2. Select only "Google Photos"
3. Download and unzip all archives
4. Locate the main folder (usually named "Google Photos" or similar)

## Step-by-Step Process

### Step 1: Navigate to Your Photos Folder

```bash
cd /path/to/your/Google-Photos-folder
```

💡 **Tip:** Drag the folder into Terminal to auto-fill the path

### Step 2: Merge JSON Metadata into Photos

This is the main command that copies metadata from `.supplemental-metadata.json` files into your photos:

```bash
exiftool -r -d %s -tagsfromfile "%d/%F.supplemental-metadata.json" \
  "-GPSAltitude<GeoDataAltitude" \
  "-GPSLatitude<GeoDataLatitude" \
  "-GPSLatitudeRef<GeoDataLatitude" \
  "-GPSLongitude<GeoDataLongitude" \
  "-GPSLongitudeRef<GeoDataLongitude" \
  "-Keywords<Tags" \
  "-Subject<Tags" \
  "-Caption-Abstract<Description" \
  "-ImageDescription<Description" \
  "-DateTimeOriginal<PhotoTakenTimeTimestamp" \
  -ext "*" -overwrite_original -progress --ext json .
```

**What it does:**
- `-r` - Recursively processes all subdirectories
- `-d %s` - Uses Unix timestamp format
- `-tagsfromfile` - Copies tags from JSON to image files
- All the GPS and date mappings copy specific metadata fields
- `-overwrite_original` - Modifies files in place (no backup copies)
- `-progress` - Shows progress as it works
- `--ext json` - Excludes JSON files themselves

**Expected output:**
```
29 directories scanned
6518 image files updated
1439 image files unchanged
817 files weren't updated due to errors
```

### Step 3: Restore Dates from Filenames

Many photos have dates in their filenames (e.g., `2013-06-11 16.19.16.jpg`). This command extracts and applies those dates:

```bash
exiftool "-DateTimeOriginal<Filename" -d "%Y-%m-%d %H.%M.%S.%%e" -r .
```

**Expected output:**
```
29 directories scanned
1071 image files updated
6697 image files unchanged
998 files weren't updated due to errors
```

### Step 4: Check for HEIC/JPEG Format Mismatches

Some files may have incorrect extensions. Check if this is an issue:

```bash
exiftool -r . 2>&1 | grep -i "looks more like" | wc -l
```

- **If result = 0:** No problem, skip to Step 5
- **If result > 10:** Fix with the commands below

#### Fix HEIC files that are actually JPEG:
```bash
exiftool -r -ext heic '-filename<${filename;s/\.heic$/.jpg/i}' -if '$mimetype eq "image/jpeg"' .
```

#### Fix JPEG files that are actually HEIC:
```bash
exiftool -r -ext jpg '-filename<${filename;s/\.jpg$/.heic/i}' -if '$mimetype eq "image/heif"' .
```

### Step 5: Import into Apple Photos

1. Open **Photos.app** on your Mac
2. Go to **File > Import** (or press **⌘⇧I**)
3. Select your processed Google Photos folder
4. Click **Review for Import** then **Import All**
5. ⏱️ Wait for import to complete (can take 1-4 hours for large libraries)

### Step 6: Verify Results (Recommended)

After import:
1. Open a random photo in Apple Photos
2. Press **⌘I** to view info
3. Check that date, time, and location are correct
4. View the map view to confirm GPS locations appear

## Troubleshooting

### Multiple Google Takeout Archives

If you have multiple Takeout folders (Takeout-1, Takeout-2, etc.), you can either:

**Option A: Process each separately** (follow Steps 1-6 for each)

**Option B: Process all at once** (recommended)
1. Create a parent folder
2. Move all Takeout folders into it
3. Run Steps 2-4 once from the parent folder (the `-r` flag processes all subdirectories)

### Photos in Wrong Year Folders

Google Takeout sometimes places photos in incorrect year folders (e.g., a 2013 photo in "Photos from 2002"). This happens when Google Photos didn't have correct metadata. The scripts in Step 2 and 3 will fix the metadata regardless of folder location.

### Verify Specific Photo

To check metadata of a specific photo:

```bash
exiftool "/path/to/photo.jpg" | grep -E "Date|GPS"
```

### Fix Single Photo Date Manually

If a specific photo has the wrong date:

```bash
exiftool "-DateTimeOriginal=2013:06:11 16:19:16" "/path/to/photo.jpg"
```

### Search for a Photo by Name

```bash
find . -name "photo-name.jpg"
```

### View JSON Metadata Contents

To see what metadata Google Photos had:

```bash
cat "/path/to/photo.jpg.supplemental-metadata.json"
```

## Common Errors Explained

These errors are **normal** and usually not a problem:

### ✅ "Error opening file - .json"
**Meaning:** The JSON metadata file doesn't exist for this photo.

**Why:** Google Photos didn't have metadata for this photo, or you uploaded it with embedded EXIF data.

**Action:** None needed. Photo will keep its original EXIF data.

### ✅ "No writable tags set from .json"
**Meaning:** The JSON file exists but is empty or contains no usable metadata.

**Why:** Photo had no metadata in Google Photos (e.g., GPS location was 0,0).

**Action:** None needed.

### ✅ "Invalid date/time"
**Meaning:** ExifTool couldn't parse the date from the filename.

**Why:** Filename doesn't contain a date (e.g., `IMG_1234.JPG` instead of `2013-06-11 16.19.16.jpg`).

**Action:** None needed. These photos keep their existing metadata.

### ✅ "IPTCDigest is not current"
**Meaning:** Technical warning about XMP metadata synchronization.

**Action:** None needed. This doesn't affect photo functionality.

### ⚠️ "Nothing changed"
**Meaning:** ExifTool found nothing to update for this file.

**Why:** Usually appears with "Error opening file" - no JSON metadata available.

**Action:** Verify the photo has correct metadata with:
```bash
exiftool "/path/to/photo.jpg" | grep -E "Date|GPS"
```

## Quick Checklist

For each folder you need to process:

- [ ] Navigate to folder (`cd` command)
- [ ] Run Step 2: Merge JSON metadata
- [ ] Run Step 3: Restore dates from filenames
- [ ] Run Step 4: Check HEIC/JPEG mismatches (usually returns 0)
- [ ] Import into Apple Photos
- [ ] Spot-check a few random photos

## Performance Notes

### Typical Processing Times
- **Small library** (< 1,000 photos): 5-10 minutes
- **Medium library** (1,000-5,000 photos): 15-30 minutes
- **Large library** (5,000-10,000 photos): 30-60 minutes
- **Very large library** (> 10,000 photos): 1-2 hours

### Import Times into Apple Photos
- Depends on library size and whether you sync to iCloud
- Typical: 1-4 hours for 8,000 photos
- You can use your Mac during import (keep Photos.app open)

## Alternative: Automated Tool

The original tutorial author created a GUI tool to automate this process:
- [Metadata Fixer](http://metadatafixer.com/)

However, the command-line approach in this guide gives you more control and visibility into the process.

## Credits

This guide is based on:
- [Original tutorial by Mathieu Legault](https://legault.me/post/correctly-migrate-away-from-google-photos-to-icloud)
- [ExifTool by Phil Harvey](https://exiftool.org/)
- Community experience and troubleshooting

## Contributing

Found this helpful? Have improvements or encountered other issues? Feel free to:
- Open an issue
- Submit a pull request
- Share your experience

## License

This guide is provided as-is for educational purposes. Use at your own risk. Always keep backups of your original Google Takeout downloads.

---

**Last Updated:** January 2026

**Tested on:** macOS Sonoma/Sequoia with ExifTool 12.x
