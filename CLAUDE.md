# UniWindowController (UniWinC) - Plugin Reference

> **Package:** `com.kirurobo.uniwinc` v0.9.4 | **Namespace:** `Kirurobo`
> **Platforms:** Windows 10/11, macOS | **Native:** `LibUniWinC.dll` / `.bundle`
> **Status:** Installed via UPM (`Library/PackageCache/com.kirurobo.uniwinc@...`)

## Desktop Pet Essentials

UniWinC controls the app's OS window — the foundation for a desktop pet:
- **Transparent window** — pet character floats on the desktop
- **Click-through** on transparent pixels — interact with desktop normally
- **Always-on-top** — pet stays visible above other windows
- **Drag-to-move** — reposition the pet via UI handle
- **File drop** — feed files to the pet
- **Window position/size** — programmatic movement for pet locomotion

**Transparency only works in standalone builds, not the Unity Editor.**

---

## Setup Checklist

1. Add `UniWindowController` prefab to scene
2. Add `DragMoveCanvas` prefab for drag-to-move (needs `EventSystem`)
3. Player Settings (use inspector "Fix all" button, or set manually):
   - **Run in Background:** true
   - **Resizable Window:** true
   - **Fullscreen Mode:** Windowed
   - **Allow Fullscreen Switch:** false
   - **Use DXGI Flip Mode Swapchain:** false (Windows, required for transparency)
4. Build as PC/Mac Standalone

---

## API Reference

### UniWindowController (MonoBehaviour, Singleton)

Access the singleton instance via:
```csharp
UniWindowController uwc = UniWindowController.current;
```

#### Properties

| Property | Type | Default | Description |
|:---------|:-----|:--------|:------------|
| `isTransparent` | `bool` | `false` | Toggle transparent (non-rectangular) window. Only works in builds. |
| `alphaValue` | `float` | `1.0` | Window opacity (0.0 fully transparent to 1.0 fully opaque). |
| `isTopmost` | `bool` | `false` | Always-on-top. Mutually exclusive with `isBottommost`. |
| `isBottommost` | `bool` | `false` | Always-on-bottom. Mutually exclusive with `isTopmost`. |
| `isZoomed` | `bool` | `false` | Maximize/restore the window (called "zoomed" on macOS). |
| `isClickThrough` | `bool` | `false` | Pass mouse events through to windows behind. |
| `isHitTestEnabled` | `bool` | `true` | Enable automatic click-through based on `hitTestType`. When false, manually control `isClickThrough`. |
| `hitTestType` | `HitTestType` | `Opacity` | Automatic click-through detection method. |
| `opacityThreshold` | `float` | `0.1` | Alpha threshold for opacity-based hit test (0.0-1.0). |
| `allowDropFiles` | `bool` | `false` | Accept file/folder drag-and-drop onto window. |
| `shouldFitMonitor` | `bool` | `false` | Fit (maximize) the window to a specific monitor. |
| `monitorToFit` | `int` | `0` | Target monitor index when `shouldFitMonitor` is true. |
| `windowPosition` | `Vector2` | â€” | Get/set window position. Origin is lower-left of main monitor, Y-up. |
| `windowSize` | `Vector2` | â€” | Get/set window size (including frame). |
| `clientSize` | `Vector2` | â€” | Get client area size (read-only). |
| `cursorPosition` | `Vector2` | â€” | Get/set mouse cursor position in screen coordinates. |
| `pickedColor` | `Color` | â€” | Color under cursor when opacity hit test is active (read-only). |
| `transparentType` | `TransparentType` | `Alpha` | Transparency method (Windows only). |
| `keyColor` | `Color32` | `(1,0,1,0)` | Key color for ColorKey transparency (Windows only). |
| `autoSwitchCameraBackground` | `bool` | `true` | Auto-switch camera to transparent background when transparent. |
| `forceWindowed` | `bool` | `false` | Force windowed mode on startup if launched fullscreen. |
| `currentCamera` | `Camera` | `null` | Camera to manage. Uses `Camera.main` if null. |

#### Methods

| Method | Returns | Description |
|:-------|:--------|:------------|
| `SetTransparentType(TransparentType type)` | `void` | Change transparency method at runtime (Windows only). |
| `SetCamera(Camera newCamera)` | `void` | Change the managed camera, restoring previous camera's background. |
| `GetMonitorCount()` | `int` | Static. Get number of connected monitors. |
| `GetMonitorRect(int index)` | `Rect` | Static. Get monitor position and size. |
| `GetCursorPosition()` | `Vector2` | Static. Get mouse cursor screen position. |
| `SetCursorPosition(Vector2 pos)` | `void` | Static. Set mouse cursor screen position. |

#### Enums

**`TransparentType`** (Windows only)
| Value | Description |
|:------|:------------|
| `None` | No transparency |
| `Alpha` | Standard alpha-based transparency (default) |
| `ColorKey` | Single-color transparency via layered window. No semi-transparency but better touch support. |

**`HitTestType`**
| Value | Description |
|:------|:------------|
| `None` | No automatic hit test. Manually control `isClickThrough`. |
| `Opacity` | Check pixel alpha under cursor. Natural but higher CPU cost. |
| `Raycast` | Check colliders and UI raycasts. Lower CPU cost, requires colliders. Respects "Ignore Raycast" layer. |

**`WindowStateEventType`** (Flags enum)
| Value | Description |
|:------|:------------|
| `StyleChanged` | Window style was modified |
| `Resized` | Window size changed |
| `TopMostEnabled` / `TopMostDisabled` | Topmost state changed |
| `BottomMostEnabled` / `BottomMostDisabled` | Bottommost state changed |
| `WallpaperModeEnabled` / `WallpaperModeDisabled` | Wallpaper mode toggled |

#### Events

| Event | Signature | Description |
|:------|:----------|:------------|
| `OnStateChanged` | `OnStateChangedDelegate(WindowStateEventType type)` | Window style, maximize, topmost, etc. changed. |
| `OnDropFiles` | `FilesDelegate(string[] files)` | Files or folders were dropped on the window. |
| `OnMonitorChanged` | `OnMonitorChangedDelegate()` | Monitor configuration or resolution changed. |

---

### UniWindowMoveHandle (MonoBehaviour)

Attach to any UI element (must be a Raycast Target) to enable drag-to-move of the window.

| Property | Type | Default | Description |
|:---------|:-----|:--------|:------------|
| `disableOnZoomed` | `bool` | `true` | Disable drag-move when the window is maximized or fit-to-monitor. |
| `IsDragging` | `bool` | â€” | Whether a drag is in progress (read-only). |

**Behavior:**
- Only responds to left mouse button drag
- Ignores drag when Shift, Ctrl, or Alt is held
- Automatically disables hit test during drag and restores it after
- On macOS uses native cursor position; on Windows uses `eventData.position` for touch compatibility
- Implements `IDragHandler`, `IBeginDragHandler`, `IEndDragHandler`, `IPointerUpHandler`

**DragMoveCanvas prefab pattern:**
- A full-screen Panel on the "Ignore Raycast" layer with `UniWindowMoveHandle` attached
- Since it's on "Ignore Raycast", the automatic hit test ignores it
- Set a lower Sort Order so other UI elements receive events first

---

### FilePanel (Static Class)

Native cross-platform file open/save dialogs.

#### Opening Files

```csharp
FilePanel.Settings settings = new FilePanel.Settings
{
    title = "Open Image",
    filters = new FilePanel.Filter[]
    {
        new FilePanel.Filter("Image files", "png", "jpg", "jpeg"),
        new FilePanel.Filter("All files", "*"),
    },
    flags = FilePanel.Flag.AllowMultipleSelection,
    initialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyPictures),
    initialFile = "photo.png",
};

FilePanel.OpenFilePanel(settings, (string[] files) =>
{
    foreach (var file in files)
        Debug.Log("Selected: " + file);
});
```

#### Saving Files

```csharp
FilePanel.Settings settings = new FilePanel.Settings
{
    title = "Save File",
    filters = new FilePanel.Filter[]
    {
        new FilePanel.Filter("Text files", "txt", "log"),
        new FilePanel.Filter("All files", "*"),
    },
    initialFile = "output.txt",
    initialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments),
};

FilePanel.SaveFilePanel(settings, (string[] files) =>
{
    Debug.Log("Save to: " + files[0]);
});
```

#### FilePanel.Settings

| Field | Type | Description |
|:------|:-----|:------------|
| `title` | `string` | Dialog title |
| `filters` | `Filter[]` | File type filters. `Filter("Label", "ext1", "ext2", ...)` |
| `initialDirectory` | `string` | Starting directory |
| `initialFile` | `string` | Default filename |
| `defaultExtension` | `string` | Not implemented yet |
| `flags` | `Flag` | Bitfield flags (see below) |

#### FilePanel.Flag

| Flag | Value | Description |
|:-----|:------|:------------|
| `None` | 0 | Default |
| `FileMustExist` | 1 | Windows only |
| `FolderMustExist` | 2 | Windows only |
| `AllowMultipleSelection` | 4 | Allow selecting multiple files |
| `CanCreateDirectories` | 16 | Allow creating new directories |
| `OverwritePrompt` | 256 | Warn on overwrite (always enabled on macOS) |
| `CreatePrompt` | 512 | Confirm file creation (always enabled on macOS) |
| `ShowHiddenFiles` | 4096 | Show hidden files |
| `RetrieveLink` | 8192 | Resolve shortcuts/symlinks |

---

---

## Desktop Pet Scripting Patterns

### Minimal Desktop Pet Init

```csharp
using Kirurobo;

void Start()
{
    var uwc = UniWindowController.current;
    uwc.isTransparent = true;
    uwc.isTopmost = true;
    uwc.allowDropFiles = true;
    uwc.OnDropFiles += OnFilesDropped;
}
```

### Move Pet Window Programmatically (e.g. walking on taskbar)

```csharp
var uwc = UniWindowController.current;
uwc.windowPosition += new Vector2(speed * Time.deltaTime, 0); // walk horizontally
uwc.windowSize = new Vector2(256, 256); // resize pet window
```

### React to Dropped Files

```csharp
void OnFilesDropped(string[] files)
{
    foreach (var path in files)
        Debug.Log("Pet received: " + path);
}
```

### Monitor Awareness (multi-monitor pet roaming)

```csharp
int monitorCount = UniWindowController.GetMonitorCount();
Rect monitorRect = UniWindowController.GetMonitorRect(0); // main monitor bounds
```

### Toggle Visibility / Interactivity

```csharp
var uwc = UniWindowController.current;
uwc.isClickThrough = true;  // ghost mode: clicks pass through entirely
uwc.alphaValue = 0.5f;      // semi-transparent
```

---

## Input System

Supports both via preprocessor directives (`ENABLE_INPUT_SYSTEM` / `ENABLE_LEGACY_INPUT_MANAGER`), auto-defined by Unity's Active Input Handling setting. This project uses the new InputSystem.

---

## Platform Notes

| | Windows | macOS | Editor |
|:--|:--------|:------|:-------|
| Transparency | Alpha (default) or ColorKey | Alpha only | **Not supported** |
| Click-through | Yes | Yes | Disabled |
| Topmost/Position/Size | Yes | Yes | Yes |
| File drop/dialogs | Yes | Yes | Yes |

- **Windows:** DXGI Flip Mode Swapchain must be off. `ColorKey` mode is better for touch but loses semi-transparency.
- **macOS:** Retina scaling differences handled by `UniWindowMoveHandle`.
- **Editor:** Transparency and click-through don't work — must test in standalone builds.

---

## Limitations

- Transparency **only in standalone builds**
- Single window only (no multi-window)
- `ColorKey` transparency: no semi-transparency, reduced visual quality
- API may change in future versions
