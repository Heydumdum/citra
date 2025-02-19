# Enable modules to include each other's files
include_directories(.)

# CMake seems to only define _DEBUG on Windows
set_property(DIRECTORY APPEND PROPERTY
    COMPILE_DEFINITIONS $<$<CONFIG:Debug>:_DEBUG> $<$<NOT:$<CONFIG:Debug>>:NDEBUG>)

# Set compilation flags
if (MSVC)
    set(CMAKE_CONFIGURATION_TYPES Debug Release CACHE STRING "" FORCE)

    # Silence "deprecation" warnings
    add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE -D_SCL_SECURE_NO_WARNINGS)

    # Avoid windows.h junk
    add_definitions(-DNOMINMAX)

    # Avoid windows.h from including some usually unused libs like winsocks.h, since this might cause some redefinition errors.
    add_definitions(-DWIN32_LEAN_AND_MEAN)

    # Ensure that projects build with Unicode support.
    add_definitions(-DUNICODE -D_UNICODE)

    # /W3                 - Level 3 warnings
    # /MP                 - Multi-threaded compilation
    # /Zi                 - Output debugging information
    # /Zo                 - Enhanced debug info for optimized builds
    # /permissive-        - Enables stricter C++ standards conformance checks
    # /EHsc               - C++-only exception handling semantics
    # /volatile:iso       - Use strict standards-compliant volatile semantics.
    # /Zc:externConstexpr - Allow extern constexpr variables to have external linkage, like the standard mandates
    # /Zc:inline          - Let codegen omit inline functions in object files
    # /Zc:throwingNew     - Let codegen assume `operator new` (without std::nothrow) will never return null
    # /external:*         - Suppress warnings from external headers
    if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        # Ignore /Zc:externConstexpr /Zc:throwingNew /experimental:external when using clang-cl
        add_compile_options(
            /MP
            /permissive-
            /EHsc
            /volatile:iso
            /Zc:inline
            /external:I "${CMAKE_SOURCE_DIR}/externals"
            /external:anglebrackets
            /external:W0

            # Warnings
            /W3
            /we4062 # enumerator 'identifier' in a switch of enum 'enumeration' is not handled
            /we4101 # 'identifier': unreferenced local variable
            /we4265 # 'class': class has virtual functions, but destructor is not virtual
            /we4267 # 'var': conversion from 'size_t' to 'type', possible loss of data
            /we4388 # signed/unsigned mismatch
            /we4547 # 'operator' : operator before comma has no effect; expected operator with side-effect
            /we4549 # 'operator1': operator before comma has no effect; did you intend 'operator2'?
            /we4555 # Expression has no effect; expected expression with side-effect
            /we4834 # Discarding return value of function with 'nodiscard' attribute
            /we5038 # data member 'member1' will be initialized after data member 'member2'
        )
    else()
        add_compile_options(
            /MP
            /Zo
            /permissive-
            /EHsc
            /volatile:iso
            /Zc:externConstexpr
            /Zc:inline
            /Zc:throwingNew
            /experimental:external
            /external:I "${CMAKE_SOURCE_DIR}/externals"
            /external:anglebrackets
            /external:W0

            # Warnings
            /W3
            /we4062 # enumerator 'identifier' in a switch of enum 'enumeration' is not handled
            /we4101 # 'identifier': unreferenced local variable
            /we4265 # 'class': class has virtual functions, but destructor is not virtual
            /we4267 # 'var': conversion from 'size_t' to 'type', possible loss of data
            /we4388 # signed/unsigned mismatch
            /we4547 # 'operator' : operator before comma has no effect; expected operator with side-effect
            /we4549 # 'operator1': operator before comma has no effect; did you intend 'operator2'?
            /we4555 # Expression has no effect; expected expression with side-effect
            /we4834 # Discarding return value of function with 'nodiscard' attribute
            /we5038 # data member 'member1' will be initialized after data member 'member2'
        )
    endif()

    # Since MSVC's debugging information is not very deterministic, so we have to disable it
    # when using ccache or other caching tools
    if (CITRA_USE_CCACHE OR CITRA_USE_PRECOMPILED_HEADERS)
    # Precompiled headers are deleted if not using /Z7. See https://github.com/nanoant/CMakePCHCompiler/issues/21
        add_compile_options(/Z7)
    else()
        add_compile_options(/Zi)
    endif()

    # /GS- - No stack buffer overflow checks
    add_compile_options("$<$<CONFIG:Release>:/GS->")

    set(CMAKE_EXE_LINKER_FLAGS_DEBUG   "/DEBUG /MANIFEST:NO" CACHE STRING "" FORCE)
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "/DEBUG /MANIFEST:NO /INCREMENTAL:NO /OPT:REF,ICF" CACHE STRING "" FORCE)
else()
    if (NOT IOS)
        # iOS uses Swift for GUI code, but SwiftDriver can't handle -Wall
        add_compile_options(
            -Wall
            -Wno-attributes
        )
    endif()

    if (APPLE AND CMAKE_CXX_COMPILER_ID STREQUAL Clang)
        add_compile_options("-stdlib=libc++")
    endif()

    # Set file offset size to 64 bits.
    #
    # On modern Unixes, this is typically already the case. The lone exception is
    # glibc, which may default to 32 bits. glibc allows this to be configured
    # by setting _FILE_OFFSET_BITS.
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR MINGW)
        add_definitions(-D_FILE_OFFSET_BITS=64)
    endif()

    if (MINGW)
        add_definitions(-DMINGW_HAS_SECURE_API)
        add_compile_options("-Wa,-mbig-obj")
        if (COMPILE_WITH_DWARF)
            add_compile_options("-gdwarf")
        endif()

        if (MINGW_STATIC_BUILD)
            add_definitions(-DQT_STATICPLUGIN)
            add_compile_options("-static")
        endif()
    endif()

    if(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR MINGW)
        # GNU ar: Create thin archive files.
        # Requires binutils-2.19 or later.
        set(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> qcTP <TARGET> <LINK_FLAGS> <OBJECTS>")
        set(CMAKE_C_ARCHIVE_APPEND   "<CMAKE_AR> qTP  <TARGET> <LINK_FLAGS> <OBJECTS>")
        set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> qcTP <TARGET> <LINK_FLAGS> <OBJECTS>")
        set(CMAKE_CXX_ARCHIVE_APPEND "<CMAKE_AR> qTP  <TARGET> <LINK_FLAGS> <OBJECTS>")
    endif()
endif()

add_subdirectory(common)
add_subdirectory(core)
add_subdirectory(video_core)
add_subdirectory(audio_core)
add_subdirectory(network)
add_subdirectory(input_common)
add_subdirectory(tests)

if (ENABLE_SDL2)
    add_subdirectory(citra)
endif()

if (ENABLE_QT)
    add_subdirectory(citra_qt)
endif()

if (ANDROID)
    add_subdirectory(android/app/src/main/jni)
    target_include_directories(citra-android PRIVATE android/app/src/main)
elseif(IOS)
    add_subdirectory(citra_ios)
else()
    add_subdirectory(dedicated_room)
endif()

if (ENABLE_WEB_SERVICE)
    add_subdirectory(web_service)
endif()
