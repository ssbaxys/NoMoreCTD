# NoMoreCTD Mod Compatibility Guide

## Overview
NoMoreCTD has been designed with maximum compatibility in mind. This guide explains how NoMoreCTD works with other popular mods.

## Compatibility Features

### 1. Mixin Support
NoMoreCTD uses Mixins with low priority to ensure compatibility with other mods. Our mixins are designed to:
- Run after most other mods' mixins
- Use non-intrusive injection points
- Have proper error handling to prevent conflicts

### 2. Event Priority Management
All event handlers use appropriate priorities:
- **LOW priority** for tick events - runs after most mods
- **LOWEST priority** for error detection - captures errors other mods might miss
- Non-cancellable events to avoid breaking other mods' functionality

### 3. Soft Dependencies
NoMoreCTD declares soft dependencies for popular mods, ensuring proper load order:
- Always loads AFTER rendering mods (OptiFine, Rubidium, Embeddium)
- Loads AFTER major content mods (Create, Mekanism, etc.)
- Compatible with utility mods (JEI, Waystones, etc.)

### 4. Inter-Mod Communication (IMC)
Other mods can interact with NoMoreCTD through IMC:

```java
// Register a custom crash handler
InterModComms.sendTo("nomorectd", "register_crash_handler", () -> myCrashHandler);

// Register a compatibility layer
InterModComms.sendTo("nomorectd", "register_compat_layer", () -> myCompatLayer);

// Request an emergency save
InterModComms.sendTo("nomorectd", "request_emergency_save", () -> "reason");
```

### 5. Public API
NoMoreCTD provides a public API for other mods:

```java
import com.nomorectd.api.*;

// Get the API instance
Optional<INoMoreCTDAPI> api = NoMoreCTDAPI.getAPI();

api.ifPresent(nomorectd -> {
    // Register a crash handler
    nomorectd.registerCrashHandler("mymod", (exception, context) -> {
        // Handle crash
        return true; // Return true if handled
    });

    // Check if a feature is enabled
    if (nomorectd.isFeatureEnabled("hot_reload")) {
        // Do something
    }

    // Get performance metrics
    PerformanceData perf = nomorectd.getPerformanceMetrics();
    System.out.println("FPS: " + perf.getAverageFps());
});
```

## Compatible Mods

### ✅ Fully Compatible (HIGH)

#### Rendering & Performance
- **OptiFine** - Rendering optimizer (don't use with Rubidium/Embeddium)
- **Rubidium** - Sodium port for Forge (don't use with OptiFine/Embeddium)
- **Embeddium** - Alternative Sodium port (don't use with OptiFine/Rubidium)
- **Oculus** - Shader support for Rubidium/Embeddium
- **FerriteCore** - Memory optimization
- **ModernFix** - Performance improvements
- **EntityCulling** - Performance optimization
- **LazyDFU** - Faster startup times
- **Spark** - Performance profiler

#### Major Content Mods
- **Create** - Mechanical engineering
- **Mekanism** - Technology and machinery
- **Immersive Engineering** - Industrial technology
- **Applied Energistics 2** - Storage and automation
- **Botania** - Magic and nature
- **Twilight Forest** - Dimension mod

#### Utility Mods
- **Just Enough Items (JEI)** - Recipe viewer
- **Waystones** - Teleportation
- **JourneyMap** - Mapping
- **Xaero's Minimap** - Minimap
- **Curios API** - Equipment slots
- **Patchouli** - Guide books

#### Storage & Backpacks
- **Sophisticated Backpacks** - Advanced backpacks
- **Sophisticated Storage** - Advanced storage

#### Animals & Mobs
- **Alex's Mobs** - New creatures
- **GeckoLib** - Animation library (hot-reload may cause issues)

### ⚠️ Compatible with Warnings (MEDIUM)

#### Crash Handling Mods
- **BetterCrashes** - Similar functionality, may conflict
  - *Recommendation:* Use only one crash handler mod
  - *Note:* Can coexist but may have duplicate features

- **Crash Utilities** - Similar functionality, may conflict
  - *Recommendation:* Use only one crash handler mod
  - *Note:* Can coexist but may have duplicate features

### ❌ Known Conflicts

#### Multiple Rendering Optimizers
**CONFLICT:** Do not use multiple rendering optimizers together!
- ❌ OptiFine + Rubidium
- ❌ OptiFine + Embeddium
- ❌ Rubidium + Embeddium

**Solution:** Choose ONE:
- OptiFine (for vanilla-like experience)
- Rubidium/Embeddium + Oculus (for better performance and shader support)

## Hot-Reload Compatibility

Some mods may require special handling during hot-reload:

### Safe to Hot-Reload
- Most utility mods (JEI, Waystones, etc.)
- Performance mods (Spark, ModernFix, etc.)
- Simple content mods

### Requires Caution
- **GeckoLib mods** - May cause animation glitches after hot-reload
- **Large content mods** - May require full restart (Create, Mekanism, etc.)
- **Shader mods** - May need resource reload after hot-reload

### Not Recommended for Hot-Reload
- Core library mods
- Mods that modify world generation
- Mods with complex native code integration

## Troubleshooting

### Issue: Rendering glitches with OptiFine/Rubidium
**Solution:** Ensure you're using only ONE rendering optimizer

### Issue: Crash handler conflicts
**Solution:** Disable other crash handling mods or disable conflicting features in NoMoreCTD config

### Issue: Hot-reload causes animation issues
**Solution:** Restart the game for mods using GeckoLib or similar animation libraries

### Issue: Performance monitoring conflicts with Spark
**Solution:** Both can coexist - they monitor different aspects

## For Mod Developers

### Adding NoMoreCTD Support to Your Mod

1. **Add to build.gradle:**
```gradle
repositories {
    maven { url = "https://maven.yourrepo.com" }
}

dependencies {
    compileOnly "com.nomorectd:nomorectd:${nomorectd_version}"
}
```

2. **Implement compatibility layer:**
```java
public class MyModCompat implements ICompatibilityLayer {
    @Override
    public void onInit() {
        // Initialize compatibility
    }

    @Override
    public boolean canHotReload() {
        return true; // or false if your mod can't handle hot-reload
    }

    @Override
    public void onHotReload() {
        // Cleanup/reset state after hot-reload
    }

    @Override
    public CompatibilityLevel getCompatibilityLevel() {
        return CompatibilityLevel.HIGH;
    }
}
```

3. **Register via IMC:**
```java
private void enqueueIMC(InterModEnqueueEvent event) {
    InterModComms.sendTo("nomorectd", "register_compat_layer", MyModCompat::new);
}
```

### API Usage Examples

```java
// Check if NoMoreCTD is loaded
if (ModList.get().isLoaded("nomorectd")) {
    NoMoreCTDAPI.getAPI().ifPresent(api -> {
        // Your integration code
    });
}

// Register crash handler for specific errors
api.registerCrashHandler("mymod", (exception, context) -> {
    if (exception instanceof MyCustomException) {
        MyMod.LOGGER.error("Handled crash: {}", exception.getMessage());
        return true; // Crash handled
    }
    return false; // Let NoMoreCTD handle it
});

// Request emergency save before risky operation
api.requestEmergencySave();
try {
    dangerousOperation();
} catch (Exception e) {
    // Handle error
}
```

## Reporting Compatibility Issues

If you encounter compatibility issues:

1. Check this document for known conflicts
2. Check the NoMoreCTD logs for compatibility warnings
3. Try disabling conflicting features in the config
4. Report issues at: https://github.com/yourusername/NoMoreCTD/issues

Include:
- List of all installed mods
- NoMoreCTD version
- Minecraft version
- Crash report or log file
- Steps to reproduce

## Version History

### v1.5.0+
- Added Mixin support
- Added public API
- Added IMC support
- Added compatibility checker
- Added access transformers
- Improved event priority management
- Added soft dependencies for 20+ popular mods

---

**Last Updated:** 2025-12-19
**NoMoreCTD Version:** 1.5.0+
**Minecraft Version:** 1.20.1
