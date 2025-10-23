Aperio SVS Format
=================

The Aperio ScanScope Virtual Slide (SVS) format is one of the most widely used formats in digital pathology. Developed by Aperio Technologies (now part of Leica Biosystems), it's a TIFF-based format that stores pyramidal whole slide images with JPEG compression.

Format Specification
--------------------

File Extensions
~~~~~~~~~~~~~~~

- ``.svs``

Detection Criteria
~~~~~~~~~~~~~~~~~~

FastSlide detects a file as Aperio SVS if:

1. The file is a valid TIFF (starts with TIFF magic bytes)
2. The file is BigTIFF (supports files > 4GB)
3. The first IFD contains an ImageDescription tag starting with "Aperio"

File Structure
--------------

SVS files are organized as multi-page TIFF files with the following typical structure:

.. code-block:: text

   slide.svs (BigTIFF File)
   ├── Page 0: Full resolution image (largest, tiled, JPEG-compressed)
   ├── Page 1: Thumbnail (typically 1024px wide, stripped)
   ├── Page 2: Pyramid level 1 (downsampled ~4×, tiled)
   ├── Page 3: Pyramid level 2 (downsampled ~16×, tiled)
   ├── Page 4: Macro image (label photo, stripped, JPEG)
   └── Page 5: Label image (barcode label, stripped, JPEG)

**Pyramid Organization:**

- **Page 0**: Full resolution scan (level 0), usually 20× or 40× magnification
- **Pages 2, 3, ...**: Downsampled levels for efficient low-resolution viewing
- Each pyramid level typically downsampled by factor of 2-4× from previous level
- Downsample factors may vary between levels

**Associated Images:**

- **Thumbnail** (Page 1): Small preview image, typically 1024px wide
- **Macro** (Page 4): Photo of entire glass slide including label
- **Label** (Page 5): Cropped image of slide barcode label

TIFF Structure Details
~~~~~~~~~~~~~~~~~~~~~~~

Each pyramid level is stored as a TIFF directory (IFD) with:

.. list-table::
   :header-rows: 1
   :widths: 25 50 25

   * - Property
     - Description
     - Typical Value
   * - Tile Size
     - Tile dimensions in pixels
     - 240×240 (Aperio default)
   * - Compression
     - JPEG (OLD_JPEG tag)
     - Tag 6
   * - Photometric
     - YCbCr (JPEG colorspace)
     - Tag 6
   * - Samples Per Pixel
     - Number of channels
     - 3 (RGB after decode)
   * - Bits Per Sample
     - Bits per channel
     - 8
   * - Planar Configuration
     - Contiguous (interleaved RGB)
     - 1

**Tiled vs Stripped:**

- **Tiled Format** (most common for main levels):
    - Fixed tile size (typically 240×240)
    - Random access to any tile
    - Tile index: ``tile_y * tiles_across + tile_x``
    - Read via: ``TIFFReadEncodedTile(tif, tile_index, buffer, size)``

- **Stripped Format** (typical for associated images):
    - Full width strips with variable height (rows per strip)
    - Sequential access pattern
    - Strip index: ``strip_y``
    - Read via: ``TIFFReadEncodedStrip(tif, strip_index, buffer, size)``

Metadata Format
---------------

Aperio stores rich metadata in the ImageDescription TIFF tag (tag 270) of the first IFD. The metadata is a pipe-delimited string with key=value pairs.

Example Metadata
~~~~~~~~~~~~~~~~

.. code-block:: text

   Aperio Image Library v11.2.1
   46000x32914 [0,100 46000x32914] (256x256) JPEG/RGB Q=30
   |AppMag = 20
   |StripeWidth = 2040
   |ScanScope ID = CPAPERIOCS
   |Filename = CMU-1
   |Date = 12/29/09
   |Time = 09:59:15
   |User = b414003d-95c6-48b0-9369-8010ed517ba7
   |Parmset = USM Filter
   |MPP = 0.4990
   |Left = 25.691574
   |Top = 23.449873
   |LineCameraSkew = -0.000424
   |LineAreaXOffset = 0.019265
   |LineAreaYOffset = -0.000313
   |Focus Offset = 0.000000
   |ImageID = 1004486
   |OriginalWidth = 46000
   |Originalheight = 32914
   |Filtered = 5
   |ICC Profile = ScanScope v1

Key Metadata Fields
~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 60 20

   * - Field
     - Description
     - Example
   * - AppMag
     - Apparent magnification (objective power)
     - 20, 40
   * - MPP
     - Microns per pixel at level 0
     - 0.4990
   * - Date / Time
     - Scan acquisition timestamp
     - 12/29/09 09:59:15
   * - ScanScope ID
     - Scanner identifier
     - CPAPERIOCS
   * - OriginalWidth / Originalheight
     - Dimensions of scanned region
     - 46000 × 32914
   * - JPEG Quality
     - JPEG compression quality (from header line)
     - Q=30

Known Properties
----------------

FastSlide exposes Aperio metadata as standardized properties:

- **openslide.mpp-x**: Microns per pixel in X direction (from MPP field)
- **openslide.mpp-y**: Microns per pixel in Y direction (typically same as mpp-x)
- **openslide.objective-power**: Objective magnification (from AppMag field)
- **aperio.Date**: Scan date
- **aperio.Time**: Scan time
- **aperio.ScanScope ID**: Scanner identifier
- **aperio.ImageID**: Unique image identifier

FastSlide Implementation
------------------------

FastSlide implements Aperio SVS support through a modular architecture with three main components plus a handle pool for thread safety.

Components
~~~~~~~~~~

1. **AperioReader**: Main reader class implementing ``SlideReader`` interface
2. **AperioPlanBuilder**: Planning stage - analyzes regions and enumerates tiles
3. **AperioTileExecutor**: Execution stage - reads tiles with parallel JPEG decoding
4. **TiffHandlePool**: Thread-safe TIFF handle management

**Implementation Files:**
  - ``aifo/fastslide/src/readers/aperio/aperio.cpp`` (main reader)
  - ``aifo/fastslide/src/readers/aperio/aperio_plan_builder.cpp`` (planning)
  - ``aifo/fastslide/src/readers/aperio/aperio_tile_executor.cpp`` (execution)
  - ``aifo/fastslide/utilities/tiff/tiff_handle_pool.cpp`` (handle pool)

Initialization Algorithm
~~~~~~~~~~~~~~~~~~~~~~~~

The initialization phase opens the TIFF file and parses metadata:

**Pseudocode:**

.. code-block:: text

   function Initialize(file_path):
     // 1. Open TIFF file with libtiff
     tiff_handle = TIFFOpen(file_path, "r")
     if not tiff_handle:
       return error("Failed to open TIFF")
     
     // 2. Verify it's a BigTIFF
     if not IsBigTIFF(tiff_handle):
       return error("Not a BigTIFF file")
     
     // 3. Read ImageDescription from first directory
     TIFFSetDirectory(tiff_handle, 0)
     image_desc = TIFFGetField(tiff_handle, TIFFTAG_IMAGEDESCRIPTION)
     
     if not image_desc.starts_with("Aperio"):
       return error("Not an Aperio SVS file")
     
     // 4. Parse Aperio metadata (pipe-delimited key=value pairs)
     metadata = ParseAperioMetadata(image_desc)
     
     // 5. Build pyramid level information
     directory_count = TIFFNumberOfDirectories(tiff_handle)
     pyramid_levels = []
     
     for dir_index in 0 to directory_count-1:
       TIFFSetDirectory(tiff_handle, dir_index)
       
       width, height = TIFFGetField(tiff_handle, TIFFTAG_IMAGEWIDTH, TIFFTAG_IMAGELENGTH)
       subfile_type = TIFFGetField(tiff_handle, TIFFTAG_SUBFILETYPE)
       
       // Check if this is a pyramid level (not thumbnail/macro/label)
       if subfile_type == 0:  // Main pyramid level
         level_info = LevelInfo()
         level_info.page = dir_index
         level_info.dimensions = (width, height)
         level_info.downsample = CalculateDownsample(width, height, pyramid_levels[0])
         pyramid_levels.append(level_info)
     
     // 6. Create TiffHandlePool for thread-safe reading
     handle_pool = TiffHandlePool(file_path, num_handles=8)
     
     // 7. Store metadata and pyramid structure
     return AperioReader(metadata, pyramid_levels, handle_pool)

Planning Algorithm (AperioPlanBuilder)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The planning phase enumerates tiles using simple grid arithmetic:

**Pseudocode:**

.. code-block:: text

   function BuildPlan(request, reader, tiff_metadata):
     // 1. Validate request
     if request.level < 0 or request.level >= reader.level_count:
       return error("Invalid level")
     
     // 2. Get level information
     level_info = reader.GetLevelInfo(request.level)
     pyramid_levels = reader.GetPyramidLevels()
     page = pyramid_levels[request.level].page
     
     // 3. Query TIFF structure (once, reused by executor)
     tiff_file = TiffFile.Create(reader.handle_pool)
     tiff_file.SetDirectory(page)
     
     tile_width, tile_height = tiff_file.GetTileDimensions()
     samples_per_pixel = tiff_file.GetSamplesPerPixel()
     is_tiled = tiff_file.IsTiled()
     
     // Store for executor
     tiff_metadata.page = page
     tiff_metadata.tile_width = tile_width
     tiff_metadata.tile_height = tile_height
     tiff_metadata.samples_per_pixel = samples_per_pixel
     tiff_metadata.is_tiled = is_tiled
     
     // 4. Determine region bounds
     if request.has_region_bounds():
       x, y = request.region_bounds.x, request.region_bounds.y
       width, height = ceil(request.region_bounds.width), ceil(request.region_bounds.height)
     else:
       x, y = 0, 0
       width, height = level_info.dimensions
     
     // 5. Clamp region to level bounds
     if x >= level_info.dimensions[0] or y >= level_info.dimensions[1]:
       return empty_plan  // Completely outside bounds
     
     if x + width > level_info.dimensions[0]:
       width = level_info.dimensions[0] - x
     if y + height > level_info.dimensions[1]:
       height = level_info.dimensions[1] - y
     
     // 6. Calculate tile grid coordinates
     region_x = uint32(x)
     region_y = uint32(y)
     
     first_tile_x = region_x / tile_width
     first_tile_y = region_y / tile_height
     last_tile_x = (region_x + width - 1) / tile_width
     last_tile_y = (region_y + height - 1) / tile_height
     
     // 7. Create tile operations
     operations = []
     tiles_across = (level_info.dimensions[0] + tile_width - 1) / tile_width
     
     for tile_y in first_tile_y to last_tile_y:
       for tile_x in first_tile_x to last_tile_x:
         // Calculate tile bounds in level coordinates
         tile_left = tile_x * tile_width
         tile_top = tile_y * tile_height
         tile_right = min(tile_left + tile_width, level_info.dimensions[0])
         tile_bottom = min(tile_top + tile_height, level_info.dimensions[1])
         
         // Calculate intersection with requested region
         inter_left = max(tile_left, region_x)
         inter_top = max(tile_top, region_y)
         inter_right = min(tile_right, region_x + width)
         inter_bottom = min(tile_bottom, region_y + height)
         
         if inter_left >= inter_right or inter_top >= inter_bottom:
           continue  // No intersection
         
         inter_width = inter_right - inter_left
         inter_height = inter_bottom - inter_top
         
         // Create tile operation
         op = TileReadOp()
         op.level = request.level
         op.tile_coord = (tile_x, tile_y)
         op.source_id = page
         
         // For TIFF, byte_offset is the tile index
         if is_tiled:
           op.byte_offset = tile_y * tiles_across + tile_x
         else:
           op.byte_offset = tile_y  // Strip number
         
         // Estimate compressed tile size
         op.byte_size = EstimateJPEGSize(tile_width, tile_height, samples_per_pixel)
         
         // Transform: source crop within tile, destination in output
         src_x = inter_left - tile_left
         src_y = inter_top - tile_top
         dest_x = inter_left - region_x
         dest_y = inter_top - region_y
         
         op.transform.source = (src_x, src_y, inter_width, inter_height)
         op.transform.dest = (dest_x, dest_y, inter_width, inter_height)
         
         operations.append(op)
     
     // 8. Create output specification
     output = OutputSpec()
     output.dimensions = (width, height)
     output.channels = samples_per_pixel
     output.pixel_format = kUInt8
     output.background = (255, 255, 255, 255)  // White background
     
     // 9. Estimate costs
     cost.total_tiles = len(operations)
     cost.total_bytes_to_read = sum(op.byte_size for op in operations)
     cost.tiles_to_decode = cost.total_tiles
     cost.estimated_time_ms = cost.total_tiles * 1.5  // ~1.5ms per tile
     
     // 10. Return plan
     return TilePlan(operations, output, cost, tiff_metadata)

**Key Features:**

- Simple grid-based tile enumeration (no spatial index needed)
- Calculates intersections and clipping for edge tiles
- Queries TIFF structure once and passes to executor
- Handles both tiled and stripped formats

Execution Algorithm (AperioTileExecutor)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The execution phase uses strategy selection based on tile count:

**Pseudocode:**

.. code-block:: text

   function ExecutePlan(plan, reader, writer, tiff_metadata):
     if plan.operations.empty():
       writer.FillWithBackground(plan.output.background)
       return success
     
     // Choose execution strategy based on tile count
     if len(plan.operations) <= 8:
       return ExecuteSequential(plan, reader, writer, tiff_metadata)
     else:
       return ExecuteParallel(plan, reader, writer, tiff_metadata)
   
   function ExecuteSequential(plan, reader, writer, tiff_metadata):
     // Single-threaded execution in calling thread
     for op in plan.operations:
       // Acquire TIFF handle (from pool, reused across tiles)
       tiff_file = TiffFile.Create(reader.handle_pool)
       
       // Read and decode tile
       tile_data = ReadAndDecodeTile(tiff_file, op, tiff_metadata)
       
       // Extract sub-region if clipped
       cropped_data = ExtractSubRegion(tile_data, op.transform.source, tiff_metadata)
       
       // Write to output (no mutex needed)
       writer.WriteTile(op, cropped_data)
     
     return success
   
   function ExecuteParallel(plan, reader, writer, tiff_metadata):
     // Parallel execution via global thread pool
     thread_pool = GetGlobalThreadPool()
     mutex = Mutex()
     
     futures = thread_pool.submit_all(plan.operations, lambda op:
       // Each thread gets its own TIFF handle from pool
       tiff_file = TiffFile.Create(reader.handle_pool)
       
       // Read and decode tile
       tile_data = ReadAndDecodeTile(tiff_file, op, tiff_metadata)
       
       // Extract sub-region if clipped
       cropped_data = ExtractSubRegion(tile_data, op.transform.source, tiff_metadata)
       
       // Write to output with mutex protection
       lock(mutex):
         writer.WriteTile(op, cropped_data)
     )
     
     futures.wait()  // Synchronize all threads
     return success
   
   function ReadAndDecodeTile(tiff_file, op, tiff_metadata):
     // 1. Set TIFF directory to correct page (cached, usually no-op)
     tiff_file.SetDirectory(tiff_metadata.page)
     
     // 2. Allocate buffer for tile (thread-local, reused)
     tiff_handle = tiff_file.GetHandle()
     tile_size = tiff_metadata.is_tiled ? TIFFTileSize(tiff_handle) : TIFFStripSize(tiff_handle)
     buffer = GetThreadLocalBuffer(tile_size)
     
     // 3. Read JPEG-compressed tile via libtiff
     if tiff_metadata.is_tiled:
       bytes_read = TIFFReadEncodedTile(tiff_handle, op.byte_offset, buffer, tile_size)
     else:
       bytes_read = TIFFReadEncodedStrip(tiff_handle, op.byte_offset, buffer, tile_size)
     
     if bytes_read <= 0:
       return error("Failed to read tile")
     
     // libtiff automatically decodes JPEG to RGB
     // 4. Return span view of decoded data (no copy)
     return span(buffer, bytes_read)
   
   function ExtractSubRegion(tile_data, source_region, tiff_metadata):
     // Extract crop rectangle from full tile
     src_x, src_y = source_region.x, source_region.y
     crop_width, crop_height = source_region.width, source_region.height
     tile_width = tiff_metadata.tile_width
     samples_per_pixel = tiff_metadata.samples_per_pixel
     
     // Allocate output buffer (thread-local)
     crop_size = crop_width * crop_height * samples_per_pixel
     cropped = GetThreadLocalCropBuffer(crop_size)
     
     // Copy row by row (IMPORTANT: use tile_width as stride)
     for y in 0 to crop_height-1:
       src_offset = ((src_y + y) * tile_width + src_x) * samples_per_pixel
       dst_offset = y * crop_width * samples_per_pixel
       memcpy(cropped + dst_offset, tile_data + src_offset, crop_width * samples_per_pixel)
     
     return span(cropped, crop_size)

**Key Features:**

- **Sequential Threshold**: Avoids thread pool overhead for small reads (≤8 tiles)
- **Handle Pool**: Each thread gets its own ``TIFF*`` handle (zero contention)
- **Directory Caching**: libtiff caches directory info across reads
- **Thread-Local Buffers**: Eliminates per-tile allocations
- **Parallel JPEG Decoding**: 3-4× speedup on 8-core CPU

Handle Pool Architecture
~~~~~~~~~~~~~~~~~~~~~~~~

The handle pool enables thread-safe parallel reading:

**Design:**

.. code-block:: text

   class TiffHandlePool:
     file_path: string
     handles: vector<TIFF*>
     mutex: Mutex
     condition: ConditionVariable
     
     function Create():
       handle = AcquireHandle()  // Blocks if all in use
       return TiffFile(handle, this)
     
     function AcquireHandle() -> TIFF*:
       lock(mutex):
         while handles.empty():
           condition.wait(mutex)  // Wait for available handle
         
         handle = handles.pop_back()
         return handle
     
     function ReleaseHandle(handle):
       lock(mutex):
         handles.push_back(handle)
         condition.notify_one()  // Wake waiting thread

**Benefits:**

- Multiple ``TIFF*`` handles to same file (libtiff supports this)
- Thread-local acquisition (each thread gets its own)
- Zero contention during tile reading (read-only operations)
- Directory caching optimization (libtiff caches per handle)
- Auto-scaling based on thread pool size

Sub-Region Extraction
~~~~~~~~~~~~~~~~~~~~~

Edge tiles often need cropping:

**Why Cropping is Needed:**

- Tiles at image boundaries may be partial
- Requested region may not align with tile grid
- TIFF allocates full tile dimensions in memory

**Extraction Algorithm:**

.. code-block:: text

   // Source: full decoded tile (tile_width × tile_height × channels)
   // Extract: crop rectangle (src_x, src_y, crop_width, crop_height)
   
   for y in 0 to crop_height-1:
     // CRITICAL: Use tile_width as stride, not crop_width
     // TIFF allocates full tile dimensions even for edge tiles
     src_offset = ((src_y + y) * tile_width + src_x) * channels
     dst_offset = y * crop_width * channels
     memcpy(dst + dst_offset, src + src_offset, crop_width * channels)

Performance Characteristics
---------------------------

JPEG Decode Parallelism
~~~~~~~~~~~~~~~~~~~~~~~~

JPEG decoding is the primary bottleneck:

- **Single-threaded**: ~50-100 tiles/sec (depends on CPU, JPEG quality)
- **Multi-threaded (8 cores)**: ~300-500 tiles/sec (near-linear scaling)
- **Typical tile**: 240×240 pixels, ~20-40 KB compressed, ~170 KB uncompressed

**Optimization Strategies:**

1. **Parallel Execution**: Use all CPU cores for JPEG decoding
2. **Handle Pool**: Eliminate handle creation overhead
3. **Directory Caching**: libtiff caches directory info across reads
4. **Thread-Local Buffers**: Eliminate per-tile allocations
5. **Sequential Tile Order**: Improves disk I/O patterns (prefetch)

Handle Pool Performance
~~~~~~~~~~~~~~~~~~~~~~~

Without handle pool, operations would serialize:

.. code-block:: text

   Thread 1: ████████████████████  Open → Read → Close
   Thread 2:                     ████████████████████  
   Thread 3:                                         ████████████████████

With handle pool, true parallel I/O:

.. code-block:: text

   Thread 1: ████████████  Read tile 0, 3, 6, ...
   Thread 2: ████████████  Read tile 1, 4, 7, ...
   Thread 3: ████████████  Read tile 2, 5, 8, ...

Cache Behavior
~~~~~~~~~~~~~~

Tile caching is very effective for SVS:

- **Cache Key**: ``(filename, level, page, tile_index)``
- **Hit Rate**: High for repeated region reads (e.g., viewer panning)
- **Memory**: Cached tiles stored as decoded RGB (~170 KB per tile)
- **Eviction**: LRU policy, configurable memory limit

Typical speedup with cache:
  - First read: 100ms (cold cache)
  - Subsequent reads: 5ms (warm cache, 20× faster)

Compatibility
-------------

FastSlide's Aperio implementation is compatible with:

- **OpenSlide**: Produces identical output for most slides
- **Bio-Formats**: Compatible metadata parsing
- **QuPath**: Can open slides processed by FastSlide
- **ImageJ/Fiji**: Via Bio-Formats plugin

**Known Differences from OpenSlide:**

- FastSlide uses parallel tile reading (faster)
- Slightly different JPEG decode implementation (libtiff vs libjpeg-turbo)
- FastSlide's quickhash includes more metadata fields

Limitations
-----------

Current limitations of the Aperio implementation:

1. **JPEG Quality**: Locked to quality factor in file (no runtime override)
2. **Color Profiles**: ICC profiles are parsed but not applied during decode
3. **Fluorescence**: Optimized for brightfield RGB, fluorescence support untested
4. **Very Large Files**: Files >100GB may have degraded performance

Test Data
---------

Sample Aperio SVS files are available at:
  https://openslide.cs.cmu.edu/download/openslide-testdata/Aperio/

References
----------

- `OpenSlide Aperio Format Documentation <https://openslide.org/formats/aperio/>`_
- `BigTIFF Specification <https://www.awaresystems.be/imaging/tiff/bigtiff.html>`_
- `TIFF Tag Reference <https://www.awaresystems.be/imaging/tiff/tifftags.html>`_
- `LibTIFF Documentation <http://www.libtiff.org/>`_
