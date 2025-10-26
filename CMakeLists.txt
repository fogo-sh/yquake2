cmake_minimum_required(VERSION 3.31 FATAL_ERROR)

# Enforce "Debug" as standard build type.
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

# CMake project configuration.
project(yquake2 C)

# Cmake module search path.
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/stuff/cmake/modules ${CMAKE_MODULE_PATH})
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED OFF)

if(YQUAKE2LIBS)
	if(CMAKE_CROSSCOMPILING)
		set(CMAKE_FIND_ROOT_PATH ${YQUAKE2LIBS})
	else()
		set(ENV{CMAKE_PREFIX_PATH} ${YQUAKE2LIBS})
	endif()

    set(ENV{OPENALDIR} ${YQUAKE2LIBS})
endif()

# Add extended path for homebrew on macOS.
list(APPEND CMAKE_PREFIX_PATH /usr/local)
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    # Prefer homebrew-provided packages over system ones
    set(CMAKE_FIND_FRAMEWORK LAST)
    list(APPEND CMAKE_PREFIX_PATH /opt/homebrew /usr/local /opt/local /opt/homebrew/opt/openal-soft /usr/local/opt/openal-soft)
endif()

if (MSVC)
	add_compile_options(/MP) # parallel build (use all cores, or as many as configured in VS)

	# ignore some compiler warnings
	add_compile_options(/wd4244 /wd4305) # possible loss of data/truncation (double to float etc; ignore)
	add_compile_options(/wd4018) # signed/unsigned mismatch
	add_compile_options(/wd4996) # 'function': was declared deprecated (like all that secure CRT stuff)
	# don't show me warnings for system headers, why the fuck isn't this default
	add_compile_options(/experimental:external /external:W0)
else() # GCC/clang/mingw
# Enforce compiler flags:
#  -Wall                -> More warnings
#  -fno-strict-aliasing -> Quake 2 is far away from strict aliasing
#  -fwrapv              -> Make signed integer overflows defined
#  -fvisibility=hidden	-> Force defaultsymbol visibility to hidden
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -fno-strict-aliasing -fwrapv -fvisibility=hidden")

# Use -O2 as maximum optimization level. -O3 has it's problems with yquake2.
string(REPLACE "-O3" "-O2" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
endif() # MSVC'S else-case

# Switch off some annoying warnings
if (${CMAKE_C_COMPILER_ID} STREQUAL "Clang")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-missing-braces")
elseif (${CMAKE_C_COMPILER_ID} STREQUAL "GNU")
	if (CMAKE_C_COMPILER_VERSION GREATER 7.99)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format-truncation -Wno-format-overflow")
	endif()
endif()

# Compilation time options.
option(CURL_SUPPORT "cURL support" ON)
option(OPENAL_SUPPORT "OpenAL support" ON)
option(SYSTEMWIDE_SUPPORT "Enable systemwide installation of game assets" OFF)
option(GL3_RENDERER "Build the GL3 renderer" ON)

set(SYSTEMDIR "" CACHE STRING "Override the system default directory")

# These variables will act as our list of include folders and linker flags.
set(yquake2IncludeDirectories)
set(yquake2LinkerDirectories)
set(yquake2LinkerFlags)
set(yquake2ClientLinkerFlags)
set(yquake2ServerLinkerFlags)
set(yquake2OpenGLLinkerFlags)
set(yquake2VulkanLinkerFlags)
set(yquake2SDLLinkerFlags)
set(yquake2ZLibLinkerFlags)

# Set directory locations (allowing us to move directories easily)
set(SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/src)
set(BACKENDS_SRC_DIR ${SOURCE_DIR}/backends)
set(COMMON_SRC_DIR ${SOURCE_DIR}/common)
set(GAME_SRC_DIR ${SOURCE_DIR}/game)
set(SERVER_SRC_DIR ${SOURCE_DIR}/server)
set(CLIENT_SRC_DIR ${SOURCE_DIR}/client)
set(REF_SRC_DIR ${SOURCE_DIR}/client/refresh)

# Operating system.
set(YQ2OSTYPE "${CMAKE_SYSTEM_NAME}" CACHE STRING "Override operation system type")
add_definitions(-DYQ2OSTYPE="${YQ2OSTYPE}")

# Architecture string
# work around CMake's useless/broken CMAKE_SYSTEM_PROCESSOR (taken from dhewm3)

set(cpu ${CMAKE_SYSTEM_PROCESSOR})

# Originally, ${CMAKE_SYSTEM_PROCESSOR} was supposed to contain the *target* CPU, according to CMake's documentation.
# As far as I can tell this has always been broken (always returns host CPU) at least on Windows
# (see e.g. https://cmake.org/pipermail/cmake-developers/2014-September/011405.html) and wasn't reliable on
# other systems either, for example on Linux with 32bit userland but 64bit kernel it returned the kernel CPU type
# (e.g. x86_64 instead of i686). Instead of fixing this, CMake eventually updated their documentation in 3.20,
# now it's officially the same as CMAKE_HOST_SYSTEM_PROCESSOR except when cross-compiling (where it's explicitly set)
# So we gotta figure out the actual target CPU type ourselves..
if(NOT (CMAKE_SYSTEM_PROCESSOR STREQUAL CMAKE_HOST_SYSTEM_PROCESSOR))
	# special case: cross-compiling, here CMAKE_SYSTEM_PROCESSOR should be correct, hopefully
	# (just leave cpu at ${CMAKE_SYSTEM_PROCESSOR})
elseif(MSVC)
	# because all this wasn't ugly enough, it turned out that, unlike standalone CMake, Visual Studio's
	# integrated CMake doesn't set CMAKE_GENERATOR_PLATFORM, so I gave up on guessing the CPU arch here
	# and moved the CPU detection to MSVC-specific code in neo/sys/platform.h
else() # not MSVC and not cross-compiling, assume GCC or clang (-compatible), seems to work for MinGW as well
	execute_process(COMMAND ${CMAKE_C_COMPILER} "-dumpmachine"
	                RESULT_VARIABLE cc_dumpmachine_res
	                OUTPUT_VARIABLE cc_dumpmachine_out)
	if(cc_dumpmachine_res EQUAL 0)
		string(STRIP ${cc_dumpmachine_out} cc_dumpmachine_out) # get rid of trailing newline
		message(DEBUG "`${CMAKE_C_COMPILER} -dumpmachine` says: \"${cc_dumpmachine_out}\"")
		# gcc -dumpmachine and clang -dumpmachine seem to print something like "x86_64-linux-gnu" (gcc)
		# or "x64_64-pc-linux-gnu" (clang) or "i686-w64-mingw32" (32bit mingw-w64) i.e. starting with the CPU,
		# then "-" and then OS or whatever - so use everything up to first "-"
		string(REGEX MATCH "^[^-]+" cpu ${cc_dumpmachine_out})
		message(DEBUG "  => CPU architecture extracted from that: \"${cpu}\"")
	else()
		message(WARNING "${CMAKE_C_COMPILER} -dumpmachine failed with error (code) ${cc_dumpmachine_res}")
		message(WARNING "will use the (sometimes incorrect) CMAKE_SYSTEM_PROCESSOR (${cpu}) to determine YQ2ARCH")
	endif()
endif()

if(cpu STREQUAL "powerpc")
	set(cpu "ppc")
elseif(cpu STREQUAL "aarch64")
	# "arm64" is more obvious, and some operating systems (like macOS) use it instead of "aarch64"
	set(cpu "arm64")
elseif(cpu MATCHES "[aA][mM][dD]64" OR cpu MATCHES "[xX].*64")
	set(cpu "x86_64")
elseif(cpu MATCHES "i.86" OR cpu MATCHES "[xX]86")
	set(cpu "i386")
elseif(cpu MATCHES "[aA][rR][mM].*") # some kind of arm..
	# On 32bit Raspbian gcc -dumpmachine returns sth starting with "arm-",
	# while clang -dumpmachine says "arm6k-..." - try to unify that to "arm"
	if(CMAKE_SIZEOF_VOID_P EQUAL 8) # sizeof(void*) == 8 => must be arm64
		set(cpu "arm64")
	else() # should be 32bit arm then (probably "armv7l" "armv6k" or sth like that)
		set(cpu "arm")
	endif()
endif()

if(MSVC)
	# for MSVC YQ2ARCH is set in code (in src/common/header/common.h)
	message(STATUS "Setting YQ2OSTYPE to \"${YQ2OSTYPE}\" - NOT setting YQ2ARCH, because we're targeting MSVC (VisualC++)")
else()
	set(ARCH "${cpu}")
	add_definitions(-DYQ2ARCH="${ARCH}")
	message(STATUS "Setting YQ2OSTYPE to \"${YQ2OSTYPE}\" and YQ2ARCH to \"${ARCH}\".")
endif()
# make sure that ${cpu} isn't used below - if at all use ${ARCH}, but not when compiling with MSVC!
unset(cpu)

# END OF workarounds for CMake's poor choices regarding CPU architecture detection


# Systemwide installation of game assets.
if(${SYSTEMWIDE_SUPPORT})
    add_definitions(-DSYSTEMWIDE)
    if(NOT ${SYSTEMDIR} STREQUAL "")
      add_definitions(-DSYSTEMDIR="${SYSTEMDIR}")
    endif()
endif()

# We need to pass some options to minizip / unzip.
add_definitions(-DNOUNCRYPT)

if(NOT (CMAKE_SYSTEM_NAME MATCHES "Linux") AND NOT (CMAKE_SYSTEM_NAME MATCHES "Windows"))
	add_definitions(-DIOAPI_NO_64)
endif()

# Required libraries to build the different components of the binaries. Find
# them and add the include/linker directories and flags (in case the package
# manager find it in a weird place).
find_package(SDL3 REQUIRED)
add_definitions(-DUSE_SDL3)

# backtrace lookup
# Some systems like Linux has it within the libc some like the BSD, Haiku ...
# into an external libexecinfo library
include(CheckFunctionExists)
include(CheckLibraryExists)
check_function_exists(backtrace HAVE_EXECINFO_SYS)
IF (NOT HAVE_EXECINFO_SYS)
	check_library_exists(execinfo backtrace "" HAVE_EXECINFO_LIB)
	if (HAVE_EXECINFO_LIB)
		list(APPEND yquake2ClientLinkerFlags execinfo)
		list(APPEND yquake2ServerLinkerFlags execinfo)
		add_definitions(-DHAVE_EXECINFO)
	endif()
else()
	add_definitions(-DHAVE_EXECINFO)
endif()

# cURL support.
if (${CURL_SUPPORT})
	find_package(CURL REQUIRED)
	include_directories(${CURL_INCLUDE_DIR})
	add_definitions(-DUSE_CURL)
endif()

# OpenAL support.
if(${OPENAL_SUPPORT})
	find_package(OpenAL REQUIRED)

    if(${OPENAL_FOUND})
        # Normalize include dirs: if CMake reports include path ending in
        # 'AL', use its parent so '#include <AL/al.h>' resolves correctly.
        set(_OPENAL_INCLUDES)
        if(DEFINED OPENAL_INCLUDE_DIRS)
            list(APPEND _OPENAL_INCLUDES ${OPENAL_INCLUDE_DIRS})
        elseif(DEFINED OPENAL_INCLUDE_DIR)
            list(APPEND _OPENAL_INCLUDES ${OPENAL_INCLUDE_DIR})
        endif()
        foreach(_inc_dir IN LISTS _OPENAL_INCLUDES)
            get_filename_component(_tail "${_inc_dir}" NAME)
            if(_tail STREQUAL "AL")
                get_filename_component(_parent "${_inc_dir}" DIRECTORY)
                list(APPEND yquake2IncludeDirectories "${_parent}")
            else()
                list(APPEND yquake2IncludeDirectories "${_inc_dir}")
            endif()
        endforeach()

        if(TARGET OpenAL::OpenAL)
            list(APPEND yquake2ClientLinkerFlags OpenAL::OpenAL)
        else()
            list(APPEND yquake2ClientLinkerFlags ${OPENAL_LIBRARY})
        endif()

        if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="openal32.dll")
		elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.dylib")
		elseif((${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD") OR (${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD"))
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.so")
		else()
			add_definitions(-DUSE_OPENAL -DDEFAULT_OPENAL_DRIVER="libopenal.so.1")
		endif()
	endif()
endif()

# General linker flags.
if(NOT MSVC)
	list(APPEND yquake2LinkerFlags m)
endif()
list(APPEND yquake2LinkerFlags ${CMAKE_DL_LIBS})

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	if(!MSVC)
		list(APPEND yquake2LinkerFlags "-static-libgcc")
	endif()
else()
	if (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Haiku")
		list(APPEND yquake2LinkerFlags "-rdynamic")
	else()
		list(APPEND yquake2LinkerFlags "-lnetwork")
                set(CMAKE_POSITION_INDEPENDENT_CODE ON)
	endif()
	if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
		list(APPEND yquake2LinkerFlags "-lsocket -lnsl")
	endif()
endif()

if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD" AND NOT WIN32)
	list(APPEND yquake2LinkerFlags "-Wl,--no-undefined")
endif()

# With all of those libraries and user defined paths
# added, lets give them to the compiler and linker.
include_directories(${yquake2IncludeDirectories})
link_directories(${yquake2LinkerDirectories})

# these settings only work for GCC and clang
if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
	# If we're building with gcc for i386 let's define -ffloat-store.
	# This helps the old and crappy x87 FPU to produce correct values.
	# Would be nice if Clang had something comparable.
	if ("${ARCH}" STREQUAL "i386" AND ${CMAKE_C_COMPILER_ID} STREQUAL "GNU")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffloat-store")
	endif()

	# Force SSE math on x86_64. All sane compilers should do this
	# anyway, just to protect us from broken Linux distros.
	if ("${ARCH}" STREQUAL "x86_64")
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpmath=sse")
	endif()

	if ("${ARCH}" STREQUAL "arm")
		if (CMAKE_SIZEOF_VOID_P EQUAL 4)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=armv6k")
		endif()
	endif()
endif()

add_subdirectory(${BACKENDS_SRC_DIR})


add_subdirectory(${CLIENT_SRC_DIR})
add_subdirectory(${SERVER_SRC_DIR})
## Executables moved to src/client and src/server

add_subdirectory(${GAME_SRC_DIR})


if(${GL3_RENDERER})
	add_subdirectory(${REF_SRC_DIR}/gl3)
endif()
