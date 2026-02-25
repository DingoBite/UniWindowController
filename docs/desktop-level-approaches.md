# Desktop-Level Window Approaches for UniWinC Desktop Pets

> **Audience:** Developers building desktop pets with UniWindowController (UniWinC)
> **Platform focus:** Windows 10/11 (most techniques are Windows-specific)
> **UniWinC version:** 0.9.4+

## Problem Statement

A desktop pet should:
1. **Stay at desktop level** — below all normal application windows, sitting "on" the desktop/taskbar
2. **Never minimize** — remain visible even when the user presses Win+D ("Show Desktop") or clicks "Show Desktop" on the taskbar

UniWinC provides building blocks for this, but no single toggle solves both requirements. This document describes five approaches, their trade-offs, and recommended combinations.

---

## Quick Comparison

| Approach | Desktop-Level | Survives Win+D | Complexity | Requires DLL Changes |
|:---------|:-------------:|:---------------:|:----------:|:--------------------:|
| A. Bottommost Mode | Yes | No | None | No |
| B. Wallpaper Mode | Yes | Yes | Low | C# binding only |
| C. WM_SYSCOMMAND Block | No | Yes | Medium | Yes |
| D. Polling + Auto-Restore | Partial | Partial | Low | No |
| E. WS_EX_TOOLWINDOW + Tray | No | Partial | Medium | Yes |

---

## Approach A: Bottommost Mode (Existing)

### What It Does

Sets the window to `HWND_BOTTOM` z-order and enforces it on every `WM_WINDOWPOSCHANGING` message. The window stays below all normal windows.

### How to Use

```csharp
var uwc = UniWindowController.current;
uwc.isBottommost = true;
```

That's it — this is already fully exposed in the C# API.

### How It Works (Native)

In the DLL's subclassed WndProc (`libuniwinc.cpp:1423-1428`):

```cpp
case WM_WINDOWPOSCHANGING:
    if (bIsBottommost_) {
        ((WINDOWPOS*)lParam)->hwndInsertAfter = HWND_BOTTOM;
    }
    break;
```

Every time Windows is about to reposition the window, the hook forces it to `HWND_BOTTOM`. This is robust — other windows activating, user clicks, focus changes — the pet stays at the bottom.

### Limitations

- **Does NOT survive Win+D.** The "Show Desktop" command minimizes all windows (including bottommost ones). Once minimized, the pet disappears until manually restored.
- **Does NOT survive Win+M.** Same issue — "Minimize All" minimizes the pet.
- The pet sits *below* the desktop icons layer. If the user has desktop icons, the pet will be behind them.

### When to Use

Good enough if you don't need Win+D resilience, or as one part of a combined strategy.

---

## Approach B: Wallpaper Mode / SetBackground (Existing Native, Needs C# Exposure)

### What It Does

Reparents the Unity window as a child of the desktop's `WorkerW` window — the same layer that displays the wallpaper. The pet becomes part of the desktop itself. Win+D has no effect because the window is no longer a top-level window that can be minimized.

### Current Status

The native DLL **already implements this** (`libuniwinc.cpp:884-919`), exported as `SetBackground()`. However, **it is NOT exposed in the C# UniWindowController class.** You must add the binding yourself.

### How It Works (Native)

1. **Desktop window discovery** (`libuniwinc.cpp:209-234`): Enumerates top-level windows looking for `WorkerW` or `Progman` class names, then finds the one *after* the `SHELLDLL_DefView` child — this is the wallpaper layer.

   ```cpp
   // Pseudocode of the search:
   // 1. Find WorkerW/Progman that contains SHELLDLL_DefView
   // 2. The NEXT WorkerW enumerated is the target desktop window
   ```

2. **Reparenting** (`libuniwinc.cpp:893`):
   ```cpp
   SetParent(hTargetWnd_, hDesktopWnd_);
   ```

3. **Disabling** (`libuniwinc.cpp:906`):
   ```cpp
   SetParent(hTargetWnd_, hParentWnd_);  // restore original parent
   ```

### Adding the C# Binding

In your `UniWinCoreWindowsApi.cs` (or equivalent native interop file), add:

```csharp
[DllImport("LibUniWinC")]
private static extern void SetBackground(bool isBackground);

[DllImport("LibUniWinC")]
private static extern bool IsBackground();
```

Then expose it in `UniWindowController.cs`:

```csharp
public bool isBackground
{
    get { return _uniWinCore != null && _uniWinCore.IsBackground(); }
    set { _uniWinCore?.SetBackground(value); }
}
```

### Usage

```csharp
var uwc = UniWindowController.current;
uwc.isTransparent = true;
uwc.isBackground = true;  // (after adding the binding)
```

### Limitations

- **No input events from Unity's event system.** As a child of the desktop window, the Unity window may not receive normal mouse input. You'll need alternative input handling (global mouse hooks or polling `GetCursorPos`).
- **Coordinate system changes.** The window's coordinates become relative to the desktop window, not the screen. `windowPosition` may not behave as expected.
- **Desktop icon interaction.** The pet renders in the wallpaper layer, *behind* desktop icons.
- **Multi-monitor quirks.** The WorkerW window may span all monitors or only the primary, depending on Windows version and wallpaper settings.
- **Windows updates can break this.** The WorkerW/Progman/SHELLDLL_DefView hierarchy is undocumented and has changed across Windows versions. This is the same technique used by Wallpaper Engine and Lively Wallpaper, but it's fundamentally fragile.

### When to Use

Best approach for a true "lives on the desktop" pet that must survive Win+D. Accept the input handling complexity as a trade-off.

---

## Approach C: WM_SYSCOMMAND Minimize Prevention (DLL Modification)

### What It Does

Intercepts `WM_SYSCOMMAND` with `SC_MINIMIZE` in the DLL's WndProc and blocks it. The window simply refuses to minimize, regardless of the source (Win+D, Win+M, taskbar click, etc.).

### Implementation

Add this case to the WndProc in `libuniwinc.cpp` (around line 1428, near the other message handlers):

```cpp
case WM_SYSCOMMAND:
    if (bPreventMinimize_ && (wParam & 0xFFF0) == SC_MINIMIZE) {
        return 0;  // swallow the minimize command
    }
    break;
```

Add the flag and exported functions:

```cpp
// Global state
static BOOL bPreventMinimize_ = FALSE;

// Exported API
void UNIWINC_API SetPreventMinimize(const BOOL bEnabled) {
    bPreventMinimize_ = bEnabled;
}

BOOL UNIWINC_API IsPreventMinimize() {
    return bPreventMinimize_;
}
```

**Important:** The `return 0` must happen *before* `CallWindowProc`. This means you need to return early rather than break:

```cpp
case WM_SYSCOMMAND:
    if (bPreventMinimize_ && (wParam & 0xFFF0) == SC_MINIMIZE) {
        return 0;  // do NOT pass to CallWindowProc
    }
    // fall through to CallWindowProc for other syscommands
    break;
```

### C# Binding

```csharp
[DllImport("LibUniWinC")]
private static extern void SetPreventMinimize(bool enabled);

[DllImport("LibUniWinC")]
private static extern bool IsPreventMinimize();
```

### Limitations

- **Win+D is special.** Win+D doesn't just send `SC_MINIMIZE` to each window — it uses a shell-level "toggle desktop" mechanism. Blocking `WM_SYSCOMMAND` may not catch all code paths that Win+D uses. You may also need to handle `WM_SIZE` with `SIZE_MINIMIZED` and call `ShowWindow(SW_RESTORE)` immediately.
- **User confusion.** The window cannot be minimized at all — including from the taskbar button or title bar. Users may find this frustrating. Consider only enabling this when the pet is in "desktop mode."
- **Does not place the window at desktop level.** This only prevents minimize — combine with Approach A (bottommost) or B (wallpaper) for full desktop-level behavior.

### When to Use

As a complement to Approach A (bottommost). Prevents the main weakness of bottommost mode (minimize via Win+D).

---

## Approach D: Polling + Auto-Restore (C#-Only Workaround)

### What It Does

Periodically checks if the window has been minimized and immediately restores it. No DLL modifications needed — works entirely in C# using the existing `IsMinimized()` native API (which is exported but not currently bound in C#).

### Implementation

First, add the missing C# binding for `IsMinimized`:

```csharp
// In your native interop class
[DllImport("LibUniWinC")]
private static extern bool IsMinimized();
```

Then, in a MonoBehaviour:

```csharp
using System.Runtime.InteropServices;
using UnityEngine;
using Kirurobo;

public class DesktopPetKeepAlive : MonoBehaviour
{
    [DllImport("user32.dll")]
    private static extern bool ShowWindow(System.IntPtr hWnd, int nCmdShow);

    [DllImport("user32.dll")]
    private static extern System.IntPtr GetActiveWindow();

    private const int SW_RESTORE = 9;

    // Alternatively, use UniWinC's IsMinimized if you add the binding
    [DllImport("LibUniWinC")]
    private static extern bool IsMinimized();

    private float checkInterval = 0.1f;  // 100ms polling
    private float timer;

    void Update()
    {
        timer += Time.unscaledDeltaTime;
        if (timer >= checkInterval)
        {
            timer = 0f;
            if (IsMinimized())
            {
                // Restore the window immediately
                ShowWindow(GetActiveWindow(), SW_RESTORE);

                // Re-apply bottommost if using Approach A
                var uwc = UniWindowController.current;
                if (uwc != null && uwc.isBottommost)
                {
                    uwc.isBottommost = true;  // re-enforce
                }
            }
        }
    }
}
```

### Limitations

- **Visible flicker.** There's a brief moment where the window is minimized before being restored. The user may see the pet disappear and reappear — typically a single-frame flash but noticeable.
- **`Update()` may not run while minimized.** Unity may pause or reduce update frequency for minimized windows. Set **Run in Background: true** in Player Settings (should already be set for desktop pets). Even so, there may be a delay.
- **Race condition with Win+D.** Win+D toggles desktop visibility. If the pet auto-restores, Win+D might enter a fight loop — repeatedly trying to minimize while the pet restores. Workaround: add a cooldown or detect rapid minimize cycles.
- **Not suitable as sole solution.** Best used as a fallback safety net alongside other approaches.

### Alternative: OnStateChanged Event

Instead of polling, use the `OnStateChanged` event which fires on `WM_SIZE`:

```csharp
void Start()
{
    UniWindowController.current.OnStateChanged += OnWindowStateChanged;
}

void OnWindowStateChanged(UniWindowController.WindowStateEventType type)
{
    if (type == UniWindowController.WindowStateEventType.Resized)
    {
        // Check if minimized and restore
        // Note: the callback fires but doesn't distinguish minimize from other resizes
    }
}
```

The event-based approach avoids polling overhead but has the same flicker and Unity-pausing caveats.

### When to Use

Quick workaround when you can't modify the DLL. Acceptable for prototyping but not ideal for production.

---

## Approach E: WS_EX_TOOLWINDOW + Tray Icon (Hide from Taskbar/Alt+Tab)

### What It Does

Adds the `WS_EX_TOOLWINDOW` extended window style, which:
- Removes the window from the taskbar
- Removes the window from the Alt+Tab switcher
- Makes Win+D *ignore* the window (it only targets windows visible in the taskbar/Alt+Tab list)

A system tray icon provides an alternative way to interact with/close the pet.

### Implementation (DLL)

Add to `libuniwinc.cpp`:

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

    // Force Windows to re-evaluate the window
    ShowWindow(hTargetWnd_, SW_HIDE);
    ShowWindow(hTargetWnd_, SW_SHOW);
}
```

### Tray Icon (C# Side)

UniWinC does not include tray icon support. Options:
- **P/Invoke `Shell_NotifyIcon`** directly (complex but no dependencies)
- **Use a managed library** like [H.NotifyIcon](https://github.com/HadrielWonda/H.NotifyIcon) (Unity-compatible with some effort)
- **Minimal tray icon** for right-click quit menu only

### Limitations

- **WS_EX_TOOLWINDOW changes the window frame.** Tool windows have a smaller title bar. With a transparent borderless Unity window this shouldn't matter, but verify visually.
- **The Show/Hide flicker.** Changing `WS_EX_TOOLWINDOW` requires hiding and re-showing the window for the taskbar to update. This causes a brief flicker.
- **No taskbar presence.** Users can't find the pet in the taskbar, which may be confusing. The tray icon is essential for discoverability.
- **Win+D behavior is version-dependent.** While `WS_EX_TOOLWINDOW` windows are generally excluded from Win+D, this is not guaranteed across all Windows versions.
- **Does NOT place the window at desktop level.** Still needs Approach A or B for z-ordering.

### When to Use

When you want the pet to be "invisible" to window management (taskbar, Alt+Tab, Win+D). Combine with Approach A for desktop-level placement.

---

## Recommended Combinations

### Production Desktop Pet (Best Overall)

**Approach B (Wallpaper Mode) + custom input handling**

- Pet lives in the desktop layer
- Immune to Win+D, Win+M, and all minimize commands
- Requires solving the input problem (global mouse hook or cursor polling)
- Most robust but most complex to implement

### Simple Desktop Pet (Easy to Implement)

**Approach A (Bottommost) + Approach C (Minimize Prevention)**

```csharp
var uwc = UniWindowController.current;
uwc.isTransparent = true;
uwc.isBottommost = true;
SetPreventMinimize(true);  // your added native function
```

- Pet stays below all windows
- Refuses to minimize
- Requires DLL modification for Approach C
- May need additional Win+D handling if SC_MINIMIZE blocking is insufficient

### No-DLL-Modification Pet (Quick Start)

**Approach A (Bottommost) + Approach D (Auto-Restore)**

```csharp
var uwc = UniWindowController.current;
uwc.isTransparent = true;
uwc.isBottommost = true;
// Add DesktopPetKeepAlive MonoBehaviour to the scene
```

- Works with unmodified UniWinC
- Slight flicker on Win+D but recovers automatically
- Good for prototyping

### Stealth Pet (Hidden from Task Management)

**Approach A (Bottommost) + Approach C (Minimize Prevention) + Approach E (ToolWindow)**

- Pet is invisible to taskbar, Alt+Tab, and Win+D
- Stays at desktop level
- Tray icon for user control
- Most modification work, but most "pet-like" behavior

---

## Native Code Reference

Key locations in `VisualStudio/LibUniWinC/libuniwinc.cpp`:

| Feature | Location | Status |
|:--------|:---------|:-------|
| `SetBackground()` (wallpaper mode) | Lines 884-919 | Native only, no C# binding |
| Desktop window discovery (WorkerW/Progman) | Lines 209-234 | Internal to DLL |
| `WM_WINDOWPOSCHANGING` bottommost enforcement | Lines 1423-1428 | Fully working |
| `WM_SIZE` minimize/maximize detection | Lines 1437-1449 | Fires callback but doesn't prevent |
| `IsMinimized()` (via `IsIconic`) | Lines 532-534 | Exported, no C# binding |
| `SetBottommost()` | Exported in header line 101 | Fully working, exposed in C# |

Header exports: `VisualStudio/LibUniWinC/libuniwinc.h`

---

## Platform Notes

- All approaches described here are **Windows-specific**. macOS has different window management (no `HWND_BOTTOM` equivalent, different "desktop level" via `NSWindow.level`).
- The WorkerW/Progman technique (Approach B) is undocumented Windows behavior. It works on Windows 10 and 11 as of early 2026 but could break in future updates.
- `WS_EX_TOOLWINDOW` is a stable, documented Win32 API.
- Win+D behavior is ultimately controlled by the Windows Shell (`explorer.exe`) and its exact mechanism is not fully documented.
