# Fix: Desktop UI 'Install OpenClaw' Button Resource Resolution

## Problem
When running OpenClaw Desktop on Linux and clicking the **"Install OpenClaw"** button on the setup screen, the installation fails with:
```
bash: /usr/lib/OpenClaw/install-cli.sh: No such file or directory
Installer exited with exit status: 127
```

## Root Cause
**File:** `apps/linux/src-tauri/src/installer.rs`

The installer attempts to resolve `install-cli.sh` using Tauri's `BaseDirectory::Resource`:
```rust
let script = app
    .path()
    .resolve("install-cli.sh", BaseDirectory::Resource)
    .map_err(|error| format!("Bundled installer is unavailable: {error}"))?;
```

**Issue:** On Linux, `BaseDirectory::Resource` resolves to `/usr/lib/OpenClaw/` (system package path), which doesn't exist in:
- **AppImage distributions** - bundled resources are in `$APPDIR/usr/share/openclaw-desktop/`
- **Standalone builds** - resources may be relative to the executable directory
- **Development builds** - resources are in the `src-tauri/` directory tree

## Solution

### 1. Ensure Resources Are Bundled
**File:** `apps/linux/src-tauri/tauri.conf.json`

Verify that `install-cli.sh` is included in the bundle:
```json
{
  "bundle": {
    "resources": [
      "../../scripts/install-cli.sh"
    ]
  }
}
```

### 2. Implement Fallback Resolution
**File:** `apps/linux/src-tauri/src/installer.rs`

Add a multi-strategy resource resolver:

```rust
fn resolve_installer_script(app: &AppHandle) -> Result<PathBuf, String> {
    // Strategy 1: Try bundled resource via Tauri path resolver
    if let Ok(path) = app
        .path()
        .resolve("install-cli.sh", BaseDirectory::Resource)
    {
        if path.exists() {
            return Ok(path);
        }
    }

    // Strategy 2: Look in AppDir (for AppImage)
    if let Ok(appdir) = std::env::var("APPDIR") {
        let appimage_path = PathBuf::from(appdir)
            .join("usr/share/openclaw-desktop/install-cli.sh");
        if appimage_path.exists() {
            return Ok(appimage_path);
        }
    }

    // Strategy 3: Look relative to executable (standalone)
    if let Ok(exe_path) = std::env::current_exe() {
        let exe_dir = exe_path.parent().ok_or("Cannot determine executable directory")?;
        let local_path = exe_dir.join("install-cli.sh");
        if local_path.exists() {
            return Ok(local_path);
        }
    }

    // Strategy 4: Fetch from remote (fallback)
    // Download from https://openclaw.ai/install-cli.sh and execute directly
    Err("Bundled installer not found. Falling back to remote download.".to_string())
}
```

### 3. Provide Remote Fallback
If the local script is not found, offer a remote execution option:

```rust
async fn install_from_remote(channel: InstallChannel) -> Result<(), String> {
    let url = "https://openclaw.ai/install-cli.sh";
    let response = reqwest::get(url)
        .await
        .map_err(|e| format!("Failed to download installer: {e}"))?;
    
    let script_content = response.text()
        .await
        .map_err(|e| format!("Failed to read installer: {e}"))?;
    
    // Execute script with appropriate channel
    execute_installer_script(&script_content, channel)
}
```

## Implementation Checklist

- [ ] Verify `install-cli.sh` is in `tauri.conf.json` bundle resources
- [ ] Update `installer.rs` to use multi-strategy resolution
- [ ] Add fallback to remote download if bundled resource unavailable
- [ ] Test with:
  - [ ] AppImage distribution
  - [ ] `.deb` package
  - [ ] Standalone binary
  - [ ] Development build (`cargo run`)
- [ ] Document the installer script path requirements

## Testing
```bash
# Test with AppImage
./OpenClaw-*.AppImage
# Navigate to setup screen and click "Install OpenClaw"

# Test with .deb package
sudo apt install ./OpenClaw-*.deb
openclaw-desktop
# Navigate to setup screen and click "Install OpenClaw"

# Test development build
cd apps/linux/src-tauri
cargo run
# Navigate to setup screen and click "Install OpenClaw"
```

## Workaround for End Users
Until fixed, manually install via terminal:
```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash -s -- --version beta --prefix ~/.openclaw
```

Then restart the desktop app - it will detect the CLI at `~/.openclaw/bin/openclaw`.

## References
- Issue: #2 - Desktop UI 'Install OpenClaw' button fails with missing `/usr/lib/OpenClaw/install-cli.sh`
- Tauri Path Resolver: https://tauri.app/en/v1/api/js/path/
- BaseDirectory Reference: https://tauri.app/en/v1/api/rust/tauri/path/enum.BaseDirectory/
