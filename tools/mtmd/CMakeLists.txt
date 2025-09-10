# mtmd

find_package(Threads REQUIRED)

# CoreML support option
option(ENABLE_COREML "Enable CoreML support" OFF)
option(ENABLE_TRT    "Enable TensorRT support" OFF)

add_library(mtmd
            mtmd.cpp
            mtmd-audio.cpp
            mtmd.h
            clip.cpp
            clip.h
            clip-impl.h
            mtmd-helper.cpp
            mtmd-helper.h
            )

# Add CoreML related files when enabled
if(ENABLE_COREML)
    target_sources(mtmd PRIVATE
        coreml/mtmd_coreml.h
        coreml/mtmd_coreml.mm
        coreml/ane_minicpmv4_vit_f16.h
        coreml/ane_minicpmv4_vit_f16.m
    )
    # Define compile-time macro for code guards
    target_compile_definitions(mtmd PRIVATE ENABLE_COREML)
    
    # Enable ARC for Objective-C files
    set_source_files_properties(coreml/mtmd_coreml.mm PROPERTIES COMPILE_FLAGS "-fobjc-arc")
    set_source_files_properties(coreml/ane_minicpmv4_vit_f16.m PROPERTIES COMPILE_FLAGS "-fobjc-arc")
endif()

# TensorRT support (Linux)
if(ENABLE_TRT)
    # CUDA runtime
    find_package(CUDAToolkit REQUIRED)

    # Try to locate TensorRT headers and libs
    find_path(TENSORRT_INCLUDE_DIR NvInfer.h
        HINTS
            ${TENSORRT_DIR}/include
            $ENV{TENSORRT_DIR}/include
            /usr/include
            /usr/include/x86_64-linux-gnu
            /usr/local/include
            /usr/local/cuda/include)

    find_library(NVINFER_LIBRARY nvinfer
        HINTS
            ${TENSORRT_DIR}/lib
            ${TENSORRT_DIR}/lib64
            $ENV{TENSORRT_DIR}/lib
            $ENV{TENSORRT_DIR}/lib64
            /usr/lib
            /usr/lib/x86_64-linux-gnu
            /usr/local/lib
            /usr/local/lib64)

    # Optional plugin lib (link if available)
    find_library(NVINFER_PLUGIN_LIBRARY nvinfer_plugin
        HINTS
            ${TENSORRT_DIR}/lib
            ${TENSORRT_DIR}/lib64
            $ENV{TENSORRT_DIR}/lib
            $ENV{TENSORRT_DIR}/lib64
            /usr/lib
            /usr/lib/x86_64-linux-gnu
            /usr/local/lib
            /usr/local/lib64)

    if(NOT TENSORRT_INCLUDE_DIR OR NOT NVINFER_LIBRARY)
        message(FATAL_ERROR "TensorRT not found. Set TENSORRT_DIR or specify TENSORRT_INCLUDE_DIR and NVINFER_LIBRARY.")
    endif()

    target_include_directories(mtmd PRIVATE ${TENSORRT_INCLUDE_DIR})
    target_link_libraries(mtmd PRIVATE ${NVINFER_LIBRARY} CUDA::cudart)
    if(NVINFER_PLUGIN_LIBRARY)
        target_link_libraries(mtmd PRIVATE ${NVINFER_PLUGIN_LIBRARY})
    endif()
    target_compile_definitions(mtmd PRIVATE ENABLE_TRT)
endif()

target_link_libraries     (mtmd PUBLIC ggml llama common)
target_link_libraries     (mtmd PRIVATE Threads::Threads)
target_include_directories(mtmd PUBLIC  .)
target_include_directories(mtmd PRIVATE ../..)
target_include_directories(mtmd PRIVATE ../../common)
target_include_directories(mtmd PRIVATE ../../include)
target_include_directories(mtmd PRIVATE ../../ggml/include)
target_include_directories(mtmd PRIVATE ../../src)
target_include_directories(mtmd PRIVATE ../../vendor)
target_compile_features   (mtmd PRIVATE cxx_std_17)

# Link CoreML and Accelerate frameworks when CoreML is enabled
if(ENABLE_COREML)
    target_link_libraries(mtmd PRIVATE 
        "-framework Foundation" 
        "-framework CoreML" 
        "-framework Accelerate"
        "-ObjC"
    )
endif()

if (BUILD_SHARED_LIBS)
    set_target_properties     (mtmd PROPERTIES POSITION_INDEPENDENT_CODE ON)
    target_compile_definitions(mtmd PRIVATE LLAMA_BUILD)
    target_compile_definitions(mtmd PUBLIC  LLAMA_SHARED)
endif()

set(MTMD_PUBLIC_HEADERS
    ${CMAKE_CURRENT_SOURCE_DIR}/mtmd.h
    ${CMAKE_CURRENT_SOURCE_DIR}/mtmd-helper.h
    )

# Add CoreML public headers when enabled
if(ENABLE_COREML)
    list(APPEND MTMD_PUBLIC_HEADERS
        ${CMAKE_CURRENT_SOURCE_DIR}/coreml/coreml.h
    )
endif()

set_target_properties(mtmd
    PROPERTIES
    PUBLIC_HEADER "${MTMD_PUBLIC_HEADERS}")

install(TARGETS mtmd LIBRARY PUBLIC_HEADER)

if (NOT MSVC)
    # for stb_image.h and miniaudio.h
    target_compile_options(mtmd PRIVATE -Wno-cast-qual)
endif()

if (TARGET BUILD_INFO)
    add_dependencies(mtmd        BUILD_INFO)
    add_dependencies(mtmd-helper BUILD_INFO)
endif()

add_executable(llama-llava-cli    deprecation-warning.cpp)
add_executable(llama-gemma3-cli   deprecation-warning.cpp)
add_executable(llama-minicpmv-cli deprecation-warning.cpp)
add_executable(llama-qwen2vl-cli  deprecation-warning.cpp)

set(TARGET llama-mtmd-cli)
add_executable         (${TARGET} mtmd-cli.cpp)
set_target_properties  (${TARGET} PROPERTIES OUTPUT_NAME llama-mtmd-cli)
install                (TARGETS ${TARGET} RUNTIME BUNDLE DESTINATION .)
target_link_libraries  (${TARGET} PRIVATE common mtmd Threads::Threads)
target_compile_features(${TARGET} PRIVATE cxx_std_17)
