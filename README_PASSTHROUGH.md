# Audio/Video Pass-Through Feature

This directory contains the implementation of audio/video pass-through for RustDesk, allowing users to forward their local webcam and microphone to remote computers.

## Quick Start

### What This Feature Does

When you're remotely controlling Computer B from Computer A:
- Normally: You can see/control Computer B's screen
- With pass-through: Computer B can use Computer A's webcam and microphone

**Use Cases:**
- Video conferencing on a remote machine using your local camera
- Using your microphone for voice calls on the remote computer
- Remote presentations or recordings using local media devices

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Computer A (Controller)                     │
│                                                                   │
│  ┌──────────────┐         ┌──────────────────────────┐         │
│  │   Webcam     │────────▶│  MediaPassThroughManager │         │
│  └──────────────┘         │                          │         │
│                           │  • Captures frames       │         │
│  ┌──────────────┐         │  • Encodes if needed     │         │
│  │  Microphone  │────────▶│  • Sends to server       │         │
│  └──────────────┘         └────────────┬─────────────┘         │
│                                         │                        │
└─────────────────────────────────────────┼────────────────────────┘
                                          │
                          ClientAudioFrame / ClientVideoFrame
                                          │
                                          ▼
┌─────────────────────────────────────────┼────────────────────────┐
│                      Computer B (Controlled)                      │
│                                         │                         │
│                           ┌─────────────▼──────────────┐         │
│                           │   Connection Handler       │         │
│                           │                            │         │
│                           │  • Receives frames         │         │
│                           │  • Routes to virtual dev   │         │
│                           └─────────────┬──────────────┘         │
│                                         │                         │
│     ┌───────────────────────────────────┴────────────────┐       │
│     │                                                     │       │
│     ▼                                                     ▼       │
│  ┌─────────────────────┐                    ┌──────────────────┐ │
│  │ Virtual Microphone  │                    │ Virtual Webcam   │ │
│  │ (OS sees this as    │                    │ (OS sees this as │ │
│  │  real microphone)   │                    │  real camera)    │ │
│  └─────────────────────┘                    └──────────────────┘ │
│            │                                           │          │
│            └───────────────────┬───────────────────────┘          │
│                                ▼                                  │
│                     ┌────────────────────┐                        │
│                     │  Any Application   │                        │
│                     │  (Zoom, Teams,     │                        │
│                     │   recording, etc)  │                        │
│                     └────────────────────┘                        │
└───────────────────────────────────────────────────────────────────┘
```

## File Structure

```
rustdesk/
├── libs/hbb_common/
│   └── protos/
│       └── message.proto              # Protocol definitions (new messages added)
│
├── src/
│   ├── client/
│   │   └── media_passthrough.rs       # Client-side capture and management
│   │
│   └── server/
│       └── connection.rs              # Server-side message handling (modified)
│
├── IMPLEMENTATION_SUMMARY.md          # High-level overview of the feature
├── PASSTHROUGH_IMPLEMENTATION.md      # Detailed implementation guide
└── README_PASSTHROUGH.md             # This file
```

## Implementation Status

### ✅ Completed (Foundation)
- Protocol message definitions
- Client-side management infrastructure  
- Server-side message handlers
- Message routing between client and server
- Comprehensive documentation

### 🚧 To Be Implemented
- Platform-specific audio capture (Windows/Linux/macOS)
- Platform-specific video capture (Windows/Linux/macOS)
- Virtual device creation (complex - may need drivers)
- UI controls in Flutter client
- Testing and optimization

## Documentation

1. **[IMPLEMENTATION_SUMMARY.md](./IMPLEMENTATION_SUMMARY.md)**
   - What was accomplished
   - How the system works
   - What needs to be done
   - Challenges and alternatives

2. **[PASSTHROUGH_IMPLEMENTATION.md](./PASSTHROUGH_IMPLEMENTATION.md)**
   - Detailed code examples for each platform
   - Virtual device strategies
   - Integration steps
   - Dependencies and references

## Protocol Messages

Four new message types were added to `message.proto`:

```protobuf
// Client sends audio from its microphone
message ClientAudioFrame {
  bytes data = 1;
  AudioFormat format = 2;
}

// Client sends video from its webcam
message ClientVideoFrame {
  bytes data = 1;
  int32 width = 2;
  int32 height = 3;
  string format = 4;
}

// Request to enable/disable pass-through
message PassThroughRequest {
  enum Type { Audio = 0; Video = 1; }
  Type type = 1;
  bool enable = 2;
  string device = 3;  // Optional device name
}

// Response from server
message PassThroughResponse {
  enum Type { Audio = 0; Video = 1; }
  Type type = 1;
  bool success = 2;
  string error = 3;
  repeated string available_devices = 4;
}
```

## Code Examples

### Client-Side: Enable Audio Pass-Through

```rust
use crate::client::media_passthrough::{MediaPassThroughManager, PassThroughType};

// Create manager (usually in Remote struct)
let mut manager = MediaPassThroughManager::new(sender.clone());

// Start audio capture with default device
manager.start_audio_capture(None);

// Or with specific device
manager.start_audio_capture(Some("USB Microphone".to_string()));

// Stop when done
manager.stop_audio_capture();
```

### Server-Side: Handle Received Frames

The server automatically handles incoming frames through the connection handler:

```rust
// In src/server/connection.rs
async fn handle_client_audio_frame(&mut self, frame: ClientAudioFrame) {
    // TODO: Route to virtual audio device
    // This is where you'd write audio data to a virtual sink
}

async fn handle_client_video_frame(&mut self, frame: ClientVideoFrame) {
    // TODO: Route to virtual video device
    // This is where you'd write video frames to a virtual webcam
}
```

## Building

When building in a proper environment (not the sandboxed agent environment):

```bash
# Initialize submodules
git submodule update --init --recursive

# Build
cargo build --release

# Or with specific features
cargo build --release --features hwcodec
```

This will automatically regenerate the protobuf code with the new message types.

## Security Considerations

1. **User Consent**: Pass-through only activates with explicit user action
2. **Visual Indicators**: UI should show when devices are being forwarded
3. **Permissions**: Proper OS-level permission requests for camera/microphone
4. **Encryption**: All data uses RustDesk's existing secure channel
5. **Auto-Stop**: Capture stops automatically on disconnection

## Platform-Specific Notes

### Windows
- **Audio**: Use WASAPI
- **Video**: Use Media Foundation or DirectShow
- **Virtual Devices**: May need VB-Cable (audio) and DirectShow filter (video)
- **Challenge**: May require signed drivers

### Linux
- **Audio**: Use PulseAudio
- **Video**: Use V4L2
- **Virtual Devices**: 
  - Audio: `module-pipe-sink` or ALSA loopback
  - Video: `v4l2loopback` kernel module
- **Challenge**: User must install kernel module

### macOS
- **Audio**: Use CoreAudio
- **Video**: Use AVFoundation
- **Virtual Devices**: CoreMediaIO (video) and Audio HAL (audio)
- **Challenge**: May require system extensions (SIP implications)

## Performance Tips

1. **Audio Compression**: Use Opus codec (already in RustDesk)
2. **Video Compression**: MJPEG for simplicity, H.264 for efficiency
3. **Frame Rate**: Limit to 15-30 fps for video
4. **Resolution**: Allow user to select (e.g., 640x480, 1280x720)
5. **Buffering**: Add jitter buffer for smoother playback

## Troubleshooting

### "No devices found"
- Check OS permissions for camera/microphone
- Ensure devices are not in use by other apps

### "Virtual device not appearing"
- Windows: Check if virtual audio cable is installed
- Linux: Ensure v4l2loopback module is loaded
- macOS: System extension may need approval

### "Poor video quality"
- Adjust resolution settings
- Check network bandwidth
- Enable compression

## Contributing

To contribute to this feature:

1. Pick a platform to implement (recommend Linux first)
2. See `PASSTHROUGH_IMPLEMENTATION.md` for code examples
3. Implement capture for that platform
4. Add tests
5. Submit PR

## Future Enhancements

Potential improvements:
- [ ] Multiple video sources (screen + webcam)
- [ ] Audio mixing (multiple sources)
- [ ] Video filters (virtual backgrounds)
- [ ] Hardware encoding acceleration
- [ ] Adaptive bitrate based on network
- [ ] Recording capability
- [ ] Picture-in-picture mode

## License

This feature is part of RustDesk and follows the same license terms.

## Questions?

See the detailed implementation guides:
- [IMPLEMENTATION_SUMMARY.md](./IMPLEMENTATION_SUMMARY.md) - Overview
- [PASSTHROUGH_IMPLEMENTATION.md](./PASSTHROUGH_IMPLEMENTATION.md) - Technical details

---

**Status**: Foundation complete, platform-specific implementation pending  
**Complexity**: High (requires platform-specific APIs and virtual device drivers)  
**Impact**: Enables true media device forwarding for remote collaboration
