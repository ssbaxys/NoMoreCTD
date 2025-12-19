# NoMoreCTD API Documentation

## Overview
NoMoreCTD provides a comprehensive API for other mod developers to integrate with our crash prevention and hot-reload features.

## Getting Started

### Maven/Gradle Setup

Add NoMoreCTD to your development environment:

**build.gradle:**
```gradle
repositories {
    maven {
        name = "NoMoreCTD"
        url = "https://maven.yourrepo.com/releases"
    }
}

dependencies {
    // Compile-only dependency (NoMoreCTD is not required)
    compileOnly "com.nomorectd:nomorectd:1.5.0"
}
```

### Basic Usage

```java
import com.nomorectd.api.*;

public class MyMod {
    public void init() {
        // Check if NoMoreCTD is available
        NoMoreCTDAPI.getAPI().ifPresent(api -> {
            LOGGER.info("NoMoreCTD API v{} available", api.getAPIVersion());
            registerIntegration(api);
        });
    }
}
```

## API Interfaces

### INoMoreCTDAPI

Main API interface for interacting with NoMoreCTD.

#### Methods

##### getAPIVersion()
```java
String getAPIVersion()
```
Returns the API version string (e.g., "1.0.0").

##### registerCrashHandler()
```java
void registerCrashHandler(String modId, ICrashHandler handler)
```
Register a custom crash handler for your mod.

**Parameters:**
- `modId` - Your mod's ID
- `handler` - Implementation of ICrashHandler

**Example:**
```java
api.registerCrashHandler("mymod", (exception, context) -> {
    if (exception instanceof MyCustomException) {
        LOGGER.error("Handled custom exception: {}", exception.getMessage());
        performCleanup();
        return true; // Crash handled, continue game
    }
    return false; // Let NoMoreCTD handle it
});
```

##### unregisterCrashHandler()
```java
void unregisterCrashHandler(String modId)
```
Remove your crash handler.

##### hasKnownIssues()
```java
boolean hasKnownIssues(String modId)
```
Check if a specific mod has known compatibility issues.

##### getDetectedConflicts()
```java
List<String> getDetectedConflicts()
```
Get list of detected mod conflicts as formatted strings.

##### registerCompatibilityLayer()
```java
void registerCompatibilityLayer(String modId, ICompatibilityLayer layer)
```
Register a compatibility layer for your mod.

**Example:**
```java
api.registerCompatibilityLayer("mymod", new ICompatibilityLayer() {
    @Override
    public void onInit() {
        LOGGER.info("MyMod compatibility layer initialized");
    }

    @Override
    public boolean canHotReload() {
        return !hasActiveTransactions(); // Check if safe to reload
    }

    @Override
    public void onHotReload() {
        clearCaches();
        reloadConfigs();
    }

    @Override
    public CompatibilityLevel getCompatibilityLevel() {
        return CompatibilityLevel.HIGH;
    }
});
```

##### isFeatureEnabled()
```java
boolean isFeatureEnabled(String feature)
```
Check if a NoMoreCTD feature is enabled.

**Feature names:**
- `"crash_prevention"` - Crash prevention system
- `"hot_reload"` - Hot-reload functionality
- `"performance_monitor"` - Performance monitoring
- `"auto_save"` - Automatic emergency saves
- `"safe_mode"` - Safe mode system

##### getPerformanceMetrics()
```java
PerformanceData getPerformanceMetrics()
```
Get current performance metrics.

**Returns:** PerformanceData object with:
- `getAverageFps()` - Average FPS
- `getMemoryUsed()` - Memory usage in bytes
- `getMemoryTotal()` - Total available memory
- `getMemoryUsagePercent()` - Memory usage percentage
- `getTickRate()` - Server tick rate
- `getEntityCount()` - Entity count
- `getChunkCount()` - Loaded chunk count

##### requestEmergencySave()
```java
void requestEmergencySave()
```
Request an emergency world save. Use before risky operations.

**Example:**
```java
api.requestEmergencySave();
try {
    performRiskyOperation();
} catch (Exception e) {
    LOGGER.error("Operation failed, but world was saved", e);
}
```

##### isSafeMode()
```java
boolean isSafeMode()
```
Check if the game is running in safe mode.

---

### ICrashHandler

Functional interface for handling crashes.

```java
@FunctionalInterface
public interface ICrashHandler {
    boolean handleCrash(Throwable exception, CrashContext context);
}
```

#### CrashContext

Provides context about a crash:
```java
public class CrashContext {
    public String getLocation()   // Where the crash occurred
    public String getModId()      // Mod that caused the crash
    public long getTimestamp()    // When the crash occurred
}
```

#### Example Implementation

```java
ICrashHandler myHandler = (exception, context) -> {
    // Log crash details
    LOGGER.error("Crash at {} from mod {}",
        context.getLocation(), context.getModId());

    // Handle specific exception types
    if (exception instanceof NullPointerException) {
        resetNullableFields();
        return true; // Crash handled
    }

    if (exception instanceof OutOfMemoryError) {
        clearLargeCaches();
        System.gc();
        return true; // Attempt recovery
    }

    return false; // Let NoMoreCTD handle it
};

api.registerCrashHandler("mymod", myHandler);
```

---

### ICompatibilityLayer

Interface for defining mod-specific compatibility behavior.

```java
public interface ICompatibilityLayer {
    void onInit();
    boolean canHotReload();
    void onHotReload();
    CompatibilityLevel getCompatibilityLevel();
    default String[] getDisabledFeatures() { return new String[0]; }
}
```

#### Methods

##### onInit()
Called when NoMoreCTD initializes. Set up your compatibility layer here.

##### canHotReload()
Return `true` if your mod can safely be hot-reloaded, `false` otherwise.

**Considerations:**
- Do you have active network connections?
- Are there running timers/scheduled tasks?
- Is there unsaved state that would be lost?
- Do you use native code that can't be unloaded?

##### onHotReload()
Called after a hot-reload operation. Clean up and reinitialize as needed.

**Common tasks:**
- Clear caches
- Reload configurations
- Reset state machines
- Reinitialize resources
- Reestablish connections

##### getCompatibilityLevel()
Return your compatibility level with NoMoreCTD.

**Levels:**
- `HIGH` - Fully compatible, all features work
- `MEDIUM` - Compatible with minor issues
- `LOW` - Limited compatibility, some features may not work
- `INCOMPATIBLE` - Not compatible

##### getDisabledFeatures()
Return array of NoMoreCTD features that should be disabled for compatibility.

**Example:**
```java
@Override
public String[] getDisabledFeatures() {
    return new String[]{"hot_reload", "performance_monitor"};
}
```

---

### PerformanceData

Data class containing performance metrics.

```java
public class PerformanceData {
    public double getAverageFps()
    public long getMemoryUsed()
    public long getMemoryTotal()
    public double getMemoryUsagePercent()
    public int getTickRate()
    public int getEntityCount()
    public int getChunkCount()
}
```

#### Example Usage

```java
PerformanceData perf = api.getPerformanceMetrics();

if (perf.getAverageFps() < 30) {
    LOGGER.warn("Low FPS detected: {}", perf.getAverageFps());
    reduceLOD();
}

if (perf.getMemoryUsagePercent() > 80) {
    LOGGER.warn("High memory usage: {}%", perf.getMemoryUsagePercent());
    clearCaches();
}

LOGGER.info("Entities: {}, Chunks: {}",
    perf.getEntityCount(), perf.getChunkCount());
```

---

## Inter-Mod Communication (IMC)

You can also interact with NoMoreCTD using Forge's IMC system.

### Sending IMC Messages

```java
import net.minecraftforge.fml.InterModComms;

private void enqueueIMC(InterModEnqueueEvent event) {
    // Register crash handler
    InterModComms.sendTo("nomorectd", "register_crash_handler",
        () -> myCrashHandler);

    // Register compatibility layer
    InterModComms.sendTo("nomorectd", "register_compat_layer",
        () -> myCompatLayer);

    // Request emergency save
    InterModComms.sendTo("nomorectd", "request_emergency_save",
        () -> "Before risky operation");

    // Check compatibility
    InterModComms.sendTo("nomorectd", "check_compatibility",
        () -> "mymod");
}
```

### IMC Methods

| Method | Parameter Type | Description |
|--------|---------------|-------------|
| `register_crash_handler` | ICrashHandler | Register crash handler |
| `register_compat_layer` | ICompatibilityLayer | Register compatibility layer |
| `request_emergency_save` | String (reason) | Request emergency save |
| `check_compatibility` | String (modId) | Check compatibility |

---

## Advanced Examples

### Example 1: Complex Crash Handler

```java
public class MyModCrashHandler implements ICrashHandler {
    private int crashCount = 0;
    private long lastCrash = 0;

    @Override
    public boolean handleCrash(Throwable exception, CrashContext context) {
        long now = System.currentTimeMillis();

        // Track crash frequency
        if (now - lastCrash < 5000) { // 5 seconds
            crashCount++;
        } else {
            crashCount = 1;
        }
        lastCrash = now;

        // Too many crashes - don't handle
        if (crashCount > 3) {
            LOGGER.error("Too many crashes, letting game crash");
            return false;
        }

        // Handle specific exceptions
        if (exception instanceof MyModException) {
            MyModException mme = (MyModException) exception;

            switch (mme.getSeverity()) {
                case CRITICAL:
                    performEmergencyShutdown();
                    return false; // Can't recover

                case HIGH:
                    resetSubsystem(mme.getSubsystem());
                    return true; // Recovered

                case MEDIUM:
                    clearCache(mme.getCache());
                    return true; // Recovered

                case LOW:
                    logWarning(mme);
                    return true; // Ignored
            }
        }

        return false;
    }
}
```

### Example 2: Full Compatibility Layer

```java
public class MyModCompatibility implements ICompatibilityLayer {
    private boolean initialized = false;
    private final Set<String> activeConnections = new HashSet<>();

    @Override
    public void onInit() {
        LOGGER.info("Initializing MyMod compatibility with NoMoreCTD");

        // Register for NoMoreCTD events
        NoMoreCTDAPI.getAPI().ifPresent(api -> {
            api.registerCrashHandler("mymod", new MyModCrashHandler());
        });

        initialized = true;
    }

    @Override
    public boolean canHotReload() {
        // Check for active operations
        if (!activeConnections.isEmpty()) {
            LOGGER.warn("Cannot hot-reload: {} active connections",
                activeConnections.size());
            return false;
        }

        if (MyMod.getInstance().hasUnsavedData()) {
            LOGGER.warn("Cannot hot-reload: unsaved data present");
            return false;
        }

        return true;
    }

    @Override
    public void onHotReload() {
        LOGGER.info("MyMod hot-reload triggered");

        // Clean up resources
        MyMod.getInstance().clearCaches();
        MyMod.getInstance().reloadConfigs();

        // Reinitialize systems
        MyMod.getInstance().reinitializeNetworking();
        MyMod.getInstance().resetState();

        LOGGER.info("MyMod hot-reload complete");
    }

    @Override
    public CompatibilityLevel getCompatibilityLevel() {
        // Check system state
        if (!initialized) {
            return CompatibilityLevel.LOW;
        }

        if (MyMod.getInstance().hasKnownIssues()) {
            return CompatibilityLevel.MEDIUM;
        }

        return CompatibilityLevel.HIGH;
    }

    @Override
    public String[] getDisabledFeatures() {
        List<String> disabled = new ArrayList<>();

        // Disable features based on config or state
        if (!MyMod.CONFIG.allowPerformanceMonitoring) {
            disabled.add("performance_monitor");
        }

        if (!MyMod.CONFIG.allowHotReload) {
            disabled.add("hot_reload");
        }

        return disabled.toArray(new String[0]);
    }
}
```

### Example 3: Performance Monitoring Integration

```java
public class MyModPerformanceIntegration {
    private INoMoreCTDAPI api;
    private Timer monitorTimer;

    public void init() {
        NoMoreCTDAPI.getAPI().ifPresent(nomorectd -> {
            this.api = nomorectd;
            startMonitoring();
        });
    }

    private void startMonitoring() {
        monitorTimer = new Timer("MyMod-Performance", true);
        monitorTimer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                PerformanceData perf = api.getPerformanceMetrics();

                // Adjust quality based on performance
                if (perf.getAverageFps() < 30) {
                    MyMod.getInstance().decreaseQuality();
                } else if (perf.getAverageFps() > 60) {
                    MyMod.getInstance().increaseQuality();
                }

                // Memory management
                if (perf.getMemoryUsagePercent() > 85) {
                    MyMod.getInstance().aggressiveCacheClearing();
                }

                // Entity optimization
                if (perf.getEntityCount() > 1000) {
                    MyMod.getInstance().enableEntityCulling();
                }
            }
        }, 5000, 5000); // Every 5 seconds
    }

    public void shutdown() {
        if (monitorTimer != null) {
            monitorTimer.cancel();
        }
    }
}
```

---

## Best Practices

### 1. Always Check Availability
```java
// Good
NoMoreCTDAPI.getAPI().ifPresent(api -> {
    // Use API
});

// Also good
if (ModList.get().isLoaded("nomorectd")) {
    NoMoreCTDAPI.getAPI().ifPresent(api -> {
        // Use API
    });
}
```

### 2. Handle Failures Gracefully
```java
try {
    api.registerCrashHandler("mymod", handler);
} catch (Exception e) {
    LOGGER.error("Failed to register crash handler", e);
    // Continue without NoMoreCTD integration
}
```

### 3. Clean Up on Shutdown
```java
@SubscribeEvent
public void onServerStopping(ServerStoppingEvent event) {
    NoMoreCTDAPI.getAPI().ifPresent(api -> {
        api.unregisterCrashHandler("mymod");
    });
}
```

### 4. Use Compatibility Layers for Complex Mods
If your mod has complex state or resources, always implement ICompatibilityLayer.

### 5. Test Hot-Reload Thoroughly
If you claim hot-reload support, test it extensively:
- Test with active operations
- Test with loaded resources
- Test with network connections
- Test with scheduled tasks

---

## Troubleshooting

### API Not Available
**Cause:** NoMoreCTD not loaded or wrong version

**Solution:**
```java
if (!ModList.get().isLoaded("nomorectd")) {
    LOGGER.warn("NoMoreCTD not available, integration disabled");
    return;
}
```

### Crash Handler Not Called
**Cause:** Exception occurs before handler registration or in different thread

**Solution:** Register handlers as early as possible (during mod construction or FMLCommonSetupEvent)

### Hot-Reload Causes Issues
**Cause:** Improper cleanup in onHotReload()

**Solution:** Implement proper cleanup:
- Clear all caches
- Cancel scheduled tasks
- Close connections
- Reset state variables

---

## Version Compatibility

| NoMoreCTD Version | API Version | Minecraft Version |
|------------------|-------------|-------------------|
| 1.5.0+           | 1.0.0       | 1.20.1            |

---

## Support

- GitHub: https://github.com/yourusername/NoMoreCTD
- Discord: [Your Discord]
- Wiki: [Your Wiki]

---

**Last Updated:** 2025-12-19
**API Version:** 1.0.0
**NoMoreCTD Version:** 1.5.0+
