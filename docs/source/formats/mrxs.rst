MIRAX (MRXS) Format
===================

The MIRAX (MRXS) format is a proprietary whole slide imaging format developed by 3DHISTECH (now part of Sysmex Corporation). It's widely used in digital pathology for storing high-resolution microscopy images from PANNORAMIC series scanners.

Format Specification
--------------------

File Extensions
~~~~~~~~~~~~~~~

- ``.mrxs``

Detection Criteria
~~~~~~~~~~~~~~~~~~

FastSlide detects a file as MIRAX if:

1. The file is not a TIFF
2. The filename ends with ``.mrxs``
3. A directory exists in the same location as the file, with the same name as the file minus the extension
4. A file named ``Slidedat.ini`` exists in that directory

File Structure
--------------

MIRAX slides consist of multiple files organized in a hierarchical structure:

.. code-block:: text

   Slide001.mrxs                    # Main index file (INI format)
   Slide001/                        # Data directory
   ├── Slidedat.ini                 # Metadata (INI format with sections)
   ├── Data/                        # Data subdirectory
   │   ├── Index.dat                # Tile index (binary, 32-bit integers)
   │   ├── Data0000.dat             # Tile data file 0
   │   ├── Data0001.dat             # Tile data file 1
   │   ├── ...                      # Additional data files
   │   └── DataNNNN.dat
   └── [optional position files]    # Camera position data

Overview
--------

MIRAX stores slides in JPEG, PNG, or BMP formats. Because JPEG does not allow for large images, and JPEG and PNG provide very poor support for random-access decoding of part of an image, multiple images are needed to encode a slide. To avoid having many individual files, MIRAX packs these images into a small number of data files. The index file provides offsets into the data files for each required piece of data.

**Key Characteristics:**

- **Compression**: JPEG (most common), PNG, or BMP per tile
- **Organization**: Multi-file with separate index and data files
- **Tile Layout**: Non-uniform, overlapping tiles from camera positions
- **Pyramid**: Hierarchical levels with progressive downsampling (2× per level)
- **Metadata**: INI-style Slidedat.ini with comprehensive scan parameters

Tile Organization
~~~~~~~~~~~~~~~~~

The camera on MIRAX scanners takes overlapping photos and records the position of each one. Each photo is then split into multiple images which do not overlap. Overlaps only occur between images that come from different photos.

To generate level n + 1, each image from level n is downsampled by 2 and then concatenated into a new image, 4 old images per new image (2 × 2). This process is repeated for each level, irrespective of image overlaps. Therefore, at sufficiently high levels, a single image can contain one or more embedded overlaps of non-integral width.

Slidedat.ini Metadata File
--------------------------

The ``Slidedat.ini`` file contains scan metadata in INI format with multiple sections:

.. code-block:: ini

   [GENERAL]
   SLIDE_ID = 123456789
   IMAGENUMBER_X = 8
   IMAGENUMBER_Y = 6
   OBJECTIVE_MAGNIFICATION = 20
   MICROMETER_PER_PIXEL_X = 0.2432
   MICROMETER_PER_PIXEL_Y = 0.2432
   
   [HIERARCHICAL]
   HIER_COUNT = 9                  # Number of pyramid levels
   HIER_0_COUNT = 48               # Number of tiles in level 0
   HIER_0_VAL = 1                  # Value identifier for level 0
   HIER_1_COUNT = 12
   HIER_1_VAL = 2
   ...
   
   [DATAFILE]
   FILE_0 = Data0000.dat
   FILE_1 = Data0001.dat
   ...
   
   [NONHIER_*_VAL_*]                # Non-hierarchical data (labels, macros)
   SECTION = ScanDataLayer_SlidePreview
   ...

**Key Sections:**

- ``[GENERAL]``: Basic slide information (dimensions, magnification, pixel size)
- ``[HIERARCHICAL]``: Pyramid structure (level count, tile counts per level)
- ``[DATAFILE]``: Mapping of file numbers to data file names
- ``[LAYER_*_LEVEL_*]``: Per-level metadata (dimensions, position, colors)
- ``[NONHIER_*]``: Associated images and auxiliary data

Index.dat File
--------------

The index file starts with a five-character ASCII version string, followed by the SLIDE_ID from the slidedat file. The rest of the file consists of 32-bit little-endian integers (unaligned), which can be data values or pointers to byte offsets within the index file.

**Structure:**

1. **Header**: 5-byte version + SLIDE_ID
2. **Root Pointers**: First two integers point to hierarchical and non-hierarchical root tables
3. **Offset Tables**: Tables contain one record per VAL in HIERARCHICAL/NONHIER sections
4. **Record Format**: Each record is a pointer to a linked list of data pages
5. **Data Pages**: Contain tile metadata (image index, file number, offset, length)

The first two integers point to offset tables for the hierarchical and nonhierarchical roots, respectively. These tables contain one record for each VAL in the HIERARCHICAL slidedat section. For example, the record for NONHIER_1_VAL_2 would be stored at ``nonhier_root + 4 * (NONHIER_0_COUNT + 2)``.

Each record is a pointer to a linked list of data pages. The first two values in a data page are the number of data items in the page and a pointer to the next page. The first page always has 0 data items, and the last page has a 0 next pointer.

**Hierarchical Records** (pyramid levels):

There is one hierarchical record for each zoom level. The record contains data items consisting of:
  - Image index: ``image_y * GENERAL.IMAGENUMBER_X + image_x``
  - Offset and length within data file
  - File number (maps to DATAFILE section)
  
Image coordinates which are not multiples of the zoom level's downsample factor are omitted.

**Nonhierarchical Records** (associated images):

Nonhierarchical records refer to associated images and additional metadata. Nonhierarchical data items consist of two values which are usually zero, followed by an offset, length, and file number as in hierarchical records.

Data Files (Data*.dat)
----------------------

A data file begins with a header containing:
  - Five-character ASCII version string
  - SLIDE_ID from slidedat file
  - File number encoded into three ASCII characters
  - 256 bytes of padding

In newer slides, the SLIDE_ID and file number are encoded as UTF-16LE, so the second half of each value is truncated away.

The remainder of the file contains packed data referenced by the index file. Each tile is stored as a compressed image (JPEG/PNG/BMP) at the byte offset specified in the index.

Slide Position File
-------------------

The slide position file is referenced by the ``VIMSLIDE_POSITION_BUFFER.default`` nonhierarchical section. It contains one entry for each camera position (not each image position) in row-major order.

**Entry Format** (9 bytes per camera position):
  - Flag byte (1 byte): 1 if slide contains images for this position, 0 otherwise (always 0 in versions < 1.9)
  - X pixel coordinate (4 bytes, little-endian, may be negative)
  - Y pixel coordinate (4 bytes, little-endian, may be negative)

In slides with ``CURRENT_SLIDE_VERSION`` ≥ 2.2, the slide position file is compressed with DEFLATE and referenced by the ``StitchingIntensityLayer.StitchingIntensityLevel`` nonhierarchical section. In some such slides, the nonhierarchical record has a second data item which points to a DEFLATE-compressed blob with four bytes of additional metadata per camera position.

Associated Images
-----------------

MIRAX supports several associated images:

- **label**: The image named "ScanDataLayer_SlideBarcode" in Slidedat.ini (optional)
- **macro**: The image named "ScanDataLayer_SlideThumbnail" in Slidedat.ini (optional)
- **thumbnail**: The image named "ScanDataLayer_SlidePreview" in Slidedat.ini (optional)

Known Properties
----------------

All key-value data stored in the Slidedat.ini file are encoded as properties prefixed with "mirax.".

**Standard Properties:**

- **openslide.mpp-x**: Normalized MICROMETER_PER_PIXEL_X from level 0 (typically ``mirax.LAYER_0_LEVEL_0_SECTION.MICROMETER_PER_PIXEL_X``)
- **openslide.mpp-y**: Normalized MICROMETER_PER_PIXEL_Y from level 0 (typically ``mirax.LAYER_0_LEVEL_0_SECTION.MICROMETER_PER_PIXEL_Y``)
- **openslide.objective-power**: Normalized ``mirax.GENERAL.OBJECTIVE_MAGNIFICATION``

FastSlide Implementation
------------------------

FastSlide implements MRXS support through a modular architecture with three main components plus a spatial indexing system.

Components
~~~~~~~~~~

1. **MrxsReader**: Main reader class implementing ``SlideReader`` interface
2. **MrxsPlanBuilder**: Planning stage - queries spatial index and creates tile plans
3. **MrxsTileExecutor**: Execution stage - reads tiles in parallel with blending
4. **MrxsSpatialIndex**: R-tree spatial index for fast tile lookup

**Implementation Files:**
  - ``aifo/fastslide/src/readers/mrxs/mrxs.cpp`` (main reader)
  - ``aifo/fastslide/src/readers/mrxs/mrxs_plan_builder.cpp`` (planning)
  - ``aifo/fastslide/src/readers/mrxs/mrxs_tile_executor.cpp`` (execution)
  - ``aifo/fastslide/src/readers/mrxs/spatial_index.cpp`` (R-tree index)

Initialization Algorithm
~~~~~~~~~~~~~~~~~~~~~~~~

The initialization phase parses metadata and builds spatial indexes:

**Step 1: Parse Slidedat.ini**

.. code-block:: text

   1. Open and parse Slidedat.ini as INI file
   2. Read [GENERAL] section:
      - SLIDE_ID
      - IMAGENUMBER_X, IMAGENUMBER_Y (tile grid dimensions)
      - OBJECTIVE_MAGNIFICATION
      - MICROMETER_PER_PIXEL_X, MICROMETER_PER_PIXEL_Y
   3. Read [HIERARCHICAL] section:
      - HIER_COUNT (number of pyramid levels)
      - HIER_*_COUNT (tile count per level)
      - HIER_*_VAL (value identifiers)
   4. Read [DATAFILE] section:
      - FILE_* mappings (file numbers to filenames)
   5. Read [LAYER_*_LEVEL_*] sections:
      - Dimensions, position, background color per level

**Step 2: Parse Index.dat**

.. code-block:: text

   1. Read 5-byte version string
   2. Read SLIDE_ID and verify against Slidedat.ini
   3. Read two root pointers:
      - hierarchical_root_offset
      - nonhierarchical_root_offset
   4. For each pyramid level:
      a. Read hierarchical record pointer from root table
      b. Follow linked list of data pages
      c. For each data item in pages:
         - image_index = data[i]
         - offset = data[i+1]
         - length = data[i+2]
         - file_number = data[i+3]
         - Convert image_index to (x, y) tile coordinates
         - Store TileInfo(x, y, offset, length, file_number)
   5. Parse nonhierarchical records for associated images

**Step 3: Build Spatial R-tree Index**

.. code-block:: text

   For each pyramid level:
     1. Create empty R-tree index
     2. For each tile in level:
        a. Read camera position from position file (if available)
        b. Calculate tile bounding box:
           - Use camera position + tile dimensions
           - Account for fractional positioning
        c. Create SpatialTile entry:
           - bbox = (min_x, min_y, max_x, max_y)
           - tile_info = (x, y, offset, length, file_number, etc.)
        d. Insert into R-tree: Insert(bbox, tile_info)
     3. Store index for this level

The R-tree enables O(log n) queries for tiles intersecting a region.

Planning Algorithm (MrxsPlanBuilder)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The planning phase queries the spatial index and creates tile operations:

**Pseudocode:**

.. code-block:: text

   function BuildPlan(request, reader):
     // 1. Validate request
     if request.level < 0 or request.level >= reader.level_count:
       return error("Invalid level")
     
     // 2. Get level information
     level_info = reader.GetLevelInfo(request.level)
     zoom_level = reader.slide_info.zoom_levels[request.level]
     
     // 3. Get spatial index for this level
     spatial_index = reader.GetSpatialIndex(request.level)
     
     // 4. Determine region bounds (preserves fractional coordinates)
     if request.has_region_bounds():
       x, y = request.region_bounds.x, request.region_bounds.y
       width, height = ceil(request.region_bounds.width), ceil(request.region_bounds.height)
     else:
       x, y = 0.0, 0.0
       width, height = level_info.dimensions
     
     // 5. Query spatial index for intersecting tiles
     tile_indices = spatial_index.QueryRegion(x, y, width, height)
     
     if tile_indices.empty():
       return empty_plan  // No tiles, fill with background
     
     // 6. Create tile operations with fractional positioning
     operations = []
     for idx in tile_indices:
       spatial_tile = spatial_index.spatial_tiles[idx]
       
       // Calculate tile position relative to region origin
       tile_x_in_level = spatial_tile.bbox.min[0]
       tile_y_in_level = spatial_tile.bbox.min[1]
       rel_x = tile_x_in_level - x
       rel_y = tile_y_in_level - y
       
       // Extract integer and fractional components
       dest_x = floor(rel_x)
       dest_y = floor(rel_y)
       frac_x = rel_x - dest_x  // Subpixel offset (0.0 to 1.0)
       frac_y = rel_y - dest_y
       
       // Calculate clipping for tiles extending outside region
       src_width = ceil(spatial_tile.tile_width)
       src_height = ceil(spatial_tile.tile_height)
       src_offset_x = round(spatial_tile.subregion_x)
       src_offset_y = round(spatial_tile.subregion_y)
       
       final_dest_x, final_dest_y = 0, 0
       final_width, final_height = src_width, src_height
       
       // Clip left/top if tile extends before region origin
       if dest_x < 0:
         clip_amount = -dest_x
         src_offset_x += clip_amount
         final_width = max(0, src_width - clip_amount)
         final_dest_x = 0
       else:
         final_dest_x = dest_x
       
       if dest_y < 0:
         clip_amount = -dest_y
         src_offset_y += clip_amount
         final_height = max(0, src_height - clip_amount)
         final_dest_y = 0
       else:
         final_dest_y = dest_y
       
       // Clip right/bottom if tile extends beyond region
       if final_dest_x + final_width > width:
         final_width = max(0, width - final_dest_x)
       if final_dest_y + final_height > height:
         final_height = max(0, height - final_dest_y)
       
       // Skip tiles completely outside region
       if final_width == 0 or final_height == 0:
         continue
       
       // Create tile operation
       op = TileReadOp()
       op.level = request.level
       op.tile_coord = (spatial_tile.x, spatial_tile.y)
       op.source_id = spatial_tile.data_file_number
       op.byte_offset = spatial_tile.offset
       op.byte_size = spatial_tile.length
       
       // Set transform (source crop, destination position)
       op.transform.source = (src_offset_x, src_offset_y, final_width, final_height)
       op.transform.dest = (final_dest_x, final_dest_y, final_width, final_height)
       
       // Populate blend metadata for fractional positioning
       op.blend_metadata = BlendMetadata()
       op.blend_metadata.fractional_x = frac_x
       op.blend_metadata.fractional_y = frac_y
       op.blend_metadata.weight = 1.0
       op.blend_metadata.mode = BlendMode.kAverage  // Average overlaps
       op.blend_metadata.enable_subpixel_resampling = true
       
       operations.append(op)
     
     // 7. Create output specification
     output = OutputSpec()
     output.dimensions = (width, height)
     output.channels = 3  // RGB
     output.pixel_format = kUInt8
     output.background = zoom_level.background_color_rgb
     
     // 8. Estimate costs
     cost.total_tiles = len(operations)
     cost.total_bytes_to_read = sum(op.byte_size for op in operations)
     cost.tiles_to_decode = cost.total_tiles
     
     // 9. Return plan
     return TilePlan(operations, output, cost, actual_region)

**Key Features:**

- Preserves fractional coordinates throughout planning
- Calculates subpixel offsets for each tile
- Handles clipping for tiles at region boundaries
- Sets blend mode to ``kAverage`` for seamless stitching

Execution Algorithm (MrxsTileExecutor)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The execution phase reads and decodes tiles in parallel with blending:

**Pseudocode:**

.. code-block:: text

   function ExecutePlan(plan, reader, writer):
     if plan.operations.empty():
       writer.FillWithBackground(plan.output.background)
       return success
     
     // Submit all tiles to global thread pool for parallel processing
     thread_pool = GetGlobalThreadPool()
     mutex = Mutex()
     error_count = AtomicInt(0)
     
     futures = thread_pool.submit_all(plan.operations, lambda op:
       status = ExecuteTileOperation(op, reader, writer, mutex)
       if not status.ok():
         error_count.increment()
         log_warning("Tile failed: " + status.message())
     )
     
     // Wait for all tiles to complete
     futures.wait()
     
     if error_count > 0:
       log_warning(error_count + " tile(s) failed during execution")
     
     return success
   
   function ExecuteTileOperation(op, reader, writer, mutex):
     // 1. Read compressed tile data from Data*.dat file
     data_file_path = reader.GetDataFilePath(op.source_id)
     compressed_data = ReadBytesFromFile(data_file_path, op.byte_offset, op.byte_size)
     
     // 2. Decode based on compression type (JPEG/PNG/BMP)
     image = DecodeImage(compressed_data)  // Format auto-detected
     
     // 3. Extract sub-region if needed (for clipped tiles)
     if NeedsSubRegionExtraction(image, op.transform.source):
       image = ExtractSubRegion(image, op.transform.source)
     
     // 4. Apply fractional resampling if needed
     if op.blend_metadata.has_value() and 
        (op.blend_metadata.fractional_x != 0 or op.blend_metadata.fractional_y != 0):
       
       if op.blend_metadata.enable_subpixel_resampling:
         // Apply Magic Kernel resampling for subpixel shifts
         image = ResampleWithMagicKernel(
           image,
           op.blend_metadata.fractional_x,
           op.blend_metadata.fractional_y
         )
     
     // 5. Write to output buffer with mutex protection (for thread safety)
     lock(mutex):
       writer.WriteTile(
         op,
         image.data(),
         image.width(),
         image.height(),
         3  // RGB channels
       )
     
     return success

**Key Features:**

- **Parallel Execution**: All tiles submitted to thread pool
- **Format Detection**: Auto-detects JPEG/PNG/BMP from magic bytes
- **Subpixel Resampling**: Magic Kernel for fractional shifts
- **Thread-Safe Writing**: Mutex protects shared output buffer
- **Weighted Blending**: TileWriter handles overlap averaging

Tile Overlap Handling
~~~~~~~~~~~~~~~~~~~~~~

MRXS tiles overlap at boundaries due to camera positioning. FastSlide handles this with:

**1. Fractional Positioning**
   - Tiles positioned at fractional pixel coordinates (e.g., x=100.3, y=200.7)
   - ``fractional_x`` and ``fractional_y`` store subpixel offsets
   - Preserved through planning pipeline

**2. Subpixel Resampling**
   - Magic Kernel resampling shifts tiles by fractional amounts
   - Provides smooth, high-quality stitching
   - Enabled via ``blend_metadata.enable_subpixel_resampling``

**3. Weighted Averaging**
   - Overlapping regions averaged pixel-by-pixel
   - ``TileWriter`` accumulates: ``accumulator[x,y] += pixel_value * weight``
   - Final output: ``output[x,y] = accumulator[x,y] / total_weight[x,y]``
   - Produces seamless blending without visible seams

**4. Blend Mode**
   - Set to ``BlendMode::kAverage`` for all MRXS tiles
   - Other modes (kOverwrite, kMaxIntensity) available but unused

Spatial Indexing
~~~~~~~~~~~~~~~~

FastSlide uses an R-tree spatial index for efficient tile queries:

**Benefits:**
  - O(log n) query time for region intersection
  - Handles non-uniform tile layouts
  - Scales to large slides (thousands of tiles)
  - Critical for performance

**Algorithm:**
  - Tiles inserted with bounding boxes during initialization
  - Query returns indices of tiles intersecting region
  - Bulk-loaded for optimal tree structure

Performance Characteristics
---------------------------

- **Parallel Execution**: Always parallel (global thread pool)
- **Typical Speedup**: 4-6× on 8-core CPU
- **Tile Cache**: Effective due to small tile sizes (~64×64 to 256×256 pixels)
- **Spatial Index**: Fast queries even for large regions
- **Blend Overhead**: 10-20% for subpixel resampling and averaging
- **I/O Pattern**: Random access to data files (offset-based reads)

Compatibility
-------------

FastSlide's MRXS implementation is compatible with:

- **OpenSlide**: Produces visually identical output for most slides
- **3DHISTECH CaseViewer**: Can read slides processed by FastSlide
- **QuPath**: Compatible slide opening and metadata parsing

Test Data
---------

Sample MIRAX files are available at:
  https://openslide.cs.cmu.edu/download/openslide-testdata/Mirax/

References
----------

- `OpenSlide MIRAX Format Documentation <https://openslide.org/formats/mirax/>`_
- `Introduction to MIRAX/MRXS <https://lists.andrew.cmu.edu/pipermail/openslide-users/2012-July/000373.html>`_
- 3DHISTECH PANNORAMIC Scanner Documentation

.. note::
   The MIRAX format is proprietary. This documentation is based on reverse engineering by the OpenSlide project and FastSlide's implementation experience.
