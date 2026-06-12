# Determine current Git SHA
execute_process(
    COMMAND git rev-parse HEAD
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE KOSMICKRISP_GIT_SHA
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# Download the pre-built zip bundle
set(KOSMICKRISP_ZIP_PATH "${CMAKE_CURRENT_BINARY_DIR}/kosmickrisp-${KOSMICKRISP_GIT_SHA}.zip")
if (NOT EXISTS "${KOSMICKRISP_ZIP_PATH}")
    message(STATUS "Downloading KosmicKrisp pre-builts...")
    file(
        DOWNLOAD https://github.com/shadexternals/mesa-kosmickrisp/releases/download/release-${KOSMICKRISP_GIT_SHA}/kosmickrisp-universal.zip
        "${KOSMICKRISP_ZIP_PATH}" SHOW_PROGRESS
	STATUS FILE_STATUS
    )
    list(GET FILE_STATUS 0 STATUS_CODE)
    if (NOT STATUS_CODE EQUAL 0)
        file(REMOVE "${KOSMICKRISP_ZIP_PATH}")
        message(FATAL_ERROR "No KosmicKrisp pre-built found with corresponding commit SHA (${KOSMICKRISP_GIT_SHA})")
    endif()
endif()

# Extract the pre-built zip bundle
set(KOSMICKRISP_FILES_PATH "${CMAKE_CURRENT_BINARY_DIR}/kosmickrisp-${KOSMICKRISP_GIT_SHA}")
if (NOT EXISTS "${KOSMICKRISP_FILES_PATH}")
    file(MAKE_DIRECTORY "${KOSMICKRISP_FILES_PATH}")
    execute_process(
        COMMAND ${CMAKE_COMMAND} -E tar xzf "${KOSMICKRISP_ZIP_PATH}"
        WORKING_DIRECTORY "${KOSMICKRISP_FILES_PATH}"
    )
endif()

set(KOSMICKRISP_LOADER "${KOSMICKRISP_FILES_PATH}/libvulkan.dylib" PARENT_SCOPE)
set(KOSMICKRISP_DRIVER "${KOSMICKRISP_FILES_PATH}/libvulkan_kosmickrisp.dylib" PARENT_SCOPE)
set(KOSMICKRISP_ICD "${KOSMICKRISP_FILES_PATH}/kosmickrisp_mesa_icd.json" PARENT_SCOPE)

# Adds a dependency target to the provided parent target which copies
# KosmicKrisp to the specified loader and driver directories.
# libvulkan is placed in the loader directory, and the KosmicKrisp driver
# and ICD is placed in the driver directory.
function (AddCopyKosmicKrisp PARENT_TARGET LOADER_DEST DRIVER_DEST)
    set(FULL_LOADER_DEST "${LOADER_DEST}/libvulkan.dylib")
    set(FULL_DRIVER_DEST "${DRIVER_DEST}/libvulkan_kosmickrisp.dylib")
    set(FULL_ICD_DEST "${DRIVER_DEST}/kosmickrisp_mesa_icd.json")

    add_custom_command(
        OUTPUT ${FULL_LOADER_DEST}
        DEPENDS ${KOSMICKRISP_LOADER} ${LOADER_DEST}
        COMMAND ${CMAKE_COMMAND} -E copy ${KOSMICKRISP_LOADER} ${FULL_LOADER_DEST})
    add_custom_command(
        OUTPUT ${FULL_DRIVER_DEST}
        DEPENDS ${KOSMICKRISP_DRIVER} ${DRIVER_DEST}
        COMMAND ${CMAKE_COMMAND} -E copy ${KOSMICKRISP_DRIVER} ${FULL_DRIVER_DEST})
    add_custom_command(
        OUTPUT ${FULL_ICD_DEST}
        DEPENDS ${KOSMICKRISP_ICD} ${DRIVER_DEST}
        COMMAND ${CMAKE_COMMAND} -E copy ${KOSMICKRISP_ICD} ${FULL_ICD_DEST})

    add_custom_target(CopyKosmicKrisp-${PARENT_TARGET} DEPENDS ${FULL_LOADER_DEST} ${FULL_DRIVER_DEST} ${FULL_ICD_DEST})
    add_dependencies(${PARENT_TARGET} CopyKosmicKrisp-${PARENT_TARGET})
endfunction()

