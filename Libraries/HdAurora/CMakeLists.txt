# Specify the library (for dependents), project (IDE), and output (binary file) names.
project(hdAurora)

find_package(glm REQUIRED) # Find the GLM vector maths package.
if(NOT AIAD_INTEGRATION) # 4AIAD: no OpenGL/Vulkan when embedded in AIAD (D3D12 host).
find_package(OpenGL REQUIRED) # Find OpenGL library for pxr targets
if (NOT APPLE)
    find_package(Vulkan QUIET MODULE COMPONENTS shaderc_combined) # Find Vulkan SDK for pxr targets
endif()
find_package(GLEW REQUIRED)
endif()
find_package(MaterialX REQUIRED) # MaterialX SDK
find_package(pxr REQUIRED) # Find the Universal Scene Description (USD) library
# Alias the namespace to meet the cmake convention on imported targets
add_library(MaterialX::GenGlsl ALIAS MaterialXGenGlsl)
if(ENABLE_EXPERIMENTAL_MDL)
    add_library(MaterialX::GenMdl ALIAS MaterialXGenMdl)
endif()
# Include system python libraries if needed by USD.
if(USD_BUILT_WITH_PYTHON)
  find_package (Python3 COMPONENTS Development)
  set(PYTHON_DEPENDENCY Python3::Python)
endif()


# On windows, the version resource and header files that add the version information to the DLL.
if(WIN32)
set(VERSION_FILES
    ${VERSION_FOLDER}/AuroraVersion.h
    HdAurora.rc
)
endif()

# 4AIAD: static library without DllMain/version resource when embedded in AIAD.
set(HDAURORA_LIBRARY_TYPE SHARED)
set(HDAURORA_DLL_SOURCE "DLL.cpp")
if(AIAD_INTEGRATION)
    set(HDAURORA_LIBRARY_TYPE STATIC)
    set(HDAURORA_DLL_SOURCE "HdAuroraStaticRegistration.cpp")
    set(VERSION_FILES ${VERSION_FOLDER}/AuroraVersion.h)
endif()

# Add shard library with all source files.
add_library(${PROJECT_NAME} ${HDAURORA_LIBRARY_TYPE}
    ${HDAURORA_DLL_SOURCE}
    "HdAuroraImageCache.cpp"
    "HdAuroraImageCache.h"
    "HdAuroraInstancer.cpp"
    "HdAuroraInstancer.h"
    "HdAuroraLight.cpp"
    "HdAuroraLight.h"
    "HdAuroraMaterial.cpp"
    "HdAuroraMaterial.h"
    "HdAuroraMesh.cpp"
    "HdAuroraMesh.h"
    "HdAuroraPlugin.cpp"
    "HdAuroraPlugin.h"
    "HdAuroraRenderBuffer.cpp"
    "HdAuroraRenderBuffer.h"
    "HdAuroraRenderDelegate.cpp"
    "HdAuroraRenderDelegate.h"
    "HdAuroraRenderPass.cpp"
    "HdAuroraRenderPass.h"
    "HdAuroraAssetPath.h"
    "HdAuroraTokens.h"
    "pch.h"
    ${VERSION_FILES}
)

# Set custom output properties.
set_target_properties(${PROJECT_NAME} PROPERTIES
	FOLDER "Libraries"
    RUNTIME_OUTPUT_DIRECTORY "${RUNTIME_OUTPUT_DIR}"
    LIBRARY_OUTPUT_DIRECTORY "${LIBRARY_OUTPUT_DIR}"
    ARCHIVE_OUTPUT_DIRECTORY "${LIBRARY_OUTPUT_DIR}"
    PDB_OUTPUT_DIRECTORY "${RUNTIME_OUTPUT_DIR}"

    # Follow the USD convention of lower-case initial on libraries.
    OUTPUT_NAME "hdAurora"
)

# Add dependencies.
# 4AIAD: without OpenGL there is no GLEW, no OpenGL::GL and no hdx (hdx requires a GPU backend).
if(AIAD_INTEGRATION)
    set(HDAURORA_GL_LIBRARIES)
else()
    set(HDAURORA_GL_LIBRARIES OpenGL::GL GLEW::glew hdx)
endif()
target_link_libraries(${PROJECT_NAME}
PRIVATE
    glm::glm
    ${HDAURORA_GL_LIBRARIES}
    MaterialX::GenGlsl
    usd
    hf
    hd
    Foundation
    ${PYTHON_DEPENDENCY}
    Aurora
)

if(ENABLE_EXPERIMENTAL_MDL)
    target_link_libraries(${PROJECT_NAME} PRIVATE MaterialX::GenMdl)
endif()

# Add default compile definitions (set in root CMakefile)
target_compile_definitions(${PROJECT_NAME} PRIVATE ${DEFAULT_COMPILE_DEFINITIONS})
if(AIAD_INTEGRATION)
    target_compile_definitions(${PROJECT_NAME} PRIVATE HDAURORA_NO_GL)
endif()

# Config the plug information file.
if(AIAD_INTEGRATION)
    # 4AIAD: statically linked plugin - the code lives in the host executable.
    set(PLUG_INFO_LIBRARY_NAME "")
elseif(WIN32)
    set(PLUG_INFO_LIBRARY_NAME "hdAurora.dll")
elseif(APPLE)
    set(PLUG_INFO_LIBRARY_NAME "libhdAurora.dylib")
else()
    set(PLUG_INFO_LIBRARY_NAME "libhdAurora.so")
endif()
if(PLUG_INFO_LIBRARY_NAME)
    set(PLUG_INFO_LIBRARY_PATH "../${PLUG_INFO_LIBRARY_NAME}")
else()
    set(PLUG_INFO_LIBRARY_PATH "")
endif()
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/resources/plugInfo.json.in"
    "${CMAKE_CURRENT_SOURCE_DIR}/resources/plugInfo.json"
    @ONLY
)

if(WIN32 AND NOT AIAD_INTEGRATION)
    # Create a custom target that will copy USD executables to the runtime folder.
    file(GLOB USD_EXECUTABLES ${pxr_ROOT}/bin/*)
    # USD v26.08 installs its Python modules to lib/site-packages;
    # v25.08 and earlier used lib/python.
    if(EXISTS "${pxr_ROOT}/lib/site-packages")
        set(USD_PYTHON_LIB_DIR "${pxr_ROOT}/lib/site-packages")
    else()
        set(USD_PYTHON_LIB_DIR "${pxr_ROOT}/lib/python")
    endif()
    set(USD_LIB_PLUGIN_DIR "${pxr_ROOT}/lib/usd")
    set(USD_PLUGIN_DIR "${pxr_ROOT}/plugin/usd")
    add_custom_target(CopyUSDExecutables ALL
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${USD_EXECUTABLES} ${RUNTIME_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${USD_LIB_PLUGIN_DIR} ${RUNTIME_OUTPUT_DIR}/usd
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${USD_PLUGIN_DIR} ${RUNTIME_OUTPUT_DIR}/usd
    )
    set_property(TARGET CopyUSDExecutables PROPERTY FOLDER "Deployment")
    add_dependencies(CopyUSDExecutables MakeRuntimeDir)
    add_dependencies(${PROJECT_NAME} CopyUSDExecutables)

    # If USD was built with python, add a custom build step to copy PXR python libs.
    if(USD_BUILT_WITH_PYTHON)
        message(STATUS "USD built with python, adding python copy step")
        add_custom_target(CopyUSDPythonExecutables ALL
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${USD_PYTHON_LIB_DIR} ${RUNTIME_OUTPUT_DIR}/python
        )
        set_property(TARGET CopyUSDPythonExecutables PROPERTY FOLDER "Deployment")
        add_dependencies(CopyUSDPythonExecutables MakeRuntimeDir)
        add_dependencies(${PROJECT_NAME} CopyUSDPythonExecutables)
    endif()

    set(HDAURORA_PLUGIN_RESOURCES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/resources)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${PROJECT_NAME}> $<$<CONFIG:Debug>:$<TARGET_PDB_FILE:${PROJECT_NAME}>> ${RUNTIME_OUTPUT_DIR}/usd
        COMMAND ${CMAKE_COMMAND} -E copy ${HDAURORA_PLUGIN_RESOURCES_DIR}/plugInfo.json ${RUNTIME_OUTPUT_DIR}/usd/hdAurora/resources/plugInfo.json
    )

elseif(APPLE)
    # Deploy the hdAurora render delegate plugin for USD discovery.
    set(HDAURORA_PLUGIN_RESOURCES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/resources)
    # Top-level plugInfo.json with "Includes" directive required for USD scanning.
    configure_file(
        "${HDAURORA_PLUGIN_RESOURCES_DIR}/pluginPath.plugInfo.json.in"
        "${HDAURORA_PLUGIN_RESOURCES_DIR}/pluginPath.plugInfo.json"
        @ONLY
    )
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        # Ensure the plugin directory structure exists.
        COMMAND ${CMAKE_COMMAND} -E make_directory ${RUNTIME_OUTPUT_DIR}/usd/hdAurora/resources
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${PROJECT_NAME}> ${RUNTIME_OUTPUT_DIR}/usd
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${HDAURORA_PLUGIN_RESOURCES_DIR}/plugInfo.json ${RUNTIME_OUTPUT_DIR}/usd/hdAurora/resources/plugInfo.json
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${HDAURORA_PLUGIN_RESOURCES_DIR}/pluginPath.plugInfo.json ${RUNTIME_OUTPUT_DIR}/usd/plugInfo.json
    )
else() # Linux
    #TODO anything we need to set to support the Aurora execution?
endif()

# Add the version folder required by HdAurora.rc
target_include_directories(${PROJECT_NAME}
PRIVATE
    ${VERSION_FOLDER}
    ${PYTHON_INCLUDE_DIRS}
)

# Install the binaries.
install(TARGETS ${PROJECT_NAME}  DESTINATION ${INSTALL_BIN}/usd)
install(FILES "resources/plugInfo.json" DESTINATION ${INSTALL_BIN}/usd/hdAurora/resources)
