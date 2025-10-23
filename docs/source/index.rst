FastSlide Documentation
=======================

.. toctree::
   :maxdepth: 2
   :caption: Contents:

   overview
   architecture
   caching
   formats/index
   api/index
   guides/index
   examples/index

Documentation Structure
-----------------------

FastSlide provides two complementary documentation systems:

**This Sphinx Documentation** (You are here)
   - **Overview and Guides**: High-level architecture, tutorials, and examples
   - **Python API**: Complete Python interface documentation
   - **C++ API Overview**: Quick reference with Breathe integration
   - **Cross-references**: Links between C++ and Python APIs

**`Doxygen Documentation <../doxygen/index.html>`_** (Detailed C++ Reference)
   - **Interactive Diagrams**: Zoomable SVG dependency graphs
   - **Complete C++ API**: Every class, function, and namespace
   - **Call Graphs**: Function dependencies and relationships
   - **Inheritance Diagrams**: Visual class hierarchies
   - **Source Browser**: Browse C++ source code with syntax highlighting

.. tip::
   **For C++ Developers**: Start with the `Doxygen Documentation <../doxygen/index.html>`_ 
   for comprehensive API details and interactive visualizations.
   
   **For Python Users**: Continue with the :doc:`api/python_api` for complete Python interface docs.
   
   **For New Contributors**: See the :doc:`guides/index` for step-by-step implementation guides.

.. image:: https://img.shields.io/badge/version-2.0-blue.svg
   :alt: Version 0.0.1

.. image:: https://img.shields.io/badge/C++-20-green.svg
   :alt: C++20

.. image:: https://img.shields.io/badge/Python-3.11+-green.svg
   :alt: Python 3.11+

FastSlide is a high-performance whole slide image reader library with both C++ and Python APIs.

Features
--------

🚀 **High Performance**
   - Memory-efficient tile caching
   - Multi-threaded processing
   - Optimized I/O operations

🔧 **Multi-Format Support**
   - MRXS (3DHistech)
   - Aperio SVS
   - QPTIFF
   - Generic TIFF

🌐 **Cross-Platform**
   - macOS, Linux
   - x86_64 and ARM64 architectures

🐍 **Dual APIs**
   - Native C++20 API with modern features
   - Python bindings with NumPy integration

Quick Start
-----------

C++ API
~~~~~~~

.. code-block:: cpp

   #include <fastslide/slide_reader.h>
   
   auto reader = fastslide::SlideReader::Open("slide.mrxs");
   if (!reader) {
       // Handle error
       return;
   }
   
   // Read a region at level 0
   auto image = reader->ReadRegion(0, {0, 0}, {1024, 1024});
   
   // Get slide properties
   auto properties = reader->GetProperties();
   std::cout << "Slide dimensions: " 
             << properties.base_dimensions.width << "x" 
             << properties.base_dimensions.height << std::endl;

Python API
~~~~~~~~~~

.. code-block:: python

   import fastslide
   
   # Open a slide
   slide = fastslide.SlideReader("slide.mrxs")
   
   # Read a region
   region = slide.read_region((0, 0), 0, (1024, 1024))
   
   # Get slide properties
   print(f"Slide dimensions: {slide.dimensions}")
   print(f"Available levels: {len(slide.level_dimensions)}")

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`