# Shutter Compare - Architecture & Code Documentation

## Overview

Shutter Compare is a single-file vanilla JavaScript web application for comparing regions of photographs with different camera settings. Everything is contained in `index.html` - no build step, no dependencies except exif-js from CDN.

## File Structure

```
index.html          # Complete application (HTML + CSS + JS)
README.md           # User-facing documentation
ARCHITECTURE.md     # This file - technical documentation
```

## Technology Stack

- **Vanilla JavaScript** (ES6+)
- **exif-js** (CDN) - EXIF extraction for JPEG files
- **Canvas API** - Image cropping and histogram generation
- **CSS Grid** - Responsive layout for comparison items

## Application State

All state is stored in global variables at the top of the `<script>` section:

```javascript
let images = [];              // Array of { file, url, exif, img } objects
let hiddenImages = new Set(); // Indices of hidden images
let selections = [];          // Array of { x, y, width, height } in percentages (0-1)
let zoomLevel = 100;          // Current zoom percentage (10-500)
let isDrawing = false;        // Whether user is currently drawing a selection
let startX, startY;           // Starting coordinates for selection drawing
let currentSelectionBox = null; // DOM element being drawn
const MAX_REGIONS = 5;        // Maximum allowed selection regions
const REGION_COLORS = ['#4a9eff', '#81c784', '#ffb74d', '#e57373', '#ba68c8'];
```

## Key Functions

### File Loading

#### `handleFiles(files)`
- Filters for JPEG and CR2 files
- Sorts by filename
- Resets application state
- Calls `extractCR2Preview()` or `extractExif()` based on file type
- Loads images and stores in `images` array

#### `extractCR2Preview(file)`
- Parses Canon CR2 RAW files (TIFF-based format)
- Finds embedded JPEG preview by scanning for JPEG markers (FFD8)
- Validates JPEG structure (checks for JFIF/EXIF/DQT markers)
- Returns blob URL and parsed EXIF data

#### `extractExif(file)`
- Uses exif-js library for JPEG files
- Returns promise with formatted EXIF object

#### `parseCR2Exif(data)`
- Manual TIFF/IFD parser for CR2 files
- Handles both little-endian and big-endian byte order
- Extracts: FNumber, ExposureTime, ISO, FocalLength, ExposureBias, Camera Model

### Selection System

#### `createSelectionBox(index)`
- Creates a DOM element for selection region
- Adds colored border based on region index
- Includes label ("Region N") and delete button (×)
- Attaches mousedown handler to prevent event bubbling on delete

#### `deleteRegion(index)`
- Removes selection from `selections` array
- Updates visual selection boxes
- Re-renders comparisons

#### `updateSelectionBoxes()`
- Clears all selection box DOM elements
- Recreates them from `selections` array
- Used after deletion or window resize

#### `clearAllSelectionBoxes()`
- Removes all `.selection-box` elements from reference container

### Mouse Event Handlers

- `mousedown` on reference container: Starts new selection (if < MAX_REGIONS)
- `mousemove` on document: Updates selection box size while drawing
- `mouseup` on document: Finalizes selection, adds to `selections` array

### Rendering

#### `renderComparisons()`
1. Filters visible images (excludes hidden)
2. Calculates which EXIF fields differ across images
3. Displays common EXIF values below reference image
4. Calculates cell width based on widest selection region
5. Renders each image with all its regions using `createComparisonItem()`

#### `createComparisonItem(imageData, imageIndex, prevEV, currentEV, selectionsArray)`
- Creates comparison card for one image
- Loops through all selections, creating crop canvas for each
- Adds region indicator label (R1, R2, etc.) if multiple regions
- Creates histogram overlay for each crop
- Adds EXIF display (only differing fields)
- Adds EV difference badge
- Adds hide button

#### `createHistogram(canvas, ctx)`
- Reads pixel data from crop canvas
- Calculates luminosity histogram (256 bins)
- Formula: `lum = 0.299 * R + 0.587 * G + 0.114 * B`
- Draws histogram as white bars on transparent canvas (100x40px)

### EV Calculation

#### `calculateEV(exif)`
- Parses aperture string (e.g., "f/2.8" → 2.8)
- Parses shutter speed (e.g., "1/250s" → 0.004)
- Parses ISO (e.g., "ISO 100" → 100)
- Formula: `EV = log₂(f² / shutter) - log₂(ISO / 100)`
- Returns null if any value is missing

EV difference display:
- `prevEV - currentEV` (inverted so positive = brighter)
- Green badge for brighter (+EV)
- Red badge for darker (-EV)
- Gray badge for same (±0.01)

## CSS Architecture

### Layout
- `.container` - Max width 1200px, centered
- `.comparison-list` - CSS Grid with dynamic `--cell-min-width` variable
- `.comparison-item` - Card with padding, contains multiple `.crop-container` elements

### Selection Boxes
- `.selection-box` - Absolute positioned on reference image
- `.region-N` classes (1-5) for different colors
- `.region-label` - Top-left label
- `.region-delete` - Top-right delete button (has `pointer-events: auto`)

### Responsive
- Grid auto-fills columns based on `--cell-min-width`
- Single column fallback at 600px

## Data Flow

```
User drops files
    ↓
handleFiles() - parse, extract EXIF, load images
    ↓
User draws selection on reference image
    ↓
selections[] updated, renderComparisons() called
    ↓
For each visible image:
    For each selection:
        - Crop image to canvas
        - Generate histogram
        - Create comparison item
    ↓
Display in grid with EXIF, EV diff, histogram
```

## EXIF Fields

| Field | EXIF Tag | Format Example |
|-------|----------|----------------|
| Aperture | FNumber (0x829D) | "f/2.8" |
| Shutter | ExposureTime (0x829A) | "1/250s" |
| ISO | ISOSpeedRatings (0x8827) | "ISO 100" |
| Focal Length | FocalLength (0x920A) | "50mm" |
| Exp Comp | ExposureBias (0x9204) | "+0.3 EV" |
| Camera | Model (0x0110) | "Canon EOS R5" |

## Browser Compatibility

- Requires: ES6, Canvas API, File API, Blob URLs
- Tested: Chrome, Firefox, Edge (modern versions)
- No IE11 support

## Known Limitations

- CR2 support only (no other RAW formats)
- Large files may be slow (no web workers)
- Histogram is luminosity only (no RGB channels)
- No persistence (refresh = lose all state)

## Future Enhancement Ideas

- Add more RAW format support (NEF, ARW, etc.)
- Save/load session to localStorage
- Export comparison as image
- RGB histogram option
- Keyboard shortcuts
- Drag to reorder images
- Side-by-side A/B comparison mode
