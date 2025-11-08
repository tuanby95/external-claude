# libvips Architecture Documentation

## Overview

libvips is a fast, memory-efficient image processing library written in C. It uses a demand-driven, streaming architecture that processes images in small regions, making it capable of handling very large images without loading them entirely into memory. It's built on GObject and provides bindings for multiple languages.

---

## C4 Model Diagrams for libvips

### Level 1: System Context Diagram for libvips

```mermaid
graph TB
    subgraph "Image Processing Applications"
        imgproxy[imgproxy<br/>Go application]
        sharpLib[sharp<br/>Node.js library]
        pyVips[pyvips<br/>Python library]
        rubyVips[ruby-vips<br/>Ruby library]
        phpVips[php-vips<br/>PHP extension]
    end

    libvips[libvips<br/>Fast image processing library<br/>Demand-driven streaming<br/>architecture]

    subgraph "System Libraries & Dependencies"
        glib[GLib/GObject<br/>Core object system<br/>and utilities]
        imageLibs[Image Format Libraries<br/>libjpeg, libpng, libwebp,<br/>libheif, libtiff, etc.]
        mathLibs[Math Libraries<br/>FFTW, OpenEXR,<br/>LAPACK, etc.]
        fileSystem[File System<br/>Local storage,<br/>memory buffers]
    end

    subgraph "Hardware Resources"
        cpu[CPU<br/>SIMD operations<br/>Multi-threading]
        memory[Memory<br/>Buffer management<br/>Partial image loading]
        disk[Disk I/O<br/>Sequential read/write<br/>Tiled access]
    end

    imgproxy -->|FFI/CGo calls| libvips
    sharpLib -->|Node-API bindings| libvips
    pyVips -->|Python C API| libvips
    rubyVips -->|Ruby FFI| libvips
    phpVips -->|PHP extension API| libvips

    libvips -->|Uses for object system| glib
    libvips -->|Delegates format encoding/decoding| imageLibs
    libvips -->|Uses for mathematical operations| mathLibs
    libvips -->|Reads/writes image data| fileSystem

    libvips -.->|Utilizes| cpu
    libvips -.->|Manages| memory
    libvips -.->|Accesses| disk

    classDef application fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef external fill:#999,stroke:#666,color:#fff
    classDef hardware fill:#6c8ebf,stroke:#4a6078,color:#fff

    class imgproxy,sharpLib,pyVips,rubyVips,phpVips application
    class libvips system
    class glib,imageLibs,mathLibs,fileSystem external
    class cpu,memory,disk hardware
```

### Level 2: Container Diagram for libvips

```mermaid
graph TB
    Client[Application Code<br/>imgproxy, sharp, pyvips, etc.]

    subgraph libvips[libvips Library]
        API[Public API Layer<br/>C functions<br/>vips_image_new_*,<br/>vips_operation_*]

        subgraph "Core Components"
            VipsImage[VipsImage<br/>Image representation<br/>Metadata, pixels, regions]
            VipsOperation[VipsOperation<br/>Operation registry<br/>and execution]
            VipsRegion[VipsRegion<br/>Partial image access<br/>Demand-driven evaluation]
            VipsBuffer[VipsBuffer<br/>Memory management<br/>Buffer pools]
        end

        subgraph "Image I/O Layer"
            Loaders[Format Loaders<br/>JPEG, PNG, WebP,<br/>TIFF, AVIF, HEIF, etc.]
            Savers[Format Savers<br/>Encode to various formats]
            Foreign[Foreign API<br/>Format detection<br/>and delegation]
        end

        subgraph "Operations Layer"
            Arithmetic[Arithmetic Ops<br/>add, subtract, multiply]
            Conversion[Conversion Ops<br/>resize, crop, rotate]
            Color[Color Ops<br/>colourspace, ICC profiles]
            Morphology[Morphology Ops<br/>erode, dilate, convolution]
            Frequency[Frequency Ops<br/>FFT, filtering]
        end

        subgraph "Threading & Memory"
            ThreadPool[Thread Pool<br/>vips_concurrency_set<br/>Work distribution]
            MemoryManager[Memory Manager<br/>Tile cache, buffers<br/>Memory limits]
        end

        subgraph "GObject Foundation"
            GObject[GObject System<br/>Type system, properties<br/>signals, reference counting]
        end
    end

    subgraph "External Libraries"
        libjpeg[libjpeg-turbo<br/>JPEG codec]
        libpng[libpng<br/>PNG codec]
        libwebp[libwebp<br/>WebP codec]
        libheif[libheif<br/>HEIF/AVIF codec]
        glib2[GLib 2.0<br/>Core utilities]
    end

    Client -->|Function calls| API
    API --> VipsImage
    API --> VipsOperation
    VipsImage --> VipsRegion
    VipsImage --> VipsBuffer
    VipsOperation --> Arithmetic
    VipsOperation --> Conversion
    VipsOperation --> Color
    VipsOperation --> Morphology
    VipsOperation --> Frequency
    VipsRegion --> MemoryManager
    VipsBuffer --> MemoryManager
    Conversion --> ThreadPool
    Arithmetic --> ThreadPool
    API --> Foreign
    Foreign --> Loaders
    Foreign --> Savers
    Loaders --> libjpeg
    Loaders --> libpng
    Loaders --> libwebp
    Loaders --> libheif
    Savers --> libjpeg
    Savers --> libpng
    Savers --> libwebp
    Savers --> libheif
    VipsImage --> GObject
    VipsOperation --> GObject
    VipsRegion --> GObject
    GObject --> glib2
    MemoryManager --> glib2

    classDef client fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef external fill:#999,stroke:#666,color:#fff

    class Client client
    class API,VipsImage,VipsOperation,VipsRegion,VipsBuffer,Loaders,Savers,Foreign container
    class Arithmetic,Conversion,Color,Morphology,Frequency,ThreadPool,MemoryManager,GObject component
    class libjpeg,libpng,libwebp,libheif,glib2 external
```

### Level 3: Component Diagram - Image Processing Pipeline

```mermaid
graph TB
    subgraph Pipeline[libvips Processing Pipeline]
        subgraph "Input Stage"
            FileLoader[File Loader<br/>Detect format<br/>Open file handle]
            HeaderReader[Header Reader<br/>Read metadata<br/>dimensions, bands, format]
            ImageBuilder[Image Builder<br/>Create VipsImage object<br/>Set up pipeline]
        end

        subgraph "Processing Graph"
            OperationGraph[Operation Graph<br/>Build computation DAG<br/>Lazy evaluation]
            Optimizer[Graph Optimizer<br/>Combine operations<br/>Eliminate redundancy]
        end

        subgraph "Execution Engine"
            Scheduler[Demand Scheduler<br/>Determine required regions<br/>Schedule computation]
            RegionGenerator[Region Generator<br/>Generate image regions<br/>on demand]
            TileProcessor[Tile Processor<br/>Process tiles in parallel<br/>Apply operations]
        end

        subgraph "Memory Management"
            RegionCache[Region Cache<br/>Cache computed regions<br/>LRU eviction]
            BufferPool[Buffer Pool<br/>Reuse memory buffers<br/>Reduce allocations]
            MemoryLimiter[Memory Limiter<br/>Track memory usage<br/>Enforce limits]
        end

        subgraph "Output Stage"
            FormatConverter[Format Converter<br/>Convert pixel format<br/>Apply color profiles]
            Encoder[Image Encoder<br/>Encode to target format<br/>Apply compression]
            FileWriter[File Writer<br/>Write to disk or memory<br/>Streaming output]
        end

        subgraph "Threading"
            WorkQueue[Work Queue<br/>Tile computation tasks]
            ThreadPool[Thread Pool<br/>Worker threads<br/>VIPS_CONCURRENCY]
        end
    end

    FormatLibs[Format Libraries<br/>libjpeg, libpng, libwebp]
    GLibUtils[GLib Utilities<br/>Memory allocation,<br/>thread management]

    FileLoader -->|Read header| HeaderReader
    HeaderReader -->|Create| ImageBuilder
    ImageBuilder -->|Build| OperationGraph
    OperationGraph -->|Optimize| Optimizer
    Optimizer -->|Schedule| Scheduler
    Scheduler -->|Request regions| RegionGenerator
    RegionGenerator -->|Generate tiles| TileProcessor
    TileProcessor -->|Submit work| WorkQueue
    WorkQueue -->|Dispatch| ThreadPool
    ThreadPool -->|Process| TileProcessor
    TileProcessor -->|Cache results| RegionCache
    RegionCache -->|Use| BufferPool
    BufferPool -->|Enforce| MemoryLimiter
    TileProcessor -->|Convert| FormatConverter
    FormatConverter -->|Encode| Encoder
    Encoder -->|Write| FileWriter
    FileLoader -.->|Delegate| FormatLibs
    Encoder -.->|Delegate| FormatLibs
    BufferPool -.->|Use| GLibUtils
    ThreadPool -.->|Use| GLibUtils

    classDef input fill:#c3e6cb,stroke:#28a745,color:#000
    classDef process fill:#d1ecf1,stroke:#17a2b8,color:#000
    classDef memory fill:#fff3cd,stroke:#ffc107,color:#000
    classDef output fill:#f8d7da,stroke:#dc3545,color:#000
    classDef thread fill:#e2e3e5,stroke:#6c757d,color:#000
    classDef external fill:#999,stroke:#666,color:#fff

    class FileLoader,HeaderReader,ImageBuilder input
    class OperationGraph,Optimizer,Scheduler,RegionGenerator,TileProcessor process
    class RegionCache,BufferPool,MemoryLimiter memory
    class FormatConverter,Encoder,FileWriter output
    class WorkQueue,ThreadPool thread
    class FormatLibs,GLibUtils external
```

---

## Class Diagram for libvips

```mermaid
classDiagram
    class GObject {
        <<GLib Base Class>>
        +gint ref_count
        +GData *qdata
        +g_object_ref()
        +g_object_unref()
        +g_object_get_property()
        +g_object_set_property()
    }

    class VipsObject {
        <<Abstract Base>>
        +gboolean constructed
        +gboolean static_object
        +char *nickname
        +char *description
        +vips_object_build()
        +vips_object_print_class()
        +vips_object_sanity()
    }

    class VipsImage {
        +gint Xsize
        +gint Ysize
        +gint Bands
        +VipsBandFormat BandFmt
        +VipsCoding Coding
        +VipsInterpretation Type
        +double Xres
        +double Yres
        +VipsImageType dtype
        +void *data
        +VipsRegion **regions
        +vips_image_new()
        +vips_image_new_from_file()
        +vips_image_new_from_buffer()
        +vips_image_write_to_file()
        +vips_image_get_width()
        +vips_image_get_height()
        +vips_image_wio_input()
        +vips_image_pio_input()
    }

    class VipsRegion {
        +VipsImage *im
        +VipsRect valid
        +RegionType type
        +VipsPel *data
        +int bpl
        +vips_region_new()
        +vips_region_prepare()
        +vips_region_fetch()
        +vips_region_copy()
        +vips_region_position()
    }

    class VipsOperation {
        <<Abstract>>
        +VipsOperationFlags flags
        +gboolean sequential
        +gint64 pixels
        +vips_operation_new()
        +vips_operation_build()
        +vips_call()
        +vips_cache_operation_build()
    }

    class VipsArithmetic {
        <<Arithmetic Operations>>
        +VipsImage *left
        +VipsImage *right
        +VipsImage *out
        +vips_add()
        +vips_subtract()
        +vips_multiply()
        +vips_divide()
        +vips_linear()
    }

    class VipsConversion {
        <<Conversion Operations>>
        +VipsImage *in
        +VipsImage *out
        +vips_resize()
        +vips_crop()
        +vips_embed()
        +vips_extract_area()
        +vips_rot()
        +vips_flip()
    }

    class VipsResample {
        <<Resampling Operations>>
        +VipsImage *in
        +VipsImage *out
        +VipsKernel kernel
        +vips_resize()
        +vips_shrink()
        +vips_reduce()
        +vips_affine()
        +vips_mapim()
    }

    class VipsColour {
        <<Color Operations>>
        +VipsImage *in
        +VipsImage *out
        +VipsInterpretation space
        +vips_colourspace()
        +vips_Lab2XYZ()
        +vips_XYZ2Lab()
        +vips_sRGB2HSV()
        +vips_icc_import()
        +vips_icc_transform()
    }

    class VipsForeign {
        <<Format I/O>>
        +char *filename
        +VipsImage *out
        +vips_foreign_find_load()
        +vips_foreign_find_save()
        +vips_foreign_load()
        +vips_foreign_save()
    }

    class VipsForeignLoad {
        <<Abstract Loader>>
        +VipsImage *out
        +VipsForeignFlags flags
        +gboolean sequential
        +vips_foreign_load_*()
        #header()
        #load()
        #generate()
    }

    class VipsForeignLoadJpeg {
        +char *filename
        +gboolean shrink
        +gboolean autorotate
        +vips_jpegload()
        +vips_jpegload_buffer()
        -vips_foreign_load_jpeg_file()
        -vips_foreign_load_jpeg_buffer()
    }

    class VipsForeignLoadPng {
        +char *filename
        +vips_pngload()
        +vips_pngload_buffer()
        -read_png_header()
        -read_png_image()
    }

    class VipsForeignLoadWebp {
        +char *filename
        +gint page
        +gint n
        +vips_webpload()
        +vips_webpload_buffer()
        -read_webp()
    }

    class VipsForeignSave {
        <<Abstract Saver>>
        +VipsImage *in
        +gint page_height
        +vips_foreign_save_*()
        #write()
    }

    class VipsForeignSaveJpeg {
        +char *filename
        +gint Q
        +gboolean optimize_coding
        +gboolean strip
        +vips_jpegsave()
        +vips_jpegsave_buffer()
    }

    class VipsBuffer {
        +VipsImage *im
        +VipsRect area
        +gboolean done
        +VipsBufferCache *cache
        +VipsPel *buf
        +size_t bsize
        +vips_buffer_new()
        +vips_buffer_unref()
        +vips_buffer_print()
    }

    class VipsThreadState {
        +VipsImage *im
        +VipsRegion *reg
        +VipsRect pos
        +int x
        +int y
        +vips_threadstate_new()
        +vips_get_tile_size()
    }

    class VipsThreadPool {
        +VipsImage *im
        +VipsStartFn start
        +VipsStopFn stop
        +VipsAllocateFn allocate
        +VipsWorkFn work
        +vips_threadpool_run()
        -vips_threadpool_allocate()
        -vips_threadpool_work()
    }

    GObject <|-- VipsObject
    VipsObject <|-- VipsImage
    VipsObject <|-- VipsRegion
    VipsObject <|-- VipsOperation
    VipsOperation <|-- VipsArithmetic
    VipsOperation <|-- VipsConversion
    VipsOperation <|-- VipsResample
    VipsOperation <|-- VipsColour
    VipsOperation <|-- VipsForeign
    VipsForeign <|-- VipsForeignLoad
    VipsForeign <|-- VipsForeignSave
    VipsForeignLoad <|-- VipsForeignLoadJpeg
    VipsForeignLoad <|-- VipsForeignLoadPng
    VipsForeignLoad <|-- VipsForeignLoadWebp
    VipsForeignSave <|-- VipsForeignSaveJpeg

    VipsImage --> VipsRegion : contains many
    VipsRegion --> VipsImage : references
    VipsBuffer --> VipsImage : references
    VipsThreadState --> VipsRegion : uses
    VipsThreadPool --> VipsThreadState : manages
    VipsOperation --> VipsImage : operates on

    class GObject gobject
    class VipsObject,VipsImage,VipsRegion,VipsBuffer base
    class VipsOperation,VipsArithmetic,VipsConversion,VipsResample,VipsColour operations
    class VipsForeign,VipsForeignLoad,VipsForeignLoadJpeg,VipsForeignLoadPng,VipsForeignLoadWebp,VipsForeignSave,VipsForeignSaveJpeg io
    class VipsThreadState,VipsThreadPool threading

    classDef gobject fill:#f9f,stroke:#333,color:#000
    classDef base fill:#bbf,stroke:#333,color:#000
    classDef operations fill:#bfb,stroke:#333,color:#000
    classDef io fill:#ffb,stroke:#333,color:#000
    classDef threading fill:#fbb,stroke:#333,color:#000
```

---

## Data Structure & Memory Architecture Diagram

```mermaid
graph TB
    subgraph "VipsImage Memory Layout"
        subgraph "Header/Metadata"
            ImageHeader[Image Header<br/>Dimensions: Xsize, Ysize<br/>Bands, BandFormat<br/>Coding, Interpretation<br/>Resolution: Xres, Yres]
            ImageMeta[Metadata<br/>EXIF, ICC Profile<br/>XMP, IPTC<br/>Custom fields]
        end

        subgraph "Image Data Storage"
            DataType{Image Type}

            MemoryImage[VIPS_IMAGE_SETBUF<br/>Fully in memory<br/>data pointer to buffer]

            PartialImage[VIPS_IMAGE_PARTIAL<br/>Demand-driven<br/>Generated on request]

            MMapImage[VIPS_IMAGE_MMAPIN<br/>Memory mapped file<br/>Lazy loading]

            OpenFileImage[VIPS_IMAGE_OPENIN<br/>File descriptor<br/>Sequential access]
        end

        ImageHeader --> DataType
        ImageMeta --> DataType
        DataType -->|Type: SETBUF| MemoryImage
        DataType -->|Type: PARTIAL| PartialImage
        DataType -->|Type: MMAPIN| MMapImage
        DataType -->|Type: OPENIN| OpenFileImage
    end

    subgraph "Region-based Access"
        VipsRegion1[VipsRegion<br/>Region 1<br/>valid: x=0,y=0,w=512,h=512<br/>data: pixel buffer pointer<br/>bpl: bytes per line]

        VipsRegion2[VipsRegion<br/>Region 2<br/>valid: x=512,y=0,w=512,h=512<br/>data: pixel buffer pointer<br/>bpl: bytes per line]

        VipsRegion3[VipsRegion<br/>Region 3<br/>valid: x=0,y=512,w=512,h=512<br/>data: pixel buffer pointer<br/>bpl: bytes per line]

        PartialImage -->|Create region| VipsRegion1
        PartialImage -->|Create region| VipsRegion2
        PartialImage -->|Create region| VipsRegion3
    end

    subgraph "Buffer Management"
        BufferPool[Buffer Pool<br/>Cached memory buffers<br/>Size: configurable]

        Buffer1[Buffer 1<br/>Size: 512x512x3<br/>Status: In use]

        Buffer2[Buffer 2<br/>Size: 512x512x3<br/>Status: Free]

        Buffer3[Buffer 3<br/>Size: 512x512x3<br/>Status: Free]

        BufferPool --> Buffer1
        BufferPool --> Buffer2
        BufferPool --> Buffer3

        VipsRegion1 -.->|Uses| Buffer1
    end

    subgraph "Tile Cache"
        TileCache[Tile Cache<br/>LRU eviction<br/>Max memory limit]

        CachedTile1[Tile x=0,y=0<br/>Computed result<br/>Last access: recent]

        CachedTile2[Tile x=512,y=0<br/>Computed result<br/>Last access: old]

        TileCache --> CachedTile1
        TileCache --> CachedTile2

        VipsRegion2 -.->|Cache hit| CachedTile2
    end

    subgraph "Pixel Format Layout"
        PixelLayout[Pixel Data Layout]

        Interleaved[Interleaved<br/>RGBRGBRGB...<br/>Band-interleaved by pixel]

        Planar[Planar<br/>RRR...GGG...BBB...<br/>Band-sequential]

        PixelLayout --> Interleaved
        PixelLayout --> Planar
    end

    subgraph "Operation Pipeline Graph"
        SourceImage[Source Image<br/>VipsImage*<br/>input.jpg]

        ResizeOp[Resize Operation<br/>VipsOperation*<br/>scale: 0.5]

        CropOp[Crop Operation<br/>VipsOperation*<br/>area: 100,100,800,600]

        BlurOp[Blur Operation<br/>VipsOperation*<br/>sigma: 2.0]

        OutputImage[Output Image<br/>VipsImage*<br/>result]

        SourceImage -->|input| ResizeOp
        ResizeOp -->|output| CropOp
        CropOp -->|output| BlurOp
        BlurOp -->|output| OutputImage
    end

    subgraph "Memory Tracking"
        MemStats[Memory Statistics<br/>vips_tracked_get_mem()<br/>vips_tracked_get_allocs()]

        MemLimit[Memory Limit<br/>VIPS_MAX_MEMORY<br/>Default: 100MB]

        MemStats -.->|Enforce| MemLimit
    end

    style ImageHeader fill:#d4edda,stroke:#28a745
    style ImageMeta fill:#d4edda,stroke:#28a745
    style MemoryImage fill:#cce5ff,stroke:#004085
    style PartialImage fill:#fff3cd,stroke:#856404
    style MMapImage fill:#cce5ff,stroke:#004085
    style OpenFileImage fill:#cce5ff,stroke:#004085
    style VipsRegion1 fill:#f8d7da,stroke:#721c24
    style VipsRegion2 fill:#f8d7da,stroke:#721c24
    style VipsRegion3 fill:#f8d7da,stroke:#721c24
    style BufferPool fill:#d1ecf1,stroke:#0c5460
    style TileCache fill:#e2e3e5,stroke:#383d41
    style SourceImage fill:#c3e6cb,stroke:#155724
    style OutputImage fill:#c3e6cb,stroke:#155724
```

---

## Memory Management Flow Diagram

```mermaid
graph LR
    subgraph "Application Request"
        App[Application<br/>requests pixel region<br/>x=1000, y=1000, w=512, h=512]
    end

    subgraph "Region Preparation"
        PrepareRegion[vips_region_prepare<br/>Calculate required<br/>input regions]

        CheckCache{Region<br/>in cache?}

        CacheHit[Return cached data<br/>Update LRU]

        CacheMiss[Compute region<br/>Allocate buffer]
    end

    subgraph "Computation"
        TraceBack[Trace computation graph<br/>Find source regions needed]

        RequestInput[Request input regions<br/>from parent operations]

        AllocBuffer[Allocate output buffer<br/>from buffer pool]

        Execute[Execute operation<br/>on input data]

        StoreCache[Store result in cache<br/>if cacheable]
    end

    subgraph "Buffer Pool Management"
        CheckPool{Buffer<br/>available?}

        ReuseBuffer[Reuse existing buffer<br/>from pool]

        AllocNew[Allocate new buffer<br/>Track memory]

        CheckLimit{Exceeds<br/>memory limit?}

        EvictOld[Evict old buffers<br/>Free memory]

        OOMError[Out of memory error<br/>Return error]
    end

    subgraph "Result Delivery"
        ReturnData[Return pixel data<br/>to application]
    end

    App --> PrepareRegion
    PrepareRegion --> CheckCache
    CheckCache -->|Hit| CacheHit
    CheckCache -->|Miss| CacheMiss
    CacheHit --> ReturnData
    CacheMiss --> TraceBack
    TraceBack --> RequestInput
    RequestInput --> AllocBuffer
    AllocBuffer --> CheckPool
    CheckPool -->|Available| ReuseBuffer
    CheckPool -->|Not available| AllocNew
    AllocNew --> CheckLimit
    CheckLimit -->|OK| Execute
    CheckLimit -->|Exceeds| EvictOld
    EvictOld --> CheckLimit
    CheckLimit -->|Still exceeds| OOMError
    ReuseBuffer --> Execute
    Execute --> StoreCache
    StoreCache --> ReturnData

    style App fill:#e1f5ff,stroke:#01579b
    style CacheHit fill:#c8e6c9,stroke:#2e7d32
    style Execute fill:#fff9c4,stroke:#f57f17
    style OOMError fill:#ffcdd2,stroke:#c62828
    style ReturnData fill:#c8e6c9,stroke:#2e7d32
```

---

## Pixel Data Storage Schema

```mermaid
erDiagram
    VIPS_IMAGE ||--o{ VIPS_REGION : "provides access via"
    VIPS_IMAGE ||--|| IMAGE_HEADER : contains
    VIPS_IMAGE ||--o| PIXEL_DATA : "may contain"
    VIPS_IMAGE ||--o{ METADATA : has
    VIPS_REGION ||--|| BUFFER : "uses"
    BUFFER ||--|| MEMORY_POOL : "allocated from"

    VIPS_IMAGE {
        int Xsize "Image width"
        int Ysize "Image height"
        int Bands "Number of bands (channels)"
        VipsBandFormat BandFmt "Pixel format (uchar, float, etc)"
        VipsCoding Coding "Encoding (NONE, LABQ, RAD)"
        VipsInterpretation Type "Color space interpretation"
        double Xres "Horizontal resolution"
        double Yres "Vertical resolution"
        VipsImageType dtype "Storage type (SETBUF, PARTIAL, etc)"
        void_ptr data "Pointer to pixel data (if in memory)"
        size_t length "Size of pixel data in bytes"
    }

    IMAGE_HEADER {
        int magic "VIPS magic number"
        int width "Image width in pixels"
        int height "Image height in pixels"
        int bands "Number of bands"
        int format "Band format code"
        int coding "Coding type"
        int type "Interpretation"
        float xres "X resolution"
        float yres "Y resolution"
        int xoffset "X offset"
        int yoffset "Y offset"
    }

    PIXEL_DATA {
        void_ptr pixels "Raw pixel buffer"
        size_t size "Total size in bytes"
        int stride "Bytes per line (row stride)"
        bool is_contiguous "Contiguous in memory"
        allocation_type alloc "malloc, mmap, external"
    }

    METADATA {
        string key "Metadata key (exif-*, icc-profile)"
        GValue value "GLib value (string, int, blob)"
        size_t size "Size of metadata value"
    }

    VIPS_REGION {
        VipsRect valid "Valid area (x, y, width, height)"
        int bpl "Bytes per line in buffer"
        void_ptr data "Pointer to pixel data"
        RegionType type "NONE, BUFFER, OTHER_REGION, etc"
        bool invalid "Region invalidated flag"
    }

    BUFFER {
        VipsRect area "Covered area"
        void_ptr buf "Buffer memory"
        size_t bsize "Buffer size"
        bool done "Computation complete"
        int ref_count "Reference count"
    }

    MEMORY_POOL {
        GHashTable buffers "Hash table of buffers"
        size_t total_allocated "Total allocated memory"
        size_t max_memory "Maximum allowed memory"
        int num_buffers "Number of buffers"
        GMutex lock "Thread safety lock"
    }
```

---

## Supported Image Formats & Codecs

```mermaid
graph TB
    subgraph "Input Formats (Loaders)"
        JPEG[JPEG / JPG<br/>libjpeg-turbo<br/>Fast baseline & progressive]
        PNG[PNG<br/>libpng / libspng<br/>All bit depths, interlaced]
        WebP[WebP<br/>libwebp<br/>Lossy & lossless, animation]
        TIFF[TIFF<br/>libtiff<br/>Multipage, tiled, BigTIFF]
        HEIF[HEIF / HEIC<br/>libheif<br/>HEVC compression]
        AVIF[AVIF<br/>libheif<br/>AV1 compression]
        GIF[GIF<br/>giflib<br/>Animation support]
        SVG[SVG<br/>librsvg<br/>Vector graphics]
        PDF[PDF<br/>poppler<br/>Rasterization]
        RAW[Camera RAW<br/>libraw<br/>CR2, NEF, DNG, etc.]
        FITS[FITS<br/>cfitsio<br/>Scientific images]
        OpenEXR[OpenEXR<br/>IlmImlib<br/>HDR images]
        PPM[PPM/PGM/PBM<br/>Native<br/>Netpbm formats]
        VIPS[VIPS Native<br/>Native<br/>Fast save/load]
    end

    subgraph "Output Formats (Savers)"
        JPEG_OUT[JPEG<br/>Quality, progressive,<br/>optimize coding]
        PNG_OUT[PNG<br/>Compression level,<br/>interlacing, palette]
        WebP_OUT[WebP<br/>Quality, lossless,<br/>near lossless]
        TIFF_OUT[TIFF<br/>Compression (LZW, JPEG),<br/>tiling, pyramids]
        HEIF_OUT[HEIF<br/>Quality, compression]
        AVIF_OUT[AVIF<br/>Quality, speed,<br/>lossless]
        GIF_OUT[GIF<br/>Dithering,<br/>animation]
        JXL[JPEG XL<br/>libjxl<br/>Next-gen format]
        VIPS_OUT[VIPS Native<br/>Zero-copy<br/>intermediate format]
    end

    subgraph "libvips Format System"
        FormatDetect[Format Detection<br/>Magic bytes, extension]
        LoaderRegistry[Loader Registry<br/>vips_foreign_find_load]
        SaverRegistry[Saver Registry<br/>vips_foreign_find_save]
    end

    JPEG --> LoaderRegistry
    PNG --> LoaderRegistry
    WebP --> LoaderRegistry
    TIFF --> LoaderRegistry
    HEIF --> LoaderRegistry
    AVIF --> LoaderRegistry
    GIF --> LoaderRegistry
    SVG --> LoaderRegistry
    PDF --> LoaderRegistry
    RAW --> LoaderRegistry
    FITS --> LoaderRegistry
    OpenEXR --> LoaderRegistry
    PPM --> LoaderRegistry
    VIPS --> LoaderRegistry

    LoaderRegistry --> FormatDetect
    FormatDetect --> SaverRegistry

    SaverRegistry --> JPEG_OUT
    SaverRegistry --> PNG_OUT
    SaverRegistry --> WebP_OUT
    SaverRegistry --> TIFF_OUT
    SaverRegistry --> HEIF_OUT
    SaverRegistry --> AVIF_OUT
    SaverRegistry --> GIF_OUT
    SaverRegistry --> JXL
    SaverRegistry --> VIPS_OUT

    style JPEG fill:#d4edda,stroke:#28a745
    style PNG fill:#d4edda,stroke:#28a745
    style WebP fill:#d4edda,stroke:#28a745
    style AVIF fill:#d1ecf1,stroke:#17a2b8
    style HEIF fill:#d1ecf1,stroke:#17a2b8
    style FormatDetect fill:#fff3cd,stroke:#ffc107
```

---

## Threading Architecture

```mermaid
graph TB
    subgraph "Main Thread"
        MainThread[Main Thread<br/>Application entry<br/>API calls]
        BuildGraph[Build Operation Graph<br/>Lazy evaluation setup]
    end

    subgraph "Thread Pool"
        ThreadPool[vips_threadpool_run<br/>Concurrency level:<br/>VIPS_CONCURRENCY]

        Worker1[Worker Thread 1<br/>Process tiles]
        Worker2[Worker Thread 2<br/>Process tiles]
        Worker3[Worker Thread 3<br/>Process tiles]
        WorkerN[Worker Thread N<br/>Process tiles]
    end

    subgraph "Work Distribution"
        TileQueue[Tile Work Queue<br/>Pending regions<br/>to compute]

        Allocator[Work Allocator<br/>vips_get_tile_size<br/>Determines tile boundaries]

        Tile1[Tile 1<br/>x=0, y=0<br/>128x128]
        Tile2[Tile 2<br/>x=128, y=0<br/>128x128]
        Tile3[Tile 3<br/>x=0, y=128<br/>128x128]
        TileM[Tile M<br/>x=..., y=...<br/>128x128]
    end

    subgraph "Synchronization"
        Mutex[GMutex Locks<br/>Protect shared state]
        Semaphore[Semaphores<br/>Limit concurrency]
        Atomic[Atomic Operations<br/>Reference counting]
    end

    subgraph "Per-Thread Resources"
        ThreadState1[Thread State 1<br/>VipsThreadState<br/>Private region buffer]
        ThreadState2[Thread State 2<br/>VipsThreadState<br/>Private region buffer]
        ThreadState3[Thread State 3<br/>VipsThreadState<br/>Private region buffer]
    end

    MainThread --> BuildGraph
    BuildGraph --> ThreadPool
    ThreadPool --> Worker1
    ThreadPool --> Worker2
    ThreadPool --> Worker3
    ThreadPool --> WorkerN

    ThreadPool --> Allocator
    Allocator --> Tile1
    Allocator --> Tile2
    Allocator --> Tile3
    Allocator --> TileM

    Tile1 --> TileQueue
    Tile2 --> TileQueue
    Tile3 --> TileQueue
    TileM --> TileQueue

    TileQueue --> Worker1
    TileQueue --> Worker2
    TileQueue --> Worker3
    TileQueue --> WorkerN

    Worker1 --> ThreadState1
    Worker2 --> ThreadState2
    Worker3 --> ThreadState3

    Worker1 -.->|Sync| Mutex
    Worker2 -.->|Sync| Mutex
    Worker3 -.->|Sync| Mutex

    ThreadPool -.->|Control| Semaphore
    ThreadState1 -.->|Update| Atomic
    ThreadState2 -.->|Update| Atomic
    ThreadState3 -.->|Update| Atomic

    style MainThread fill:#e1f5ff,stroke:#01579b
    style Worker1 fill:#c8e6c9,stroke:#2e7d32
    style Worker2 fill:#c8e6c9,stroke:#2e7d32
    style Worker3 fill:#c8e6c9,stroke:#2e7d32
    style WorkerN fill:#c8e6c9,stroke:#2e7d32
    style TileQueue fill:#fff9c4,stroke:#f57f17
    style Mutex fill:#ffcdd2,stroke:#c62828
```

---

## Performance Characteristics

| Aspect | Implementation | Benefit |
|--------|----------------|---------|
| **Memory Efficiency** | Streaming, region-based processing | Process images larger than RAM |
| **CPU Utilization** | Thread pool, tile-level parallelism | Multi-core speedup |
| **Cache Optimization** | Tile cache, buffer pool | Reduce redundant computation |
| **I/O Optimization** | Memory mapping, sequential access | Minimize disk I/O overhead |
| **Pipeline Fusion** | Operation graph optimization | Combine operations, reduce passes |
| **SIMD** | ORC, hand-coded SIMD (ARM NEON, x86 SSE) | Vectorized operations |
| **Format Libraries** | Delegates to specialized codecs | Optimal encoding/decoding |
| **Lazy Evaluation** | Demand-driven computation | Only compute what's needed |

---

## Key Design Patterns

### 1. **Demand-Driven Evaluation (Pull Model)**
- Operations build a computation graph without executing
- Evaluation happens when output is requested
- Only required regions are computed

### 2. **Region-Based Processing**
- Images divided into small tiles/regions
- Each region processed independently
- Enables parallel processing and memory efficiency

### 3. **Object-Oriented C (via GObject)**
- Type system, inheritance, polymorphism
- Reference counting for memory management
- Property system for configuration

### 4. **Pipeline Optimization**
- Graph optimization combines operations
- Eliminates redundant computation
- Reduces memory bandwidth

### 5. **Thread Pool Pattern**
- Fixed number of worker threads
- Work queue for tile processing
- Per-thread state for safety

### 6. **Buffer Pool**
- Reuse memory buffers
- Reduce allocation overhead
- Configurable memory limits

---

## Configuration Options

| Environment Variable | Purpose | Default |
|---------------------|---------|---------|
| `VIPS_CONCURRENCY` | Number of worker threads | Number of CPU cores |
| `VIPS_MAX_MEMORY` | Maximum memory for cache | 100 MB |
| `VIPS_DISC_THRESHOLD` | Threshold to use disk temp files | 100 MB |
| `VIPS_NOVECTOR` | Disable SIMD vectorization | Enabled |
| `VIPS_STALL` | Stall on first warning/error | Disabled |
| `VIPS_PROGRESS` | Enable progress reporting | Disabled |

---

## Integration Points

### C API
```c
VipsImage *image = vips_image_new_from_file("input.jpg", NULL);
vips_resize(image, &output, 0.5, NULL);
vips_image_write_to_file(output, "output.jpg", NULL);
```

### Go (via CGo)
```go
C.vips_resize(image.VipsImage, &out, C.double(scale), nil)
```

### Python (pyvips)
```python
image = pyvips.Image.new_from_file('input.jpg')
image = image.resize(0.5)
image.write_to_file('output.jpg')
```

### Node.js (sharp)
```javascript
sharp('input.jpg')
  .resize(800, 600)
  .toFile('output.jpg');
```

---

This documentation provides a comprehensive architectural view of libvips, showing its internal structure, memory management, threading model, and data flow patterns. The library's design prioritizes memory efficiency, performance, and scalability for high-volume image processing workloads.
