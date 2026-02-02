# Audio/Video Pass-Through Implementation Guide

This document provides guidance for completing the audio/video pass-through feature in RustDesk.

## Overview

The audio/video pass-through feature allows a user controlling a remote computer to share their local webcam and microphone with the controlled computer, making these devices available as virtual devices on the remote system.

## Current Status

### ✅ Completed
1. **Protocol Definitions** (`libs/hbb_common/protos/message.proto`)
   - Added `ClientAudioFrame` message for forwarding microphone audio
   - Added `ClientVideoFrame` message for forwarding webcam video
   - Added `PassThroughRequest`/`PassThroughResponse` for enabling/disabling

2. **Client Infrastructure** (`src/client/media_passthrough.rs`)
   - Created `MediaPassThroughManager` for managing pass-through state
   - Added methods for starting/stopping audio and video capture
   - Created skeleton for device enumeration

3. **Server Infrastructure** (`src/server/connection.rs`)
   - Added message handlers for pass-through requests
   - Added handlers for receiving client audio/video frames
   - Created response mechanism

### 🚧 To Be Implemented

## Platform-Specific Capture Implementation

### Windows Audio Capture
Use **WASAPI** (Windows Audio Session API):
```rust
// Example pseudo-code
use windows::Win32::Media::Audio::*;

// Enumerate devices
IMMDeviceEnumerator::CoCreateInstance()
device_enumerator.EnumAudioEndpoints(eCapture, DEVICE_STATE_ACTIVE)

// Capture audio
IAudioClient::Initialize()
IAudioClient::GetService::<IAudioCaptureClient>()
loop {
    capture_client.GetBuffer()
    // Send data via ClientAudioFrame message
}
```

### Windows Video Capture
Use **Media Foundation** or **DirectShow**:
```rust
// Example with Media Foundation
use windows::Win32::Media::MediaFoundation::*;

MFStartup()
// Enumerate cameras
MFCreateAttributes()
MFEnumDeviceSources()

// Capture video
IMFSourceReader::Create()
source_reader.ReadSample()
```

### Linux Audio Capture
Use **PulseAudio** (already partially implemented in `src/server/audio_service.rs`):
```rust
// Adapt existing PulseAudio code for input capture
use libpulse_binding as pulse;

let spec = pulse::sample::Spec {
    format: pulse::sample::SAMPLE_S16LE,
    channels: 2,
    rate: 48000,
};

let mut simple = pulse::simple::Simple::new(
    None,
    "RustDesk",
    pulse::stream::Direction::Record,
    None,
    "Microphone Capture",
    &spec,
    None,
    None,
).unwrap();
```

### Linux Video Capture
Use **V4L2** (Video4Linux2):
```rust
use v4l::prelude::*;

// List cameras
let devices = v4l::context::enum_devices();

// Capture from camera
let dev = v4l::Device::new(0)?;
let stream = v4l::io::mmap::Stream::new(&dev, Type::VideoCapture)?;
for frame in stream {
    // Send via ClientVideoFrame message
}
```

### macOS Audio Capture
Use **CoreAudio**:
```objective-c
// Swift/Objective-C bridging needed
AVCaptureDevice.requestAccess(for: .audio)
let session = AVCaptureSession()
let device = AVCaptureDevice.default(for: .audio)
let input = try AVCaptureDeviceInput(device: device)
session.addInput(input)
```

### macOS Video Capture
Use **AVFoundation**:
```objective-c
AVCaptureDevice.requestAccess(for: .video)
let session = AVCaptureSession()
let device = AVCaptureDevice.default(for: .video)
let input = try AVCaptureDeviceInput(device: device)
session.addInput(input)
```

## Virtual Device Creation

### Windows Virtual Audio Device
**Options:**
1. **VB-CABLE / Virtual Audio Cable** - Requires user to install separately
2. **Programmatic approach using Audio Drivers** - Complex, requires kernel driver

**Recommended**: Create a custom WASAPI loopback device or use existing virtual audio solutions

### Windows Virtual Webcam
**Options:**
1. **OBS Virtual Camera** approach - DirectShow filter
2. **v4l2loopback equivalent for Windows**

**Implementation Steps:**
```cpp
// DirectShow filter registration
HRESULT RegisterFilter() {
    IFilterMapper2 *pFM2 = NULL;
    CoCreateInstance(CLSID_FilterMapper2, ...);
    pFM2->RegisterFilter(CLSID_VirtualCam, ...);
}
```

### Linux Virtual Audio Device
Use **PulseAudio module-pipe-sink**:
```bash
# Create virtual sink
pactl load-module module-pipe-sink sink_name=rustdesk_virtual_mic

# Feed audio data to /tmp/audio.input
# Your Rust code writes to this pipe
```

Or use **ALSA loopback**:
```bash
modprobe snd-aloop
```

### Linux Virtual Webcam
Use **v4l2loopback** kernel module:
```bash
# Install and load
sudo modprobe v4l2loopback devices=1 video_nr=10 card_label="RustDesk Virtual Cam"

# Write frames to /dev/video10
use v4l::Device;
let mut dev = Device::new(10)?;
dev.write(&frame_data)?;
```

### macOS Virtual Devices
**Complex** - May require:
- CoreMediaIO framework for virtual cameras
- Audio driver extensions for virtual audio
- May need to be a system extension (SIP implications)

## Integration Steps

### 1. Complete Audio Capture
In `src/client/media_passthrough.rs`:
```rust
fn audio_capture_thread(
    device: Option<String>,
    message_sender: UnboundedSender<crate::client::Data>,
    stop_receiver: std::sync::mpsc::Receiver<()>,
) {
    // Platform-specific capture
    #[cfg(target_os = "windows")]
    let capture = windows_audio_capture(device);
    
    #[cfg(target_os = "linux")]
    let capture = linux_audio_capture(device);
    
    #[cfg(target_os = "macos")]
    let capture = macos_audio_capture(device);
    
    while stop_receiver.try_recv().is_err() {
        let audio_data = capture.read_frame();
        
        let mut frame = ClientAudioFrame::new();
        frame.set_data(audio_data);
        
        let mut format = AudioFormat::new();
        format.set_sample_rate(48000);
        format.set_channels(2);
        frame.set_format(format);
        
        let mut msg = Message::new();
        msg.set_client_audio_frame(frame);
        message_sender.send(crate::client::Data::Message(msg)).ok();
    }
}
```

### 2. Complete Video Capture
Similar to audio capture, implement platform-specific video capture in the same file.

### 3. Server-Side Virtual Device Routing
In `src/server/connection.rs`:
```rust
async fn handle_client_audio_frame(&mut self, frame: ClientAudioFrame) {
    // Get or create virtual audio device
    let virtual_device = self.get_or_create_virtual_audio_device();
    
    // Write audio data to virtual device
    virtual_device.write_frame(&frame.data);
}
```

### 4. Add UI Controls
In Flutter client (`flutter/lib/desktop/pages/connection_page.dart`):
```dart
// Add pass-through controls
class PassThroughControls extends StatefulWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        IconButton(
          icon: Icon(Icons.mic),
          onPressed: () {
            // Toggle audio pass-through
            bind.sessionTogglePassThrough(
              sessionId: widget.id,
              type: 'audio',
              enable: !audioEnabled,
            );
          },
        ),
        IconButton(
          icon: Icon(Icons.videocam),
          onPressed: () {
            // Toggle video pass-through
            bind.sessionTogglePassThrough(
              sessionId: widget.id,
              type: 'video',
              enable: !videoEnabled,
            );
          },
        ),
      ],
    );
  }
}
```

## Building and Testing

### Build Requirements
1. Ensure vcpkg is set up for codec dependencies
2. Initialize submodules: `git submodule update --init --recursive`
3. Build: `cargo build --release`

### Testing Locally
1. Start a server instance
2. Connect with client
3. Enable audio pass-through
4. Use system sound settings to verify virtual microphone appears
5. Test with an application (e.g., recording software)

## Security Considerations

1. **Permissions**: Audio/video capture requires user permissions
2. **Privacy**: Add clear UI indicators when pass-through is active
3. **Bandwidth**: Consider compression for video frames (MJPEG or H.264)
4. **Authentication**: Only allow pass-through for authenticated connections

## Performance Optimization

1. **Audio**: Use Opus codec for compression (already available in RustDesk)
2. **Video**: Consider MJPEG for simplicity or H.264 for better compression
3. **Frame Rate**: Limit video to 15-30 fps to reduce bandwidth
4. **Resolution**: Allow user to select video resolution

## Dependencies to Add

Add to `Cargo.toml`:
```toml
[target.'cfg(windows)'.dependencies]
windows = { version = "0.51", features = ["Media_Audio", "Media_MediaFoundation"] }

[target.'cfg(target_os = "linux")'.dependencies]
libpulse-binding = "2.28"
v4l = "0.14"

[target.'cfg(target_os = "macos")'.dependencies]
# macOS frameworks require Objective-C bindings
core-foundation = "0.9"
```

## Known Challenges

1. **Virtual Device Drivers**: 
   - Windows and macOS may require signed drivers
   - Linux requires kernel modules (v4l2loopback, snd-aloop)
   
2. **Permission Prompts**:
   - macOS requires explicit user consent for camera/microphone
   - Windows UAC may prompt for virtual device creation
   
3. **Compatibility**:
   - Virtual devices may not work with all applications
   - Some apps require specific device types

## Alternative Approach: Network Streaming

Instead of virtual devices, consider streaming approach:
- Client streams audio/video over custom protocol
- Server-side application receives and plays back
- Simpler but requires dedicated receiver application

## References

- [WASAPI Documentation](https://docs.microsoft.com/en-us/windows/win32/coreaudio/wasapi)
- [Media Foundation](https://docs.microsoft.com/en-us/windows/win32/medfound/microsoft-media-foundation-sdk)
- [PulseAudio Documentation](https://www.freedesktop.org/wiki/Software/PulseAudio/)
- [V4L2 API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/v4l2.html)
- [AVFoundation](https://developer.apple.com/av-foundation/)

## Summary

This is a complex feature that requires:
1. Platform-specific capture implementation (most work)
2. Virtual device creation (complex, may need external tools)
3. UI integration (relatively simple)
4. Testing across platforms (time-consuming)

Consider starting with a single platform (e.g., Windows or Linux) and expanding from there.
