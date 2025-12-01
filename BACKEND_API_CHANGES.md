# Backend API Changes for Browser Fingerprinting

## Overview
The frontend now collects detailed browser fingerprint data and sends it with each answer submission. You need to update your R API to accept and store this data.

## What Data is Being Captured

The frontend now sends these additional fields (all automatic, no user input required):

### Screen & Display
- `screen_width` - Screen width in pixels
- `screen_height` - Screen height in pixels
- `screen_color_depth` - Color depth (bits)
- `screen_pixel_depth` - Pixel depth (bits)

### System Info
- `timezone_offset` - Minutes from UTC
- `platform` - OS platform string
- `hardware_concurrency` - Number of CPU cores
- `device_memory` - RAM in GB (if available)
- `max_touch_points` - Touch screen points

### Browser Config
- `do_not_track` - DNT setting
- `cookie_enabled` - Boolean (cookies enabled)
- `languages` - Comma-separated language list
- `vendor` - Browser vendor
- `product` - Browser product name
- `product_sub` - Product sub-version

### Fingerprints
- `canvas_hash` - Unique canvas rendering signature (50 chars)
- `webgl_vendor` - GPU vendor
- `webgl_renderer` - GPU renderer model

## Changes Needed in Your API

### 1. Update `check_riddle_answer` Function Signature

Change from:
```r
check_riddle_answer <- function(riddle_id, answer, req)
```

To:
```r
check_riddle_answer <- function(riddle_id, answer, req, fingerprint = NULL)
```

### 2. Update Database Schema

Add these columns to your `submissions` table:

```sql
ALTER TABLE submissions ADD COLUMN screen_width INTEGER;
ALTER TABLE submissions ADD COLUMN screen_height INTEGER;
ALTER TABLE submissions ADD COLUMN screen_color_depth INTEGER;
ALTER TABLE submissions ADD COLUMN screen_pixel_depth INTEGER;
ALTER TABLE submissions ADD COLUMN timezone_offset INTEGER;
ALTER TABLE submissions ADD COLUMN platform TEXT;
ALTER TABLE submissions ADD COLUMN hardware_concurrency INTEGER;
ALTER TABLE submissions ADD COLUMN device_memory REAL;
ALTER TABLE submissions ADD COLUMN max_touch_points INTEGER;
ALTER TABLE submissions ADD COLUMN do_not_track TEXT;
ALTER TABLE submissions ADD COLUMN cookie_enabled INTEGER;
ALTER TABLE submissions ADD COLUMN languages TEXT;
ALTER TABLE submissions ADD COLUMN vendor TEXT;
ALTER TABLE submissions ADD COLUMN product TEXT;
ALTER TABLE submissions ADD COLUMN product_sub TEXT;
ALTER TABLE submissions ADD COLUMN canvas_hash TEXT;
ALTER TABLE submissions ADD COLUMN webgl_vendor TEXT;
ALTER TABLE submissions ADD COLUMN webgl_renderer TEXT;
```

Or update your table creation code to include these fields from the start.

### 3. Extract Fingerprint Data

Add this code after extracting the request metadata (after the `session_id` line):

```r
# Extract fingerprint data (sent from frontend)
fp_screen_width <- NULL
fp_screen_height <- NULL
fp_screen_color_depth <- NULL
fp_screen_pixel_depth <- NULL
fp_timezone_offset <- NULL
fp_platform <- NULL
fp_hardware_concurrency <- NULL
fp_device_memory <- NULL
fp_max_touch_points <- NULL
fp_do_not_track <- NULL
fp_cookie_enabled <- NULL
fp_languages <- NULL
fp_vendor <- NULL
fp_product <- NULL
fp_product_sub <- NULL
fp_canvas_hash <- NULL
fp_webgl_vendor <- NULL
fp_webgl_renderer <- NULL

if (!is.null(fingerprint) && is.list(fingerprint)) {
  fp_screen_width <- fingerprint$screen_width
  fp_screen_height <- fingerprint$screen_height
  fp_screen_color_depth <- fingerprint$screen_color_depth
  fp_screen_pixel_depth <- fingerprint$screen_pixel_depth
  fp_timezone_offset <- fingerprint$timezone_offset
  fp_platform <- fingerprint$platform
  fp_hardware_concurrency <- fingerprint$hardware_concurrency
  fp_device_memory <- fingerprint$device_memory
  fp_max_touch_points <- fingerprint$max_touch_points
  fp_do_not_track <- fingerprint$do_not_track
  fp_cookie_enabled <- as.integer(fingerprint$cookie_enabled)
  fp_languages <- fingerprint$languages
  fp_vendor <- fingerprint$vendor
  fp_product <- fingerprint$product
  fp_product_sub <- fingerprint$product_sub
  fp_canvas_hash <- fingerprint$canvas_hash
  fp_webgl_vendor <- fingerprint$webgl_vendor
  fp_webgl_renderer <- fingerprint$webgl_renderer
}
```

### 4. Update INSERT Statement

Change your `dbExecute` INSERT statement from:

```r
dbExecute(
  con,
  "INSERT INTO submissions (riddle_id, answer_given, is_correct, submitted_at,
   ip_address, user_agent, referer, accept_language, x_forwarded_for, origin, session_id)
   VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)",
  params = list(
    riddle_id,
    user_answer,
    as.integer(is_correct),
    format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
    ip_address,
    user_agent,
    referer,
    accept_language,
    x_forwarded_for,
    origin,
    session_id
  )
)
```

To:

```r
dbExecute(
  con,
  "INSERT INTO submissions (
     riddle_id, answer_given, is_correct, submitted_at,
     ip_address, user_agent, referer, accept_language, x_forwarded_for, origin, session_id,
     screen_width, screen_height, screen_color_depth, screen_pixel_depth,
     timezone_offset, platform, hardware_concurrency, device_memory, max_touch_points,
     do_not_track, cookie_enabled, languages, vendor, product, product_sub,
     canvas_hash, webgl_vendor, webgl_renderer
   )
   VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17, $18, $19, $20, $21, $22, $23, $24, $25, $26, $27, $28, $29)",
  params = list(
    riddle_id,
    user_answer,
    as.integer(is_correct),
    format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
    ip_address,
    user_agent,
    referer,
    accept_language,
    x_forwarded_for,
    origin,
    session_id,
    fp_screen_width,
    fp_screen_height,
    fp_screen_color_depth,
    fp_screen_pixel_depth,
    fp_timezone_offset,
    fp_platform,
    fp_hardware_concurrency,
    fp_device_memory,
    fp_max_touch_points,
    fp_do_not_track,
    fp_cookie_enabled,
    fp_languages,
    fp_vendor,
    fp_product,
    fp_product_sub,
    fp_canvas_hash,
    fp_webgl_vendor,
    fp_webgl_renderer
  )
)
```

### 5. Update Your Plumber API Annotation

Make sure the `@param` documentation includes the fingerprint parameter:

```r
#* Check riddle answer (with tracking)
#* @param riddle_id The ID of the riddle
#* @param answer The user's answer
#* @param fingerprint Browser fingerprint data (optional JSON object)
#* @post /riddles/check
#* @serializer unboxedJSON
#* @tag riddles
```

## Testing

After making these changes:

1. Test that submissions still work without fingerprint data (backward compatibility)
2. Test with the updated frontend - fingerprint data should be captured
3. Check your database to see the new columns populated

## Viewing User Fingerprints

To analyze unique users, you can query your database like this:

```sql
SELECT
  COUNT(*) as submission_count,
  MIN(submitted_at) as first_seen,
  MAX(submitted_at) as last_seen,
  ip_address,
  screen_width || 'x' || screen_height as screen_resolution,
  platform,
  hardware_concurrency as cpu_cores,
  device_memory as ram_gb,
  canvas_hash,
  webgl_renderer as gpu
FROM submissions
GROUP BY
  ip_address,
  screen_width,
  screen_height,
  canvas_hash,
  webgl_renderer,
  hardware_concurrency,
  timezone_offset
ORDER BY submission_count DESC;
```

This will show you distinct users even if they're on the same network/IP address!

## Privacy Note

All this data is collected passively from the browser - no user input required. The data helps distinguish between different devices/browsers on the same network, which is exactly what you wanted for your corporate users scenario.

The canvas and WebGL fingerprints are particularly effective at distinguishing different machines, even when they have identical configurations.
