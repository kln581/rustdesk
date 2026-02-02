# Audio/Video Pass-Through Feature - Implementation Summary

## What Was Accomplished

I've successfully laid the foundation for the audio/video pass-through feature in RustDesk. This feature will allow users to forward their local webcam and microphone from the controlling computer to the controlled computer, making them available as virtual devices.

### Changes Made

1. **Protocol Extensions** (`libs/hbb_common/protos/message.proto`)
   - Added 4 new message types:
     - `ClientAudioFrame`: Carries audio data from controller's microphone
     - `ClientVideoFrame`: Carries video data from controller's webcam
     - `PassThroughRequest`: Request to enable/disable pass-through
     - `PassThroughResponse`: Response with success/failure status
   - Updated the main `Message` union to include these new types

2. **Client-Side Infrastructure** (`src/client/media_passthrough.rs`)
   - Created `MediaPassThroughManager` class to manage pass-through state
   - Implemented methods to:
     - Start/stop audio capture
     - Start/stop video capture
     - Request pass-through from server
     - Track active state
   - Added placeholder for device enumeration
   - Included skeleton for capture threads

3. **Server-Side Infrastructure** (`src/server/connection.rs`)
   - Added message handlers for:
     - `PassThroughRequest`: Enable/disable pass-through
     - `ClientAudioFrame`: Receive audio from client
     - `ClientVideoFrame`: Receive video from client
   - Implemented response mechanism
   - Added placeholders for virtual device routing

4. **Documentation**
   - Created `PASSTHROUGH_IMPLEMENTATION.md` with:
     - Platform-specific capture examples (Windows/Linux/macOS)
     - Virtual device creation strategies
     - Code examples and integration steps
     - Dependencies and build requirements
     - Security and performance considerations

## How It Works (Design)

### Flow for Audio Pass-Through

1. **Client Side**:
   ```
   User clicks "Enable Microphone" 
   → MediaPassThroughManager.start_audio_capture()
   → Send PassThroughRequest to server
   → Start capture thread
   → Capture audio frames from microphone
   → Send ClientAudioFrame messages to server
   ```

2. **Server Side**:
   ```
   Receive PassThroughRequest
   → Create/prepare virtual audio device
   → Send PassThroughResponse (success/fail)
   → Receive ClientAudioFrame messages
   → Write audio data to virtual device
   → Apps on controlled computer can use virtual microphone
   ```

### Flow for Video Pass-Through

Same as audio, but with video frames from webcam.

## What Still Needs to Be Done

### Critical Implementation Work

1. **Platform-Specific Capture** (Medium-High Complexity)
   - Windows: Use WASAPI for audio, Media Foundation for video
   - Linux: Use PulseAudio for audio, V4L2 for video
   - macOS: Use CoreAudio for audio, AVFoundation for video
   - Implement actual capture loops in the thread functions

2. **Virtual Device Creation** (High Complexity - Hardest Part)
   - **Windows**:
     - Virtual audio: May need VB-Cable or similar driver
     - Virtual webcam: DirectShow filter (complex)
   - **Linux**:
     - Virtual audio: PulseAudio module or ALSA loopback
     - Virtual webcam: v4l2loopback kernel module (user needs to install)
   - **macOS**:
     - Both: May require system extensions (SIP complications)

3. **UI Integration** (Medium Complexity)
   - Add controls to Flutter client
   - Device selection dropdowns
   - Enable/disable buttons
   - Status indicators

4. **Build System**
   - Protobuf regeneration (automatic when building in proper environment)
   - Add platform-specific dependencies to Cargo.toml

### Challenges & Considerations

**Major Challenges**:
- Virtual device creation on Windows/macOS may require signed drivers
- Linux requires kernel modules that users must install separately
- Permission handling (camera/microphone access)
- Bandwidth usage (need compression for video)

**Recommended Approach**:
1. Start with Linux (easiest - v4l2loopback and PulseAudio)
2. Then Windows (moderate - existing drivers available)
3. macOS last (hardest - system extension requirements)

## Build and Test Instructions

### Building

The code won't compile yet because:
1. The protobuf changes need to be regenerated (requires proper build environment)
2. Platform-specific capture code is not implemented (will cause compile errors)

To build when ready:
```bash
# Initialize submodules
git submodule update --init --recursive

# Build (will regenerate protobuf)
cargo build --release

# Or build with specific features
cargo build --release --features hwcodec
```

### Testing When Complete

1. Build RustDesk with the changes
2. Start server on one machine
3. Connect with client from another machine
4. Enable audio pass-through from client
5. Check system settings on server to see virtual microphone
6. Test with any app that uses microphone
7. Repeat for video pass-through

## Alternative Approaches

If virtual device creation proves too difficult:

### Alternative 1: Simpler Streaming
Instead of virtual devices, create a receiver application:
- Client streams audio/video
- Server has a dedicated "RustDesk Receiver" app
- App displays video and plays audio
- Simpler but less integrated

### Alternative 2: Browser-Based
Use WebRTC:
- Client opens a browser-based interface
- WebRTC handles capture and streaming
- Server side receives via browser API
- Modern and well-supported but requires browser

### Alternative 3: External Tools
Document how to use existing tools:
- OBS Virtual Camera for video
- VoiceMeeter or VB-Cable for audio
- RustDesk forwards streams to these tools
- User needs to install separately

## Security & Privacy Notes

Important considerations:
1. **User Consent**: Always require explicit user action to enable
2. **Visual Indicators**: Show when camera/mic is being forwarded
3. **Permissions**: Request OS permissions properly
4. **Encryption**: Audio/video data should use RustDesk's existing encryption
5. **Disconnection**: Automatically stop capture when connection ends

## Current Limitations

1. **No actual capture**: Placeholder code only
2. **No virtual devices**: Handler functions are stubs
3. **No UI**: Command-line only (need Flutter integration)
4. **Single platform**: Need separate implementations for each OS
5. **No compression**: Need to add codec support for bandwidth efficiency

## Next Steps for Developer

Recommended order of implementation:

1. **Choose a platform** (recommend Linux first)
2. **Implement audio capture**:
   - Complete the `audio_capture_thread` function
   - Add device enumeration
   - Test sending frames to server
3. **Implement virtual audio device**:
   - Research platform-specific method
   - Implement device creation
   - Test routing audio to virtual device
4. **Repeat for video**
5. **Add UI controls**
6. **Test end-to-end**
7. **Port to other platforms**

## Conclusion

A solid foundation has been established with:
- Complete protocol definitions
- Message routing infrastructure
- Management classes
- Comprehensive documentation

The remaining work is primarily platform-specific implementation, which is substantial but well-documented in `PASSTHROUGH_IMPLEMENTATION.md`.

This is an advanced feature that will significantly enhance RustDesk's capabilities by enabling true media device forwarding for remote support and collaboration scenarios.
