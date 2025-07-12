# Understanding ALVR

## System Diagram

### Overall Architecture

```mermaid
graph TD
    subgraph "PC / Streamer"
        A[SteamVR] -- Raw Frames --> B(<b>alvr/server/</b><br>Driver Compositor & Encoder);
        B -- Encoded NALs --> C(<b>alvr/server_core/c_api.rs</b><br>alvr_send_video_nal);
        C -- VideoPacket --> D(<b>alvr/server_core/lib.rs</b><br>ServerCoreContext);
        D -- Reports Stats --> E(<b>alvr/server_core/statistics.rs</b><br>StatisticsManager);
        D -- Pushes to Channel --> F(Video Send Channel);
        G(<b>alvr/server_core/connection.rs</b><br>video_send_thread) -- Pulls from --> F;
        G -- Sends via --> H(<b>alvr/sockets/</b><br>StreamSender);

        I(<b>alvr/server_core/bitrate.rs</b><br>BitrateManager) -- DynamicEncoderParams --> B;
        E -- Latency/Size Stats --> I;
    end

    subgraph "Network (UDP/TCP)"
        H -- Video Packets --> J(<b>alvr/sockets/</b><br>StreamReceiver);
        K(<b>alvr/sockets/</b><br>StreamSender) -- ClientStatistics --> L(<b>alvr/sockets/</b><br>StreamReceiver);
    end

    subgraph "VR Headset / Client"
        J -- Encoded Packets --> M(<b>alvr/client_core/</b><br>Video Decoder<br><i>Uses Android MediaCodec</i>);
        M -- Decoded Frames --> N(<b>alvr/client_core/</b><br>Client Compositor<br><i>Re-expands FFR</i>);
        N -- Final Frames --> O(<b>alvr/client_openxr/</b><br>OpenXR Runtime);
        O -- Presents to --> P[Display];

        Q(Client-side Stats) -- Reports --> K;
        L -- ClientStatistics --> E;
    end

    style A fill:#f9f,stroke:#333,stroke-width:2px
    style P fill:#f9f,stroke:#333,stroke-width:2px
```

### Server

`TODO`

### Client

**Network**

```mermaid
graph TD
    subgraph "alvr/sockets"
        A["Network Socket (UDP/TCP)"] --> B("<b>stream_socket.rs</b><br>StreamSocket::recv()<br><i>Reassembles shards</i>");
    end

    subgraph "alvr/client_core"
        B -- Reconstructed Packet --> C{<b>connection.rs</b><br>video_receiver<br><i>mpsc::Receiver</i>};
        D(<b>connection.rs</b><br>video_receive_thread) -- Pulls from --> C;
        D -- "data.get()" --> E{Deserialized Header & NAL};
        E -- NAL --> F(<b>lib.rs</b><br>decoder_callback);
        F -- NAL --> G(<b>video_decoder/</b><br>Hardware Decoder Sink);
    end

    style A fill:#f9f,stroke:#333,stroke-width:2px
    style G fill:#f9f,stroke:#333,stroke-width:2px
```

**Video Decoding**

```mermaid
graph TD
    subgraph "alvr/client_core/connection.rs"
        A[video_receive_thread] -- NAL unit --> B(decoder_callback);
    end

    subgraph "alvr/client_openxr/stream.rs"
        B -- "sink.push_nal()" --> C[NAL Channel];
    end

    subgraph "alvr/client_core/video_decoder.rs"
        D(Decoder Thread<br><i>run_decoder_loop</i>) -- Pulls from --> C;
        D -- Feeds NAL to --> E(<b>Android MediaCodec</b><br><i>Hardware Decoder</i>);
        E -- Decodes frame to --> F[OpenGL Texture<br><i>via Android Surface</i>];
    end

    subgraph "alvr/client_openxr/stream.rs"
        G(StreamRenderer) -- Uses --> F;
    end

    style E fill:#f9f,stroke:#333,stroke-width:2px
```

# Vocabulary

## Video

- **Network Abstraction Layer (NAL)**: A logical packet within a video stream in H.264 (AVC) and H.265 (HEVC) video compression standards. It has several types, including **parameter sets** (SPS/PPS) that contains metadata including resolution, frame rate, and encoding profile, **IDR frames**, which are self-contained frame that does not depend on any previous frames (used for re-syncing after frame drop), and other types of frames. 

## Network


## Rust
- **Atomically Reference Counted (`Arc`)**: An Arc in Rust is a thread-safe, reference-counted smart pointer. An `Arc<T>` wraps your data of type `T` and keeps track of how many active references (or "owners") point to it. The "atomic" part is crucial: it guarantees that the incrementing and decrementing of the count is a safe operation that won't be corrupted even when multiple threads are doing it at the same time.
  1. **Creation**: You create an `Arc` with `Arc::new(data)`. At this point, the reference count is 1.
  2. **Cloning**: When you want to share ownership, you call Arc::clone(&my_arc). This doesn't clone the data itself; it just creates a new pointer to the same data and atomically increments the reference count by one. This cloning is very cheap.
  3. **Dropping**: When an `Arc` goes out of scope, its destructor atomically decrements the reference count.
  4. **Deallocation**: The actual data on the heap is only deallocated and cleaned up when the reference count drops to zero, meaning the last owner has gone out of scope.

