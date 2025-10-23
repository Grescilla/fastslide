Supported File Formats
======================

FastSlide supports multiple whole slide imaging formats commonly used in digital pathology and biomedical research. Each format has unique characteristics in terms of compression, tile organization, and metadata structure.

.. toctree::
   :maxdepth: 2
   :caption: Format Documentation:

   svs
   mrxs
   qptiff

Format Overview
---------------

FastSlide currently supports the following formats:

.. list-table:: Supported Formats
   :header-rows: 1
   :widths: 15 20 20 25 20

   * - Format
     - File Extension
     - Compression
     - Tile Organization
     - Special Features
   * - Aperio SVS
     - .svs
     - JPEG (in TIFF)
     - Tiled TIFF / Strips
     - BigTIFF, associated images
   * - MIRAX (MRXS)
     - .mrxs
     - JPEG / PNG / BMP
     - Hierarchical DAT files
     - Overlapping tiles, spatial index
   * - QPTIFF
     - .tif/.tiff
     - LZW / JPEG / Uncompressed
     - Multi-page TIFF
     - Multi-channel spectral imaging

Format Detection
----------------

FastSlide automatically detects the file format based on:

1. **File Extension**: Initial format hint from filename
2. **Magic Bytes**: TIFF magic numbers, custom headers
3. **Metadata Inspection**: Format-specific metadata tags and structures

The detection process uses a plugin-based architecture where each format plugin:

- Registers supported file extensions
- Implements a ``CanRead()`` method to validate file compatibility
- Provides a factory method to create format-specific readers

Architecture
------------

All format readers follow a common two-stage pipeline:

**Stage 1: Planning** (``PrepareRequest``)
  - Analyze requested region and pyramid level
  - Determine which tiles intersect the region
  - Calculate coordinate transforms and clipping
  - Generate a ``TilePlan`` with operations to execute
  - Estimate I/O costs and execution time

**Stage 2: Execution** (``ExecutePlan``)
  - Read tiles from disk (potentially in parallel)
  - Decode compressed tile data
  - Apply transforms (crop, scale, blend)
  - Write to output buffer
  - Handle errors gracefully

This separation enables:

- **Testing**: Plan generation can be tested without I/O
- **Optimization**: Plans can be analyzed and optimized before execution
- **Caching**: Plans can be cached for repeated reads
- **Batching**: Multiple plans can be merged for efficient execution

Performance Characteristics
---------------------------

.. list-table:: Format Performance Comparison
   :header-rows: 1
   :widths: 15 25 25 35

   * - Format
     - Read Performance
     - Parallel Execution
     - Memory Usage
   * - Aperio SVS
     - Fast (JPEG decode)
     - Yes (handle pool)
     - Low (tile-based)
   * - MRXS
     - Fast (cached tiles)
     - Yes (thread pool)
     - Medium (spatial index)
   * - QPTIFF
     - Medium (LZW decode)
     - No (sequential)
     - High (multi-channel)

Adding New Formats
------------------

To add support for a new format:

1. Create format plugin in ``src/readers/<format_name>/``
2. Implement ``SlideReader`` interface
3. Create plan builder class (optional but recommended)
4. Create tile executor class (optional but recommended)
5. Register format in plugin system
6. Add comprehensive tests

See the developer guide for detailed instructions on implementing new format readers.

