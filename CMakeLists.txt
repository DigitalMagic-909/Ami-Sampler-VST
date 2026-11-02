# Ami Sampler - CMake build with legacy macOS support (down to 10.9 Mavericks)
#
# JUCE 7.0.12 is the last JUCE release that supports a macOS 10.9 deployment
# target (JUCE 8 requires 10.11+), so this build is pinned to it.
#
#   cmake -B build -G Xcode            # or -G Ninja / "Unix Makefiles"
#   cmake --build build --config Release
#
# Use -DJUCE_DIR=/path/to/JUCE to use a local JUCE checkout instead of
# downloading it.

cmake_minimum_required(VERSION 3.22)

# Must be set before project() so it applies to every target.
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9" CACHE STRING "Minimum macOS version")
# Mavericks-era Macs are all Intel. arm64 cannot target anything below 11.0.
set(CMAKE_OSX_ARCHITECTURES "x86_64" CACHE STRING "macOS architectures")
# JUCE builds its "juceaide" helper in a separate CMake run that does not see
# the cache variable above. Without this it targets the build machine's macOS
# (15+), where JUCE 7 no longer compiles (CGWindowListCreateImage obsoleted).
set(ENV{MACOSX_DEPLOYMENT_TARGET} "${CMAKE_OSX_DEPLOYMENT_TARGET}")

project(AmiSampler VERSION 0.7.2 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(JUCE_DIR "" CACHE PATH "Path to a local JUCE 7.0.12 checkout (optional)")
if(JUCE_DIR)
    add_subdirectory("${JUCE_DIR}" JUCE)
else()
    include(FetchContent)
    FetchContent_Declare(JUCE
        GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
        GIT_TAG        7.0.12
        GIT_SHALLOW    TRUE)
    FetchContent_MakeAvailable(JUCE)
endif()

if(APPLE)
    set(AMI_FORMATS VST3 AU Standalone)
else()
    set(AMI_FORMATS VST3 Standalone)
endif()

# Identifiers copied from the official Projucer build, so hosts see this
# as the same plugin.
juce_add_plugin(AmiSampler
    PRODUCT_NAME                "Ami Sampler"
    COMPANY_NAME                "_astriid_"
    BUNDLE_ID                   com.yourcompany.AmiSampler
    PLUGIN_MANUFACTURER_CODE    Manu
    PLUGIN_CODE                 Xwql
    DESCRIPTION                 "Ami Sampler"
    IS_SYNTH                    TRUE
    NEEDS_MIDI_INPUT            TRUE
    NEEDS_MIDI_OUTPUT           TRUE
    IS_MIDI_EFFECT              FALSE
    EDITOR_WANTS_KEYBOARD_FOCUS FALSE
    VST3_CATEGORIES             Instrument Sampler
    AU_MAIN_TYPE                kAudioUnitType_MusicDevice
    AU_EXPORT_PREFIX            AmiSamplerAU
    ICON_BIG                    Res/AmiLogo.png
    ICON_SMALL                  Res/AmiLogo.png
    COPY_PLUGIN_AFTER_BUILD     FALSE
    FORMATS                     ${AMI_FORMATS})

juce_generate_juce_header(AmiSampler)

target_sources(AmiSampler PRIVATE
    Source/astro_formats/astro_BrrAudioFormat.cpp
    Source/astro_formats/astro_IffAudioFormat.cpp
    Source/astro_formats/astro_MuLawFormat.cpp
    Source/AmiWindowEditor.cpp
    Source/AmiAlertWindow.cpp
    Source/ami_font.cpp
    Source/GuiComponent.cpp
    Source/AmiLookAndFeel.cpp
    Source/AmiSamplerSound.cpp
    Source/PixelBuffer.cpp
    Source/PluginEditor.cpp
    Source/PluginProcessor.cpp
    Source/RCFilters.cpp
    Source/SliderComponent.cpp)

juce_add_binary_data(AmiSamplerData SOURCES
    Res/AmiLogo.png
    Res/amidos.ttf
    Res/amiMouseCursor.png
    Res/amiwin1_1.png
    Res/amiwin1_2.png
    Res/amiwin2_1.png
    Res/amiwin2_2.png
    Res/amiwin3_1.png
    Res/amiwin3_2.png
    Res/pixelkey_black.png
    Res/amiTrashOff.png
    Res/amiTrashOn.png
    Res/astriid_amiga.png)

# Same preprocessor settings as ami.jucer
target_compile_definitions(AmiSampler PUBLIC
    JUCE_MODAL_LOOPS_PERMITTED=0
    JUCE_WAV_DO_NOT_PAD_HEADER_SIZE=1
    JUCE_STRICT_REFCOUNTEDPOINTER=1
    JUCE_VST3_CAN_REPLACE_VST2=0
    JUCE_WEB_BROWSER=0
    JUCE_USE_CURL=0
    JUCE_DISPLAY_SPLASH_SCREEN=1
    DONT_SET_USING_JUCE_NAMESPACE=1)

target_link_libraries(AmiSampler
    PRIVATE
        AmiSamplerData
        juce::juce_audio_basics
        juce::juce_audio_devices
        juce::juce_audio_formats
        juce::juce_audio_plugin_client
        juce::juce_audio_processors
        juce::juce_audio_utils
        juce::juce_core
        juce::juce_data_structures
        juce::juce_events
        juce::juce_graphics
        juce::juce_gui_basics
        juce::juce_gui_extra
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_lto_flags)

if(APPLE)
    # Xcode 15/16 ship a new linker that produces binaries which can fail to
    # load on old macOS versions. Use the classic linker when it still exists.
    include(CheckLinkerFlag)
    check_linker_flag(CXX "-Wl,-ld_classic" AMI_HAS_LD_CLASSIC)
    if(AMI_HAS_LD_CLASSIC)
        target_link_options(AmiSampler PUBLIC "-Wl,-ld_classic")
    endif()
    # Report any API use that is newer than the deployment target.
    target_compile_options(AmiSampler PRIVATE -Wunguarded-availability)
endif()
