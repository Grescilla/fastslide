QPTIFF Format
=============

QPTIFF (PerkinElmer Quantitative Pathology TIFF) is a multi-channel spectral imaging format based on TIFF. It's designed for multispectral and hyperspectral microscopy applications in quantitative pathology, immunofluorescence, and biomedical research.

Format Specification
--------------------

File Extensions
~~~~~~~~~~~~~~~

- ``.tif``
- ``.tiff``
- ``.qptiff`` (less common)

Detection Criteria
~~~~~~~~~~~~~~~~~~

FastSlide detects a file as QPTIFF if:

1. The file is a valid TIFF
2. The first IFD contains an ImageDescription tag with XML data
3. The XML contains a root element ``<PerkinElmer-QPI-ImageDescription>``
4. Multiple pages exist with spectral channel information

File Structure
--------------

QPTIFF files are organized as multi-page TIFF files with page-per-channel organization:

.. code-block:: text

   multispectral.tif (TIFF File)
   ├── Page 0: Channel 0, Level 0 (full resolution, wavelength λ₀)
   ├── Page 1: Channel 1, Level 0 (full resolution, wavelength λ₁)
   ├── Page 2: Channel 2, Level 0 (full resolution, wavelength λ₂)
   ├── ...
   ├── Page N: Channel 0, Level 1 (downsampled 2×)
   ├── Page N+1: Channel 1, Level 1 (downsampled 2×)
   ├── ...
   ├── Page M: Thumbnail, all channels merged
   └── Page M+1: Macro image (optional)

**Page-Per-Channel Organization:**

- Each spectral channel stored as separate TIFF page
- Grayscale images (1 sample per pixel)
- Higher bit depths supported (8-bit, 12-bit, 16-bit, float)
- Pyramid levels grouped by downsampling factor
- Each level contains one page per channel

Overview
--------

QPTIFF stores multispectral and hyperspectral imaging data with rich channel metadata for quantitative analysis.

**Key Characteristics:**

- **Compression**: LZW (common), JPEG, or uncompressed
- **Organization**: Page-per-channel with pyramid levels
- **Channels**: 3-100+ spectral channels
- **Bit Depth**: 8-bit, 12-bit, 16-bit, or 32-bit float
- **Metadata**: XML in ImageDescription tag with channel info
- **Typical use**: Immunofluorescence, spectral unmixing, quantitative pathology

Channel Organization
~~~~~~~~~~~~~~~~~~~~

**Multispectral Imaging** (3-10 channels):
  - Specific fluorophores (DAPI, FITC, Cy3, Cy5, etc.)
  - Fixed wavelength bands
  - Typical immunofluorescence panels

**Hyperspectral Imaging** (50-100+ channels):
  - Full spectral resolution (e.g., 400-700nm in 5nm steps)
  - Material identification via spectral signatures
  - Spectral unmixing applications

Metadata Format
---------------

Channel metadata is stored in XML within the ImageDescription TIFF tag of each page:

Example XML
~~~~~~~~~~~

.. code-block:: xml

   <?xml version="1.0" encoding="utf-8"?>
   <PerkinElmer-QPI-ImageDescription>
     <ID>12345</ID>
     <Name>Channel_DAPI</Name>
     <ImageType>FullResolution</ImageType>
     <ChannelID>0</ChannelID>
     <Biomarker>
       <Name>DAPI</Name>
       <Type>Fluorophore</Type>
       <Color>255,255,255</Color>
     </Biomarker>
     <Acquisition>
       <Wavelength>461</Wavelength>
       <WavelengthUnit>nm</WavelengthUnit>
       <ExposureTime>100</ExposureTime>
       <ExposureUnit>ms</ExposureUnit>
     </Acquisition>
     <Magnification>20</Magnification>
     <PhysicalSizeX>0.3250</PhysicalSizeX>
     <PhysicalSizeY>0.3250</PhysicalSizeY>
     <PhysicalSizeUnit>µm</PhysicalSizeUnit>
   </PerkinElmer-QPI-ImageDescription>

Key Metadata Fields
~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 50 25

   * - Field
     - Description
     - Example
   * - ImageType
     - Page type (FullResolution, Thumbnail, etc.)
     - FullResolution
   * - ChannelID
     - Channel index
     - 0, 1, 2, ...
   * - Biomarker/Name
     - Fluorophore or marker name
     - DAPI, FITC, Cy5
   * - Wavelength
     - Emission wavelength (nm)
     - 461, 520, 670
   * - ExposureTime
     - Camera exposure time (ms)
     - 100
   * - PhysicalSizeX/Y
     - Microns per pixel
     - 0.3250
   * - Magnification
     - Objective power
     - 20, 40

**Metadata Sections:**

- ``<Biomarker>``: Fluorophore/marker information (name, type, color)
- ``<Acquisition>``: Imaging parameters (wavelength, exposure, filters)
- Physical dimensions (pixel size, magnification)
- Channel identification (ID, name, type)

Known Properties
----------------

FastSlide exposes QPTIFF metadata as standardized properties:

- **openslide.mpp-x**: Microns per pixel in X (from PhysicalSizeX)
- **openslide.mpp-y**: Microns per pixel in Y (from PhysicalSizeY)
- **openslide.objective-power**: Objective magnification
- **qptiff.channel-count**: Total number of spectral channels
- **qptiff.channel.*.name**: Name of channel * (e.g., "DAPI")
- **qptiff.channel.*.wavelength**: Wavelength of channel * in nm
- **qptiff.channel.*.exposure**: Exposure time in ms

Pyramid Organization
--------------------

QPTIFF pyramid levels are organized by channel:

.. code-block:: text

   Level 0 (Full Resolution):
     Page 0: Channel 0 (40000×30000 pixels)
     Page 1: Channel 1 (40000×30000 pixels)
     Page 2: Channel 2 (40000×30000 pixels)
     ...
   
   Level 1 (Downsampled 2×):
     Page N: Channel 0 (20000×15000 pixels)
     Page N+1: Channel 1 (20000×15000 pixels)
     Page N+2: Channel 2 (20000×15000 pixels)
     ...
   
   Level 2 (Downsampled 4×):
     Page M: Channel 0 (10000×7500 pixels)
     ...

**Downsampling Strategy:**
  - Each level downsampled by 2× from previous
  - All channels at each level have identical dimensions
  - Typically 4-6 pyramid levels

FastSlide Implementation
------------------------

FastSlide implements QPTIFF support with simplified architecture compared to other formats.

Components
~~~~~~~~~~

1. **QpTiffReader**: Main reader class implementing ``SlideReader`` interface
2. **QptiffPlanBuilder**: Planning stage - enumerates channel pages and tiles
3. **QptiffTileExecutor**: Execution stage - sequential channel-by-channel reading
4. **QptiffMetadataLoader**: XML metadata parser and pyramid builder

**Implementation Files:**
  - ``aifo/fastslide/src/readers/qptiff/qptiff.cpp`` (main reader)
  - ``aifo/fastslide/src/readers/qptiff/qptiff_plan_builder.cpp`` (planning)
  - ``aifo/fastslide/src/readers/qptiff/qptiff_tile_executor.cpp`` (execution)
  - ``aifo/fastslide/src/readers/qptiff/qptiff_metadata_loader.cpp`` (metadata)

Initialization Algorithm
~~~~~~~~~~~~~~~~~~~~~~~~

The initialization phase scans all TIFF pages and parses channel metadata:

**Pseudocode:**

.. code-block:: text

   function Initialize(file_path):
     // 1. Open TIFF file
     tiff_handle = TIFFOpen(file_path, "r")
     if not tiff_handle:
       return error("Failed to open TIFF")
     
     // 2. Get total page count
     total_pages = TIFFNumberOfDirectories(tiff_handle)
     
     if total_pages < 4:
       return error("Too few pages for QPTIFF")
     
     // 3. Scan all pages and classify by ImageType
     channels = []
     pyramid = []
     associated_images = {}
     
     for page in 0 to total_pages-1:
       TIFFSetDirectory(tiff_handle, page)
       
       // Get image dimensions
       width, height = TIFFGetField(tiff_handle, TIFFTAG_IMAGEWIDTH, TIFFTAG_IMAGELENGTH)
       
       // Read XML metadata
       image_desc = TIFFGetField(tiff_handle, TIFFTAG_IMAGEDESCRIPTION)
       if not image_desc:
         continue
       
       // Parse XML
       xml_doc = ParseXML(image_desc)
       root = xml_doc.child("PerkinElmer-QPI-ImageDescription")
       if not root:
         return error("Invalid QPTIFF XML")
       
       image_type = root.child_value("ImageType")
       
       // Classify page by type
       if image_type == "FullResolution" or image_type.empty():
         // Full resolution channel
         channel_info = QpTiffChannelInfo()
         channel_info.page = page
         channel_info.width = width
         channel_info.height = height
         channel_info.channel_id = ExtractChannelID(xml_doc)
         channel_info.wavelength = ExtractWavelength(xml_doc)
         channel_info.exposure = ExtractExposure(xml_doc)
         channel_info.name = ExtractChannelName(xml_doc)
         channels.append(channel_info)
       
       elif image_type == "Thumbnail":
         // Thumbnail starts pyramid levels
         thumbnail_start_page = page
         break  // Stop processing full-res channels
     
     if channels.empty():
       return error("No full resolution channels found")
     
     // 4. Build level 0 from channels
     level0 = QpTiffLevelInfo()
     level0.size = (channels[0].width, channels[0].height)
     for ch in channels:
       level0.pages.append(ch.page)
     pyramid.append(level0)
     
     // 5. Process thumbnail and reduced levels
     num_channels = len(channels)
     current_page = thumbnail_start_page
     
     while current_page < total_pages:
       TIFFSetDirectory(tiff_handle, current_page)
       width, height = TIFFGetField(tiff_handle, TIFFTAG_IMAGEWIDTH, TIFFTAG_IMAGELENGTH)
       
       // Check if this could be start of a pyramid level
       if IsDownsampledFrom(width, height, level0.size):
         // Collect all channels at this level
         level = QpTiffLevelInfo()
         level.size = (width, height)
         
         for ch in 0 to num_channels-1:
           if current_page + ch >= total_pages:
             break
           level.pages.append(current_page + ch)
         
         pyramid.append(level)
         current_page += num_channels
       else:
         // Might be associated image (macro, etc.)
         current_page += 1
     
     // 6. Create handle pool for thread-safe reading
     handle_pool = TiffHandlePool(file_path, num_handles=4)
     
     // 7. Return reader
     return QpTiffReader(channels, pyramid, handle_pool, metadata)

**Key Features:**

- Scans all pages to classify by ImageType
- Groups channels by pyramid level
- Parses XML metadata for each channel
- Handles variable bit depths (8/12/16-bit, float)

Planning Algorithm (QptiffPlanBuilder)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The planning phase creates operations for each channel:

**Pseudocode:**

.. code-block:: text

   function BuildPlan(request, pyramid, output_planar_config, tiff_file):
     // 1. Validate request
     if request.level < 0 or request.level >= len(pyramid):
       return error("Invalid level")
     
     // 2. Get level information
     level_info = pyramid[request.level]
     num_channels = len(level_info.pages)
     
     // 3. Query TIFF structure from first channel page
     tiff_file.SetDirectory(level_info.pages[0])
     
     bits_per_sample = tiff_file.GetBitsPerSample() or 8
     bytes_per_sample = (bits_per_sample + 7) / 8
     
     // Convert to pixel format
     if bits_per_sample <= 8:
       pixel_format = kUInt8
     elif bits_per_sample <= 16:
       pixel_format = kUInt16
     else:
       pixel_format = kFloat32
     
     tile_width, tile_height = GetTileDimensions(tiff_file, level_info)
     is_tiled = tiff_file.IsTiled()
     
     // 4. Determine region bounds
     if request.has_region_bounds():
       x, y = request.region_bounds.x, request.region_bounds.y
       width, height = ceil(request.region_bounds.width), ceil(request.region_bounds.height)
     else:
       x, y = 0, 0
       width, height = level_info.size
     
     // 5. Clamp region to level bounds
     if x >= level_info.size[0] or y >= level_info.size[1]:
       return empty_plan  // Outside bounds
     
     if x + width > level_info.size[0]:
       width = level_info.size[0] - x
     if y + height > level_info.size[1]:
       height = level_info.size[1] - y
     
     // 6. Calculate which tiles intersect the region
     region_x, region_y = uint32(x), uint32(y)
     
     first_tile_x = region_x / tile_width
     first_tile_y = region_y / tile_height
     last_tile_x = (region_x + width - 1) / tile_width
     last_tile_y = (region_y + height - 1) / tile_height
     
     // 7. Create tile operations (one per channel per tile)
     operations = []
     
     for ch in 0 to num_channels-1:
       page = level_info.pages[ch]
       
       for tile_y in first_tile_y to last_tile_y:
         for tile_x in first_tile_x to last_tile_x:
           // Calculate tile bounds
           tile_left = tile_x * tile_width
           tile_top = tile_y * tile_height
           tile_right = min(tile_left + tile_width, level_info.size[0])
           tile_bottom = min(tile_top + tile_height, level_info.size[1])
           
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
           op.tile_coord = (ch, 0)  // Channel stored in X coordinate
           op.source_id = page
           
           // For TIFF, byte_offset is tile/strip index
           if is_tiled:
             tiles_across = (level_info.size[0] + tile_width - 1) / tile_width
             op.byte_offset = tile_y * tiles_across + tile_x
           else:
             op.byte_offset = tile_y  // Strip number
           
           // Estimate tile size
           op.byte_size = tile_width * tile_height * bytes_per_sample
           
           // Transform: source crop, destination position
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
     output.channels = num_channels
     output.pixel_format = pixel_format
     output.planar_config = output_planar_config
     output.background = (0, 0, 0, 255)  // Black background
     
     // 9. Estimate costs
     cost.total_tiles = len(operations)
     cost.total_bytes_to_read = sum(op.byte_size for op in operations)
     cost.tiles_to_decode = cost.total_tiles
     
     // 10. Return plan
     return TilePlan(operations, output, cost)

**Key Features:**

- Creates one operation per channel per tile
- Handles multiple bit depths (8/12/16-bit, float)
- Sequential operation ordering (channel-by-channel)
- Supports both tiled and stripped formats

Execution Algorithm (QptiffTileExecutor)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The execution phase processes channels sequentially:

**Pseudocode:**

.. code-block:: text

   function ExecutePlan(plan, pyramid, tiff_file, writer):
     if plan.operations.empty():
       writer.FillWithBackground(plan.output.background)
       return success
     
     // QPTIFF always executes sequentially (no parallelism)
     // Reason: Channel dependencies, memory constraints, TIFF directory switching
     
     level_info = pyramid[plan.request.level]
     
     // Group operations by page (channel)
     operations_by_page = GroupOperationsByPage(plan.operations)
     
     // Process each channel sequentially
     for page, ops in operations_by_page:
       // Set TIFF directory to this channel's page
       tiff_file.SetDirectory(page)
       
       // Get page state (tile dimensions, format, etc.)
       page_state = GetPageState(tiff_file)
       
       // Execute all tile operations for this channel
       for op in ops:
         // Read tile data
         tile_data = ReadTileData(op, tiff_file, page_state)
         
         // Extract sub-region if needed
         cropped_data = ExtractRegionFromTile(tile_data, op.transform.source, page_state)
         
         // Write to output (accumulate channels)
         writer.WriteTile(op, cropped_data)
     
     return success
   
   function ReadTileData(op, tiff_file, page_state):
     tiff_handle = tiff_file.GetHandle()
     
     // Get tile size
     tile_size = page_state.is_tiled ? TIFFTileSize(tiff_handle) : TIFFStripSize(tiff_handle)
     
     if tile_size <= 0:
       return error("Invalid tile size")
     
     // Use thread-local buffer (even though sequential, for consistency)
     buffer = GetThreadLocalBuffer(tile_size)
     
     // Read encoded tile/strip
     if page_state.is_tiled:
       bytes_read = TIFFReadEncodedTile(tiff_handle, op.byte_offset, buffer, tile_size)
     else:
       bytes_read = TIFFReadEncodedStrip(tiff_handle, op.byte_offset, buffer, tile_size)
     
     if bytes_read <= 0:
       return error("Failed to read tile")
     
     // libtiff automatically decodes LZW/JPEG
     return span(buffer, bytes_read)
   
   function ExtractRegionFromTile(tile_data, source_region, page_state):
     // Similar to Aperio, but handle variable bit depths
     src_x, src_y = source_region.x, source_region.y
     crop_width, crop_height = source_region.width, source_region.height
     tile_width = page_state.tile_width
     bytes_per_sample = page_state.bytes_per_sample
     
     crop_size = crop_width * crop_height * bytes_per_sample
     cropped = GetThreadLocalCropBuffer(crop_size)
     
     // Copy row by row
     for y in 0 to crop_height-1:
       src_offset = ((src_y + y) * tile_width + src_x) * bytes_per_sample
       dst_offset = y * crop_width * bytes_per_sample
       memcpy(cropped + dst_offset, tile_data + src_offset, crop_width * bytes_per_sample)
     
     return span(cropped, crop_size)

**Key Features:**

- **Sequential Execution**: No parallelism (channel dependencies)
- **Page-by-Page Processing**: Process all tiles from one channel before moving to next
- **Channel Accumulation**: Writer accumulates channels in output buffer
- **Bit Depth Handling**: Supports 8/12/16-bit and float

Why Sequential Execution?
~~~~~~~~~~~~~~~~~~~~~~~~~~

QPTIFF uses sequential execution for several reasons:

**1. Channel Dependencies**
   - Spectral unmixing may require specific channel ordering
   - Some workflows need complete channels before processing

**2. Memory Constraints**
   - High channel counts (50-100+) consume significant memory
   - Parallel execution would multiply memory usage
   - Sequential processing keeps memory footprint low

**3. TIFF Directory Switching**
   - Frequent directory changes have overhead
   - Sequential access minimizes directory switches
   - Better cache locality for TIFF structures

**4. Simpler Implementation**
   - No thread synchronization needed
   - No mutex overhead
   - Easier to reason about channel accumulation

**5. I/O Patterns**
   - Sequential page access may improve disk I/O
   - Better prefetching by OS
   - Less seeking on spinning disks

Channel Accumulation
~~~~~~~~~~~~~~~~~~~~

The ``TileWriter`` accumulates channels into the output buffer:

.. code-block:: text

   function WriteTile(op, tile_data):
     channel_index = op.tile_coord.x  // Channel stored in X coord
     dest_x, dest_y = op.transform.dest.x, op.transform.dest.y
     width, height = op.transform.dest.width, op.transform.dest.height
     
     if planar_config == kContiguous:
       // Interleaved: [R₀G₀B₀][R₁G₁B₁][R₂G₂B₂]...
       for y in 0 to height-1:
         for x in 0 to width-1:
           output_offset = ((dest_y + y) * output_width + dest_x + x) * num_channels + channel_index
           tile_offset = y * width + x
           output[output_offset] = tile_data[tile_offset]
     
     elif planar_config == kSeparate:
       // Planar: [R₀R₁R₂...][G₀G₁G₂...][B₀B₁B₂...]
       channel_offset = channel_index * output_width * output_height
       for y in 0 to height-1:
         for x in 0 to width-1:
           output_offset = channel_offset + (dest_y + y) * output_width + dest_x + x
           tile_offset = y * width + x
           output[output_offset] = tile_data[tile_offset]

Performance Characteristics
---------------------------

Sequential Execution Performance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

QPTIFF has different performance characteristics than RGB formats:

- **Sequential Only**: No parallel tile decoding
- **Channel Count Impact**: Performance linear with channel count
- **LZW Decode**: Faster than JPEG (less CPU-intensive)
- **Memory Usage**: Lower than parallel approaches

**Typical Performance:**

- 3-channel RGB equivalent: ~100-150 tiles/sec
- 10-channel multispectral: ~30-40 tiles/sec per channel
- 50-channel hyperspectral: ~20-30 tiles/sec per channel

**Bottlenecks:**

1. Sequential processing (no parallelism)
2. TIFF directory switching overhead
3. Channel accumulation (memory writes)
4. Bit depth conversion (if needed)

Cache Behavior
~~~~~~~~~~~~~~

Tile caching is less effective for QPTIFF:

- **Cache Key**: ``(filename, level, page, tile_index)``
- **Memory**: High memory usage with many channels
- **Hit Rate**: Lower than RGB formats (more unique tiles)
- **Eviction**: More frequent due to high memory pressure

Memory Usage
~~~~~~~~~~~~

QPTIFF consumes more memory than RGB formats:

.. code-block:: text

   Single tile memory:
     RGB (3 channels): 240×240×3 = 170 KB
     10-channel: 240×240×10 = 565 KB
     50-channel: 240×240×50 = 2.8 MB

   Full region (1024×1024):
     RGB: 1024×1024×3 = 3 MB
     10-channel: 1024×1024×10 = 10 MB
     50-channel: 1024×1024×50 = 50 MB

Compatibility
-------------

FastSlide's QPTIFF implementation is compatible with:

- **PerkinElmer Vectra/Phenochart**: Native format
- **Bio-Formats**: Can read QPTIFF files
- **QuPath**: Supports QPTIFF via Bio-Formats
- **ImageJ/Fiji**: Via Bio-Formats plugin

Limitations
-----------

Current limitations of the QPTIFF implementation:

1. **No Parallel Execution**: Sequential only (by design)
2. **Memory Usage**: High for many channels
3. **Spectral Operations**: No built-in spectral unmixing
4. **Bit Depth**: Float32 support tested minimally
5. **Very Large Channel Counts**: >100 channels may be slow

Future Enhancements
-------------------

Planned improvements:

1. **Parallel Channel Reading**: Parallel execution for independent channels
2. **Spectral Unmixing**: Built-in spectral processing
3. **On-Demand Channel Loading**: Load only requested channels
4. **Compressed Output**: Compress output for high channel counts
5. **GPU Acceleration**: GPU-based spectral operations

Test Data
---------

Sample QPTIFF files can be obtained from:
  - PerkinElmer Vectra sample datasets
  - Bio-Formats test data repository

References
----------

- `PerkinElmer QPTIFF Specification <https://www.perkinelmer.com/>`_
- `TIFF Specification <https://www.awaresystems.be/imaging/tiff.html>`_
- `Bio-Formats QPTIFF Support <https://docs.openmicroscopy.org/bio-formats/>`_

.. note::
   The QPTIFF format specification is not publicly documented in detail. This documentation is based on reverse engineering and implementation experience.
