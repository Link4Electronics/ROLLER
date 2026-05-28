cmake_minimum_required(VERSION 3.0...4.0)
project(roller C)

set(CMAKE_COMPILE_WARNING_AS_ERROR "OFF" CACHE BOOL "Treat warnings as errors")

if(CMAKE_C_COMPILER_ID MATCHES "AppleClang|Clang|GNU")
    add_compile_options(-Wall)
    add_compile_options(-Wextra)
    add_compile_options(-Wno-unused-parameter)
    add_compile_options(-Wno-unused-variable)
    add_compile_options(-Wno-unused-but-set-variable)
endif()

# --- macOS / Xcode / PowerPC support -----------------------------------------
option(ROLLER_MACOS_BUNDLE "Create macOS .app bundle" OFF)

if(APPLE)
    # Prefer dylibs over frameworks when both are available
    set(CMAKE_FIND_FRAMEWORK LAST)

    if(ROLLER_MACOS_BUNDLE)
        set(ROLLER_BUNDLE_NAME "${PROJECT_NAME}.app")
        set(ROLLER_BUNDLE_DIR "${CMAKE_BINARY_DIR}/${ROLLER_BUNDLE_NAME}")
    endif()

    # Detect PPC architecture (ppc or ppc64)
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(powerpc|ppc)")
        set(ROLLER_ARCH_PPC TRUE)
        message(STATUS "Roller: detected PowerPC (big-endian) architecture")
    elseif(CMAKE_OSX_ARCHITECTURES MATCHES "^(ppc|ppc64)")
        set(ROLLER_ARCH_PPC TRUE)
        message(STATUS "Roller: targeting PowerPC (big-endian) via Xcode")
    else()
        set(ROLLER_ARCH_PPC FALSE)
    endif()

    # Xcode generator: set SDK and PPC architecture when requested
    if(CMAKE_GENERATOR STREQUAL "Xcode")
        if(NOT CMAKE_OSX_ARCHITECTURES AND ROLLER_ARCH_PPC)
            set(CMAKE_OSX_ARCHITECTURES "ppc64" CACHE STRING "macOS architecture for Xcode")
        endif()
        if(NOT CMAKE_OSX_DEPLOYMENT_TARGET)
            set(CMAKE_OSX_DEPLOYMENT_TARGET "10.5" CACHE STRING "macOS deployment target")
        endif()
    endif()

    # Pass -arch flag for PPC when building outside Xcode
    if(ROLLER_ARCH_PPC AND NOT CMAKE_GENERATOR STREQUAL "Xcode")
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "ppc64")
            # Native PPC64 build — already correct
        elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "ppc")
            # Native PPC32 build — already correct
        endif()
    endif()

    # Frameworks may not be found via pkg-config on macOS
    # SDL3, SDL3_image, WildMidi, libcdio should be installed via
    # Homebrew, MacPorts, or built from source and placed in CMAKE_PREFIX_PATH.
    if(NOT CMAKE_PREFIX_PATH)
        message(STATUS "Roller: set CMAKE_PREFIX_PATH to point to your dependency install root")
    endif()
endif()

if(APPLE AND ROLLER_MACOS_BUNDLE)
    # Build output goes inside the .app bundle
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${ROLLER_BUNDLE_DIR}/Contents/MacOS")
endif()

# --- Dependencies -------------------------------------------------------------
find_package(SDL3 REQUIRED)
find_package(SDL3_image REQUIRED)
find_package(WildMidi REQUIRED)

if(UNIX AND NOT APPLE)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(LIBCDIO REQUIRED libcdio libiso9660)
    if(NOT TARGET libcdio::cdio)
        add_library(libcdio::cdio INTERFACE IMPORTED)
        set_target_properties(libcdio::cdio PROPERTIES
            INTERFACE_INCLUDE_DIRECTORIES "${LIBCDIO_INCLUDE_DIRS}"
            INTERFACE_LINK_LIBRARIES "${LIBCDIO_LIBRARIES}")
    endif()
else()
    find_package(libcdio REQUIRED)
endif()

if(WIN32)
    set(PLATFORM_SOURCES ROLLER.rc)
else()
    set(PLATFORM_SOURCES "")
endif()

add_executable(roller
    ${PLATFORM_SOURCES}
    PROJECTS/ROLLER/debug_overlay.c
    PROJECTS/ROLLER/3d.c
    PROJECTS/ROLLER/building.c
    PROJECTS/ROLLER/car.c
    PROJECTS/ROLLER/carplans.c
    PROJECTS/ROLLER/cdx.c
    PROJECTS/ROLLER/colision.c
    PROJECTS/ROLLER/comms.c
    PROJECTS/ROLLER/control.c
    PROJECTS/ROLLER/date.c
    PROJECTS/ROLLER/drawtrk3.c
    PROJECTS/ROLLER/render_queue_3d.c
    PROJECTS/ROLLER/engines.c
    PROJECTS/ROLLER/frontend.c
    PROJECTS/ROLLER/frontend_config.c
    PROJECTS/ROLLER/frontend_data.c
    PROJECTS/ROLLER/frontend_lobby.c
    PROJECTS/ROLLER/frontend_pause.c
    PROJECTS/ROLLER/frontend_screens.c
    PROJECTS/ROLLER/frontend_select_car.c
    PROJECTS/ROLLER/frontend_select_disk.c
    PROJECTS/ROLLER/frontend_select_players.c
    PROJECTS/ROLLER/frontend_select_track.c
    PROJECTS/ROLLER/frontend_select_type.c
    PROJECTS/ROLLER/frontend_util.c
    PROJECTS/ROLLER/menu_render.c
    PROJECTS/ROLLER/menu_render_gpu.c
    PROJECTS/ROLLER/menu_render_software.c
    PROJECTS/ROLLER/game_render.c
    PROJECTS/ROLLER/game_render_software.c
    PROJECTS/ROLLER/func2.c
    PROJECTS/ROLLER/func3.c
    PROJECTS/ROLLER/function.c
    PROJECTS/ROLLER/graphics.c
    PROJECTS/ROLLER/horizon.c
    PROJECTS/ROLLER/loadtrak.c
    PROJECTS/ROLLER/moving.c
    PROJECTS/ROLLER/network.c
    PROJECTS/ROLLER/plans.c
    PROJECTS/ROLLER/png_writer.c
    PROJECTS/ROLLER/polyf.c
    PROJECTS/ROLLER/polytex.c
    PROJECTS/ROLLER/replay.c
    PROJECTS/ROLLER/roller.c
    PROJECTS/ROLLER/rollercd.c
    PROJECTS/ROLLER/rollercomms.c
    PROJECTS/ROLLER/rollersound.c
    PROJECTS/ROLLER/snapshot.c
    PROJECTS/ROLLER/sound.c
    PROJECTS/ROLLER/tower.c
    PROJECTS/ROLLER/transfrm.c
    PROJECTS/ROLLER/userfns.c
    PROJECTS/ROLLER/view.c
    PROJECTS/ROLLER/crashdump.c
    PROJECTS/ROLLER/scene_render.c
    PROJECTS/ROLLER/scene_render_software.c
    PROJECTS/ROLLER/snapshot_scenes.c
)
# --- macOS .app bundle --------------------------------------------------------
if(APPLE AND ROLLER_MACOS_BUNDLE)
    set_target_properties(roller PROPERTIES
        MACOSX_BUNDLE TRUE
        MACOSX_BUNDLE_INFO_PLIST "${CMAKE_SOURCE_DIR}/cmake/Info.plist.in"
        OUTPUT_NAME "ROLLER"
    )
endif()

# --- Xcode-specific attributes ------------------------------------------------
if(CMAKE_GENERATOR STREQUAL "Xcode")
    set_target_properties(roller PROPERTIES
        XCODE_ATTRIBUTE_GCC_OPTIMIZATION_LEVEL "0"
        XCODE_ATTRIBUTE_ONLY_ACTIVE_ARCH "$(ROLLER_ARCH_PPC)"
        XCODE_ATTRIBUTE_MACOSX_DEPLOYMENT_TARGET "${CMAKE_OSX_DEPLOYMENT_TARGET}"
    )
    if(ROLLER_ARCH_PPC)
        set_target_properties(roller PROPERTIES
            XCODE_ATTRIBUTE_ARCHS "$(CMAKE_OSX_ARCHITECTURES)"
            XCODE_ATTRIBUTE_SDKROOT "macosx"
        )
    endif()
endif()

target_include_directories(roller PRIVATE external/Nuklear-4.13.2)
target_link_libraries(roller PRIVATE SDL3::SDL3)
target_link_libraries(roller PRIVATE SDL3_image::SDL3_image)
target_link_libraries(roller PRIVATE WildMidi::libwildmidi)
target_link_libraries(roller PRIVATE libcdio::cdio)

if(NOT DEFINED HOST_BIG_ENDIAN)
    include(TestBigEndian)
    test_big_endian(HOST_BIG_ENDIAN)
endif()
if(HOST_BIG_ENDIAN)
    target_compile_definitions(roller PRIVATE HOST_BIG_ENDIAN=1)
endif()

if(WIN32)
    target_compile_definitions(roller PRIVATE WILDMIDI_STATIC=1)
    target_link_libraries(roller PRIVATE ws2_32 iphlpapi)
    set_target_properties(roller PROPERTIES WIN32_EXECUTABLE TRUE)
else()
    target_compile_definitions(roller PRIVATE
        __int16=int16
        _O_RDONLY=O_RDONLY
        "_O_BINARY=0x200"
    )
endif()
if(APPLE AND ROLLER_MACOS_BUNDLE)
    # Copy midi into bundle Resources
    add_custom_command(TARGET roller POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E copy_directory "${PROJECT_SOURCE_DIR}/midi" "${ROLLER_BUNDLE_DIR}/Contents/Resources/midi"
    )
elseif(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY STREQUAL "${CMAKE_BINARY_DIR}")
    add_custom_command(TARGET roller POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E copy_directory "${PROJECT_SOURCE_DIR}/midi" "$<TARGET_FILE_DIR:roller>/midi"
    )
endif()
find_library(MATH_LIBRARY NAMES "m")
if(MATH_LIBRARY)
    target_link_libraries(roller PRIVATE "m")
endif()
