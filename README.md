**About USB-Nvim**
--------------

USB-Nvim is self contained withitn a single directory structure.
No more scattered config files all over hell.

**I am not a developer, I'm just a man with DD titties. If something goes wrong with building or using this repo, then I have no idea how to help you.**

**Highly experimental, use at your own risk.**

```
Windows config path:
usb-nvim\local\nvim\init.lua

Linux config path:
usb-nvim/.config/nvim/init.lua
```

**Features**
--------

- Portability
- Multiple Neovim configurations can be used by simply having multiple copies of USB-Nvim.
- Should be buildable by any means the official neovim repo is built.

**`"That's just about the size of it." - Rango`**

**Downsides**
---------

- I don't have the means (or desire) to test builds on Apple garbage.
- There are no managed packages on distro repos for auto updates. I'm not even sure how to go about getting that kind of attention or support.

**`"Sometimes life is bigger than means." - Mr. Tsoding`**

**No-Install Packages**
--------------------

Pre-built packages for Windows, and Linux are found on the [**Releases**](https://github.com/shoesofgold/usb-nvim/releases/) page.

**Patch layout**
--------------

```
    ├─ src/nvim/os/
    	 ├─ stdpaths.c
```

**How it works**
------------

A custom function gets the executable path of nvim on every launch.
The bin/nvim(.exe) is stripped from the path string, then custom config paths are created from the resulting string relative to that particular nvim instance.

- Inside src/nvim/os/stdpaths.c, make the changes outlined in these excerpts.

```c
// around line 18
#include "fs.h"

// around line 21
//remove 'const' from the xdg_ arrays
static char *xdg_env_vars[] = {
  [kXDGConfigHome] = "XDG_CONFIG_HOME",
  [kXDGDataHome] = "XDG_DATA_HOME",
  [kXDGCacheHome] = "XDG_CACHE_HOME",
  [kXDGStateHome] = "XDG_STATE_HOME",
  [kXDGRuntimeDir] = "XDG_RUNTIME_DIR",
  [kXDGConfigDirs] = "XDG_CONFIG_DIRS",
  [kXDGDataDirs] = "XDG_DATA_DIRS",
};

#ifdef MSWIN
static char *xdg_defaults_env_vars[] = {
  [kXDGConfigHome] = "LOCALAPPDATA",
  [kXDGDataHome] = "LOCALAPPDATA",
  [kXDGCacheHome] = "TEMP",
  [kXDGStateHome] = "LOCALAPPDATA",
  [kXDGRuntimeDir] = NULL,  // Decided by vim_mktempdir().
  [kXDGConfigDirs] = NULL,
  [kXDGDataDirs] = NULL,
};
#endif

/// Defaults for XDGVarType values
///
/// Used in case environment variables contain nothing. Need to be expanded.
static char *xdg_defaults[] = {
#ifdef MSWIN
  [kXDGConfigHome] = "~\\AppData\\Local",
  [kXDGDataHome] = "~\\AppData\\Local",
  [kXDGCacheHome] = "~\\AppData\\Local\\Temp",
  [kXDGStateHome] = "~\\AppData\\Local",
  [kXDGRuntimeDir] = NULL,  // Decided by vim_mktempdir().
  [kXDGConfigDirs] = NULL,
  [kXDGDataDirs] = NULL,
#else
  [kXDGConfigHome] = "~/.config",
  [kXDGDataHome] = "~/.local/share",
  [kXDGCacheHome] = "~/.cache",
  [kXDGStateHome] = "~/.local/state",
  [kXDGRuntimeDir] = NULL,  // Decided by vim_mktempdir().
  [kXDGConfigDirs] = "/etc/xdg/",
  [kXDGDataDirs] = "/usr/local/share/:/usr/share/",
#endif
};

// around line 150
/////////////////////////////////
//usb-nvim START
/////////////////////////////////
static bool path_parent(char *path) {
  size_t len = strlen(path);
  while (len > 1 && path[len - 1] == '/') {
    path[--len] = '\0';
  }

  char *slash = strrchr(path, '/');
  if (slash == NULL) {
    return false;
  }

#ifdef MSWIN
  // Preserve a drive root, e.g. C:/.
  if (slash == path + 2 && path[1] == ':') {
    slash[1] = '\0';
    return true;
  }
#endif

  if (slash == path) { // Preserve Unix root, /.
    path[1] = '\0';
  } else {
    *slash = '\0';
  }
  return true;
}

/// Return an allocated path under the portable root, e.g. root/local/temp.
static char *portable_path(const char *relative) {
  static bool root_checked;
  static bool root_valid;
  static char root[MAXPATHL];

  if (!root_checked) {
    root_checked = true;

    char exe[MAXPATHL];
    size_t size = sizeof(exe);
    if (os_exepath(exe, &size) == 0) {
#ifdef MSWIN
      for (char *p = exe; *p != '\0'; p++) {
        if (*p == '\\') {
          *p = '/';
        }
      }
#endif
      // THIS IS NOT A TYPO: the path_parent() logic has to run twice!
      // clangd will complain, but it will compile.
      if (path_parent(exe) && path_parent(exe)) {
        xstrlcpy(root, exe, sizeof(root));
        root_valid = true;
      }
    }
  }

  if (!root_valid) {
    return NULL;
  }

  // Returns a newly allocated path; use "/" in relative segments on all
  // platforms.
  return concat_fnames_realloc(xstrdup(root), relative, true);
}

static void init_portable_defaults(void) {
  static bool initialized;
  if (initialized) {
    return;
  }
  initialized = true;

#ifdef MSWIN
  char *config = portable_path("../local");
  char *cache = portable_path("../local/temp");

  if (config != NULL) {
    xdg_defaults[kXDGConfigHome] = config;
    xdg_defaults[kXDGDataHome] = config;
    xdg_defaults[kXDGStateHome] = config;
    xdg_defaults_env_vars[kXDGConfigHome] = config;
    xdg_defaults_env_vars[kXDGDataHome] = config;
    xdg_defaults_env_vars[kXDGStateHome] = config;
  }
  if (cache != NULL) {
    xdg_defaults_env_vars[kXDGCacheHome] = cache;
    xdg_defaults[kXDGCacheHome] = cache;
  }
#else
  char *config = portable_path("../config");
  char *data = portable_path("../local/share");
  char *cache = portable_path("../local/temp");
  char *state = portable_path("../local/state");

  if (config != NULL){
    xdg_defaults[kXDGConfigHome] = config;
    xdg_env_vars[kXDGConfigHome] = config;
}
  if (data != NULL){
    xdg_defaults[kXDGDataHome] = data;
    xdg_env_vars[kXDGDataHome] = data;
    }
  if (cache != NULL){
    xdg_defaults[kXDGCacheHome] = cache;
    xdg_env_vars[kXDGCacheHome] = cache;
    }
  if (state != NULL){
    xdg_defaults[kXDGStateHome] = state;
    xdg_env_vars[kXDGStateHome] = state;
    }
#endif
}
/////////////////////////////////
//usb-nvim END
/////////////////////////////////

char *stdpaths_get_xdg_var(const XDGVarType idx)
  FUNC_ATTR_WARN_UNUSED_RESULT
{
// around line 275
  init_portable_defaults();
// ...
}
```

**Build from source**
-------------------

- Clone the official [Neovim](https://github.com/neovim/neovim.git) repo.
- Copy code from the patch files (or the patch files themselves) into the official source files.
- Follow [Neovim build instructions](https://github.com/neovim/neovim/blob/master/BUILD.md)

**I recommend using a cmake install prefix:**

```cmake
- cmake -S cmake.deps -B .deps -G Ninja -D CMAKE_BUILD_TYPE=Release
- cmake --build .deps --config Release
- cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=install/nvim
- cmake --build build --config Release
- cmake --install build
```

**License**
-------

**Apache 2.0 license**
