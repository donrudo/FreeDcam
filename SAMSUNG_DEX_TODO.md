# Samsung DeX Compatibility TODO

## Overview
This document outlines the necessary changes to improve FreeDcam compatibility when running on Samsung S23 devices while connected to an external display through Samsung DeX.

**Current Issues:**
- App is locked to landscape orientation, preventing proper DeX window management
- No resizeable activity support (critical for DeX multi-window mode)
- Display size calculations don't account for DeX secondary display
- Camera preview may not scale properly in DeX desktop mode
- No handling of display mode transitions (phone ↔ DeX)
- Missing DeX-specific UI optimizations

**Target Device:** Samsung Galaxy S23 (SM-S911B/DS and variants)
**Android Version:** Android 13+ with One UI 5.1+
**DeX Version:** Samsung DeX 4.0+

---

## Priority Tasks

### 🔴 Critical (Must-Have)

#### 1. Enable Resizeable Activity Support
**File:** `/app/src/main/AndroidManifest.xml`

**Current State:**
```xml
<activity
    android:name="freed.cam.ActivityFreeDcamMain"
    android:screenOrientation="landscape"
    ...>
```

**Required Changes:**
- [ ] Add `android:resizeableActivity="true"` to main camera activities
- [ ] Remove or make conditional `android:screenOrientation="landscape"`
- [ ] Add support for freeform windows and split-screen mode
- [ ] Test with different window sizes (minimum 220dp x 220dp)

**Implementation Notes:**
```xml
<activity
    android:name="freed.cam.ActivityFreeDcamMain"
    android:resizeableActivity="true"
    android:supportsPictureInPicture="false"
    android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|touchscreen|layoutDirection|density|screenSize"
    ...>
    <layout android:defaultWidth="720dp"
            android:defaultHeight="480dp"
            android:gravity="center"
            android:minWidth="400dp"
            android:minHeight="300dp" />
</activity>
```

**Files to Modify:**
- `/app/src/main/AndroidManifest.xml` (lines 58-68, 70-88)

---

#### 2. Implement Display Mode Detection
**New File:** `/app/src/main/java/freed/utils/DexManager.java`

**Requirements:**
- [ ] Create DexManager class to detect and monitor DeX mode
- [ ] Detect when device enters/exits DeX mode
- [ ] Track primary vs secondary display
- [ ] Provide callbacks for display mode changes

**Implementation:**
```java
public class DexManager implements LifecycleObserver {

    public enum DisplayMode {
        PHONE,           // Normal phone mode
        DEX_STANDALONE,  // DeX on external display
        DEX_DUAL,        // DeX with phone screen active
        UNKNOWN
    }

    /**
     * Detect if running in Samsung DeX mode
     * Uses Samsung's SDK if available, falls back to display detection
     */
    public boolean isInDexMode(Context context) {
        // Method 1: Check Samsung-specific configuration
        Configuration config = context.getResources().getConfiguration();
        try {
            Class<?> configClass = config.getClass();
            int samsungDexMode = configClass.getField("SEM_DESKTOP_MODE_ENABLED").getInt(configClass);
            if (config.semDesktopModeEnabled == samsungDexMode) {
                return true;
            }
        } catch (Exception e) {
            // Samsung DeX SDK not available
        }

        // Method 2: Check display characteristics
        DisplayManager dm = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);
        Display[] displays = dm.getDisplays();
        return displays.length > 1; // Simplified check
    }

    public DisplayMode getCurrentDisplayMode(Context context);
    public Display getActiveDisplay(Context context);
    public void registerDisplayChangeListener(DisplayChangeListener listener);
}
```

**Integration Points:**
- `/app/src/main/java/freed/cam/ActivityFreeDcamMain.java`
- `/app/src/main/java/freed/FreedApplication.java`
- Create Hilt module: `/app/src/main/java/hilt/DexManagerModule.java`

---

#### 3. Dynamic Display Size Handling
**File:** `/app/src/main/java/freed/utils/DisplayUtil.java`

**Current Issues:**
- Uses `wm.getDefaultDisplay()` which may return phone display in DeX mode
- Doesn't account for multi-display scenarios
- Hardcoded orientation swapping

**Required Changes:**
- [ ] Update `getDisplaySize()` to detect and use correct display in DeX mode
- [ ] Add method to get display by ID or type
- [ ] Handle dynamic display changes (rotation, resize)
- [ ] Cache display metrics with invalidation on configuration change

**New Methods Needed:**
```java
public class DisplayUtil {

    /**
     * Get size of the display where the activity is currently shown
     * Handles DeX multi-display scenarios
     */
    public static Point getActivityDisplaySize(Activity activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            WindowMetrics metrics = activity.getWindowManager().getCurrentWindowMetrics();
            Rect bounds = metrics.getBounds();
            return new Point(bounds.width(), bounds.height());
        } else {
            // Fallback to existing logic
            return getDisplaySize();
        }
    }

    /**
     * Get all available displays (for DeX dual-screen)
     */
    public static List<Display> getAvailableDisplays(Context context) {
        DisplayManager dm = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);
        return Arrays.asList(dm.getDisplays());
    }

    /**
     * Check if activity is on external display
     */
    public static boolean isOnExternalDisplay(Activity activity) {
        // Implementation
    }
}
```

**Files to Modify:**
- `/app/src/main/java/freed/utils/DisplayUtil.java`
- All files using `DisplayUtil.getDisplaySize()` (15+ files)

---

#### 4. Camera Preview Scaling for DeX
**Files:**
- `/app/src/main/java/freed/cam/previewpostprocessing/NormalPreview.java`
- `/app/src/main/java/freed/cam/previewpostprocessing/OpenGLPreview.java`

**Current Issues:**
- Preview assumes full-screen landscape orientation
- Aspect ratio calculations may break in resized windows
- SurfaceTexture size may not match window size in DeX

**Required Changes:**
- [ ] Implement adaptive preview scaling based on window size
- [ ] Handle aspect ratio mismatches gracefully
- [ ] Update preview when window is resized in DeX mode
- [ ] Add letterboxing/pillarboxing for extreme aspect ratios
- [ ] Optimize preview resolution for external display (may be 1440p or 4K)

**Implementation Approach:**
```java
public class Preview {

    private void updatePreviewSize(int windowWidth, int windowHeight) {
        // Get optimal preview size from camera
        Size previewSize = getOptimalPreviewSize(windowWidth, windowHeight);

        // Calculate scaling to fit window while maintaining aspect ratio
        float previewRatio = (float) previewSize.getWidth() / previewSize.getHeight();
        float windowRatio = (float) windowWidth / windowHeight;

        int scaledWidth, scaledHeight;
        if (previewRatio > windowRatio) {
            // Preview is wider - fit to width
            scaledWidth = windowWidth;
            scaledHeight = (int) (windowWidth / previewRatio);
        } else {
            // Preview is taller - fit to height
            scaledHeight = windowHeight;
            scaledWidth = (int) (windowHeight * previewRatio);
        }

        // Apply to SurfaceTexture
        surfaceTexture.setDefaultBufferSize(scaledWidth, scaledHeight);
    }

    private Size getOptimalPreviewSize(int targetWidth, int targetHeight) {
        // Enhanced to consider DeX higher resolutions
        // Prefer 1080p or higher for external displays
    }
}
```

**Files to Modify:**
- `/app/src/main/java/freed/cam/previewpostprocessing/NormalPreview.java`
- `/app/src/main/java/freed/cam/previewpostprocessing/OpenGLPreview.java`
- `/app/src/main/java/freed/cam/previewpostprocessing/PreviewController.java`

---

#### 5. Handle Configuration Changes for DeX
**File:** `/app/src/main/java/freed/cam/ActivityFreeDcamMain.java`

**Required Changes:**
- [ ] Override `onConfigurationChanged()` to detect DeX mode changes
- [ ] Pause/resume camera when switching between phone and DeX
- [ ] Update UI layout for different display modes
- [ ] Handle display density changes (phone vs monitor DPI)

**Implementation:**
```java
@Override
public void onConfigurationChanged(@NonNull Configuration newConfig) {
    super.onConfigurationChanged(newConfig);

    boolean wasInDex = isCurrentlyInDexMode;
    boolean nowInDex = dexManager.isInDexMode(this);

    if (wasInDex != nowInDex) {
        // Display mode changed
        handleDisplayModeChange(nowInDex);
    }

    // Handle orientation changes (still relevant in DeX)
    if (newConfig.orientation != currentOrientation) {
        updateUIForOrientation(newConfig.orientation);
    }

    // Handle display size/density changes
    updatePreviewForDisplayChanges();
}

private void handleDisplayModeChange(boolean enteredDex) {
    if (enteredDex) {
        // Entering DeX mode
        Log.d(TAG, "Entered Samsung DeX mode");

        // May need to restart camera with different resolution
        cameraApiManager.onPause();
        updateCameraConfigForDex();
        cameraApiManager.onResume();

    } else {
        // Exiting DeX mode (back to phone)
        Log.d(TAG, "Exited Samsung DeX mode");

        cameraApiManager.onPause();
        updateCameraConfigForPhone();
        cameraApiManager.onResume();
    }
}
```

**Files to Modify:**
- `/app/src/main/java/freed/cam/ActivityFreeDcamMain.java`

---

### 🟡 High Priority (Should-Have)

#### 6. Optimize Camera Parameters for DeX
**Files:**
- `/app/src/main/java/freed/cam/apis/camera2/parameters/ParameterHandlerApi2.java`
- `/app/src/main/java/freed/cam/apis/camera1/parameters/ParametersHandler.java`

**Requirements:**
- [ ] Select higher preview resolutions for external displays (1080p/1440p/4K)
- [ ] Adjust video recording resolution based on display capabilities
- [ ] Consider monitor refresh rates for preview frame rate
- [ ] Handle Samsung-specific camera features in DeX mode

**Samsung S23 Specific Considerations:**
- Main camera: 50MP (8000x6000), supports up to 8K video
- DeX mode typically outputs 1920x1080 or 2560x1440
- Preview should match or exceed display resolution for quality

**Implementation:**
```java
public class ParameterHandlerApi2 {

    private Size selectPreviewSize(boolean isDexMode) {
        if (isDexMode) {
            // Prefer higher resolutions for external display
            // S23 supports up to 1920x1080 preview
            return getBestPreviewSize(1920, 1080);
        } else {
            // Phone display (typically 1080x2340 for S23)
            return getBestPreviewSize(1080, 2340);
        }
    }

    private void configureCameraForDex() {
        // Use Samsung-specific keys if available
        if (useSamsungExtensions) {
            // Check camera2_hidden_keys/samsung/ for available keys
        }
    }
}
```

**Files to Modify:**
- `/app/src/main/java/freed/cam/apis/camera2/parameters/ParameterHandlerApi2.java`
- `/app/src/main/java/freed/cam/apis/camera1/parameters/ParametersHandler.java`

---

#### 7. UI Layout Adaptations for DeX
**Files:**
- `/app/src/main/res/layout/activity_freedcam_main.xml` (if exists)
- `/app/src/main/java/freed/cam/ui/themenextgen/`

**Requirements:**
- [ ] Create alternative layouts for DeX mode (res/layout-large, res/layout-xlarge)
- [ ] Adjust control sizes for mouse/keyboard interaction
- [ ] Ensure touch targets are appropriate for larger displays
- [ ] Add keyboard shortcuts for common operations
- [ ] Consider desktop-style UI patterns (toolbars, menus)

**DeX UI Guidelines:**
- Minimum touch target: 48dp x 48dp (even with mouse, for accessibility)
- Use standard desktop patterns (top menu bar, etc.)
- Support keyboard navigation
- Show tooltips on hover
- Larger text for readability on external display

**New Layouts:**
- `/app/src/main/res/layout-w960dp/` - For DeX desktop mode
- Update existing fragments to be responsive

**Files to Create/Modify:**
- Create new layout variants for large screens
- `/app/src/main/java/freed/cam/ui/themenextgen/fragment/` - Update fragments
- `/app/src/main/java/freed/cam/ui/ThemeManager.java` - Add DeX theme variant

---

#### 8. Multi-Window and Split-Screen Support
**File:** `/app/src/main/AndroidManifest.xml`

**Requirements:**
- [ ] Test and fix issues in split-screen mode
- [ ] Handle partial screen coverage gracefully
- [ ] Support multi-resume (Android 10+) for multi-window
- [ ] Ensure camera releases properly when app loses focus in multi-window

**Manifest Additions:**
```xml
<meta-data
    android:name="android.allow_multiple_instances"
    android:value="false" />
<!-- Only one instance of camera app should run -->

<meta-data
    android:name="android.window.PROPERTY_ACTIVITY_EMBEDDING_ALLOW_SYSTEM_OVERRIDE"
    android:value="true" />
<!-- Allow system to control embedding -->
```

**Activity Lifecycle Updates:**
```java
// In ActivityFreeDcamMain.java
@Override
public void onMultiWindowModeChanged(boolean isInMultiWindowMode, Configuration newConfig) {
    super.onMultiWindowModeChanged(isInMultiWindowMode, newConfig);

    if (isInMultiWindowMode) {
        // Reduce camera performance if needed
        // Adjust UI for smaller window
        handleMultiWindowMode();
    } else {
        // Restore full performance
        handleFullScreenMode();
    }
}
```

**Files to Modify:**
- `/app/src/main/AndroidManifest.xml`
- `/app/src/main/java/freed/cam/ActivityFreeDcamMain.java`

---

#### 9. Samsung DeX-Specific Camera Features
**Files:**
- `/app/src/main/java/camera2_hidden_keys/samsung/CaptureRequestSamsung.java`
- `/app/src/main/java/camera2_hidden_keys/samsung/CameraCharacteristicsSamsung.java`

**Requirements:**
- [ ] Research S23-specific camera extensions available in DeX mode
- [ ] Test if manual controls work differently in DeX
- [ ] Check for DeX-specific camera restrictions
- [ ] Verify RAW capture compatibility in DeX mode

**Samsung S23 Camera Specs:**
- Triple camera: 50MP main, 12MP ultrawide, 10MP 3x telephoto
- Supports Expert RAW mode
- 8K video @ 24/30fps, 4K @ 60fps
- Director's View and other Samsung features

**Investigation Needed:**
- [ ] Test if all camera IDs are accessible in DeX mode
- [ ] Verify multi-camera support (wide, ultra-wide, telephoto)
- [ ] Check if Samsung's camera extensions (night mode, etc.) work
- [ ] Test performance of RAW/DNG capture on external display

**Files to Review:**
- `/app/src/main/java/camera2_hidden_keys/samsung/`
- `/app/src/main/java/freed/cam/apis/featuredetector/camera2/Camera2FeatureDetectorTask.java`

---

### 🟢 Medium Priority (Nice-to-Have)

#### 10. Performance Optimization for External Displays
**Multiple Files**

**Requirements:**
- [ ] Profile app performance in DeX mode
- [ ] Optimize OpenGL preview rendering for higher resolutions
- [ ] Reduce memory usage for high-res preview buffers
- [ ] Consider GPU differences when rendering to external display

**Specific Optimizations:**
```java
// In OpenGLPreview.java
private void optimizeForDexMode() {
    // Use more efficient textures for high-res displays
    // Enable GPU acceleration features
    // Reduce unnecessary rendering passes
}

// Memory management
private void adjustBufferSizesForDisplay(DisplayMode mode) {
    if (mode == DisplayMode.DEX_STANDALONE) {
        // External display - can use larger buffers
        maxBufferSize = 8 * 1024 * 1024; // 8MB
    } else {
        // Phone display
        maxBufferSize = 4 * 1024 * 1024; // 4MB
    }
}
```

**Files to Optimize:**
- `/app/src/main/java/freed/cam/previewpostprocessing/OpenGLPreview.java`
- `/app/src/main/java/freed/gl/` - OpenGL rendering pipeline
- `/app/src/main/java/freed/cam/histogram/` - Histogram processing

---

#### 11. Keyboard and Mouse Input Support
**New File:** `/app/src/main/java/freed/utils/InputDeviceManager.java`

**Requirements:**
- [ ] Add keyboard shortcuts (Space = capture, R = record, etc.)
- [ ] Handle mouse scroll for zoom
- [ ] Right-click context menus
- [ ] Keyboard navigation through settings

**Keyboard Shortcuts:**
- `Space` - Take photo
- `R` - Start/stop video recording
- `+/-` - Zoom in/out
- `Tab` - Navigate controls
- `F` - Toggle focus mode
- `M` - Switch camera mode
- `I` - Toggle ISO mode
- `S` - Toggle shutter mode

**Implementation:**
```java
@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
    // Handle keyboard shortcuts in DeX mode
    if (dexManager.isInDexMode(this)) {
        switch (keyCode) {
            case KeyEvent.KEYCODE_SPACE:
                capturePhoto();
                return true;
            case KeyEvent.KEYCODE_R:
                toggleRecording();
                return true;
            // ... more shortcuts
        }
    }
    return super.onKeyDown(keyCode, event);
}

@Override
public boolean onGenericMotionEvent(MotionEvent event) {
    // Handle mouse scroll for zoom
    if (event.isFromSource(InputDevice.SOURCE_MOUSE)) {
        float scroll = event.getAxisValue(MotionEvent.AXIS_VSCROLL);
        if (scroll != 0) {
            adjustZoom(scroll);
            return true;
        }
    }
    return super.onGenericMotionEvent(event);
}
```

**Files to Modify:**
- `/app/src/main/java/freed/cam/ActivityFreeDcamMain.java`
- Create `/app/src/main/java/freed/utils/InputDeviceManager.java`

---

#### 12. External Display Quality Settings
**File:** `/app/src/main/java/freed/settings/SettingKeys.java`

**Requirements:**
- [ ] Add setting for DeX preview quality (high/balanced/performance)
- [ ] Option to disable effects in DeX mode for performance
- [ ] Setting for preferred display when DeX is active
- [ ] Auto-switch camera API based on DeX mode

**New Settings:**
```java
public class SettingKeys {
    // DeX-specific settings
    public static final Key<String> DEX_PREVIEW_QUALITY =
        new Key<>("dex_preview_quality", "high"); // high/balanced/performance

    public static final Key<Boolean> DEX_USE_EXTERNAL_DISPLAY =
        new Key<>("dex_use_external_display", true);

    public static final Key<Boolean> DEX_DISABLE_EFFECTS =
        new Key<>("dex_disable_effects", false);

    public static final Key<String> DEX_PREFERRED_CAMERA_API =
        new Key<>("dex_preferred_camera_api", "auto"); // auto/camera1/camera2
}
```

**Files to Modify:**
- `/app/src/main/java/freed/settings/SettingKeys.java`
- `/app/src/main/java/freed/settings/SettingsManager.java`
- Add UI in settings fragment

---

#### 13. Logging and Debugging for DeX
**File:** `/app/src/main/java/freed/utils/Log.java`

**Requirements:**
- [ ] Add DeX-specific log tags
- [ ] Log display mode transitions
- [ ] Log window size changes
- [ ] Log camera configuration changes in DeX

**Enhanced Logging:**
```java
public class Log {
    private static final String TAG_DEX = "FreeDcam_DeX";

    public static void logDexState(Context context, String message) {
        if (BuildConfig.DEBUG) {
            // Log current DeX state with message
            DisplayManager dm = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);
            Display[] displays = dm.getDisplays();

            d(TAG_DEX, message + " | Displays: " + displays.length +
              " | Mode: " + getCurrentDexMode(context));
        }
    }
}
```

**Files to Modify:**
- `/app/src/main/java/freed/utils/Log.java`

---

### 🔵 Low Priority (Future Enhancements)

#### 14. Desktop-Style Menu System
- [ ] Add top menu bar for DeX mode
- [ ] File menu (New, Open, Settings, Exit)
- [ ] View menu (Toggle controls, Full screen)
- [ ] Help menu

#### 15. Drag-and-Drop Support
- [ ] Accept images dropped into app window
- [ ] Drag photos from app to other apps/desktop

#### 16. Multiple Instance Support (Optional)
- [ ] Allow multiple app windows in DeX (advanced)
- [ ] Each window with different camera

#### 17. DeX Touchpad Gestures
- [ ] Two-finger pinch for zoom
- [ ] Swipe gestures for switching modes

---

## Testing Checklist

### Samsung S23 + DeX Testing

#### Setup
- [ ] Samsung Galaxy S23 (SM-S911B, SM-S911U, or SM-S911N)
- [ ] USB-C to HDMI adapter or DeX Station/Pad
- [ ] External monitor (test at 1080p and 1440p)
- [ ] Bluetooth keyboard and mouse (optional)
- [ ] One UI 5.1 or later

#### Test Cases

**Basic Functionality:**
- [ ] App launches in DeX mode
- [ ] App window can be resized
- [ ] App works in split-screen mode
- [ ] Camera preview displays correctly
- [ ] Preview scales with window resize
- [ ] Photo capture works
- [ ] Video recording works
- [ ] Settings are accessible and functional

**Display Transitions:**
- [ ] Connect to DeX while app is running → app continues working
- [ ] Disconnect from DeX while app is running → app continues working
- [ ] Switch between phone and external display → preview follows
- [ ] Rotate phone while in DeX → app orientation handles correctly
- [ ] Change monitor resolution → app adapts

**Multi-Window:**
- [ ] App works in split-screen with another app
- [ ] Camera releases when app loses focus
- [ ] Camera resumes when app regains focus
- [ ] Multiple windows can be opened (if supported)

**Camera Features:**
- [ ] All cameras accessible (wide, ultra-wide, telephoto)
- [ ] Manual controls work (ISO, shutter, focus, WB)
- [ ] RAW/DNG capture works
- [ ] HDR mode works
- [ ] Burst mode works
- [ ] Video profiles work
- [ ] Preview effects work (histogram, focus peak, etc.)

**Performance:**
- [ ] Preview is smooth (30+ fps)
- [ ] No lag when resizing window
- [ ] Capture latency acceptable
- [ ] App doesn't crash under load
- [ ] Memory usage reasonable
- [ ] Battery life acceptable (when plugged in)

**UI/UX:**
- [ ] All controls are accessible
- [ ] Touch targets are adequate size
- [ ] Text is readable on external display
- [ ] Keyboard shortcuts work (if implemented)
- [ ] Mouse interactions work (if implemented)

**Edge Cases:**
- [ ] Hot-plug/unplug HDMI while capturing
- [ ] Switch to DeX during video recording
- [ ] Low memory conditions
- [ ] Rotate monitor (if portrait mode supported)
- [ ] Use with USB-C dock with multiple peripherals

---

## Known Issues & Limitations

### Current Limitations:
1. **Camera API Restrictions**: Some devices limit camera access in DeX mode
2. **Orientation Lock**: Current landscape lock breaks DeX window management
3. **Fixed Layout**: UI not optimized for arbitrary window sizes
4. **Preview Resolution**: May not select optimal resolution for external display
5. **No Multi-Window Support**: Camera may conflict in split-screen

### Samsung S23 Specific:
- DeX mode may limit frame rate on some camera modes
- Expert RAW mode availability in DeX unknown
- Multi-camera switching in DeX untested
- 8K recording in DeX mode untested

### DeX Platform Limitations:
- Camera2 API may have restrictions in DeX mode
- Some OEM camera features may be disabled
- External display refresh rate typically locked to 60Hz
- Touchscreen may not work on external display (depends on monitor)

---

## Implementation Order

### Phase 1: Core DeX Support (Week 1-2)
1. Enable resizeable activity
2. Implement DexManager for mode detection
3. Update DisplayUtil for multi-display
4. Handle configuration changes
5. Basic testing on S23 + DeX

### Phase 2: Camera & Preview (Week 3-4)
6. Optimize camera parameters for DeX
7. Fix preview scaling issues
8. Handle display transitions smoothly
9. Samsung-specific camera features
10. Performance profiling and optimization

### Phase 3: UI & Polish (Week 5-6)
11. UI layout adaptations
12. Multi-window support
13. Keyboard and mouse input
14. External display quality settings
15. Comprehensive testing

### Phase 4: Enhancements (Week 7+)
16. Desktop-style menus (optional)
17. Advanced DeX features
18. Additional testing on other Samsung devices
19. Documentation updates

---

## References

### Samsung DeX Resources:
- [Samsung DeX Developer Guide](https://developer.samsung.com/samsung-dex/overview.html)
- [Multi-Window Support](https://developer.android.com/guide/topics/ui/multi-window)
- [Resizeable Activities](https://developer.android.com/guide/topics/large-screens/multi-window-support)

### Android Camera APIs:
- [Camera2 API Reference](https://developer.android.com/reference/android/hardware/camera2/package-summary)
- [Supporting Different Screen Sizes](https://developer.android.com/training/multiscreen/screensizes)
- [Window Metrics API](https://developer.android.com/reference/android/view/WindowMetrics)

### Samsung S23 Specifications:
- Display: 6.1" FHD+ (1080 x 2340)
- DeX Output: Up to QHD (2560 x 1440) @ 60Hz
- Cameras: 50MP + 12MP + 10MP
- SoC: Snapdragon 8 Gen 2 (most regions) or Exynos 2200

### Testing:
- Use `adb shell dumpsys display` to check display configuration
- Use `adb shell wm size` to check window sizes
- Monitor with `adb logcat | grep FreeDcam` during transitions

---

## Notes

- All changes should maintain backward compatibility with non-DeX usage
- Test on both Snapdragon and Exynos variants of S23
- Consider that DeX experience varies by monitor/adapter
- Keep performance impact minimal for phone-only usage
- Document any Samsung-specific workarounds needed

**Last Updated:** 2025-11-16
**Version:** 1.0
**Status:** Draft - Ready for Implementation
