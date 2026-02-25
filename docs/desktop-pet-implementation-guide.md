# Desktop Pet: Bottommost + Minimize Prevention + Taskbar Hiding

> **Reference guide** for a UniWinC desktop pet window configuration.
> All features described here are **implemented and ready to use**.
>
> The pet window will:
> 1. Stay below all windows (desktop level) using **Bottommost Mode**
> 2. Survive Win+D / Win+M using **WM_SYSCOMMAND minimize prevention**
> 3. Hide from taskbar and Alt+Tab using **WS_EX_TOOLWINDOW**
>
> **Platform:** Windows 10/11 only

---

## Table of Contents

1. [Quick Start (C# Usage)](#quick-start)
2. [Feature 1: Bottommost Mode](#feature-1-bottommost-mode)
3. [Feature 2: Minimize Prevention](#feature-2-minimize-prevention)
4. [Feature 3: Hide from Taskbar/Alt+Tab](#feature-3-hide-from-taskbaralt-tab)
5. [C# API Reference](#c-api-reference)
6. [Architecture Overview](#architecture-overview)
7. [Build & Test](#build--test)
8. [Edge Cases & Troubleshooting](#edge-cases--troubleshooting)

---

## Quick Start

All three features are exposed as properties on `UniWindowController.current`:

```csharp
using Kirurobo;
using UnityEngine;

public class DesktopPetInit : MonoBehaviour
{
    void Start()
    {
        var uwc = UniWindowController.current;
        if (uwc == null) return;

        // Core desktop pet setup
        uwc.isTransparent = true;
        uwc.isBottommost = true;

        // Prevent minimize (survives Win+D)
        uwc.preventMinimize = true;

        // Hide from taskbar and Alt+Tab
        uwc.isToolWindow = true;

        // Optional: enable file drop
        uwc.allowDropFiles = true;
        uwc.OnDropFiles += files => Debug.Log($"Dropped: {files[0]}");
    }
}
```

### Toggling Desktop Mode

Switch between "normal window" and "desktop pet" modes at runtime:

```csharp
public void SetDesktopMode(bool enabled)
{
    var uwc = UniWindowController.current;

    uwc.isBottommost = enabled;
    uwc.preventMinimize = enabled;
    uwc.isToolWindow = enabled;

    if (!enabled)
    {
        uwc.isTopmost = false;
    }
}
```

### Closing the Pet (No Taskbar = No Close Button)

Since the pet has no taskbar entry, provide a way to quit:

```csharp
// Right-click context menu, keyboard shortcut, or UI button
public void QuitPet()
{
    Application.Quit();
}
```

A system tray icon is the proper solution for this, but is not yet implemented. For now, a right-click context menu on the pet or a keyboard shortcut (e.g., Ctrl+Q) works.

---

## Feature 1: Bottommost Mode

**Status:** Pre-existing in UniWinC. No modifications were needed.

### What It Does

Sets the window to `HWND_BOTTOM` z-order and continuously enforces it. The pet stays below all normal windows, behind desktop icons.

### C# API

```csharp
uwc.isBottommost = true;
```

### Native Implementation

Two mechanisms work together:

**1. Initial placement** via `SetBottommost()` at `libuniwinc.cpp:856-877`:

```cpp
void UNIWINC_API SetBottommost(const BOOL bBottommost) {
    bIsTopmost_ = FALSE;

    if (hTargetWnd_) {
        SetWindowPos(
            hTargetWnd_,
            (bBottommost ? HWND_BOTTOM : HWND_NOTOPMOST),
            0, 0, 0, 0,
            SWP_NOSIZE | SWP_NOMOVE | SWP_NOOWNERZORDER | SWP_NOACTIVATE
        );
        // ... callback omitted
    }
    bIsBottommost_ = bBottommost;
}
```

**2. Continuous enforcement** via WndProc at `libuniwinc.cpp:1484-1489`:

```cpp
case WM_WINDOWPOSCHANGING:
    if (bIsBottommost_) {
        ((WINDOWPOS*)lParam)->hwndInsertAfter = HWND_BOTTOM;
    }
    break;
```

Every time Windows attempts to reposition the window (focus, click, app activation), the hook forces it back to `HWND_BOTTOM`.

### Limitations (Solved by Features 2 and 3)

- Win+D / Win+M minimizes the window (solved by Feature 2)
- The pet appears in taskbar and Alt+Tab (solved by Feature 3)

---

## Feature 2: Minimize Prevention

**Status:** Implemented. DLL and C# bindings complete.

### What It Does

Blocks all minimize attempts by intercepting `WM_SYSCOMMAND` with `SC_MINIMIZE`. Includes a safety net that auto-restores the window if minimize somehow bypasses the block (e.g., Win+D on some Windows versions).

### C# API

```csharp
uwc.preventMinimize = true;   // enable
uwc.preventMinimize = false;  // disable
bool active = uwc.preventMinimize;  // query
```

### Native Implementation

#### Global state variable

`libuniwinc.cpp:26`:
```cpp
static BOOL bPreventMinimize_ = FALSE;
```

#### Exported functions

`libuniwinc.cpp:884-894`:
```cpp
void UNIWINC_API SetPreventMinimize(const BOOL bEnabled) {
    bPreventMinimize_ = bEnabled;
}

BOOL UNIWINC_API IsPreventMinimize() {
    return bPreventMinimize_;
}
```

#### WM_SYSCOMMAND handler (primary defense)

`libuniwinc.cpp:1491-1496`, inside `customWindowProcedure()`:

```cpp
case WM_SYSCOMMAND:
    // Block minimize when prevention is enabled
    if (bPreventMinimize_ && (wParam & 0xFFF0) == SC_MINIMIZE) {
        return 0;
    }
    break;
```

**Critical detail:** This `return 0` exits the WndProc *before* reaching `CallWindowProc` at line 1533. If this were a `break` instead, the message would still be passed to the original WndProc and the minimize would happen.

**Why `(wParam & 0xFFF0)`:** Windows reserves the low 4 bits of `wParam` for internal use in `WM_SYSCOMMAND`. The actual command ID is in the upper bits. Masking with `0xFFF0` extracts the command.

#### WM_SIZE auto-restore (safety net)

`libuniwinc.cpp:1505-1527`, inside `customWindowProcedure()`:

```cpp
case WM_SIZE:
    switch (wParam)
    {
    case SIZE_RESTORED:
    case SIZE_MAXIMIZED:
        // Run callback
        if (hWindowStyleChangedHandler_ != nullptr) {
            hWindowStyleChangedHandler_((INT32)WindowStateEventType::Resized);
        }
        break;
    case SIZE_MINIMIZED:
        // Auto-restore if minimize prevention is active
        // (safety net for Win+D which may bypass WM_SYSCOMMAND)
        if (bPreventMinimize_) {
            PostMessage(hWnd, WM_SYSCOMMAND, SC_RESTORE, 0);
        }
        // Run callback
        if (hWindowStyleChangedHandler_ != nullptr) {
            hWindowStyleChangedHandler_((INT32)WindowStateEventType::Resized);
        }
        break;
    }
    break;
```

**Why `PostMessage` instead of `ShowWindow`:** Calling `ShowWindow` directly from within WndProc can cause re-entrancy issues. `PostMessage` queues the restore for after the current message is fully processed.

**Why this is needed:** Win+D uses a shell-level "toggle desktop" mechanism that may bypass `WM_SYSCOMMAND` on some Windows versions and directly minimize the window. This fallback detects the resulting `SIZE_MINIMIZED` and immediately queues a restore.

### Header declarations

`libuniwinc.h:89`:
```cpp
UNIWINC_EXPORT BOOL UNIWINC_API IsPreventMinimize();
```

`libuniwinc.h:105`:
```cpp
UNIWINC_EXPORT void UNIWINC_API SetPreventMinimize(const BOOL bEnabled);
```

---

## Feature 3: Hide from Taskbar/Alt+Tab

**Status:** Implemented. DLL and C# bindings complete.

### What It Does

Applies the `WS_EX_TOOLWINDOW` extended window style, which:
- Removes the window from the taskbar
- Removes the window from the Alt+Tab switcher
- Makes Win+D *less likely* to target the window (it primarily targets windows visible in the taskbar/Alt+Tab list)

### C# API

```csharp
uwc.isToolWindow = true;   // enable
uwc.isToolWindow = false;  // disable
bool active = uwc.isToolWindow;  // query
```

### Native Implementation

#### SetToolWindow

`libuniwinc.cpp:900-928`:

```cpp
void UNIWINC_API SetToolWindow(const BOOL bEnabled) {
    if (!hTargetWnd_) return;

    LONG_PTR exStyle = GetWindowLongPtr(hTargetWnd_, GWL_EXSTYLE);

    if (bEnabled) {
        exStyle |= WS_EX_TOOLWINDOW;
        exStyle &= ~WS_EX_APPWINDOW;
    } else {
        exStyle &= ~WS_EX_TOOLWINDOW;
        exStyle |= WS_EX_APPWINDOW;
    }

    SetWindowLongPtr(hTargetWnd_, GWL_EXSTYLE, exStyle);

    // Windows requires a hide+show cycle to update the taskbar
    ShowWindow(hTargetWnd_, SW_HIDE);
    ShowWindow(hTargetWnd_, SW_SHOW);

    // Re-apply bottommost after the show cycle (SW_SHOW can raise the window)
    if (bIsBottommost_) {
        SetWindowPos(
            hTargetWnd_,
            HWND_BOTTOM,
            0, 0, 0, 0,
            SWP_NOSIZE | SWP_NOMOVE | SWP_NOOWNERZORDER | SWP_NOACTIVATE
        );
    }
}
```

**Key details:**

- `WS_EX_APPWINDOW` is explicitly cleared because it forces taskbar visibility even when `WS_EX_TOOLWINDOW` is set. When disabling, `WS_EX_APPWINDOW` is restored.
- The `ShowWindow(SW_HIDE)` + `ShowWindow(SW_SHOW)` cycle is required — Windows does not update the taskbar just from changing extended styles. Without this, the taskbar button remains visible until the window is next hidden/shown.
- After `SW_SHOW`, the window may jump above other windows. The `SetWindowPos(HWND_BOTTOM)` at the end re-applies bottommost positioning.

#### IsToolWindow

`libuniwinc.cpp:934-938`:

```cpp
BOOL UNIWINC_API IsToolWindow() {
    if (!hTargetWnd_) return FALSE;
    LONG_PTR exStyle = GetWindowLongPtr(hTargetWnd_, GWL_EXSTYLE);
    return (exStyle & WS_EX_TOOLWINDOW) != 0;
}
```

This reads the live window style rather than a cached flag, so it always reflects the actual state.

### Header declarations

`libuniwinc.h:90`:
```cpp
UNIWINC_EXPORT BOOL UNIWINC_API IsToolWindow();
```

`libuniwinc.h:106`:
```cpp
UNIWINC_EXPORT void UNIWINC_API SetToolWindow(const BOOL bEnabled);
```

### Flicker Considerations

The `SW_HIDE` / `SW_SHOW` cycle causes a **single-frame flicker**:

- **At startup (no flicker):** If the pet should always be taskbar-hidden, call `SetToolWindow(true)` early — before the first frame renders. The window is not yet visible, so there's nothing to flicker.
- **At runtime (brief flash):** The flicker is ~16ms at 60fps. Acceptable for an infrequent toggle, but avoid calling in a loop.

---

## C# API Reference

### Binding Layers

The implementation follows UniWinC's existing 3-layer pattern:

```
UniWindowController (public API, MonoBehaviour singleton)
    → UniWinCore (internal wrapper, instance methods)
        → LibUniWinC (static P/Invoke declarations)
```

### Layer 1: P/Invoke Declarations

**File:** `UniWinC/Assets/Kirurobo/UniWindowController/Runtime/Scripts/LowLevel/UniWinCore.cs`
**Location:** Inside `protected class LibUniWinC`, lines 129-141

```csharp
[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
public static extern void SetPreventMinimize([MarshalAs(UnmanagedType.U1)] bool bEnabled);

[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
[return: MarshalAs(UnmanagedType.Bool)]
public static extern bool IsPreventMinimize();

[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
public static extern void SetToolWindow([MarshalAs(UnmanagedType.U1)] bool bEnabled);

[DllImport("LibUniWinC", CallingConvention = CallingConvention.Winapi)]
[return: MarshalAs(UnmanagedType.Bool)]
public static extern bool IsToolWindow();
```

**Marshalling notes:**
- `[MarshalAs(UnmanagedType.U1)]` on `bool` parameters matches the native `BOOL` (4-byte int) calling convention via `__stdcall`.
- `[return: MarshalAs(UnmanagedType.Bool)]` ensures the native `BOOL` return is correctly marshalled to C# `bool`.
- `CallingConvention.Winapi` matches the DLL's `__stdcall` convention (`UNIWINC_API`).

### Layer 2: UniWinCore Wrapper Methods

**File:** `UniWinC/Assets/Kirurobo/UniWindowController/Runtime/Scripts/LowLevel/UniWinCore.cs`
**Location:** Inside `#region for Windows only`, lines 826-860

```csharp
public void SetPreventMinimize(bool enabled)
{
    LibUniWinC.SetPreventMinimize(enabled);
}

public bool GetPreventMinimize()
{
    return LibUniWinC.IsPreventMinimize();
}

public void SetToolWindow(bool enabled)
{
    LibUniWinC.SetToolWindow(enabled);
}

public bool GetToolWindow()
{
    return LibUniWinC.IsToolWindow();
}
```

These are thin wrappers. Unlike some other UniWinCore methods (e.g., `EnableTopmost` which also updates `_isTopmost`), these don't cache state — the native side is the source of truth.

### Layer 3: UniWindowController Properties

**File:** `UniWinC/Assets/Kirurobo/UniWindowController/Runtime/Scripts/UniWindowController.cs`
**Location:** After `_isBottommost`, lines 177-193

```csharp
/// <summary>
/// Prevent window from being minimized (Win+D, Win+M, taskbar click)
/// </summary>
public bool preventMinimize
{
    get { return _uniWinCore != null && _uniWinCore.GetPreventMinimize(); }
    set { _uniWinCore?.SetPreventMinimize(value); }
}

/// <summary>
/// Hide window from taskbar and Alt+Tab (WS_EX_TOOLWINDOW)
/// </summary>
public bool isToolWindow
{
    get { return _uniWinCore != null && _uniWinCore.GetToolWindow(); }
    set { _uniWinCore?.SetToolWindow(value); }
}
```

These properties are safe to call before `_uniWinCore` is initialized — the getter returns `false` and the setter is a no-op via null-conditional.

---

## Architecture Overview

### File Map

| File | What was added |
|:-----|:---------------|
| `VisualStudio/LibUniWinC/libuniwinc.cpp:26` | `bPreventMinimize_` global state variable |
| `VisualStudio/LibUniWinC/libuniwinc.cpp:884-894` | `SetPreventMinimize()` / `IsPreventMinimize()` exported functions |
| `VisualStudio/LibUniWinC/libuniwinc.cpp:900-938` | `SetToolWindow()` / `IsToolWindow()` exported functions |
| `VisualStudio/LibUniWinC/libuniwinc.cpp:1491-1496` | `WM_SYSCOMMAND` handler in WndProc |
| `VisualStudio/LibUniWinC/libuniwinc.cpp:1515-1520` | `WM_SIZE` / `SIZE_MINIMIZED` auto-restore in WndProc |
| `VisualStudio/LibUniWinC/libuniwinc.h:89-90` | `IsPreventMinimize()` / `IsToolWindow()` query declarations |
| `VisualStudio/LibUniWinC/libuniwinc.h:105-106` | `SetPreventMinimize()` / `SetToolWindow()` setter declarations |
| `UniWinCore.cs:129-141` | P/Invoke declarations in `LibUniWinC` class |
| `UniWinCore.cs:826-860` | Wrapper methods in `#region for Windows only` |
| `UniWindowController.cs:177-193` | `preventMinimize` and `isToolWindow` public properties |

### Message Flow

```
User presses Win+D
    → Windows sends WM_SYSCOMMAND (SC_MINIMIZE) to window
    → customWindowProcedure() catches it (line 1491)
    → bPreventMinimize_ is TRUE → return 0 (swallowed)
    → Minimize never happens

If WM_SYSCOMMAND is bypassed (some Win+D code paths):
    → Windows minimizes window directly
    → WM_SIZE fires with SIZE_MINIMIZED (line 1515)
    → bPreventMinimize_ is TRUE → PostMessage(SC_RESTORE)
    → Window restores immediately (may flash briefly)
```

```
uwc.isToolWindow = true
    → UniWindowController.isToolWindow.set
    → UniWinCore.SetToolWindow(true)
    → LibUniWinC.SetToolWindow(true) [P/Invoke]
    → Native SetToolWindow():
        1. Read current GWL_EXSTYLE
        2. Set WS_EX_TOOLWINDOW, clear WS_EX_APPWINDOW
        3. Apply via SetWindowLongPtr
        4. SW_HIDE + SW_SHOW (forces taskbar update)
        5. Re-apply HWND_BOTTOM if bottommost
    → Window disappears from taskbar and Alt+Tab
```

---

## Build & Test

### Rebuilding the DLL

1. Open `VisualStudio/LibUniWinC.sln` in Visual Studio
2. Build the `LibUniWinC` project (Release, x64)
3. Copy the output `LibUniWinC.dll` to replace the one in the Unity project:
   - Package location: `Packages/com.kirurobo.uniwinc/Runtime/Plugins/x86_64/LibUniWinC.dll`
   - Or if using the Assets version: `Assets/Kirurobo/UniWindowController/Plugins/x86_64/LibUniWinC.dll`

### Testing Checklist

| Test | Expected Result |
|:-----|:----------------|
| Launch build | Pet appears on desktop, below all windows |
| Click on desktop behind pet | Pet stays at bottom, doesn't come to front |
| Open/close other windows | Pet stays at bottom |
| Press Win+D | Pet does NOT minimize, stays visible |
| Press Win+M | Pet does NOT minimize, stays visible |
| Check taskbar | Pet is NOT shown in taskbar |
| Press Alt+Tab | Pet is NOT in the Alt+Tab list |
| Right-click pet / quit method | Pet closes |
| Click on pet (transparent area) | Click passes through to desktop |
| Click on pet (opaque area) | Pet receives the click |
| Drag pet via handle | Pet moves, stays at bottom after drag |

### Testing Notes

- **Transparency only works in standalone builds**, not in the Unity Editor
- Build with Player Settings: Run in Background = true, DXGI Flip Mode Swapchain = false
- Test on both single and multi-monitor setups if possible
- Test with and without desktop icons to verify the pet renders behind them

---

## Edge Cases & Troubleshooting

### Win+D Still Minimizes the Pet

The two-layer defense (WM_SYSCOMMAND block + WM_SIZE auto-restore) handles most cases. If the pet still minimizes on a specific Windows version, the shell may be using `ShowWindow(SW_MINIMIZE)` directly. Add a `WM_SHOWWINDOW` handler to `customWindowProcedure()`:

```cpp
case WM_SHOWWINDOW:
    if (bPreventMinimize_ && !wParam) {
        // Window is being hidden — cancel it
        return 0;
    }
    break;
```

### Pet Flashes When isToolWindow Is Set

This is the `SW_HIDE` / `SW_SHOW` cycle in `SetToolWindow()`. To avoid it, set `isToolWindow = true` as early as possible — ideally in `Start()` before the first frame renders. If toggling at runtime, the flash is a single frame and generally acceptable.

### Pet Appears Above Other Windows After Drag

`UniWindowMoveHandle` calls `SetWindowPos` during drag. The bottommost enforcement in `WM_WINDOWPOSCHANGING` (line 1484) should catch this. If it doesn't, re-enforce after drag:

```csharp
void OnEndDrag()
{
    var uwc = UniWindowController.current;
    uwc.isBottommost = true;
}
```

### User Can't Find or Close the Pet

Without a taskbar entry, the only ways to interact are:
- Clicking the pet directly
- A global keyboard shortcut (requires separate implementation)
- Task Manager (the process is still visible there)

Provide a visible UI element (right-click menu or close button) so the pet can be quit. A system tray icon can be added later.

### Multiple Monitors

Bottommost mode works across all monitors. The pet can be dragged to any screen and stays at the bottom everywhere. No special handling needed.

### Focus Stealing

With `WS_EX_TOOLWINDOW` and bottommost, the pet should never steal focus. If it does, add `WS_EX_NOACTIVATE` to `SetToolWindow()`:

```cpp
// In SetToolWindow, additionally:
exStyle |= WS_EX_NOACTIVATE;
```

This prevents the window from being activated when clicked. The window won't receive keyboard input, which is fine for a desktop pet.
