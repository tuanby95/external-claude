# imgproxy Architecture Documentation

## Overview

imgproxy is a fast and secure standalone server for resizing, processing, and converting images on-the-fly. Built in Go and leveraging libvips, it provides a stateless, high-performance image processing service designed for cloud-native deployments.

---

## C4 Model Diagrams

### Level 1: System Context Diagram

```mermaid
graph TB
    User[End User<br/>Website or app visitor<br/>requesting images]
    Developer[Developer<br/>Configures and deploys<br/>imgproxy]

    imgproxy[imgproxy<br/>Fast image processing server<br/>that resizes, converts, and<br/>optimizes images on-the-fly]

    CDN[CDN<br/>Caches processed images<br/>Cloudflare, Fastly]
    WebApp[Web Application<br/>Requests processed images<br/>via signed URLs]
    Storage[Storage Services<br/>S3, GCS, Azure Blob,<br/>Swift, HTTP servers]
    Monitoring[Monitoring Services<br/>Prometheus, DataDog,<br/>New Relic, OpenTelemetry]
    ErrorTracking[Error Tracking<br/>Sentry, Bugsnag,<br/>Honeybadder, Airbrake]

    User -->|Requests image<br/>HTTPS| CDN
    CDN -->|Fetches processed image<br/>HTTP/HTTPS| imgproxy
    WebApp -->|Generates signed URLs<br/>HMAC-SHA256| imgproxy
    imgproxy -->|Fetches source images<br/>HTTP/S3/GCS/Azure/Swift| Storage
    imgproxy -->|Sends metrics<br/>Various protocols| Monitoring
    imgproxy -->|Reports errors<br/>HTTP API| ErrorTracking
    Developer -->|Configures via<br/>environment variables| imgproxy

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef external fill:#999,stroke:#666,color:#fff

    class User,Developer person
    class imgproxy system
    class CDN,WebApp,Storage,Monitoring,ErrorTracking external
```

### Level 2: Container Diagram

```mermaid
graph TB
    Client[Client<br/>Browser, mobile app,<br/>or service]

    subgraph imgproxy[imgproxy System]
        Server[HTTP Server<br/>Go net/http<br/>Handles HTTP/2 requests<br/>with TLS, timeouts]
        Router[Router<br/>Go<br/>Routes requests,<br/>handles middleware chain]
        Processor[Image Processor<br/>Go + libvips<br/>Core image processing<br/>pipeline with worker pools]
        Security[Security Layer<br/>Go<br/>URL signature verification,<br/>source validation, auth]
        Transport[Transport Layer<br/>Go<br/>Multi-backend image fetching<br/>S3, GCS, Azure, HTTP]
        Config[Configuration<br/>Environment Variables<br/>200+ configuration options<br/>loaded at startup]
        Metrics[Metrics Collector<br/>Go<br/>Collects and exports<br/>metrics to monitoring]
    end

    Storage[Storage Backends<br/>S3, GCS, Azure,<br/>Swift, HTTP]
    Monitoring[Monitoring<br/>Prometheus, DataDog, etc.]
    VIPS[libvips<br/>C Library<br/>Fast image<br/>processing library]

    Client -->|GET /signature/options/<br/>image.jpg HTTPS| Server
    Server -->|Routes request| Router
    Router -->|Validates signature<br/>& source| Security
    Router -->|Processes image| Processor
    Processor -->|Fetches source image| Transport
    Transport -->|Downloads image<br/>Protocol specific| Storage
    Processor -->|Processes via CGo<br/>Function calls| VIPS
    Processor -->|Records metrics| Metrics
    Metrics -->|Exports metrics| Monitoring
    Config -->|Configures| Server
    Config -->|Configures| Processor
    Config -->|Configures| Security

    classDef client fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef external fill:#999,stroke:#666,color:#fff
    classDef database fill:#6c8ebf,stroke:#4a6078,color:#fff

    class Client client
    class Server,Router,Processor,Security,Transport,Config,Metrics container
    class Storage,Monitoring external
    class VIPS database
```

### Level 3: Component Diagram - Image Processing Pipeline

```mermaid
graph TB
    subgraph Processor[Image Processor Container]
        Handler[Processing Handler<br/>Go<br/>Orchestrates request processing,<br/>manages worker pool<br/>and semaphores]
        Options[Options Parser<br/>Go<br/>Parses URL paths,<br/>decodes base64/plain URLs,<br/>extracts processing options]
        ImageData[Image Data Manager<br/>Go<br/>Downloads, validates, and<br/>manages image data lifecycle]
        Pipeline[Processing Pipeline<br/>Go<br/>15-step sequential<br/>image transformation pipeline]
        VIPSWrapper[VIPS Wrapper<br/>Go/CGo<br/>Wraps libvips C API<br/>with Go-friendly interface]

        Prepare[Prepare<br/>Go<br/>Calculate dimensions,<br/>load metadata]
        Crop[Crop<br/>Go<br/>Apply cropping<br/>with gravity]
        Scale[Scale<br/>Go<br/>Resize image with<br/>various algorithms]
        Rotate[Rotate & Flip<br/>Go<br/>Apply transformations]
        Filters[Filters<br/>Go<br/>Blur, sharpen, pixelate]
        Watermark[Watermark<br/>Go<br/>Apply watermark overlay]
        Finalize[Finalize<br/>Go<br/>Convert colorspace,<br/>strip metadata]
    end

    Signature[Signature Validator<br/>Verifies HMAC-SHA256<br/>signatures]
    Transport[Transport Layer<br/>Multi-backend fetching]
    LibVIPS[libvips<br/>C library for<br/>image processing]

    Handler -->|Parses processing options| Options
    Handler -->|Validates signature| Signature
    Handler -->|Fetches source image| ImageData
    ImageData -->|Downloads via backend| Transport
    Handler -->|Executes pipeline| Pipeline
    Pipeline -->|Step 1-2| Prepare
    Prepare -->|Step 5| Crop
    Crop -->|Step 6| Scale
    Scale -->|Step 7| Rotate
    Rotate -->|Step 9| Filters
    Filters -->|Step 15| Watermark
    Watermark -->|Final steps| Finalize
    Pipeline -->|All steps use| VIPSWrapper
    VIPSWrapper -->|CGo calls| LibVIPS

    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef external fill:#999,stroke:#666,color:#fff
    classDef database fill:#6c8ebf,stroke:#4a6078,color:#fff

    class Handler,Options,ImageData,Pipeline,VIPSWrapper,Prepare,Crop,Scale,Rotate,Filters,Watermark,Finalize component
    class Signature,Transport external
    class LibVIPS database
```

---

## Class Diagram

### Core Domain Model

```mermaid
classDiagram
    class ProcessingHandler {
        +workerSem chan struct
        +queueSem chan struct
        +handleProcessing(http.ResponseWriter, *http.Request)
        -processRequest(http.ResponseWriter, *http.Request)
        -checkSecret(*http.Request) error
        -streamRaw(*imagedata.ImageData, http.ResponseWriter, *http.Request)
    }

    class ProcessingOptions {
        +ResizingType string
        +Width int
        +Height int
        +MinWidth int
        +MinHeight int
        +Dpr float64
        +Gravity Gravity
        +Crop Crop
        +Padding Padding
        +Trim Trim
        +Rotate int
        +Quality int
        +MaxBytes int
        +Format imagetype.Type
        +Filters []Filter
        +Watermark Watermark
        +Parse(string, string) error
        +String() string
    }

    class ImageData {
        +Type imagetype.Type
        +Data []byte
        +Headers map[string]string
        +Cancel context.CancelFunc
        +Close()
        +SetData([]byte)
        +SetHeaders(map[string]string)
    }

    class Pipeline {
        +Run(*ProcessingOptions, *ImageData) (*Image, error)
        -prepare()
        -scaleOnLoad()
        -crop()
        -scale()
        -rotateAndFlip()
        -applyFilters()
        -watermark()
        -finalize()
    }

    class Image {
        +Img *C.VipsImage
        +Type imagetype.Type
        +GifLoop int
        +GifDelay []int
        +Width() int
        +Height() int
        +Bands() int
        +HasAlpha() bool
        +Load() error
        +Save() ([]byte, error)
        +Clear()
    }

    class VipsWrapper {
        +vipsLoad([]byte, int) (*C.VipsImage, error)
        +vipsSave(*C.VipsImage, imagetype.Type) ([]byte, error)
        +vipsResize(*C.VipsImage, float64) error
        +vipsCrop(*C.VipsImage, int, int, int, int) error
        +vipsRotate(*C.VipsImage, int) error
        +vipsBlur(*C.VipsImage, float64) error
        +vipsSharpen(*C.VipsImage, float64) error
        +vipsWatermark(*C.VipsImage, *C.VipsImage, int, int) error
    }

    class Transport {
        <<interface>>
        +Download(*http.Request) (*ImageData, error)
    }

    class S3Transport {
        +client *s3.Client
        +encryption bool
        +Download(*http.Request) (*ImageData, error)
        -constructURL(string) string
    }

    class HTTPTransport {
        +client *http.Client
        +Download(*http.Request) (*ImageData, error)
        -followRedirects(*http.Request) (*http.Response, error)
    }

    class GCSTransport {
        +client *storage.Client
        +Download(*http.Request) (*ImageData, error)
    }

    class Signature {
        +Keys [][]byte
        +Salts [][]byte
        +Size int
        +Trusted []string
        +Verify(string, string, string) error
        -sign(string) string
        -compare(string, string) bool
    }

    class Security {
        +AllowedSources []*regexp.Regexp
        +BlockLoopback bool
        +BlockLinkLocal bool
        +BlockPrivate bool
        +VerifySourceURL(string) error
        -checkNetwork(net.IP) error
    }

    class Config {
        +Bind string
        +ReadTimeout int
        +WriteTimeout int
        +MaxClients int
        +Workers int
        +Quality int
        +GZipCompression int
        +EnableWebpDetection bool
        +EnableAvifDetection bool
        +Load()
        +Validate()
    }

    class Router {
        +matcher *urlmatcher
        +ServeHTTP(http.ResponseWriter, *http.Request)
        -extractPath(*http.Request) (string, string, string)
        -withMetrics(http.HandlerFunc) http.HandlerFunc
        -withPanicHandler(http.HandlerFunc) http.HandlerFunc
        -withCORS(http.HandlerFunc) http.HandlerFunc
    }

    class MetricsCollector {
        +RequestsTotal counter
        +RequestsDuration histogram
        +ImageSize histogram
        +ErrorsTotal counter
        +Collect()
        +Export()
    }

    ProcessingHandler --> ProcessingOptions : parses
    ProcessingHandler --> ImageData : fetches
    ProcessingHandler --> Pipeline : executes
    ProcessingHandler --> Signature : validates
    Pipeline --> Image : processes
    Pipeline --> VipsWrapper : uses
    Image --> VipsWrapper : uses
    ImageData --> Transport : downloads via
    Transport <|-- S3Transport : implements
    Transport <|-- HTTPTransport : implements
    Transport <|-- GCSTransport : implements
    ProcessingHandler --> Security : validates source
    Router --> ProcessingHandler : routes to
    Router --> MetricsCollector : records metrics
    ProcessingOptions --> Config : configured by
    Security --> Config : configured by
    Signature --> Config : configured by
```

---

## Sequence Diagrams

### Sequence 1: Image Processing Request Flow

```mermaid
sequenceDiagram
    actor Client
    participant CDN
    participant Router
    participant Security
    participant Handler
    participant Options
    participant ImageData
    participant Transport
    participant Storage
    participant Pipeline
    participant VIPS
    participant Metrics

    Client->>CDN: GET /signature/resize:800:600/s3://bucket/image.jpg
    CDN->>Router: Forward request (cache miss)

    Router->>Router: Generate request ID
    Router->>Router: Extract client IP
    Router->>Security: Verify bearer token (if configured)
    Security-->>Router: Authorized

    Router->>Handler: handleProcessing()
    Handler->>Options: Parse URL path
    Options->>Options: Decode base64/plain URL
    Options->>Options: Extract processing options
    Options-->>Handler: ProcessingOptions

    Handler->>Security: Verify signature
    Security->>Security: HMAC-SHA256 validation
    Security-->>Handler: Valid signature

    Handler->>Security: Verify source URL
    Security->>Security: Check allowed sources regex
    Security->>Security: Check network restrictions
    Security-->>Handler: Source allowed

    Handler->>Handler: Acquire semaphore (worker pool)
    Handler->>ImageData: Download(sourceURL)
    ImageData->>Transport: Select backend (S3)
    Transport->>Storage: GET s3://bucket/image.jpg
    Storage-->>Transport: Image bytes + headers
    Transport-->>ImageData: ImageData
    ImageData->>ImageData: Validate image type
    ImageData->>ImageData: Check dimensions
    ImageData-->>Handler: Validated ImageData

    Handler->>Pipeline: Run(options, imageData)
    Pipeline->>VIPS: vipsLoad(imageData)
    VIPS-->>Pipeline: VipsImage

    Pipeline->>Pipeline: Step 1: trim
    Pipeline->>Pipeline: Step 2: prepare (calculate dimensions)
    Pipeline->>VIPS: vipsGetWidth(), vipsGetHeight()

    Pipeline->>Pipeline: Step 3: scaleOnLoad
    Pipeline->>Pipeline: Step 5: crop
    Pipeline->>VIPS: vipsCrop(x, y, w, h)

    Pipeline->>Pipeline: Step 6: scale
    Pipeline->>VIPS: vipsResize(scale)

    Pipeline->>Pipeline: Step 7: rotateAndFlip
    Pipeline->>VIPS: vipsRotate(angle)

    Pipeline->>Pipeline: Step 9: applyFilters
    Pipeline->>VIPS: vipsBlur() / vipsSharpen()

    Pipeline->>Pipeline: Step 15: watermark
    Pipeline->>VIPS: vipsWatermark(overlay)

    Pipeline->>Pipeline: Finalize: colorspaceToResult
    Pipeline->>Pipeline: Finalize: stripMetadata

    Pipeline->>VIPS: vipsSave(format)
    VIPS-->>Pipeline: Processed image bytes
    Pipeline-->>Handler: Processed image

    Handler->>Metrics: Record processing time, size, status
    Handler->>Handler: Release semaphore

    Handler->>Router: Write response
    Router->>Router: Set Cache-Control headers
    Router->>Router: Set Content-Type
    Router-->>CDN: 200 OK + image bytes
    CDN->>CDN: Cache image
    CDN-->>Client: Processed image
```

### Sequence 2: Signature Validation Flow

```mermaid
sequenceDiagram
    actor Developer
    participant WebApp
    participant Client
    participant Router
    participant Signature
    participant Config

    Developer->>Config: Set IMGPROXY_KEY & IMGPROXY_SALT
    Config->>Config: Store as []byte arrays

    WebApp->>WebApp: Generate processing options
    Note over WebApp: resize:800:600/plain/image.jpg

    WebApp->>WebApp: Calculate HMAC-SHA256
    Note over WebApp: HMAC(key, salt + path)

    WebApp->>WebApp: Encode signature (base64 URL-safe)
    WebApp->>WebApp: Construct URL
    Note over WebApp: /{signature}/{path}

    WebApp->>Client: Return signed URL
    Client->>Router: GET /{signature}/resize:800:600/...

    Router->>Signature: Verify(signature, path)
    Signature->>Signature: Extract signature from URL
    Signature->>Signature: Get path to sign

    loop For each key/salt pair
        Signature->>Signature: Calculate HMAC-SHA256
        Note over Signature: hash = HMAC(key, salt + path)
        Signature->>Signature: Truncate to configured size
        Signature->>Signature: Base64 URL-safe encode
        Signature->>Signature: Constant-time compare
        alt Match found
            Signature-->>Router: Valid signature
        end
    end

    alt No match & trusted signatures exist
        Signature->>Signature: Check trusted signature list
        alt In trusted list
            Signature-->>Router: Valid (trusted)
        else Not trusted
            Signature-->>Router: Invalid signature
            Router-->>Client: 403 Forbidden
        end
    else No match
        Signature-->>Router: Invalid signature
        Router-->>Client: 403 Forbidden
    end
```

### Sequence 3: Multi-Backend Image Fetching

```mermaid
sequenceDiagram
    participant Handler
    participant ImageData
    participant Transport
    participant S3
    participant GCS
    participant Azure
    participant HTTP

    Handler->>ImageData: Download("s3://bucket/image.jpg")
    ImageData->>Transport: SelectTransport("s3://")
    Transport-->>ImageData: S3Transport

    ImageData->>S3: Download(url)
    S3->>S3: Parse bucket and key
    S3->>S3: Check encryption settings

    alt Encryption enabled
        S3->>S3: Decrypt with KMS key
    end

    S3->>S3: Call AWS SDK GetObject()
    S3-->>ImageData: bytes + headers (ETag, Last-Modified)

    ImageData->>ImageData: Validate image type via magic bytes
    ImageData->>ImageData: Check Content-Type header

    alt Invalid image type
        ImageData->>ImageData: Check fallback image configured
        alt Fallback enabled
            ImageData->>Transport: Download(fallback URL)
            Transport-->>ImageData: Fallback image
        else No fallback
            ImageData-->>Handler: Error: Invalid image
        end
    end

    ImageData->>ImageData: Check max file size
    alt Exceeds max size
        ImageData-->>Handler: Error: Image too large
    end

    ImageData->>ImageData: Pre-read dimensions (if JPEG/PNG)
    ImageData->>ImageData: Check max resolution
    alt Exceeds max resolution
        ImageData-->>Handler: Error: Resolution too high
    end

    ImageData-->>Handler: Valid ImageData

    Note over Handler,HTTP: Alternative backends work similarly:
    Note over GCS: gs://bucket/object
    Note over Azure: abs://container/blob
    Note over HTTP: http(s)://domain/path
```

### Sequence 4: Animation Processing Flow

```mermaid
sequenceDiagram
    participant Handler
    participant Pipeline
    participant VIPS
    participant AnimationSupport

    Handler->>Pipeline: Run(options, gifImageData)
    Pipeline->>VIPS: vipsLoad(gifData)
    VIPS->>VIPS: Detect GIF animation
    VIPS-->>Pipeline: VipsImage (frames stacked vertically)

    Pipeline->>AnimationSupport: Check animation support
    AnimationSupport->>AnimationSupport: Check format (GIF/WebP/AVIF)
    AnimationSupport->>AnimationSupport: Count frames
    AnimationSupport->>AnimationSupport: Check max frames limit

    alt Exceeds max frames
        AnimationSupport-->>Pipeline: Error: Too many frames
        Pipeline-->>Handler: Error
    end

    AnimationSupport->>AnimationSupport: Extract frame delays
    AnimationSupport->>AnimationSupport: Extract loop count
    AnimationSupport-->>Pipeline: Animation metadata

    Pipeline->>Pipeline: Calculate frame height
    Note over Pipeline: frameHeight = totalHeight / frameCount

    Pipeline->>VIPS: Process each frame through pipeline

    loop For each frame
        VIPS->>VIPS: Extract frame (crop vertically)
        VIPS->>VIPS: Apply resize
        VIPS->>VIPS: Apply crop
        VIPS->>VIPS: Apply rotation
        VIPS->>VIPS: Apply filters
        VIPS->>VIPS: Apply watermark
        VIPS->>VIPS: Stack processed frame
    end

    Pipeline->>VIPS: vipsArrayjoin(processedFrames, vertical)
    VIPS-->>Pipeline: Stacked frames

    Pipeline->>Pipeline: Restore animation metadata
    Pipeline->>Pipeline: Set delay array
    Pipeline->>Pipeline: Set loop count

    Pipeline->>VIPS: vipsSave(GIF/WebP/AVIF)
    VIPS->>VIPS: Encode animation with metadata
    VIPS-->>Pipeline: Animated image bytes

    Pipeline-->>Handler: Processed animation
```

---

## Storage & Data Flow Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        Browser[Web Browser]
        MobileApp[Mobile App]
        API[API Client]
    end

    subgraph "CDN Layer (Optional)"
        CDN[CDN / Cache<br/>Cloudflare, Fastly, etc.]
    end

    subgraph "imgproxy Server (Stateless)"
        LB[Load Balancer<br/>SO_REUSEPORT]

        subgraph "HTTP Server"
            HTTPServer[net/http Server<br/>HTTP/2 Support]
            Router[Router<br/>Request ID, IP Detection]
        end

        subgraph "Processing Layer"
            Queue[Request Queue<br/>Semaphore]
            WorkerPool[Worker Pool<br/>2×CPU cores]
            ProcessingPipeline[Image Processing<br/>15-step Pipeline]
        end

        subgraph "Transport Layer"
            TransportManager[Transport Manager]
            S3Client[S3 Transport]
            GCSClient[GCS Transport]
            AzureClient[Azure Transport]
            SwiftClient[Swift Transport]
            HTTPClient[HTTP Transport]
            FSClient[Filesystem Transport]
        end

        subgraph "VIPS Integration"
            VIPSWrapper[VIPS Wrapper<br/>CGo Interface]
            LibVIPS[libvips C Library]
        end

        subgraph "Security & Config"
            Config[Configuration<br/>Environment Variables]
            Signature[Signature Validator<br/>HMAC-SHA256]
            SourceValidator[Source URL Validator<br/>SSRF Protection]
        end

        subgraph "Observability"
            Metrics[Metrics Collector]
            ErrorReporting[Error Reporter]
        end
    end

    subgraph "Storage Backends (Read-Only)"
        S3[AWS S3<br/>Image Storage]
        GCS[Google Cloud Storage]
        Azure[Azure Blob Storage]
        Swift[OpenStack Swift]
        WebServer[Web Server<br/>HTTP/HTTPS]
        LocalFS[Local Filesystem<br/>Restricted]
    end

    subgraph "External Services"
        Prometheus[Prometheus]
        DataDog[DataDog]
        NewRelic[New Relic]
        OTEL[OpenTelemetry]
        Sentry[Sentry]
        Bugsnag[Bugsnag]
    end

    Browser --> CDN
    MobileApp --> CDN
    API --> CDN
    CDN -->|Cache Miss| LB
    LB --> HTTPServer
    HTTPServer --> Router

    Router --> Signature
    Router --> Queue
    Queue --> WorkerPool
    WorkerPool --> ProcessingPipeline

    ProcessingPipeline --> TransportManager
    TransportManager --> S3Client
    TransportManager --> GCSClient
    TransportManager --> AzureClient
    TransportManager --> SwiftClient
    TransportManager --> HTTPClient
    TransportManager --> FSClient

    S3Client -.->|Download Image| S3
    GCSClient -.->|Download Image| GCS
    AzureClient -.->|Download Image| Azure
    SwiftClient -.->|Download Image| Swift
    HTTPClient -.->|Download Image| WebServer
    FSClient -.->|Read Image| LocalFS

    ProcessingPipeline --> VIPSWrapper
    VIPSWrapper --> LibVIPS

    Config -.->|Configure| HTTPServer
    Config -.->|Configure| ProcessingPipeline
    Config -.->|Configure| Signature
    Config -.->|Configure| SourceValidator

    Router --> SourceValidator

    ProcessingPipeline --> Metrics
    Router --> Metrics
    ProcessingPipeline --> ErrorReporting

    Metrics -.->|Export| Prometheus
    Metrics -.->|Export| DataDog
    Metrics -.->|Export| NewRelic
    Metrics -.->|Export| OTEL

    ErrorReporting -.->|Report| Sentry
    ErrorReporting -.->|Report| Bugsnag

    style Browser fill:#e1f5ff
    style MobileApp fill:#e1f5ff
    style API fill:#e1f5ff
    style CDN fill:#fff4e1
    style LibVIPS fill:#ffe1e1
    style S3 fill:#e8f5e9
    style GCS fill:#e8f5e9
    style Azure fill:#e8f5e9
    style Swift fill:#e8f5e9
    style WebServer fill:#e8f5e9
    style LocalFS fill:#e8f5e9
```

---

## Component Interaction Architecture

```mermaid
graph LR
    subgraph "Request Flow"
        A[HTTP Request] --> B[Router]
        B --> C{Auth Check}
        C -->|Valid| D[Signature Verify]
        C -->|Invalid| E[403 Error]
        D -->|Valid| F[Options Parser]
        D -->|Invalid| E
        F --> G[Source Validator]
        G -->|Allowed| H[Queue]
        G -->|Blocked| E
    end

    subgraph "Processing Flow"
        H --> I[Worker Pool]
        I --> J[Image Download]
        J --> K{Validation}
        K -->|Valid| L[Pipeline Init]
        K -->|Invalid| M{Fallback?}
        M -->|Yes| J
        M -->|No| E

        L --> N[Step 1: Trim]
        N --> O[Step 2: Prepare]
        O --> P[Step 3-4: Scale on Load]
        P --> Q[Step 5: Crop]
        Q --> R[Step 6: Resize]
        R --> S[Step 7: Rotate/Flip]
        S --> T[Step 8-9: Filters]
        T --> U[Step 10-14: Extend/Pad]
        U --> V[Step 15: Watermark]
        V --> W[Finalize]
    end

    subgraph "Output Flow"
        W --> X[Encode to Format]
        X --> Y[Apply Headers]
        Y --> Z[Send Response]
    end

    subgraph "libvips Operations"
        N -.-> VIPS[libvips]
        O -.-> VIPS
        P -.-> VIPS
        Q -.-> VIPS
        R -.-> VIPS
        S -.-> VIPS
        T -.-> VIPS
        U -.-> VIPS
        V -.-> VIPS
        W -.-> VIPS
        X -.-> VIPS
    end

    style A fill:#e1f5ff
    style E fill:#ffe1e1
    style Z fill:#e8f5e9
    style VIPS fill:#fff4e1
```

---

## Key Architectural Patterns

### 1. **Stateless Design**
- No database or persistent storage
- Each request is independent
- Configuration via environment variables only
- Enables horizontal scaling without coordination

### 2. **Worker Pool Pattern**
- Semaphore-based concurrency control
- Configurable workers (default: 2×CPU cores)
- Optional request queue to prevent overload
- Per-request goroutine isolation

### 3. **Pipeline Pattern**
- 15-step sequential processing pipeline
- Each step is independent and testable
- Steps can be skipped based on options
- Animation support via frame iteration

### 4. **Transport Abstraction**
- Unified interface for multiple storage backends
- Protocol detection via URL scheme
- Pluggable backend implementations
- Consistent error handling

### 5. **Security-First**
- HMAC-SHA256 signature verification
- SSRF protection (network restriction)
- Image bomb prevention (size/dimension limits)
- Constant-time comparison for auth

### 6. **Observability**
- Multi-platform metrics export
- Request tracing with unique IDs
- Error reporting integrations
- Performance monitoring

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| **Language** | Go 1.21+ |
| **Image Processing** | libvips 8.13+ (C library) |
| **HTTP Server** | Go net/http with HTTP/2 |
| **Concurrency** | Goroutines + Semaphores |
| **Configuration** | Environment Variables |
| **Storage** | S3, GCS, Azure Blob, Swift, HTTP, Filesystem |
| **Metrics** | Prometheus, DataDog, New Relic, OpenTelemetry, CloudWatch |
| **Error Tracking** | Sentry, Bugsnag, Honeybadger, Airbrake |
| **Deployment** | Docker, Kubernetes, Serverless (AWS Lambda) |

---

## Performance Characteristics

- **Memory Efficiency**: libvips streaming processing
- **CPU Optimization**: GOMAXPROCS tuning, worker pools
- **Network**: HTTP/2, connection pooling, redirect following
- **Caching**: Client-side via Cache-Control, ETag support
- **Scalability**: Horizontal scaling, stateless design
- **Throughput**: High concurrency via goroutines and semaphores

---

## Security Features

1. **URL Signature**: HMAC-SHA256 with key rotation support
2. **Source Validation**: Regex-based URL whitelist
3. **SSRF Protection**: Network restriction (loopback, link-local, private)
4. **Size Limits**: Max file size, max resolution, max animation frames
5. **Auth**: Bearer token support
6. **SVG Sanitization**: XSS protection for SVG images
7. **Image Bombs**: Early validation prevents decompression attacks

---

## Deployment Patterns

### 1. **Containerized (Docker)**
```
docker run -p 8080:8080 -e IMGPROXY_KEY=... darthsim/imgproxy
```

### 2. **Kubernetes**
- StatefulSet not required (stateless)
- HorizontalPodAutoscaler for scaling
- Service mesh compatible

### 3. **Serverless (AWS Lambda)**
- Supports Lambda deployment
- Request ID from Lambda context
- Cold start optimization

### 4. **Behind CDN**
- Cloudflare, Fastly, CloudFront
- Cache processed images
- Reduce origin load

---

This documentation provides a comprehensive architectural overview of imgproxy using C4 model diagrams, class diagrams, sequence diagrams, and data flow visualizations. The system is designed for high performance, security, and scalability in cloud-native environments.
