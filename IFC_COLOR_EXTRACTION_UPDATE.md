# IFC Color Extraction Update

## Date: 2026-02-03

## Overview

Updated the GLTF conversion process to extract and use **actual IFC colors** from the source file instead of using hardcoded type-based colors. This ensures the 3D model displays with the same colors as defined in the original IFC file.

## Problem

Previously, the system assigned colors based purely on element type:
- IfcBeam → Light blue-gray (180, 180, 220)
- IfcColumn → Light blue (150, 200, 220)
- IfcMember → Light brown (200, 180, 150)
- IfcPlate → Light tan (220, 200, 180)
- etc.

This meant all beams had the same color, all columns had the same color, etc., regardless of what colors were actually specified in the IFC file.

## Solution

Implemented IFC color extraction using `ifcopenshell.util.style.get_style()`:

### 1. Color Extraction Function (Lines ~1252-1267)

```python
def get_ifc_color(product):
    """Extract color from IFC product styles - returns RGB tuple or None."""
    try:
        import ifcopenshell.util.style
        style = ifcopenshell.util.style.get_style(product)
        if style and hasattr(style, "Styles"):
            for rendering in style.Styles or []:
                if rendering.is_a('IfcSurfaceStyleRendering') and rendering.SurfaceColour:
                    # IFC colors are in 0-1 range, convert to 0-255
                    return (
                        int(rendering.SurfaceColour.Red * 255),
                        int(rendering.SurfaceColour.Green * 255),
                        int(rendering.SurfaceColour.Blue * 255)
                    )
    except:
        pass
    return None
```

### 2. Parallel Extraction (Lines ~1270-1287)

Colors are extracted **during parallel geometry creation** to minimize performance impact:

```python
def create_shape_parallel(product):
    """Create geometry shape for a product - thread-safe."""
    try:
        shape = ifcopenshell.geom.create_shape(settings, product)
        # Also extract color while we're at it (in parallel)
        ifc_color = get_ifc_color(product)
        return (product, shape, ifc_color, None)
    except Exception as e:
        # Fallback with color extraction
        alt_settings = ifcopenshell.geom.settings()
        alt_settings.set(alt_settings.USE_WORLD_COORDS, True)
        alt_settings.set(alt_settings.WELD_VERTICES, False)
        shape = ifcopenshell.geom.create_shape(alt_settings, product)
        ifc_color = get_ifc_color(product)
        return (product, shape, ifc_color, None)
```

### 3. Priority-Based Color Assignment (Lines ~1365-1385)

```python
# Priority 1: Use actual IFC color if extracted
if ifc_color is not None:
    color_rgb = ifc_color
    ifc_colors_found += 1
# Priority 2: Fallback to type-based colors
elif is_fastener:
    color_rgb = (139, 105, 20)  # Dark brown-gold for fasteners
else:
    element_type = product.is_a()
    color_map = {
        "IfcBeam": (180, 180, 220),
        "IfcColumn": (150, 200, 220),
        # ... etc
    }
    color_rgb = color_map.get(element_type, (190, 190, 220))
```

## Performance Impact

### Optimization Strategy
- Color extraction happens **in parallel** during geometry creation (which is already the bottleneck)
- Uses thread pool with 8 workers
- Minimal sequential overhead

### Expected Impact
- **Small files (100-500 elements)**: +0.5-2 seconds
- **Medium files (500-2000 elements)**: +2-5 seconds  
- **Large files (2000+ elements)**: +5-10 seconds

**Relative increase**: ~10-15% of total conversion time

### Why It's Worth It
1. Conversion happens **once per file upload** (results are cached)
2. Correct colors improve visual understanding significantly
3. Users expect models to look the same as in their CAD software

## Logging

New log output shows color extraction statistics:

```
[GLTF] Extracting IFC colors in parallel for accurate visualization...
[GLTF] Conversion summary: 1523 meshes created, 45 skipped, 12 failed
[GLTF] Color extraction: 1489/1523 elements used IFC colors, 34 used type-based fallback
```

This helps identify:
- How many elements have IFC color data
- How many fall back to type-based colors
- Overall quality of the color extraction

## Fallback Behavior

The system gracefully handles files without color data:
1. If IFC color is found → use it
2. If no IFC color → use type-based colors (previous behavior)
3. Never fails due to missing colors

## Benefits

✅ **Accurate Visualization**: Models display with original IFC colors  
✅ **Minimal Performance Impact**: Parallel extraction during geometry creation  
✅ **Graceful Degradation**: Falls back to type-based colors if needed  
✅ **Logging**: Clear feedback on color extraction success rate  
✅ **One-Time Cost**: Conversion is cached, only runs on upload  

## Testing

After this update:
1. Upload an IFC file with defined colors
2. Check the conversion logs for "Color extraction: X/Y elements used IFC colors"
3. View the Model tab - elements should show their original IFC colors
4. Files without color data will still work (fallback to type-based colors)

## Files Modified

- `api/main.py`: 
  - Added `get_ifc_color()` helper function
  - Updated `create_shape_parallel()` to extract colors
  - Modified color assignment logic to prioritize IFC colors
  - Added color statistics logging

