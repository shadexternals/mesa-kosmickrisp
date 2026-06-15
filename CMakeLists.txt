# ----- Setup -----

cmake_minimum_required(VERSION 3.24)
project(mesa-kosmickrisp)

if (NOT APPLE)
    message(FATAL_ERROR "Unsupported operating system")
endif()

if (NOT CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "(aarch64)|(AARCH64)|(arm64)|(ARM64)")
    message(FATAL_ERROR "Unsupported host CPU architecture: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()

# First, determine where to source the architectures from.
if (CMAKE_OSX_ARCHITECTURES)
    set(KOSMICKRISP_BASE_ARCHITECTURE "${CMAKE_OSX_ARCHITECTURES}")
elseif (CMAKE_SYSTEM_PROCESSOR)
    set(KOSMICKRISP_BASE_ARCHITECTURE "${CMAKE_SYSTEM_PROCESSOR}")
else()
    set(KOSMICKRISP_BASE_ARCHITECTURE "${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()

# Next, match common architecture strings down to a known common value.
if (KOSMICKRISP_BASE_ARCHITECTURE MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
    set(KOSMICKRISP_ARCHITECTURE "x86_64")
elseif (KOSMICKRISP_BASE_ARCHITECTURE MATCHES "(aarch64)|(AARCH64)|(arm64)|(ARM64)")
    set(KOSMICKRISP_ARCHITECTURE "aarch64")
else()
    message(FATAL_ERROR "Unsupported build CPU architecture: ${KOSMICKRISP_BASE_ARCHITECTURE}")
endif()

cmake_host_system_information(RESULT NUM_CORES QUERY NUMBER_OF_LOGICAL_CORES)

set(INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/install-prefix)

include(ExternalProject)

# ----- SPIRV-Tools -----

set(SPIRV_HEADERS_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/externals/SPIRV-Headers)
set(SPIRV_TOOLS_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/externals/SPIRV-Tools)
set(SPIRV_TOOLS_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/externals/SPIRV-Tools)

ExternalProject_Add(kosmickrisp-SPIRV-Tools
    SOURCE_DIR ${SPIRV_TOOLS_SOURCE_DIR}
    BINARY_DIR ${SPIRV_TOOLS_BINARY_DIR}
    INSTALL_DIR ${INSTALL_PREFIX}
    CONFIGURE_COMMAND cmake -S ${SPIRV_TOOLS_SOURCE_DIR} -B ${SPIRV_TOOLS_BINARY_DIR}
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DCMAKE_INSTALL_PREFIX=${INSTALL_PREFIX}
        -DCMAKE_OSX_ARCHITECTURES=arm64
        -DSPIRV-Headers_SOURCE_DIR=${SPIRV_HEADERS_SOURCE_DIR}
        -DSPIRV_SKIP_EXECUTABLES=ON
    BUILD_COMMAND cmake --build ${SPIRV_TOOLS_BINARY_DIR} --parallel${NUM_CORES}
    INSTALL_COMMAND cmake --install ${SPIRV_TOOLS_BINARY_DIR}
    BUILD_BYPRODUCTS ${INSTALL_PREFIX}/lib/libSPIRV-Tools.a
)

# ----- mesa -----

set(MESA_CROSSFILES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/crossfiles)
set(MESA_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/externals/mesa)
set(MESA_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/externals/mesa)

set(MESA_COMMON_CONFIGURE_ARGS
    --prefer-static
    -Dplatforms=macos
    -Dvulkan-drivers=kosmickrisp
    -Dgallium-drivers=
    -Dopengl=false
    -Dzstd=disabled)

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(MESA_BUILD_TYPE_TARGET "debug")
else()
    set(MESA_BUILD_TYPE_TARGET "release")
endif()

set(MESA_PROJECT_NAME_TARGET kosmickrisp-mesa-${KOSMICKRISP_ARCHITECTURE})
set(MESA_TARGETS_TARGET vulkan_kosmickrisp kosmickrisp_mesa_icd)
set(MESA_BINARY_DIR_TARGET ${MESA_BINARY_DIR}/build-${KOSMICKRISP_ARCHITECTURE})
set(MESA_DRIVER_FILE_TARGET "${MESA_BINARY_DIR_TARGET}/src/kosmickrisp/vulkan/libvulkan_kosmickrisp.dylib")
SET(MESA_ICD_FILE_TARGET "${MESA_BINARY_DIR_TARGET}/src/kosmickrisp/vulkan/kosmickrisp_mesa_icd.${KOSMICKRISP_ARCHITECTURE}.json")
set(MESA_BYPRODUCTS_TARGET
    ${MESA_DRIVER_FILE_AARCH64}
    ${MESA_ICD_FILE_AARCH64})

if (KOSMICKRISP_ARCHITECTURE STREQUAL "aarch64")
    # aarch64 is built as the final target
    set(MESA_BUILD_TYPE_AARCH64 ${MESA_BUILD_TYPE_TARGET})
    set(MESA_PROJECT_NAME_AARCH64 ${MESA_PROJECT_NAME_TARGET})
    set(MESA_TARGETS_AARCH64 ${MESA_TARGETS_TARGET})
    set(MESA_BINARY_DIR_AARCH64 ${MESA_BINARY_DIR_TARGET})
    set(MESA_DRIVER_FILE_AARCH64 ${MESA_DRIVER_FILE_TARGET})
    set(MESA_ICD_FILE_AARCH64 ${MESA_ICD_FILE_TARGET})
    set(MESA_BYPRODUCTS_AARCH64 ${MESA_BYPRODUCTS_TARGET})
else()
    # aarch64 is built for executable tools only
    set(MESA_BUILD_TYPE_AARCH64 "release")
    set(MESA_PROJECT_NAME_AARCH64 kosmickrisp-mesa-aarch64)
    set(MESA_TARGETS_AARCH64 mesa_clc vtn_bindgen2 kk_clc)
    set(MESA_BINARY_DIR_AARCH64 ${MESA_BINARY_DIR}/build-aarch64)
    set(MESA_DRIVER_FILE_AARCH64 "${MESA_BINARY_DIR_AARCH64}/src/kosmickrisp/vulkan/libvulkan_kosmickrisp.dylib")
    SET(MESA_ICD_FILE_AARCH64 "${MESA_BINARY_DIR_AARCH64}/src/kosmickrisp/vulkan/kosmickrisp_mesa_icd.aarch64.json")
    SET(MESA_BYPRODUCTS_AARCH64
        "${INSTALL_PREFIX}/bin/kk_clc"
        "${INSTALL_PREFIX}/bin/mesa_clc"
        "${INSTALL_PREFIX}/bin/vtn_bindgen2")
endif()

# Build either desired target, if aarch64, or the native tools for later.
ExternalProject_Add(${MESA_PROJECT_NAME_AARCH64}
    SOURCE_DIR ${MESA_SOURCE_DIR}
    BINARY_DIR ${MESA_BINARY_DIR_AARCH64}
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E env PKG_CONFIG_PATH=${INSTALL_PREFIX}/lib/pkgconfig --
        meson setup ${MESA_BINARY_DIR_AARCH64} ${MESA_SOURCE_DIR}
        --buildtype=${MESA_BUILD_TYPE_AARCH64}
        --prefix=${INSTALL_PREFIX}
        --cross-file=${MESA_CROSSFILES_DIR}/mesa-aarch64.ini
        -Dinstall-mesa-clc=true -Dinstall-precomp-compiler=true
        ${MESA_COMMON_CONFIGURE_ARGS}
    BUILD_COMMAND meson compile -C ${MESA_BINARY_DIR_AARCH64} ${MESA_TARGETS_AARCH64}
    INSTALL_COMMAND cmake -E echo "Skipping install step."
    BUILD_BYPRODUCTS ${MESA_BYPRODUCTS_AARCH64}
    DEPENDS kosmickrisp-SPIRV-Tools
)

if (NOT KOSMICKRISP_ARCHITECTURE STREQUAL "aarch64")
    set(MESA_CLC_PATH "${MESA_BINARY_DIR_AARCH64}/src/compiler/clc")
    set(MESA_VTN_BINDGEN2_PATH "${MESA_BINARY_DIR_AARCH64}/src/compiler/spirv")
    set(MESA_KK_CLC_PATH "${MESA_BINARY_DIR_AARCH64}/src/kosmickrisp/clc")

    # Build for the desired target using native tools
    ExternalProject_Add(${MESA_PROJECT_NAME_TARGET}
        SOURCE_DIR ${MESA_SOURCE_DIR}
        BINARY_DIR ${MESA_BINARY_DIR_TARGET}
        CONFIGURE_COMMAND ${CMAKE_COMMAND} -E env PATH=${MESA_CLC_PATH}:${MESA_VTN_BINDGEN2_PATH}:${MESA_KK_CLC_PATH}:$ENV{PATH} --
            meson setup ${MESA_BINARY_DIR_TARGET} ${MESA_SOURCE_DIR}
            --buildtype=${MESA_BUILD_TYPE_TARGET}
            --cross-file=${MESA_CROSSFILES_DIR}/mesa-${KOSMICKRISP_ARCHITECTURE}.ini
            -Dmesa-clc=system -Dprecomp-compiler=system -Dspirv-tools=disabled
            ${MESA_COMMON_CONFIGURE_ARGS}
        BUILD_COMMAND meson compile -C ${MESA_BINARY_DIR_TARGET} ${MESA_TARGETS_TARGET}
        INSTALL_COMMAND cmake -E echo "Skipping install step."
        BUILD_BYPRODUCTS ${MESA_BYPRODUCTS_TARGET}
        DEPENDS ${MESA_PROJECT_NAME_AARCH64}
    )
endif()

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
