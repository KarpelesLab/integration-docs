# Media Variations API Documentation

This document explains how to use media variations when making API requests that return Media objects (images, audio, or video).

## Overview

When making GET or POST requests to API endpoints that return Media objects, you can include variation parameters to receive transformed versions of the media. The system will automatically generate and include URLs for the requested variations in the response.

## How It Works

Media variations are requested by adding specific parameters to your API request:

- `image_variation` - for image transformations
- `audio_variation` - for audio transformations

These parameters can be included in any API endpoint that returns `Media_Image` or `Media_Audio` objects.

## Image Variations

When requesting an endpoint that returns image data, add the `image_variation` parameter to your request.

### Basic Usage

Single variation:
```http
GET /api/your-endpoint?image_variation=scale=200x150%26format=jpeg%26quality=85
```

Multiple variations (pass as array):
```http
GET /api/your-endpoint?image_variation[]=scale=100x100%26format=jpeg&image_variation[]=scale=300x300%26format=webp
```

### Response Format

When `image_variation` is included in the request, the response will include variation URLs:

```json
{
    "Media_Image__": "image-id-here",
    "Url": "https://cdn.example.com/image/default.jpg",
    "Variation": {
        "scale=200x150&format=jpeg&quality=85": "https://cdn.example.com/image/variation1.jpg",
        "scale=300x300&format=webp": "https://cdn.example.com/image/variation2.webp"
    }
}
```

### Available Image Filters

#### Sizing Operations
- `scale=WIDTHxHEIGHT` - Scale image maintaining aspect ratio
- `scale_crop=WIDTHxHEIGHT` - Scale and crop to exact dimensions (maintains aspect ratio by cropping)
- `autocrop` - Automatically crop borders
- `crop=WIDTHxHEIGHT+X+Y` - Crop specific region
- `tile=WIDTHxHEIGHT` - Create tiled pattern

#### Format Conversion
- `format=jpeg|png|png32|gif|bmp|webp|heic|jxl|favicon`
- `quality=0-100` - Compression quality (for lossy formats)

#### Visual Effects
- `blur=RADIUS` - Apply blur effect
- `charcoal=RADIUS` - Charcoal drawing effect
- `enhance` - Enhance image
- `flip` - Flip vertically
- `flop` - Flip horizontally
- `gaussian_blur=RADIUSxSIGMA` - Gaussian blur
- `sepia` - Sepia tone effect
- `modulate=BRIGHTNESS,SATURATION,HUE` - Adjust color properties
- `motion_blur=RADIUSxSIGMA+ANGLE` - Motion blur effect
- `negate` - Invert colors
- `normalize` - Normalize contrast
- `oilpaint=RADIUS` - Oil painting effect
- `polaroid=ANGLE` - Polaroid frame effect
- `radial_blur=ANGLE` - Radial blur
- `rotate=DEGREES` - Rotate image
- `round_corners=RADIUS` - Round corners
- `shadow=OFFSETxOFFSET` - Add shadow
- `sharpen=RADIUSxSIGMA` - Sharpen image
- `spread=RADIUS` - Spread pixels randomly
- `swirl=DEGREES` - Swirl effect
- `threshold=VALUE` - Threshold filter
- `wave=AMPLITUDExWAVELENGTH` - Wave distortion

#### Color Adjustments
- `overlaycolor=COLOR` - Overlay color
- `tint=COLOR` - Tint image
- `vignette=RADIUSxSIGMA` - Add vignette
- `darken=AMOUNT` - Darken image
- `background_color=COLOR` - Set background color

#### Processing Options
- `strip` - Remove metadata (default for URLs)
- `noalpha` - Remove alpha channel
- `transparent=COLOR` - Make color transparent
- `flatten` - Flatten layers
- `page=NUMBER` - Extract specific page (for PDFs)
- `page=*` - Extract all pages (creates separate variations for each page)
- `resolution=DPI` - Set resolution

#### Special Parameters
- `alias=NAME` - Provide a human-readable alias for the variation (doesn't affect the actual transformation, only used as a key in the response)

### Image Examples

Create a thumbnail:
```
?image_variation=scale=200x200%26format=jpeg%26quality=85
```

Using alias for cleaner response keys:
```
?image_variation=scale=200x200%26format=jpeg%26quality=85%26alias=thumbnail
```

This will return the variation with "thumbnail" as the key instead of the full variation string:
```json
{
    "Variation": {
        "thumbnail": "https://cdn.example.com/image/variation.jpg"
    }
}
```

Note: In URL parameters, use `%26` to separate filters. In JSON responses, the keys will contain `&`.

Convert to WebP with metadata stripped:
```
?image_variation=format=webp%26quality=80%26strip
```

Apply artistic effects:
```
?image_variation=sepia%26vignette=20x5%26rotate=5
```

Extract all PDF pages as PNGs:
```
?image_variation=page=*%26format=png
```

Multiple variations for responsive images:
```
?image_variation[]=scale=320x240%26format=webp%26quality=80
&image_variation[]=scale=768x576%26format=webp%26quality=85
&image_variation[]=scale=1920x1080%26format=webp%26quality=90
```

## Audio Variations

When requesting an endpoint that returns audio data, add the `audio_variation` parameter to your request.

### Basic Usage

Single variation:
```http
GET /api/your-endpoint?audio_variation=format=mp3
```

Multiple variations:
```http
GET /api/your-endpoint?audio_variation[]=format=mp3&audio_variation[]=format=opus
```

### Response Format

When `audio_variation` is included in the request, the response will include variation URLs:

```json
{
    "Media_Audio__": "audio-id-here",
    "Url": "https://cdn.example.com/audio/default.ogg",
    "Variation": {
        "format=mp3": "https://cdn.example.com/audio/variation1.mp3",
        "format=opus": "https://cdn.example.com/audio/variation2.opus",
        "format=aac": "https://cdn.example.com/audio/variation3.aac"
    }
}
```

### Available Audio Filters

#### Format Conversion
- `format=ogg` - Ogg Vorbis format (default)
- `format=opus` - Opus format
- `format=mp3` - MP3 format
- `format=flac` - FLAC lossless format
- `format=aac` - AAC format
- `format=sln|sln8|sln12|sln16|sln24|sln32|sln44|sln48|sln96|sln192` - Raw signed linear formats (for telephony)

#### Audio Processing
- `echo` - Add echo effect
- `mono` - Convert to mono
- `samplerate=RATE` - Change sample rate (e.g., samplerate=44100)

#### Trimming
- `start=SECONDS` - Start position in seconds
- `end=SECONDS` - End position in seconds

### Audio Examples

Convert to MP3:
```
?audio_variation=format=mp3
```

Convert to mono Opus:
```
?audio_variation=format=opus%26mono
```

Trim audio clip:
```
?audio_variation=start=10%26end=30%26format=mp3
```

Multiple formats for browser compatibility:
```
?audio_variation[]=format=opus
&audio_variation[]=format=mp3
&audio_variation[]=format=ogg
```

## Default Variations

Some variations are included by default in responses:

### Images
- The main `Url` field contains a variation with metadata stripped (`strip`)
- Additional default variations may be configured server-side

### Audio
- The main `Url` field contains the Ogg Vorbis format (`format=ogg`)
- By default, `format=aac` and `format=mp3` variations are also included

## Special Features

### Multi-page Documents

For images that contain multiple pages (like PDFs), you can use `page=*` to generate variations for all pages:

```
?image_variation=page=*%26format=png
```

This will create separate URLs for each page:
```json
{
    "Variation": {
        "page=1&format=png": "https://cdn.example.com/image/page1.png",
        "page=2&format=png": "https://cdn.example.com/image/page2.png",
        "page=3&format=png": "https://cdn.example.com/image/page3.png"
    }
}
```

### REST Endpoints for Direct Variation Access

Some objects may provide direct variation endpoints:

```http
GET /api/media/image/{id}/variation/{variation_string}
GET /api/media/audio/{id}/variation/{variation_string}
```

## Performance Considerations

1. **On-demand Generation**: Variations are generated on first request and then cached
2. **CDN Delivery**: Generated variations are served through CDN for fast delivery
3. **Batch Processing**: Request multiple variations in a single API call for efficiency

## Best Practices

1. **Request What You Need**: Only request variations you'll actually use
2. **Use Appropriate Formats**: 
   - WebP for modern web browsers (smaller file sizes)
   - JPEG for maximum compatibility
   - Opus for modern audio streaming
   - MP3 for wide audio compatibility
3. **Combine Filters**: Chain multiple filters together with `%26` to create complex transformations
4. **Cache URLs**: Store the returned variation URLs to avoid repeated API calls

## Common Use Cases

### Responsive Images
```
?image_variation[]=scale=320x240%26format=webp%26quality=80
&image_variation[]=scale=768x576%26format=webp%26quality=85
&image_variation[]=scale=1920x1080%26format=webp%26quality=90
&image_variation[]=scale=1920x1080%26format=jpeg%26quality=85
```

### Social Media Optimized Images
```
?image_variation[]=scale_crop=1200x630%26format=jpeg%26quality=85
&image_variation[]=scale_crop=1200x675%26format=jpeg%26quality=85
&image_variation[]=scale_crop=1080x1080%26format=jpeg%26quality=85
```

### Multi-format Audio for Compatibility
```
?audio_variation[]=format=opus
&audio_variation[]=format=mp3
&audio_variation[]=format=aac
```

## Error Handling

Variations are actually generated when the URL is accessed, so the system will not know if a variation is possible at request time, and instead the URL may return a HTTP 500 error.
