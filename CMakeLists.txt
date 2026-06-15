# ----- Setup -----

cmake_minimum_required(VERSION 3.24)
project(mesa-kosmickrisp)

if (NOT APPLE)
    message(FATAL_ERROR "Unsupported operating system")
endif()

# Determine host architecture.
if (CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "(aarch64)|(AARCH64)|(arm64)|(ARM64)")
    set(KK_TOOLS_ARCHITECTURE "aarch64")
else()
    message(FATAL_ERROR "Unsupported host CPU architecture: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()

# Determine where to source architecture from.
if (CMAKE_OSX_ARCHITECTURES)
    set(KK_BASE_TARGET_ARCHITECTURE "${CMAKE_OSX_ARCHITECTURES}")
elseif (CMAKE_SYSTEM_PROCESSOR)
    set(KK_BASE_TARGET_ARCHITECTURE "${CMAKE_SYSTEM_PROCESSOR}")
else()
    set(KK_BASE_TARGET_ARCHITECTURE "${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()

# Determine target architecture.
if (KK_BASE_TARGET_ARCHITECTURE MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
    set(KK_TARGET_ARCHITECTURE "x86_64")
elseif (KK_BASE_TARGET_ARCHITECTURE MATCHES "(aarch64)|(AARCH64)|(arm64)|(ARM64)")
    set(KK_TARGET_ARCHITECTURE "aarch64")
else()
    message(FATAL_ERROR "Unsupported build CPU architecture: ${KK_BASE_TARGET_ARCHITECTURE}")
endif()

if (NOT DEFINED CMAKE_OSX_DEPLOYMENT_TARGET)
    # Minimim requirement for KosmicKrisp
    set(CMAKE_OSX_DEPLOYMENT_TARGET 26.0)
endif()

include(ExternalProject)

# ----- mesa -----

# Common configuration
set(MESA_CROSSFILES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/crossfiles)
set(MESA_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/externals/mesa)
set(MESA_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/externals/mesa)
set(MESA_COMMON_CONFIGURE_ARGS
    -Dplatforms=macos
    -Dvulkan-drivers=kosmickrisp
    -Dgallium-drivers=
    -Dopengl=false
    -Dzstd=disabled)

# Tools build configuration
set(MESA_BUILD_TYPE_TOOLS "release")
set(MESA_PROJECT_NAME_TOOLS kosmickrisp-mesa-tools)
set(MESA_TARGETS_TOOLS mesa_clc vtn_bindgen2 kk_clc)
set(MESA_CROSS_FILE_TOOLS ${MESA_CROSSFILES_DIR}/mesa-${KK_TOOLS_ARCHITECTURE}.ini)
set(MESA_BINARY_DIR_TOOLS ${MESA_BINARY_DIR}/build-tools)
set(MESA_CLC_DIR_TOOLS "${MESA_BINARY_DIR_TOOLS}/src/compiler/clc")
set(MESA_VTN_BINDGEN2_DIR_TOOLS "${MESA_BINARY_DIR_TOOLS}/src/compiler/spirv")
set(MESA_KK_CLC_DIR_TOOLS "${MESA_BINARY_DIR_TOOLS}/src/kosmickrisp/clc")
SET(MESA_BYPRODUCTS_TOOLS
    "${MESA_CLC_DIR_TOOLS}/kk_clc"
    "${MESA_VTN_BINDGEN2_DIR_TOOLS}/mesa_clc"
    "${MESA_KK_CLC_DIR_TOOLS}/vtn_bindgen2")

# Target build configuration
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(MESA_BUILD_TYPE_TARGET "debug")
else()
    set(MESA_BUILD_TYPE_TARGET "release")
endif()
set(MESA_PROJECT_NAME_TARGET kosmickrisp-mesa-target)
set(MESA_TARGETS_TARGET vulkan_kosmickrisp kosmickrisp_mesa_icd)
set(MESA_CROSS_FILE_TARGET ${MESA_CROSSFILES_DIR}/mesa-${KK_TARGET_ARCHITECTURE}.ini)
set(MESA_BINARY_DIR_TARGET ${MESA_BINARY_DIR}/build-target)
set(MESA_DRIVER_FILE_TARGET "${MESA_BINARY_DIR_TARGET}/src/kosmickrisp/vulkan/libvulkan_kosmickrisp.dylib")
SET(MESA_ICD_FILE_TARGET "${MESA_BINARY_DIR_TARGET}/src/kosmickrisp/vulkan/kosmickrisp_mesa_icd.${KK_TARGET_ARCHITECTURE}.json")
set(MESA_BYPRODUCTS_TARGET
    ${MESA_DRIVER_FILE_TARGET}
    ${MESA_ICD_FILE_TARGET})
set(MESA_ENVIRONMENT_TARGET
    MACOSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}
    PATH=${MESA_CLC_DIR_TOOLS}:${MESA_VTN_BINDGEN2_DIR_TOOLS}:${MESA_KK_CLC_DIR_TOOLS}:$ENV{PATH})

# Build native tools for use when compiling.
ExternalProject_Add(${MESA_PROJECT_NAME_TOOLS}
    SOURCE_DIR ${MESA_SOURCE_DIR}
    BINARY_DIR ${MESA_BINARY_DIR_TOOLS}
    CONFIGURE_COMMAND meson setup ${MESA_BINARY_DIR_TOOLS} ${MESA_SOURCE_DIR}
        --buildtype=${MESA_BUILD_TYPE_TOOLS}
        --cross-file=${MESA_CROSS_FILE_TOOLS}
        -Dinstall-mesa-clc=true -Dinstall-precomp-compiler=true
        ${MESA_COMMON_CONFIGURE_ARGS}
    BUILD_COMMAND meson compile -C ${MESA_BINARY_DIR_TOOLS} ${MESA_TARGETS_TOOLS}
    INSTALL_COMMAND cmake -E echo "Skipping install step."
    BUILD_BYPRODUCTS ${MESA_BYPRODUCTS_TOOLS}
)

# Build for the desired target using the compiled native tools.
ExternalProject_Add(${MESA_PROJECT_NAME_TARGET}
    SOURCE_DIR ${MESA_SOURCE_DIR}
    BINARY_DIR ${MESA_BINARY_DIR_TARGET}
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E env ${MESA_ENVIRONMENT_TARGET} --
        meson setup ${MESA_BINARY_DIR_TARGET} ${MESA_SOURCE_DIR}
        --buildtype=${MESA_BUILD_TYPE_TARGET}
        --cross-file=${MESA_CROSS_FILE_TARGET}
        --prefer-static
        -Dmesa-clc=system -Dprecomp-compiler=system -Dspirv-tools=disabled
        ${MESA_COMMON_CONFIGURE_ARGS}
    BUILD_COMMAND ${CMAKE_COMMAND} -E env ${MESA_ENVIRONMENT_TARGET} --
        meson compile -C ${MESA_BINARY_DIR_TARGET} ${MESA_TARGETS_TARGET}
    INSTALL_COMMAND cmake -E echo "Skipping install step."
    BUILD_BYPRODUCTS ${MESA_BYPRODUCTS_TARGET}
    DEPENDS ${MESA_PROJECT_NAME_TOOLS}
)

# ----- Outputs -----

set(OUTPUTS_DIR ${CMAKE_CURRENT_BINARY_DIR}/outputs)
set(OUTPUT_DRIVER ${OUTPUTS_DIR}/libvulkan_kosmickrisp.dylib)
set(OUTPUT_ICD ${OUTPUTS_DIR}/kosmickrisp_mesa_icd.json)

add_custom_command(
    OUTPUT ${OUTPUTS_DIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${OUTPUTS_DIR})
add_custom_command(
    OUTPUT ${OUTPUT_DRIVER}
    DEPENDS ${OUTPUTS_DIR} ${MESA_PROJECT_NAME_TARGET}
    COMMAND ${CMAKE_COMMAND} -E copy ${MESA_DRIVER_FILE_TARGET} ${OUTPUT_DRIVER})
add_custom_command(
    OUTPUT ${OUTPUT_ICD}
    DEPENDS ${OUTPUTS_DIR} ${MESA_PROJECT_NAME_TARGET}
    COMMAND sed 's/\\/.*\\/libvulkan_kosmickrisp.dylib/.\\/libvulkan_kosmickrisp.dylib/g' ${MESA_ICD_FILE_TARGET} > ${OUTPUT_ICD})

add_custom_target(kosmickrisp ALL DEPENDS ${OUTPUT_DRIVER} ${OUTPUT_ICD})
