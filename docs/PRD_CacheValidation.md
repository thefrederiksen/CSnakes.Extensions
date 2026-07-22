# PRD: CSnakes Cache Validation Extension

## Overview

This document describes a proposed extension to validate CSnakes Python redistributable cache integrity before use, preventing cryptic runtime errors when the cache is corrupted or incomplete.

---

## Problem Statement

### Current Behavior

CSnakes' `FromRedistributable()` downloads Python from [python-build-standalone](https://github.com/astral-sh/python-build-standalone) and caches it locally. However:

1. **No integrity validation** - CSnakes only checks if the cache folder exists, not if contents are complete
2. **No self-healing** - Corrupt/incomplete cache stays corrupt forever
3. **Cryptic errors** - Users see `ModuleNotFoundError: No module named 'encodings'` instead of helpful messages
4. **No checksum verification** - Downloaded files are not verified against known hashes

### Real-World Scenario

A user's Python 3.12.9 cache contained only:
```
DLLs/
python3.dll
python312.dll
vcruntime140.dll
```

But should have contained:
```
DLLs/
Lib/              <-- MISSING (contains encodings module)
Scripts/          <-- MISSING
include/          <-- MISSING
libs/             <-- MISSING
tcl/              <-- MISSING
python.exe        <-- MISSING
python3.dll
python312.dll
vcruntime140.dll
LICENSE.txt       <-- MISSING
```

Result: Confusing Python initialization error that provides no guidance on the actual problem.

### Root Cause

CSnakes' `RedistributableLocator` only validates:
```csharp
if (!Directory.Exists(folder))
{
    throw new DirectoryNotFoundException($"Python not found in '{folder}'.");
}
```

This passes even when the folder exists but is incomplete.

---

## Proposed Solution

### CSnakes.Extensions.CacheValidator

A utility class that validates CSnakes cache integrity and provides clear guidance when issues are found.

### Core Functionality

#### 1. ValidateCache Method

```csharp
public static class CSnakesCacheValidator
{
    /// <summary>
    /// Validates the CSnakes Python cache for a specific version.
    /// </summary>
    /// <param name="version">Python version (e.g., "3.12")</param>
    /// <returns>ValidationResult with status and details</returns>
    public static CacheValidationResult ValidateCache(string version);
}
```

#### 2. Required Files/Folders Check

The validator should verify these critical components exist:

| Component | Why It's Critical |
|-----------|------------------|
| `Lib/` | Contains standard library including `encodings` module |
| `Lib/encodings/` | First module loaded during Python initialization |
| `DLLs/` | Contains compiled extension modules |
| `python3.dll` | Core Python runtime |
| `python{version}.dll` | Version-specific runtime (e.g., `python312.dll`) |

#### 3. ValidationResult Class

```csharp
public class CacheValidationResult
{
    public bool IsValid { get; set; }
    public string CachePath { get; set; }
    public string PythonVersion { get; set; }
    public List<string> MissingComponents { get; set; }
    public List<string> Warnings { get; set; }
    public string RecommendedAction { get; set; }
}
```

#### 4. RepairCache Method (Optional)

```csharp
/// <summary>
/// Deletes corrupt cache folder so CSnakes will re-download on next run.
/// </summary>
/// <param name="version">Python version to repair</param>
/// <returns>True if cache was deleted successfully</returns>
public static bool RepairCache(string version);
```

### Usage Example

```csharp
// Before initializing CSnakes, validate the cache
var validation = CSnakesCacheValidator.ValidateCache("3.12");

if (!validation.IsValid)
{
    Console.WriteLine($"CSnakes cache is invalid: {validation.RecommendedAction}");
    Console.WriteLine($"Missing: {string.Join(", ", validation.MissingComponents)}");

    // Optionally auto-repair
    if (CSnakesCacheValidator.RepairCache("3.12"))
    {
        Console.WriteLine("Cache cleared. Python will be re-downloaded on next run.");
    }
}

// Now safe to initialize CSnakes
builder.Services
    .WithPython()
    .WithHome(pythonHome)
    .FromRedistributable("3.12");
```

### Extension Method (Alternative API)

```csharp
public static IPythonEnvironmentBuilder WithCacheValidation(
    this IPythonEnvironmentBuilder builder,
    string version,
    bool autoRepair = false);
```

Usage:
```csharp
builder.Services
    .WithPython()
    .WithHome(pythonHome)
    .WithCacheValidation("3.12", autoRepair: true)  // Validate before use
    .FromRedistributable("3.12");
```

---

## Implementation Details

### Cache Location Detection

The validator must determine cache location using same logic as CSnakes:

```csharp
private static string GetCacheBasePath()
{
    // 1. Check environment variable first
    var envPath = Environment.GetEnvironmentVariable("CSNAKES_REDIST_CACHE");
    if (!string.IsNullOrEmpty(envPath))
        return envPath;

    // 2. Fall back to APPDATA
    return Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData),
        "CSnakes"
    );
}

private static string GetCachePath(string version)
{
    // CSnakes uses format: python{major}.{minor}.{patch}
    // e.g., "python3.12.9"
    return Path.Combine(GetCacheBasePath(), $"python{version}", "python", "install");
}
```

### Validation Checks

```csharp
private static readonly string[] RequiredFolders =
{
    "Lib",
    "Lib/encodings",
    "DLLs"
};

private static readonly string[] RequiredFiles =
{
    "python3.dll"
};

// Version-specific files (checked dynamically)
// e.g., python312.dll for Python 3.12
```

### Platform Considerations

- **Windows**: Check for `.dll` files and Windows-style paths
- **Linux/macOS**: Check for `.so` files and Unix-style paths (future enhancement)

---

## Out of Scope (For Now)

1. **Checksum verification** - Would require maintaining hash database
2. **Automatic re-download** - Would require deeper CSnakes integration
3. **Cross-platform support** - Initial version Windows-only
4. **Virtual environment validation** - Separate concern

---

## Success Criteria

1. Cache validation runs in < 100ms
2. Clear error messages when cache is invalid
3. `RepairCache()` successfully triggers re-download on next CSnakes init
4. No false positives on valid caches
5. Works with all Python versions supported by CSnakes (3.9-3.14)

---

## Future Enhancements

1. **Pre-flight check integration** - Hook into CSnakes startup
2. **Health check endpoint** - For monitoring in production
3. **Cache size reporting** - Help users understand disk usage
4. **Multi-version management** - Clean up old cached versions

---

## References

- [CSnakes GitHub Repository](https://github.com/tonybaloney/CSnakes)
- [CSnakes Issue #551 - encodings module error](https://github.com/tonybaloney/CSnakes/issues/551)
- [CSnakes Issue #318 - Container errors](https://github.com/tonybaloney/CSnakes/issues/318)
- [python-build-standalone](https://github.com/astral-sh/python-build-standalone)
