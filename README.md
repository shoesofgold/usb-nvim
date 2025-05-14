About USB-Nvim
--------------

- NVIM v0.12.0-dev
- Build type: RelWithDebInfo
- LuaJIT 2.1.1744317938

USB-Nvim is a patch archive to show people how to make Neovim totally portable.
USB-Nvim is self contained withitn a single directory structure; No more scattered config files everywhere on your system.

I am not a developer. I have zero dev experience. If something goes wrong with building or using this repo, then I have no idea how to help you.

Features
--------

- Portability
- Multiple Neovim configurations can be used by simply having multiple copies of USB-Nvim.

- "That's just about the size of it". - Rango

Downsides
---------

- Some folks are left out cuz' I don't have the means (or desire) to build on Apple garbage.
- There are no managed packages on distro repos. I'm not even sure how to go about getting that kind of attention or support.
- I was not able to get a succesful build using MSYS2's UCRT64 shell, so I was unfortunately forced to use VS2022 nonsense.

- "Sometimes life is bigger than means" - Mr. Tsoding

Install from package
--------------------

Pre-built packages for Windows, and Linux are found on the [Releases](https://github.com/shoesofgold/usb-nvim/releases/) page.

Patch layout
--------------

    ├─ src/nvim/        	application source code (see src/nvim/README.md in the official Neovim source)
    │ ├─ os/            	low-level platform code
	│	├─ stdpaths.c		Custom functions are placed inside of #ifdef statements for Windows and Linux.
	│	├─ stdpaths_defs.h	A single line placed to prototype a custom function.
    └─ main.c    	      	A single line placed to call a custom function.

How it works
------------

I've created a custom function for Windows and Linux that gets the executable path of nvim on every launch.
The bin/nvim(.exe) is stripped from the path string, then custom config paths are created from the resulting string relative to that particular nvim instance.

- Inside src/nvim/os/stdpaths_defs.h, place a custom function prototype.
- Starting around line 15:

    `void buildPTH(void);`
    
- Inside src/nvim/main.c, call the custom function from stdpaths_defs.h.
- Starting around line 255:

   `//Build config paths relative to nvim executable
    buildPTH();`

- Inside src/nvim/os/stdpaths.c, comment out the original env defs and place the block of custom code.
- Starting around line 33:

    `#ifdef MSWIN
     #include <Windows.h>
    
    char localPTH[MAXPATHL];
    char tempPTH[MAXPATHL];
    
    void buildPTH(){
      char buf[MAX_PATH];
    
      //use win32 api to get path to executable
      DWORD copied = GetModuleFileName(NULL, buf, (DWORD)sizeof(buf));
      if (copied == 0 || copied >= sizeof(buf)) {
        buf[0] = '\0';
      }
    
      // Find `"bin/nvim.exe"` in the path & truncate
      char *pos = strstr(buf, "bin\\nvim.exe");  // Locate substring
      if (pos != NULL) {
        *pos = '\0';
      }
      //construct path variables relative to nvim executable
      strcpy(localPTH, buf);
      strcpy(tempPTH, buf);
      strcat(localPTH, "local");
      strcat(tempPTH, "local\\temp");
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
    #else
    #include <unistd.h>
    
    char config[MAXPATHL];
    char localShare[MAXPATHL];
    char cache[MAXPATHL];
    char localState[MAXPATHL];
    
    buildPTH(){
    	//use the pid of nvim to get a file path to the executable
    	pid_t pid = getpid();
    	char command[MAXPATHL];
    	char buf[MAXPATHL];
    	char path[MAXPATHL];
    
    	//construct a command string to use with popen
    	sprintf(command, "readlink -f /proc/%d/exe", pid);
    	FILE *p;
    	p = popen(command, "r");
    
    	//pipe popen stream into string variable
    	while (fgets(buf, MAXPATHL, p) != NULL)
    		sprintf(path, "%s", buf);
    	pclose(p);
    
    	//strip "bin/nvim" from executable path
    	char *pos = strstr(path, "bin/nvim");
    	if (pos != NULL) {
    		*pos = '\0';
    	}
    	
    	//construct new paths relative to nvim executable
    	strcpy(config, path);
    	strcpy(localShare, path);
    	strcpy(cache, path);
    	strcpy(localState, path);
    	
    	strcat(config, "linux/.config");
    	strcat(localShare, "linux/.local/share");
    	strcat(cache, "linux/.cache");
    	strcat(localState, "linux/.local/state");
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
    };`

Build from source
-------------------

- Clone the official [Neovim](https://github.com/neovim/neovim.git) repo.
- Copy code from the patch files (or the patch files themselves) into the official source files.
- Follow [Neovim build instructions](https://github.com/neovim/neovim/blob/master/BUILD.md)

The build is CMake-based, but a Makefile is provided as a convenience.
After installing the dependencies, run the following command.

    `make CMAKE_BUILD_TYPE=RelWithDebInfo CMAKE_INSTALL_PREFIX=install`
    `make install`

I recommend using a cmake install prefix:

- `cmake -S cmake.deps -B .deps -G Ninja -D CMAKE_BUILD_TYPE=RelWithDebInfo`
- `cmake --build .deps --config RelWithDebInfo`
- `cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=install/nvim`
- `cmake --build build`
- `cmake --install build`

CMake hints for inspecting the build:

- `cmake --build build --target help` lists all build targets.
- `build/CMakeCache.txt` (or `cmake -LAH build/`) contains the resolved values of all CMake variables.
- `build/compile_commands.json` shows the full compiler invocations for each translation unit.

License
-------

Apache 2.0 license
