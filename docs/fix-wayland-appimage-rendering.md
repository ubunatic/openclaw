# Fix: Wayland AppImage Rendering Issues

## Problem
The OpenClaw AppImage crashes on modern Ubuntu systems running GNOME Wayland with the error:
```
Could not create default EGL display: EGL_BAD_PARAMETER. Aborting...
```

This is caused by forcing `GDK_BACKEND=x11` in the linuxdeploy GTK plugin hook, which conflicts with Wayland display servers and recent Mesa graphics drivers.

## Root Cause
**File:** `linuxdeploy-plugin-gtk.sh` (line 9)  
**Current code:**
```bash
export GDK_BACKEND=x11
```

On Wayland systems, this forces GTK to use X11 mode, causing WebKitGTK to initialize EGL with X11-specific parameters. Modern Mesa (26.0+) and AMD Radeon drivers reject these parameters, causing the WebProcess to crash.

## Solution
Replace the hardcoded `GDK_BACKEND=x11` with a fallback chain that allows Wayland-native rendering:

```bash
export GDK_BACKEND=wayland,x11
```

### Why This Works
- **On Wayland systems:** GTK uses native Wayland rendering (better performance, compatibility)
- **On X11 systems:** Falls back to X11 (backward compatible)
- **On XWayland:** Uses appropriate X11 compatibility mode

### Secondary Issues
The bundled GLib/GIO libraries conflict with host system libraries. Solutions:
1. **Recommended:** Don't bundle base system libraries (libglib-2.0.so, libgio-2.0.so, librsvg)
2. **Alternative:** Switch to Flatpak/Snap packaging (handles runtime isolation properly)

## Implementation
This fix requires updating the linuxdeploy-plugin-gtk.sh script. Since this file is downloaded as a tool during the build process, the fix must be applied:

1. **In the build system** - patch the tool after download
2. **Upstream** - contribute the fix to tauri-apps/linuxdeploy-plugin-gtk

### Build System Patch
Add a post-download patch in the AppImage build pipeline:

```bash
#!/bin/bash
# Path: apps/linux/scripts/patch-linuxdeploy-gtk.sh
gtk_plugin="$1"
sed -i 's/export GDK_BACKEND=x11/export GDK_BACKEND=wayland,x11/' "$gtk_plugin"
```

Apply during AppImage build:
```bash
./apps/linux/scripts/patch-linuxdeploy-gtk.sh "$LDAI_RUNTIME_DIR/linuxdeploy-plugin-gtk.sh"
```

## Testing
After applying the fix:

```bash
# On Wayland system
export XDG_SESSION_TYPE=wayland
./OpenClaw-*.AppImage

# Verify Wayland is active
echo $XDG_SESSION_TYPE  # should print 'wayland'

# Check logs for successful EGL initialization
./OpenClaw-*.AppImage 2>&1 | grep -E "(EGL|Wayland|GTK)"
```

## References
- Issue: #1 - AppImage fails to render UI on Ubuntu 24.04+ Wayland
- GDK Backend Documentation: https://docs.gtk.org/gdk4/running.html#environment-variables
- Mesa EGL Parameter Reference: https://docs.mesa3d.org/egl.html

## Workaround for End Users (Until Fixed)
```bash
./OpenClaw-*.AppImage --appimage-extract
export GDK_BACKEND=wayland,x11
./squashfs-root/usr/bin/openclaw-desktop
```
