# Protocol Changes - Important Note

## Submodule Changes

The protobuf definitions in `libs/hbb_common/protos/message.proto` have been modified to add the new pass-through messages. However, `hbb_common` is a git submodule pointing to the external repository `rustdesk/hbb_common`.

## Changes Made

In `libs/hbb_common/protos/message.proto`:

1. Added 4 new message types (lines 722-757):
   - `ClientAudioFrame`
   - `ClientVideoFrame`
   - `PassThroughRequest`
   - `PassThroughResponse`

2. Extended `Message` union (lines 1020-1023):
   - Added the 4 new message types to the union with field numbers 33-36

## How to Handle This

Since `hbb_common` is an external submodule, there are two approaches:

### Option 1: Fork and Modify (Recommended for Testing)
```bash
# Fork the rustdesk/hbb_common repository on GitHub
# Update .gitmodules to point to your fork
git config submodule.libs/hbb_common.url https://github.com/YOUR_USERNAME/hbb_common

# Commit and push changes in the submodule
cd libs/hbb_common
git checkout -b add-passthrough-protocol
git add protos/message.proto
git commit -m "Add audio/video pass-through protocol messages"
git push origin add-passthrough-protocol

# Back in main repo, update submodule reference
cd ../..
git add libs/hbb_common
git commit -m "Update hbb_common submodule with pass-through protocol"
```

### Option 2: Submit PR to Upstream (For Production)
1. Fork `rustdesk/hbb_common` on GitHub
2. Create a branch with your changes
3. Submit a PR to the official repository
4. Once merged, update the submodule reference in main RustDesk repo

### Option 3: Local Development Only
For local testing without pushing to a fork:
```bash
# The changes are already in the submodule locally
# Just rebuild to regenerate protobuf code
cargo clean
cargo build --release
```

The protobuf code generator will pick up the changes and generate the new Rust types.

## What Gets Generated

When you build, the protobuf compiler will generate:
- `ClientAudioFrame` struct
- `ClientVideoFrame` struct  
- `PassThroughRequest` struct with `Type` enum
- `PassThroughResponse` struct with `Type` enum
- `message::Union` variants for all four types

These will be available as:
```rust
use hbb_common::message_proto::{
    ClientAudioFrame,
    ClientVideoFrame,
    PassThroughRequest,
    PassThroughResponse,
};
```

## Current Status

✅ Protocol changes made locally in submodule  
⏸️ Not committed to submodule (requires fork or upstream PR)  
✅ Main repo code references the new types  
⏳ Will compile once protobuf is regenerated

## Building

When ready to build:
```bash
# Initialize submodules if not already done
git submodule update --init --recursive

# Build - this will regenerate protobuf
cargo build --release

# Or just build the hbb_common lib first
cd libs/hbb_common
cargo build
cd ../..
cargo build --release
```

The build process will automatically run `build.rs` which invokes the protobuf code generator.

## For Upstream Contribution

If you want to contribute this feature to the official RustDesk project:

1. **Protocol Changes**: Submit PR to `rustdesk/hbb_common` first
2. **Implementation**: After protocol PR is merged, submit PR to `rustdesk/rustdesk` with the implementation
3. **Coordination**: Coordinate with RustDesk maintainers on the approach

## Summary

- ✅ Protocol definitions are ready
- ⚠️ Submodule changes need to be handled (fork or upstream PR)
- ✅ Main codebase references the new types
- ⏳ Protobuf generation happens on build
