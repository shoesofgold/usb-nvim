**About USB-Nvim**
--------------

USB-Nvim is self contained withitn a single directory structure.
No more scattered config files all over hell.

**I am not a developer, I'm just a man with DD titties. If something goes wrong with building or using this repo, then I have no idea how to help you.**

Also, please note that Neovim on Windows is just a bitchy experience anyways; You may find yourself with more problems than just getting this to run.

**Highly experimental, use at your own risk.**

```
Windows config path:
usb-nvim\local\nvim\init.lua

Linux config path:
usb-nvim/home/.config/nvim/init.lua
```

**Features**
--------

- Portability
- Multiple Neovim configurations can be used by simply having multiple copies of USB-Nvim.
- Should be buildable with by any means the official neovim repo is built.

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
    ├─ src/nvim/
      ├─ os/
      │	 ├─ stdpaths.c
      │	 ├─ stdpaths_defs.h
      └─ main.c
```

**How it works**
------------

I've created a custom function for Windows and Linux that gets the executable path of nvim on every launch.
The bin/nvim(.exe) is stripped from the path string, then custom config paths are created from the resulting string relative to that particular nvim instance.

- Inside src/nvim/os/os.h, place a custom function prototype.

```c
//around line 29
//custom function for building relative paths to nvim executable
extern void BuildPth(char *arg0);
```    

- Inside src/nvim/main.c, call the custom function from os.h

```c
//around line 255
//Build config paths relative to nvim executable
BuildPth(argv[0]);
```

- Inside src/nvim/os/stdpaths.c, comment out the original env defs and place the block of custom code.

```c
//around line 18
#ifdef INCLUDE_GENERATED_DECLARATIONS
# include "os/fs.h.generated.h"
# include "os/stdpaths.c.generated.h"
#endif

//around line 33
#ifdef MSWIN

char localPTH[MAXPATHL];
char tempPTH[MAXPATHL];
char nvimPTH[MAXPATHL];

void BuildPth(char* arg0)
{
    char* buf = arg0;


    // Truncate at "bin\nvim.exe"
    char *pos = strstr(buf, "bin");
    if (pos != NULL) {
        *pos = '\0';
    }

    // Construct paths relative to executable
    strcpy(localPTH, buf);
    strcpy(tempPTH, buf);
    strcpy(nvimPTH, buf);

    strcat(localPTH, "local");
    strcat(tempPTH, "local\\temp");
    strcat(nvimPTH, "local\\nvim");

    // Create directories if they don't exist
    os_mkdir_recurse(tempPTH, 0700, NULL, NULL);
    os_mkdir_recurse(nvimPTH, 0700, NULL, NULL);
}

static const char *const xdg_defaults_env_vars[] = {
    [kXDGConfigHome] = localPTH,
    [kXDGDataHome] = localPTH,
    [kXDGCacheHome] = tempPTH,
    [kXDGStateHome] = localPTH,
    [kXDGRuntimeDir] = NULL,  // Decided by vim_mktempdir().
    [kXDGConfigDirs] = NULL,
    [kXDGDataDirs] = NULL,
};

#else  // Linux/Unix

char config[MAXPATHL];
char localShare[MAXPATHL];
char cache[MAXPATHL];
char localState[MAXPATHL];
char nvimPTH[MAXPATHL];

void BuildPth(char* arg0)
{
    char* buf = arg0;

    // Truncate at "bin/nvim"
    char *pos = strstr(buf, "bin");
    if (pos != NULL) {
        *pos = '\0';
    }

    // Construct paths relative to executable
    strcpy(config, buf);
    strcpy(localShare, buf);
    strcpy(cache, buf);
    strcpy(localState, buf);
    strcpy(nvimPTH, buf);

    strcat(config, "home/.config");
    strcat(localShare, "home/.local/share");
    strcat(cache, "home/.local/cache");
    strcat(localState, "home/.local/state");
    strcat(nvimPTH, "home/.config/nvim");

    // Create directories if they don't exist
    os_mkdir_recurse(config, 0700, NULL, NULL);
    os_mkdir_recurse(localShare, 0700, NULL, NULL);
    os_mkdir_recurse(cache, 0700, NULL, NULL);
    os_mkdir_recurse(localState, 0700, NULL, NULL);
    os_mkdir_recurse(nvimPTH, 0700, NULL, NULL);
}

#endif

/// Defaults for XDGVarType values
///
/// Used in case environment variables contain nothing. Need to be expanded.
static const char *const xdg_defaults[] = {
#ifdef MSWIN
    [kXDGConfigHome] = localPTH,
    [kXDGDataHome] = localPTH,
    [kXDGCacheHome] = tempPTH,
    [kXDGStateHome] = localPTH,
    [kXDGRuntimeDir] = NULL,  // Decided by vim_mktempdir().
    [kXDGConfigDirs] = NULL,
    [kXDGDataDirs] = NULL,
    #else
    [kXDGConfigHome] = config,
    [kXDGDataHome] = localShare,
    [kXDGCacheHome] = cache,
    [kXDGStateHome] = localState,
    [kXDGRuntimeDir] = NULL,  // Decided by vim_mktempdir().
    [kXDGConfigDirs] = "/etc/xdg/",
    [kXDGDataDirs] = "/usr/local/share/:/usr/share/",
    #endif
};
```

**Build from source**
-------------------

- Clone the official [Neovim](https://github.com/neovim/neovim.git) repo.
- Copy code from the patch files (or the patch files themselves) into the official source files.
- Follow [Neovim build instructions](https://github.com/neovim/neovim/blob/master/BUILD.md)

**I recommend using a cmake install prefix:**

```cmake
- cmake -S cmake.deps -B .deps -G Ninja -D CMAKE_BUILD_TYPE=RelWithDebInfo
- cmake --build .deps --config RelWithDebInfo
- cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=install/nvim
- cmake --build build
- cmake --install build
```

CMake hints for inspecting the build:

```cmake
- cmake --build build --target help	 lists all build targets.
- build/CMakeCache.txt			 (or `cmake -LAH build/`) contains the resolved values of all CMake variables.
- build/compile_commands.json		 shows the full compiler invocations for each translation unit.
```

**License**
-------

**Apache 2.0 license**
