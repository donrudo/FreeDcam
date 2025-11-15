# FreeDcam - AI Assistant Guide

This document provides comprehensive guidance for AI assistants working on the FreeDcam codebase. FreeDcam is an advanced Android camera application supporting Camera1 and Camera2 APIs with extensive device-specific customizations.

## Table of Contents
1. [Project Overview](#project-overview)
2. [Codebase Structure](#codebase-structure)
3. [Architecture](#architecture)
4. [Development Environment](#development-environment)
5. [Key Conventions](#key-conventions)
6. [Development Workflow](#development-workflow)
7. [Testing Guidelines](#testing-guidelines)
8. [Common Patterns](#common-patterns)
9. [Things to Avoid](#things-to-avoid)
10. [Device-Specific Code](#device-specific-code)

---

## Project Overview

**FreeDcam** is a professional camera application for Android that provides manual camera controls and RAW/DNG capture support. It abstracts both Camera1 (deprecated) and Camera2 APIs to support a wide range of devices.

### Key Features
- Manual camera controls (ISO, shutter, focus, white balance)
- RAW/DNG image capture and processing
- Device-specific optimizations for 20+ manufacturers
- Video recording with custom profiles
- OpenGL-based preview with post-processing
- JNI integration for LibRaw and LibTIFF

### Technology Stack
- **Language**: Java 8
- **Build System**: Gradle 7.2.1
- **Min SDK**: 14 (Android 4.0)
- **Target SDK**: 31 (Android 12)
- **Dependency Injection**: Dagger Hilt 2.42
- **Native Code**: NDK with LibRaw, LibTIFF
- **UI Framework**: AndroidX with Data Binding

---

## Codebase Structure

```
FreeDcam/
├── app/                              # Main application module
│   ├── src/main/
│   │   ├── java/
│   │   │   ├── freed/                # Main application code
│   │   │   │   ├── cam/              # Camera core functionality
│   │   │   │   │   ├── apis/         # Camera API abstractions
│   │   │   │   │   │   ├── basecamera/      # Base classes for camera
│   │   │   │   │   │   ├── camera1/         # Camera1 API implementation
│   │   │   │   │   │   ├── camera2/         # Camera2 API implementation
│   │   │   │   │   │   ├── featuredetector/ # Device feature detection
│   │   │   │   │   │   └── sonyremote/      # Sony remote API
│   │   │   │   │   ├── event/        # Event handling system
│   │   │   │   │   ├── histogram/    # Histogram processing
│   │   │   │   │   ├── previewpostprocessing/ # Preview rendering
│   │   │   │   │   └── ui/           # User interface
│   │   │   │   ├── dng/              # DNG file handling
│   │   │   │   ├── file/             # File operations
│   │   │   │   ├── gl/               # OpenGL rendering
│   │   │   │   ├── image/            # Image processing
│   │   │   │   ├── jni/              # JNI wrappers
│   │   │   │   ├── settings/         # Settings management
│   │   │   │   ├── utils/            # Utility classes
│   │   │   │   └── viewer/           # Image viewer
│   │   │   ├── camera2_hidden_keys/  # OEM-specific Camera2 keys
│   │   │   ├── Camera2EXT/           # Camera2 extensions
│   │   │   ├── com/                  # Third-party integrations
│   │   │   │   ├── lge/              # LG-specific APIs
│   │   │   │   ├── sonyericsson/     # Sony-specific APIs
│   │   │   │   └── ortiz/            # TouchImageView library
│   │   │   └── hilt/                 # Dependency injection modules
│   │   ├── jni/                      # Native code (C/C++)
│   │   │   ├── LibRaw/               # LibRaw library
│   │   │   ├── tiff/                 # LibTIFF library
│   │   │   ├── freedcam/             # Custom native code
│   │   │   └── include/              # Native headers
│   │   ├── res/                      # Android resources
│   │   └── AndroidManifest.xml       # App manifest
│   ├── build.gradle                  # App-level Gradle config
│   └── proguard-rules.pro            # ProGuard rules
├── Camera1Parameters/                # Camera1 parameters submodule
├── renderscript-intrinsics-replacement-toolkit/  # RenderScript toolkit
├── build.gradle                      # Project-level Gradle config
├── settings.gradle                   # Gradle settings
├── gradle.properties                 # Gradle properties
└── README.md                         # User documentation
```

### Key Directories Explained

#### `/freed/cam/apis/`
Contains all camera API abstractions and implementations:
- **basecamera/**: Abstract base classes defining the camera interface
- **camera1/**: Legacy Camera API implementation with device-specific variants
- **camera2/**: Modern Camera2 API implementation
- **featuredetector/**: Detects available features on device startup

#### `/freed/cam/ui/`
User interface components:
- **themenextgen/**: Modern UI implementation with fragments and adapters
- **themesample/**: Legacy UI implementation
- **videoprofileeditor/**: Video profile customization UI

#### `/freed/jni/`
Java wrappers for native code:
- `LibRawJniWrapper.java`: RAW image processing
- `RawToDng.java`: RAW to DNG conversion
- `DngStack.java`: DNG stacking operations

#### `/hilt/`
Dependency injection modules organized by scope:
- Singleton components (app-level)
- Activity-scoped components
- Entry points for non-Activity classes

#### `/camera2_hidden_keys/`
OEM-specific Camera2 keys for manufacturers:
- `qcom/`: Qualcomm Snapdragon devices
- `samsung/`: Samsung devices
- `xiaomi/`: Xiaomi devices
- `mtk/`: MediaTek devices
- `huawei/`: Huawei devices

---

## Architecture

### Design Patterns

FreeDcam uses a sophisticated multi-layered architecture:

#### 1. **Generic Template Method Pattern**
The core abstraction uses Java generics to unify Camera1 and Camera2 APIs:

```java
public abstract class AbstractCamera<P, C, M, F> implements CameraWrapperInterface {
    // P = ParameterHandler type
    // C = CameraHolder type
    // M = ModuleHandler type
    // F = FocusHandler type
}

// Camera1 implementation
public class Camera1 extends AbstractCamera<
    ParametersHandler,
    CameraHolder,
    ModuleHandler,
    FocusHandler
>

// Camera2 implementation
public class Camera2 extends AbstractCamera<
    ParameterHandlerApi2,
    CameraHolderApi2,
    ModuleHandlerApi2,
    FocusHandler
>
```

**Key File**: `/freed/cam/apis/basecamera/AbstractCamera.java`

#### 2. **Dependency Injection (Hilt/Dagger)**
All major components are managed through Hilt:

```java
@HiltAndroidApp
public class FreedApplication extends Application {
    // Entry points provide access to Hilt dependencies
}

@AndroidEntryPoint
public class ActivityFreeDcamMain extends ActivityAbstract {
    @Inject CameraApiManager cameraApiManager;
    @Inject SettingsManager settingsManager;
}
```

**Key Files**:
- `/freed/FreedApplication.java` - Application class
- `/hilt/*Module.java` - Dependency modules

#### 3. **Strategy Pattern for Camera Selection**
The camera implementation is selected at runtime:

```java
public class CameraApiManager<C extends CameraWrapperInterface> {
    public void switchCamera() {
        switch (api) {
            case SettingsManager.API_2:
                camera = (C) new Camera2();
                break;
            default:
                camera = (C) new Camera1();
                break;
        }
    }
}
```

**Key File**: `/freed/cam/apis/CameraApiManager.java`

#### 4. **Module System (Capture Modes)**
Capture modes are implemented as pluggable modules:

```java
public interface ModuleInterface {
    void InitModule();
    void DestroyModule();
    void DoWork();  // Execute capture
    boolean IsWorking();
    String ModuleName();
}
```

Available modules:
- Picture capture (standard, HDR, burst)
- Video recording (normal, high-speed)
- RAW capture
- Interval shooting

**Key Files**:
- `/freed/cam/apis/basecamera/modules/ModuleAbstract.java`
- `/freed/cam/apis/camera1/modules/*`
- `/freed/cam/apis/camera2/modules/*`

#### 5. **Custom Event System**
Instead of EventBus, FreeDcam uses a custom observer pattern:

```java
public abstract class BaseEventHandler<E extends MyEvent> {
    protected List<E> eventListners;

    public void setEventListner(E listner) {
        if (!eventListners.contains(listner))
            eventListners.add(listner);
    }
}
```

Event types:
- `CameraHolderEvent`: Camera state changes
- `CaptureStateChangedEvent`: Capture progress
- `ModuleChangedEvent`: Module switching

**Key Files**: `/freed/cam/event/*`

### Threading Model

FreeDcam uses a multi-threaded architecture:

1. **Main UI Thread**: UI updates and user interaction
2. **Camera Background Thread**: Camera operations (open/close/configure)
3. **Module Background Thread**: Capture operations
4. **Image Processing Thread**: File I/O and image processing

**Key File**: `/freed/utils/BackgroundHandlerThread.java`

### Lifecycle Management

Components implement lifecycle awareness:

```java
public class ActivityFreeDcamMain extends ActivityAbstract {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(null);
        getLifecycle().addObserver(locationManager);
        getLifecycle().addObserver(orientationManager);
        cameraApiManager.init();
    }

    @Override
    protected void onResume() {
        cameraApiManager.onResume();
    }

    @Override
    protected void onPause() {
        cameraApiManager.onPause();
    }
}
```

---

## Development Environment

### Prerequisites

1. **Android Studio**: Latest version (Arctic Fox or newer recommended)
2. **JDK**: Java 8 or higher
3. **NDK**: For native code compilation
4. **SDK Components**:
   - Android SDK 31 (target)
   - Android SDK 14+ (minimum)
   - Build Tools 30.0.3

### Build Configuration

#### Gradle Files

**Project-level** (`build.gradle`):
- Gradle plugin: 7.2.1
- Hilt plugin: 2.42
- Repositories: Google, JCenter, JitPack

**App-level** (`app/build.gradle`):
- Min SDK: 14
- Target SDK: 31
- Compile SDK: 31
- Java 8 compatibility
- Data Binding enabled
- NDK build enabled

### Building the Project

```bash
# Clean build
./gradlew clean

# Build debug APK
./gradlew assembleDebug

# Build release APK
./gradlew assembleRelease

# Install debug on connected device
./gradlew installDebug
```

### Native Code Compilation

Native libraries are built automatically via NDK:

```gradle
externalNativeBuild {
    ndkBuild {
        path 'src/main/jni/Android.mk'
    }
}
```

Supported ABIs:
- armeabi-v7a
- arm64-v8a
- x86
- x86_64

### Signing Configuration

Debug and release builds use signing configuration from:
- Keystore: `key/freedcamkey.jks`
- Password: `freedcam`
- Key alias: `freedcamkey`

**Note**: For production builds, use your own signing key.

---

## Key Conventions

### Naming Conventions

#### Classes
- **Camera1 classes**: `CameraHolder`, `ParametersHandler`, `ModuleHandler`
- **Camera2 classes**: `CameraHolderApi2`, `ParameterHandlerApi2`, `ModuleHandlerApi2`
- **Abstract classes**: `AbstractCamera`, `ModuleAbstract`, `ActivityAbstract`
- **Interfaces**: Suffix with `Interface` (e.g., `CameraWrapperInterface`)
- **Event handlers**: Suffix with `EventHandler` (e.g., `CameraHolderEventHandler`)

#### Packages
- **freed.***: Main application code
- **freed.cam.***: Camera-specific functionality
- **freed.cam.apis.***: API abstractions
- **camera2_hidden_keys.***: OEM-specific keys
- **hilt.***: Dependency injection modules

#### Methods
- **Lifecycle methods**: Match Android conventions (`onCreate`, `onResume`)
- **Event methods**: Prefix with `on` (e.g., `onCameraOpen`)
- **Fire methods**: Event dispatchers (e.g., `fireOnCameraOpen`)
- **Public methods**: Use camelCase
- **Module methods**: PascalCase (legacy, e.g., `InitModule`, `DestroyModule`)

### Code Style

1. **Indentation**: 4 spaces
2. **Braces**: K&R style (opening brace on same line)
3. **Line length**: No strict limit, but keep readable
4. **Logging**: Use Android Log class with TAG constants
5. **Null checks**: Perform null checks before accessing objects
6. **Thread safety**: Document thread requirements in comments

### File Organization

Within Java files:
1. Package declaration
2. Imports (Android, Java, third-party, app)
3. Class documentation (if applicable)
4. Class declaration
5. Constants
6. Static fields
7. Instance fields
8. Constructors
9. Lifecycle methods
10. Public methods
11. Protected methods
12. Private methods
13. Inner classes

### Resource Naming

**Drawables**:
- Icons: `ic_feature_variant.png` (e.g., `ic_launcher_foreground.png`)
- UI elements: `drawable_purpose.png`
- Manual controls: `manual_parameter.png` (e.g., `manual_iso.png`)

**Layouts**:
- Activities: `activity_name.xml`
- Fragments: `fragment_name.xml`
- Items: `item_description.xml`

**Strings**:
- Features: `feature_description`
- Errors: `error_description`

---

## Development Workflow

### Adding New Features

#### 1. Feature Detection
For camera features, add detection in:
- `/freed/cam/apis/featuredetector/camera1/Camera1FeatureDetector.java` (Camera1)
- `/freed/cam/apis/featuredetector/camera2/Camera2FeatureDetector.java` (Camera2)

#### 2. Parameter Implementation
Add parameter handling:
- Camera1: `/freed/cam/apis/camera1/parameters/ParametersHandler.java`
- Camera2: `/freed/cam/apis/camera2/parameters/ParameterHandlerApi2.java`

#### 3. UI Integration
Add UI controls in:
- `/freed/cam/ui/themenextgen/` for modern UI
- Update adapters and view models

#### 4. Settings Persistence
Add settings keys in:
- `/freed/settings/SettingKeys.java`
- `/freed/settings/SettingsManager.java`

### Adding Device-Specific Code

#### Camera1 Device Support
1. Create camera holder in `/freed/cam/apis/camera1/cameraholder/`:
   ```java
   public class CameraHolderYourDevice extends CameraHolderLegacy {
       // Device-specific implementations
   }
   ```

2. Register in camera holder factory:
   ```java
   public CameraHolder createCameraHolder() {
       if (Build.MANUFACTURER.equals("YourOEM"))
           return new CameraHolderYourDevice();
       // ...
   }
   ```

#### Camera2 Hidden Keys
1. Add keys in `/camera2_hidden_keys/devices/yourdevice/`:
   ```java
   public class YourDeviceKeys {
       public static final CaptureRequest.Key<Integer> CUSTOM_KEY =
           new CaptureRequest.Key<>("com.yourdevice.parameter", Integer.class);
   }
   ```

### Modifying Native Code

Native code is in `/app/src/main/jni/`:

1. Edit C/C++ files
2. Update `Android.mk` if adding new files
3. Rebuild: `./gradlew assembleDebug`
4. Update JNI wrapper in `/freed/jni/` if changing interfaces

### Testing Changes

1. **Manual Testing**:
   - Test on physical device (emulator has limited camera support)
   - Test both Camera1 and Camera2 APIs
   - Test different capture modes
   - Test settings persistence

2. **Device-Specific Testing**:
   - Test on target device manufacturer
   - Verify device-specific parameters work
   - Check for crashes in feature detection

3. **Performance Testing**:
   - Monitor memory usage (app uses `largeHeap`)
   - Check camera preview frame rate
   - Verify capture operations complete

### Debugging

#### Enable Logging
Add logging throughout code:
```java
private static final String TAG = YourClass.class.getSimpleName();
Log.d(TAG, "Message");
```

#### Camera Blob Logging (Qualcomm)
For Qualcomm devices, enable detailed camera HAL logging:

Create `/data/misc/camera/camera_dbg.txt`:
```
cam_dbglevel=debug
mct_dbglevel=debug
sensor_dbglevel=debug
iface_dbglevel=debug
isp_dbglevel=debug
stats_dbglevel=debug
pproc_dbglevel=debug
imglib_dbglevel=debug
cpp_dbglevel=debug
hal_dbglevel=debug
jpeg_dbglevel=debug
c2d_dbglevel=debug
```

Set permissions: `chmod 770 /data/misc/camera/camera_dbg.txt`

#### Common Issues

**Camera won't open**:
- Check permissions in AndroidManifest.xml
- Verify camera is not in use by another app
- Check logcat for errors

**Parameters not working**:
- Verify device supports the parameter
- Check feature detection results
- Test parameter values are in supported range

**Native crashes**:
- Check JNI wrapper matches native signatures
- Verify memory management in C++ code
- Use NDK debugger: `ndk-gdb`

---

## Testing Guidelines

### Manual Testing Checklist

When making changes, verify:

#### Camera Operations
- [ ] Camera opens successfully
- [ ] Camera switches (front/back)
- [ ] Preview displays correctly
- [ ] Focus works (tap to focus)
- [ ] Exposure works

#### Capture Operations
- [ ] Photo capture works
- [ ] Video recording works
- [ ] DNG capture works (if device supports)
- [ ] Burst mode works
- [ ] HDR mode works

#### Settings
- [ ] Settings persist after app restart
- [ ] Settings UI displays correctly
- [ ] Manual controls work (ISO, shutter, etc.)
- [ ] Video profiles work

#### Viewer
- [ ] Images display in viewer
- [ ] DNG conversion works
- [ ] Image details display
- [ ] Sharing works

### Device Testing Matrix

Test on devices from different categories:
- **Qualcomm Snapdragon**: Most common, extensive manual controls
- **MediaTek**: Different parameter handling
- **Samsung Exynos**: Samsung-specific keys
- **Huawei Kirin**: Huawei extensions
- **Legacy devices**: Android 4.x-5.x for Camera1
- **Modern devices**: Android 10+ for Camera2

### Regression Testing

Before releases, test:
1. Clean install on fresh device
2. Upgrade from previous version
3. Permission handling on different Android versions
4. Storage access (internal/external/SAF)

---

## Common Patterns

### Pattern 1: Adding a Camera Parameter

**Example**: Adding a new manual parameter

1. **Add to parameter handler**:
```java
// In ParametersHandler.java or ParameterHandlerApi2.java
public class ParametersHandler {
    private ManualParameter yourParameter;

    public void Init() {
        // ...
        yourParameter = new ManualParameter(
            parameters,
            "parameter-key",
            1,  // min value
            100,  // max value
            1   // step
        );
    }

    public ManualParameter getYourParameter() {
        return yourParameter;
    }
}
```

2. **Add UI control** in adapter:
```java
// In ParametersAdapter.java
if (cameraApiManager.getParameterHandler().getYourParameter() != null) {
    addView(new ManualView(
        context,
        cameraApiManager.getParameterHandler().getYourParameter()
    ));
}
```

3. **Add settings persistence**:
```java
// In SettingKeys.java
public static final Key<Integer> YOUR_PARAMETER = new Key<>("your_parameter", 50);

// In ParametersHandler.java
yourParameter.setValueChangedListener(value -> {
    settingsManager.set(SettingKeys.YOUR_PARAMETER, value);
});
```

### Pattern 2: Adding a Capture Module

**Example**: Adding a new capture mode

1. **Create module class**:
```java
public class YourModule extends ModuleAbstract<Camera1> {
    public YourModule(Camera1 cameraWrapper) {
        super(cameraWrapper);
    }

    @Override
    public void InitModule() {
        // Setup module
    }

    @Override
    public void DestroyModule() {
        // Cleanup
    }

    @Override
    public void DoWork() {
        // Execute capture
        changeCaptureState(CaptureStates.CAPTURING);
        // ... capture logic
        changeCaptureState(CaptureStates.DONE);
    }

    @Override
    public String ModuleName() {
        return "YourMode";
    }
}
```

2. **Register in module handler**:
```java
// In ModuleHandler.java
public void Init() {
    moduleList.put("YourMode", new YourModule(cameraWrapper));
}
```

3. **Add UI for module selection** in mode switcher.

### Pattern 3: Device-Specific Camera Holder

**Example**: Adding support for a new OEM

1. **Create camera holder**:
```java
public class CameraHolderYourOEM extends CameraHolderLegacy {
    public CameraHolderYourOEM(Context context, int camera) {
        super(context, camera);
    }

    @Override
    public void SetParameters(Camera.Parameters parameters) {
        // Apply OEM-specific parameters
        parameters.set("vendor-parameter", value);
        super.SetParameters(parameters);
    }
}
```

2. **Register in factory**:
```java
// Detect device and return appropriate holder
if (Build.MANUFACTURER.equals("YourOEM")) {
    return new CameraHolderYourOEM(context, cameraId);
}
```

### Pattern 4: Using Hilt Dependency Injection

**Example**: Accessing dependencies in non-Activity class

1. **Define entry point**:
```java
@EntryPoint
@InstallIn(ActivityComponent.class)
public interface YourEntryPoint {
    YourDependency yourDependency();
}
```

2. **Access from non-Activity**:
```java
public class YourClass {
    private YourDependency dependency;

    public YourClass(Context context) {
        YourEntryPoint entryPoint = EntryPointAccessors.fromActivity(
            (Activity) context,
            YourEntryPoint.class
        );
        dependency = entryPoint.yourDependency();
    }
}
```

### Pattern 5: Native Code Integration

**Example**: Adding a new native method

1. **Declare in Java**:
```java
public class YourJniWrapper {
    static {
        System.loadLibrary("freedcam");
    }

    private native void yourNativeMethod(ByteBuffer buffer, int param);
}
```

2. **Implement in C++**:
```cpp
// In freedcam/ directory
extern "C"
JNIEXPORT void JNICALL
Java_freed_jni_YourJniWrapper_yourNativeMethod(
    JNIEnv *env,
    jobject thiz,
    jobject buffer,
    jint param) {
    // Implementation
}
```

3. **Update Android.mk** to include the file.

### Pattern 6: Event Handling

**Example**: Listening to camera events

1. **Implement event interface**:
```java
public class YourClass implements CameraHolderEvent {
    @Override
    public void onCameraOpen() {
        // Camera opened
    }

    @Override
    public void onCameraOpenFinished() {
        // Camera fully initialized
    }

    @Override
    public void onCameraClose() {
        // Camera closed
    }

    @Override
    public void onCameraError(String error) {
        // Handle error
    }
}
```

2. **Register listener**:
```java
cameraApiManager.setCameraHolderEventHandler(yourClass);
```

---

## Things to Avoid

### Anti-Patterns

1. **DON'T use EventBus**: The project uses a custom event system. Don't add EventBus dependencies or usage.

2. **DON'T call camera APIs on main thread**: Always use background threads for camera operations:
   ```java
   // BAD
   camera.open();

   // GOOD
   backgroundThread.execute(() -> {
       camera.open();
   });
   ```

3. **DON'T assume camera features exist**: Always check if features are supported:
   ```java
   // BAD
   parameters.getManualIso().setValue(400);

   // GOOD
   if (parameters.getManualIso() != null) {
       parameters.getManualIso().setValue(400);
   }
   ```

4. **DON'T hardcode device detection**: Use Build properties:
   ```java
   // BAD
   if (deviceModel.equals("SM-G950F")) { ... }

   // GOOD
   if (Build.MANUFACTURER.equals("samsung") &&
       Build.DEVICE.contains("dreamlte")) { ... }
   ```

5. **DON'T leak camera resources**: Always release camera in finally blocks or use try-with-resources patterns.

6. **DON'T modify settings storage format**: Breaking changes affect all users' settings.

7. **DON'T add heavyweight dependencies**: The app targets min SDK 14 and needs to stay lean.

### Common Mistakes

1. **Forgetting null checks**: Many parameters are device-dependent and may be null.

2. **Not handling permissions**: Always check runtime permissions before camera access.

3. **Ignoring threading**: UI updates must be on main thread, camera ops on background.

4. **Not testing on real devices**: Emulator camera support is limited and unrealistic.

5. **Breaking backward compatibility**: Users upgrade from older versions.

6. **Not cleaning up resources**: Memory leaks are easy with cameras and bitmaps.

### Security Considerations

1. **Storage permissions**: Handle scoped storage on Android 10+
2. **Camera permissions**: Request at runtime, handle denials gracefully
3. **Location permissions**: For geotagging, optional feature
4. **File provider**: Use FileProvider for sharing images
5. **Clear text traffic**: Allowed for Sony remote API, minimize usage

---

## Device-Specific Code

### Supported Device Frameworks

FreeDcam detects and supports these frameworks:

1. **Qualcomm (Qcom)**: Most extensive support
   - Custom ISO controls
   - Custom shutter speed
   - Manual white balance
   - Focus modes
   - **Location**: `/camera2_hidden_keys/qcom/`

2. **MediaTek (MTK)**:
   - MTK-specific parameters
   - **Location**: `/camera2_hidden_keys/mtk/`, `/freed/cam/apis/camera1/cameraholder/CameraHolderMTK.java`

3. **Samsung**:
   - Samsung hidden keys
   - **Location**: `/camera2_hidden_keys/samsung/`

4. **LG**:
   - LG camera extensions
   - **Location**: `/camera2_hidden_keys/lg/`, `/com/lge/`

5. **Sony**:
   - Sony camera extensions
   - Sony remote API
   - **Location**: `/com/sonyericsson/`, `/freed/cam/apis/sonyremote/`

6. **Huawei**:
   - Huawei extensions
   - **Location**: `/camera2_hidden_keys/huawei/`

7. **Xiaomi**:
   - Xiaomi-specific keys
   - **Location**: `/camera2_hidden_keys/xiaomi/`

8. **Motorola**:
   - Motorola extensions
   - **Location**: `/freed/cam/apis/camera1/cameraholder/CameraHolderMotoX.java`

### Adding Device Support

When adding support for a new device:

1. **Identify the SoC**: Qualcomm, MediaTek, Exynos, etc.
2. **Research available parameters**: Check OEM camera app capabilities
3. **Detect the framework**: Add detection logic in `CameraFeatureDetector`
4. **Implement camera holder**: Create device-specific holder if needed
5. **Add hidden keys**: For Camera2, add vendor keys
6. **Test thoroughly**: Verify all parameters work correctly
7. **Document findings**: Add device to supported list in README

### OEM-Specific Permissions

Some OEMs require special permissions:

**Huawei**:
```xml
<uses-permission android:name="android.permission.HW_CAMCFG_SERVICE" />
<uses-permission android:name="com.huawei.camera.permission.PRIVATE" />
```

**Sony**:
```xml
<uses-permission android:name="com.sonyericsson.permission.CAMERA_EXTENDED" />
```

**OnePlus**:
```xml
<uses-permission android:name="com.oneplus.camera.CAMERA_SERVICE" />
```

These are already in AndroidManifest.xml but won't harm on other devices.

---

## Resources and References

### Internal Documentation
- `README.md`: User-facing documentation
- Code comments: Inline documentation in complex areas
- This file: AI assistant guidance

### External Resources
- [Camera1 API](http://developer.android.com/reference/android/hardware/Camera.html)
- [Camera2 API](http://developer.android.com/reference/android/hardware/camera2/package-summary.html)
- [LibRaw](https://github.com/LibRaw/LibRaw)
- [LibTIFF](http://www.remotesensing.org/libtiff/)
- [Hilt Documentation](https://dagger.dev/hilt/)

### Key Contacts
- Project: FreeDcam
- License: GNU GPL v2
- Repository: Check git remotes

---

## Changelog

### Document Version History

- **v1.0** (2025-11-15): Initial comprehensive documentation
  - Complete architecture overview
  - Development workflows
  - Common patterns and conventions
  - Device-specific guidance

---

## Quick Reference

### Essential Files

| Purpose | File Path |
|---------|-----------|
| Application entry | `/freed/FreedApplication.java` |
| Main camera activity | `/freed/cam/ActivityFreeDcamMain.java` |
| Camera API manager | `/freed/cam/apis/CameraApiManager.java` |
| Camera1 implementation | `/freed/cam/apis/camera1/Camera1.java` |
| Camera2 implementation | `/freed/cam/apis/camera2/Camera2.java` |
| Settings manager | `/freed/settings/SettingsManager.java` |
| Feature detection | `/freed/cam/apis/featuredetector/` |
| Hilt modules | `/hilt/` |
| Native wrappers | `/freed/jni/` |
| Event system | `/freed/cam/event/` |

### Command Quick Reference

```bash
# Build
./gradlew assembleDebug              # Build debug APK
./gradlew assembleRelease            # Build release APK
./gradlew clean                      # Clean build

# Install
./gradlew installDebug               # Install debug build
adb install -r app/build/outputs/apk/debug/FreeDcam_debug_*.apk

# Debugging
adb logcat | grep FreeDcam          # View app logs
adb logcat -s TAG_NAME              # View specific tag
ndk-gdb                             # Debug native code

# Device Info
adb shell getprop ro.product.manufacturer  # Get manufacturer
adb shell getprop ro.build.product         # Get device model
```

### Key Classes Hierarchy

```
Application
└── FreedApplication (Hilt entry point)

Activity
└── AppCompatActivity
    └── ActivityAbstract (permissions, settings)
        └── ActivityFreeDcamMain (camera activity)

Camera Wrapper
└── CameraWrapperInterface
    └── AbstractCamera<P,C,M,F>
        ├── Camera1
        └── Camera2

Module (Capture Modes)
└── ModuleInterface
    └── ModuleAbstract<CW>
        ├── PictureModule
        ├── VideoModule
        ├── HDRModule
        └── IntervalModule
```

---

**Last Updated**: 2025-11-15
**Document Maintainer**: AI Assistant
**Version**: 1.0
