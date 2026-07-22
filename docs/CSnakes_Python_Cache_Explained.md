# CSnakes Python Cache: How It Works and Known Limitations

## How CSnakes Downloads and Caches Python

### The FromRedistributable() Approach

When you use `FromRedistributable()`, CSnakes:

1. **Checks if cache exists** - Looks for folder at cache location
2. **Downloads if missing** - Fetches from [python-build-standalone](https://github.com/astral-sh/python-build-standalone)
3. **Extracts to cache** - Decompresses `.tar.zst` archive
4. **Uses cached version** - On subsequent runs, skips download

### Cache Location

**Default location (Windows):**
```
C:\Users\{username}\AppData\Roaming\CSnakes\python{version}\python\install\
```

**Example:**
```
C:\Users\soren\AppData\Roaming\CSnakes\python3.12.9\python\install\
```

**Override with environment variable:**
```
set CSNAKES_REDIST_CACHE=D:\MyApp\PythonCache
```

Then cache becomes:
```
D:\MyApp\PythonCache\python3.12.9\python\install\
```

---

## Important: Python is NOT in Your Application Directory

### What CSnakes Does NOT Do

CSnakes does **NOT** download Python into your application's directory like a virtual environment would. Instead:

- Python is downloaded to a **shared user-profile location**
- Multiple applications on the same machine share the same cached Python
- This is NOT isolated per-application

### Comparison with Virtual Environments

| Aspect | Virtual Environment | CSnakes Redistributable |
|--------|--------------------|-----------------------|
| Location | `{app}/.venv/` | `%APPDATA%/CSnakes/` |
| Per-application | Yes | No (shared) |
| Isolation | Complete | Shared cache |
| Portable with app | Yes | No |
| Works offline | Yes (after creation) | Yes (after first download) |

### Implications

1. **Deployment**: You cannot simply xcopy deploy your app with Python included
2. **First run**: Requires internet access to download Python (~50-80MB)
3. **Shared state**: If one app corrupts the cache, all apps using that Python version are affected
4. **User-specific**: Cache is tied to user profile, not machine-wide

---

## Service Account Problem

### The Issue

When running as a Windows Service under `LOCAL SYSTEM`, `NETWORK SERVICE`, or a custom service account:

```csharp
Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData)
```

May return:
- **Empty string** - No roaming profile
- **Inaccessible path** - Like `C:\Windows\System32\config\systemprofile\AppData\Roaming\`
- **Non-existent folder** - Profile never created

### What Happens

| Service Account | APPDATA Location | Problem |
|----------------|------------------|---------|
| `LOCAL SYSTEM` | `C:\Windows\System32\config\systemprofile\AppData\Roaming` | May not exist, permission issues |
| `NETWORK SERVICE` | `C:\Windows\ServiceProfiles\NetworkService\AppData\Roaming` | May not exist |
| `LOCAL SERVICE` | `C:\Windows\ServiceProfiles\LocalService\AppData\Roaming` | May not exist |
| Custom service account | Depends on profile creation | May not have roaming profile |

### CSnakes Behavior

CSnakes has **no fallback** if APPDATA is unavailable:
- Will throw exception or fail silently
- No automatic creation of alternative cache location
- No helpful error message explaining the issue

### Workarounds

**Option 1: Set CSNAKES_REDIST_CACHE**

Configure the service with environment variable:
```
CSNAKES_REDIST_CACHE=C:\ProgramData\CSnakes
```

Ensure the service account has read/write permissions to this folder.

**Option 2: Pre-populate cache**

1. Run application as regular user first (creates cache)
2. Copy cache to service-accessible location
3. Set `CSNAKES_REDIST_CACHE` to that location

**Option 3: Use different locator**

Instead of `FromRedistributable()`, use:
```csharp
.FromFolder("C:\\Python\\Python312")  // Explicit path to Python
```

This requires Python to be pre-installed at a known location.

---

## Why the Cache Gets Corrupted

### Known Causes

1. **Interrupted download** - Network failure during download
2. **Interrupted extraction** - Process killed during tar extraction
3. **Disk full** - Extraction fails partway through
4. **Antivirus interference** - Files quarantined during extraction
5. **Permission changes** - Folder made read-only after partial extraction

### Why CSnakes Doesn't Detect Corruption

CSnakes only checks:
```csharp
if (Directory.Exists(folder))
    return; // Assumes cache is valid
```

It does NOT check:
- Required files exist
- File sizes are correct
- Checksums match
- Python actually runs

### The Symptom

When cache is corrupt, Python initialization fails:
```
Fatal Python error: init_fs_encoding: failed to get the Python codec of the filesystem encoding
ModuleNotFoundError: No module named 'encodings'
```

This error provides no indication that the cache is the problem.

---

## Complete Python Cache Structure

A valid CSnakes cache should contain:

```
C:\Users\{user}\AppData\Roaming\CSnakes\python3.12.9\python\install\
+-- DLLs\
|   +-- _asyncio.pyd
|   +-- _bz2.pyd
|   +-- _lzma.pyd
|   +-- _socket.pyd
|   +-- _ssl.pyd
|   +-- libcrypto-3-x64.dll
|   +-- libssl-3-x64.dll
|   +-- select.pyd
|   +-- ... (many more .pyd files)
+-- include\
|   +-- (Python C header files)
+-- Lib\
|   +-- encodings\           <-- CRITICAL: First module loaded
|   |   +-- __init__.py
|   |   +-- utf_8.py
|   |   +-- ... (many encoding modules)
|   +-- site-packages\
|   +-- (entire Python standard library)
+-- libs\
|   +-- python312.lib
|   +-- python3.lib
+-- Scripts\
+-- tcl\
+-- LICENSE.txt
+-- python.exe
+-- python.pdb
+-- python3.dll
+-- python3.pdb
+-- python312.dll
+-- python312.pdb
+-- pythonw.exe
+-- pythonw.pdb
+-- vcruntime140.dll
+-- vcruntime140_1.dll
```

**Key critical components:**
- `Lib/encodings/` - Without this, Python cannot even start
- `python3.dll` and `python{version}.dll` - Core runtime
- `DLLs/` - Extension modules

---

## Recommendations for Production Deployment

### Option A: Use FromRedistributable with Validation

1. Set `CSNAKES_REDIST_CACHE` to a known, accessible location
2. Validate cache integrity before initializing CSnakes
3. Auto-repair (delete) corrupt cache if detected

### Option B: Pre-install Python

1. Install Python on target machine using standard installer
2. Use `FromWindowsInstaller()` or `FromFolder()` instead
3. No runtime download needed

### Option C: Bundle Python with Application

1. Include python-build-standalone in your deployment package
2. Extract to application directory during install
3. Use `FromFolder()` pointing to extracted location
4. Provides true application isolation

### Option D: Docker/Container Deployment

1. Use CSnakes.Stage to pre-create Python environment in container image
2. No runtime download needed
3. Consistent environment across deployments

---

## Summary of CSnakes Limitations

| Limitation | Impact | Workaround |
|------------|--------|------------|
| No cache validation | Corrupt cache causes cryptic errors | Use cache validator (this extension) |
| User-profile cache | Won't work for services | Set CSNAKES_REDIST_CACHE |
| No self-healing | Manual deletion required | Auto-repair on validation failure |
| Shared cache | Multiple apps affected by corruption | Use app-specific cache location |
| Requires internet | First run needs download | Pre-populate cache or bundle Python |
| No checksums | Can't verify download integrity | Trust extraction success |

---

## References

- [CSnakes Documentation](https://tonybaloney.github.io/CSnakes/)
- [CSnakes GitHub](https://github.com/tonybaloney/CSnakes)
- [python-build-standalone](https://github.com/astral-sh/python-build-standalone)
- [CSnakes.Stage for Docker](https://tonybaloney.github.io/CSnakes/v1/user-guide/deployment/)
