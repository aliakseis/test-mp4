# test-mp4

A minimal **GStreamer RTSP server for streaming an MP4 file over RTSP**.

The project is based on the GStreamer `gst-rtsp-server` `test-mp4.c` example and demonstrates how to create an RTSP server programmatically, construct a media pipeline from a GStreamer launch description, expose an MP4 file as an RTSP stream, and monitor RTCP/SSRC activity.

The application is intentionally small and focused on demonstrating the essential pieces required to turn a local MP4 file into an RTSP stream.

## Features

* Streams a local MP4 file through an RTSP server.
* Uses **GStreamer** and **gst-rtsp-server**.
* Supports configurable RTSP port.
* Provides a `/test` RTSP mount point.
* Streams:

  * H.264 video through `rtph264pay`
  * MPEG-4 audio through `rtpmp4apay`
* Uses the GStreamer RTSP media factory mechanism.
* Monitors RTCP activity through:

  * `on-ssrc-active`
  * `on-sender-ssrc-active`
* Prints RTCP statistics to the console.
* Uses CMake for building.
* Includes custom GStreamer discovery logic for Windows.
* Supports both 32-bit and 64-bit Windows GStreamer installations.

## How It Works

The application creates an RTSP server and attaches a media factory to the `/test` mount point.

The media factory uses the following GStreamer pipeline:

```text
(
    filesrc location="<file.mp4>" !
    qtdemux name=d
    d. ! queue ! rtph264pay pt=96 name=pay0
    d. ! queue ! rtpmp4apay pt=97 name=pay1
)
```

Conceptually, the pipeline looks like this:

```text
                    +----------------+
                    |    MP4 file    |
                    +-------+--------+
                            |
                        filesrc
                            |
                        qtdemux
                       /       \
                      /         \
                 H.264          MPEG-4
                    |              |
                  queue          queue
                    |              |
              rtph264pay       rtpmp4apay
                    |              |
                  pay0           pay1
                    \              /
                     \            /
                      RTSP Media Factory
                              |
                        RTSP Server
                              |
                    rtsp://127.0.0.1:8554/test
```

The `pay%d` naming convention is important: `gst-rtsp-server` uses elements named `pay0`, `pay1`, etc. as the RTP streams exposed by the media factory.

## Usage

The application expects an MP4 filename as its positional argument.

### Basic usage

```text
test-mp4 video.mp4
```

The default RTSP port is:

```text
8554
```

The resulting stream is available at:

```text
rtsp://127.0.0.1:8554/test
```

### Custom port

Use the `--port` / `-p` option:

```text
test-mp4 --port 8555 video.mp4
```

The stream is then available at:

```text
rtsp://127.0.0.1:8555/test
```

### Example with VLC

The stream can be opened with VLC using:

```text
rtsp://127.0.0.1:8554/test
```

Other RTSP-compatible players and GStreamer pipelines can also be used as clients.

## Command-Line Options

```text
Usage:
  test-mp4 [OPTION...] <filename.mp4>

Options:
  -p, --port=PORT    Port to listen on (default: 8554)
```

GStreamer also contributes its own standard command-line options through `gst_init_get_option_group()`.

## RTCP Statistics

The example goes beyond simply serving the media by monitoring RTCP activity.

When an RTCP source becomes active, the application retrieves its `stats` property:

```cpp
g_object_get(source, "stats", &stats, NULL);
```

The resulting `GstStructure` is converted to a human-readable string and printed to the console.

Two signals are monitored:

### `on-ssrc-active`

Triggered when a receiver/source SSRC becomes active.

### `on-sender-ssrc-active`

Triggered when a sender SSRC becomes active.

This provides a simple way to observe RTP/RTCP activity associated with connected clients.

## Media Lifecycle

The application connects to the RTSP media factory's `media-configure` signal:

```cpp
g_signal_connect(
    factory,
    "media-configure",
    (GCallback) media_configure_cb,
    factory);
```

When the media is configured, the application connects to the `prepared` signal.

Once the media has been prepared, the individual RTSP streams are inspected:

```cpp
n_streams = gst_rtsp_media_n_streams(media);
```

For each stream, the associated RTP session is retrieved:

```cpp
session = gst_rtsp_stream_get_rtpsession(stream);
```

The RTCP activity signals are then connected to that session.

This provides a useful example of how to move from:

```text
RTSP Server
    ↓
Media Factory
    ↓
RTSP Media
    ↓
RTSP Stream
    ↓
RTP Session
    ↓
RTCP statistics
```

## Requirements

### Runtime

The application requires a GStreamer installation containing the GStreamer core libraries and `gst-rtsp-server`.

The following components are used:

* GStreamer Core
* GStreamer Base
* GStreamer App
* GLib
* GObject
* GStreamer SDP
* GStreamer RTP
* GStreamer RTSP
* GStreamer RTSP Server

The MP4 file must also contain compatible streams.

The supplied pipeline expects:

* H.264 video
* MPEG-4 audio

## Build Requirements

* CMake 3.10 or newer
* C++17-compatible compiler
* GStreamer development packages
* GStreamer RTSP Server development package

The project sets:

```cmake
set(CMAKE_CXX_STANDARD 17)
```

## Building on Windows

The project contains a dedicated GStreamer detection module:

```text
cmake/detect_gstreamer.cmake
```

On Windows it searches for GStreamer installations using environment variables.

For a 64-bit build, the following variables are considered:

```text
GSTREAMER_1_0_ROOT_X86_64
GSTREAMER_ROOT_X86_64
GSTREAMER_1_0_ROOT_MSVC_X86_64
```

For a 32-bit build:

```text
GSTREAMER_1_0_ROOT_X86
GSTREAMER_ROOT_X86
GSTREAMER_1_0_ROOT_MSVC_X86
```

The detection code searches for GStreamer headers and libraries directly rather than requiring a conventional CMake package configuration.

### Example

After installing the GStreamer development/runtime packages and configuring the appropriate environment variables:

```bat
cmake -S . -B build
cmake --build build --config Release
```

The resulting executable can then be run with an MP4 file:

```bat
build\Release\test-mp4.exe video.mp4
```

## GStreamer Detection

`cmake/detect_gstreamer.cmake` supports two discovery mechanisms.

### Windows environment-based discovery

On Windows, the project explicitly searches known GStreamer installation locations.

It locates:

* `gst/gst.h`
* `glib.h`
* `glibconfig.h`
* GStreamer libraries
* GLib libraries
* RTSP-related libraries

The detected GStreamer version is also extracted from:

```text
gst/gstversion.h
```

### pkg-config

A secondary detection path is provided for environments where `pkg-config` is available.

This is particularly useful on Linux and other Unix-like environments.

## Linux

On Linux, install the GStreamer development packages appropriate for your distribution, including the RTSP server development package.

The project can then be configured with:

```bash
cmake -S . -B build
cmake --build build
```

Run:

```bash
./build/test-mp4 video.mp4
```

Then connect to:

```text
rtsp://127.0.0.1:8554/test
```

## Testing the Stream

A convenient way to test the server is with GStreamer itself.

For example:

```bash
gst-launch-1.0 rtspsrc location=rtsp://127.0.0.1:8554/test latency=100 ! decodebin ! autovideosink
```

You can also use VLC, FFplay, or another RTSP-capable player.

## Project Structure

```text
test-mp4/
├── CMakeLists.txt
├── README.md
├── cmake/
│   └── detect_gstreamer.cmake
└── src/
    └── main.cpp
```

### `src/main.cpp`

Contains the complete RTSP server implementation.

Responsibilities include:

* Command-line parsing
* GStreamer initialization
* RTSP server creation
* RTSP mount configuration
* MP4 pipeline construction
* RTSP media configuration
* RTP/RTCP session monitoring
* Main event loop

### `cmake/detect_gstreamer.cmake`

Provides GStreamer dependency detection.

It contains special handling for Windows installations and supports both 32-bit and 64-bit environments.

### `CMakeLists.txt`

Defines the executable and connects the discovered GStreamer headers, libraries, and compiler options.

## Example Session

Start the server:

```text
test-mp4 video.mp4
```

The application prints:

```text
stream ready at rtsp://127.0.0.1:8554/test
```

Connect an RTSP client to:

```text
rtsp://127.0.0.1:8554/test
```

When RTCP activity is detected, diagnostic information is printed by the application.

## Important Limitations

This is a **minimal test/example application**, rather than a general-purpose RTSP server.

In particular:

* The pipeline assumes specific MP4 codecs.
* The MP4 file is supplied directly to `qtdemux`.
* There is no automatic transcoding.
* There is no authentication or authorization.
* There is no TLS/RTSPS configuration.
* There is no custom session management.
* The server exposes a single `/test` mount point.
* Error handling is intentionally minimal.
* The server is intended primarily for experimentation and testing.

If an MP4 contains codecs incompatible with the hard-coded pipeline, the stream will not work without modifying the pipeline.

## Extending the Project

The project provides a useful starting point for experimenting with `gst-rtsp-server`.

Possible extensions include:

### Multiple files

Create multiple RTSP mount points:

```text
/test1
/test2
/test3
```

Each mount point can use a different `GstRTSPMediaFactory`.

### Transcoding

Insert decoders, converters, and encoders into the pipeline when the source MP4 is not already in an RTP-friendly format.

For example:

```text
filesrc
  ↓
qtdemux
  ↓
decoder
  ↓
videoconvert
  ↓
encoder
  ↓
rtph264pay
```

### Authentication

`GstRTSPAuth` can be introduced to restrict access to the RTSP server.

### Dynamic pipelines

Instead of constructing a pipeline directly from the input filename, the application can generate different pipelines depending on the detected media streams.

### Client/session monitoring

The existing RTCP callbacks provide a natural starting point for implementing:

* Client monitoring
* Connection statistics
* Packet-loss reporting
* Bitrate monitoring
* Stream health diagnostics
* Session logging

## Relation to the Original GStreamer Example

The implementation is based on the GStreamer RTSP server example:

```text
gst-rtsp-server/examples/test-mp4.c
```

The original example provides the basic architecture for serving an MP4 file through `gst-rtsp-server`.

This project retains that architecture while providing a standalone CMake project and additional RTCP/SSRC monitoring.

## License

`src/main.cpp` originates from the GStreamer example and contains the original GStreamer LGPL/GNU Library General Public License notice.

When redistributing or substantially modifying this project, review the licensing requirements of the GStreamer source and libraries used by the application.

## Summary

`test-mp4` is a compact example of using **GStreamer + gst-rtsp-server** to expose a local MP4 file as an RTSP stream.

Its main purpose is educational and experimental: it demonstrates how a GStreamer launch pipeline can be embedded into an RTSP media factory and how RTP/RTCP sessions can be inspected once a client connects.

The resulting architecture is deliberately simple:

```text
MP4
 │
 ▼
qtdemux
 │
 ├───────────────┐
 ▼               ▼
H.264          MPEG-4
 │               │
 ▼               ▼
rtph264pay   rtpmp4apay
 │               │
 └───────┬───────┘
         ▼
  gst-rtsp-server
         │
         ▼
rtsp://127.0.0.1:8554/test
```

It is therefore a useful starting point for building more sophisticated GStreamer-based RTSP streaming applications.

https://gitlab.freedesktop.org/gstreamer/gstreamer/-/blob/main/subprojects/gst-rtsp-server/examples/test-mp4.c

The pkg-config executable from the gstreamer installation must be in the path on Windows.

https://askubuntu.com/questions/1095521/cant-build-gst-rtsp-server/1095607#1095607
