cmake_minimum_required(VERSION 3.19)
project(breeze_tts_cpp LANGUAGES C CXX)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ggml hard fails when the Vulkan SDK is missing, and its own CUDA fallback compiles 8
# architectures. So both backends are probed here, against the toolchain that is really
# installed, and CMAKE_CUDA_ARCHITECTURES is narrowed to the local GPU. A plain
# `cmake -B build` then works on a CUDA only machine and on a Vulkan only machine alike.
# An explicit -D on the command line always wins, the probes only supply the defaults.

# the same judgement ggml's Vulkan backend makes: a loader, glslc and the SPIRV headers
function(breeze_vulkan_ready out)
    find_package(Vulkan QUIET COMPONENTS glslc)
    find_package(SPIRV-Headers CONFIG QUIET)
    set(ready FALSE)
    if(Vulkan_FOUND AND Vulkan_GLSLC_EXECUTABLE AND SPIRV-Headers_FOUND)
        set(ready TRUE)
    endif()
    set(${out} ${ready} PARENT_SCOPE)
endfunction()

function(breeze_cuda_ready out)
    find_package(CUDAToolkit QUIET)
    set(${out} ${CUDAToolkit_FOUND} PARENT_SCOPE)
    set(BREEZE_CUDA_VERSION ${CUDAToolkit_VERSION} PARENT_SCOPE)
endfunction()

if(NOT DEFINED BREEZE_VULKAN)
    breeze_vulkan_ready(BREEZE_VULKAN_DEFAULT)
    if(BREEZE_VULKAN_DEFAULT)
        message(STATUS "breeze: Vulkan SDK found (glslc + SPIRV-Headers), enabling the Vulkan backend")
    else()
        message(STATUS "breeze: Vulkan SDK not found (needs glslc + SPIRV-Headers), building without the Vulkan backend")
    endif()
else()
    set(BREEZE_VULKAN_DEFAULT ${BREEZE_VULKAN})
endif()
option(BREEZE_VULKAN "Build ggml with the Vulkan backend" ${BREEZE_VULKAN_DEFAULT})

if(NOT DEFINED BREEZE_CUDA)
    breeze_cuda_ready(BREEZE_CUDA_DEFAULT)
    if(BREEZE_CUDA_DEFAULT)
        message(STATUS "breeze: CUDA toolkit ${BREEZE_CUDA_VERSION} found, enabling the CUDA backend")
    else()
        message(STATUS "breeze: CUDA toolkit not found, building without the CUDA backend")
    endif()
else()
    set(BREEZE_CUDA_DEFAULT ${BREEZE_CUDA})
endif()
option(BREEZE_CUDA "Build ggml with the CUDA backend" ${BREEZE_CUDA_DEFAULT})
option(BREEZE_CUDA_NCCL "Link ggml-cuda against NCCL, only needed for multi GPU" OFF)
option(BREEZE_BUILD_CLI "Build the command line app" ON)
option(BREEZE_BUILD_SERVER "Build the HTTP server" ON)
option(BREEZE_MP3 "Let the server encode mp3 with the shine submodule" ON)
option(BREEZE_BUILD_SHARED "Build the shared C library" ON)
option(BREEZE_BUILD_TESTS "Build inference regression tests" OFF)

if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/third_party/ggml/CMakeLists.txt")
    message(FATAL_ERROR "third_party/ggml is missing. Run: git submodule update --init --recursive")
endif()

# an explicit request has to fail here and say why, not somewhere deep inside ggml
if(BREEZE_VULKAN)
    breeze_vulkan_ready(breeze_vulkan_ok)
    if(NOT breeze_vulkan_ok)
        message(FATAL_ERROR
            "BREEZE_VULKAN=ON needs the Vulkan SDK (glslc) and the SPIRV-Headers CMake package.\n"
            "Install them, or configure with -DBREEZE_VULKAN=OFF to build the CUDA or the CPU backend instead.")
    endif()
    unset(breeze_vulkan_ok)
endif()

if(BREEZE_CUDA)
    breeze_cuda_ready(breeze_cuda_ok)
    if(NOT breeze_cuda_ok)
        message(FATAL_ERROR
            "BREEZE_CUDA=ON needs the CUDA toolkit (nvcc not found or CUDAToolkit_FOUND is false).\n"
            "Install the toolkit, or configure with -DBREEZE_CUDA=OFF.")
    endif()
    unset(breeze_cuda_ok)
endif()

# ggml only picks its own 8 architecture list when this is undefined, so build for the
# local GPUs instead and leave the variable alone when the user set it on the command line
if(BREEZE_CUDA AND NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
    find_program(BREEZE_NVIDIA_SMI nvidia-smi)
    mark_as_advanced(BREEZE_NVIDIA_SMI)
    if(BREEZE_NVIDIA_SMI)
        execute_process(COMMAND ${BREEZE_NVIDIA_SMI} --query-gpu=compute_cap --format=csv,noheader
                        OUTPUT_VARIABLE breeze_compute_caps
                        RESULT_VARIABLE breeze_compute_caps_rc
                        ERROR_QUIET
                        TIMEOUT 15)
        if(breeze_compute_caps_rc EQUAL 0)
            # one line per gpu, windows ends them with \r\n
            string(REPLACE "\r" "" breeze_compute_caps "${breeze_compute_caps}")
            string(REPLACE "\n" ";" breeze_compute_caps "${breeze_compute_caps}")
            set(breeze_cuda_archs "")
            foreach(breeze_cap ${breeze_compute_caps})
                string(STRIP "${breeze_cap}" breeze_cap)
                if(breeze_cap MATCHES "^([0-9]+)\\.([0-9]+)$")
                    list(APPEND breeze_cuda_archs "${CMAKE_MATCH_1}${CMAKE_MATCH_2}")
                endif()
            endforeach()
            if(breeze_cuda_archs)
                list(REMOVE_DUPLICATES breeze_cuda_archs)
                set(CMAKE_CUDA_ARCHITECTURES ${breeze_cuda_archs} CACHE STRING
                    "CUDA architectures to generate code for" FORCE)
                set(breeze_cuda_archs_sm "")
                foreach(breeze_arch ${breeze_cuda_archs})
                    string(APPEND breeze_cuda_archs_sm " sm_${breeze_arch}")
                endforeach()
                message(STATUS "breeze: building CUDA for the local GPU:${breeze_cuda_archs_sm}")
            endif()
        endif()
    endif()
endif()

set(GGML_VULKAN ${BREEZE_VULKAN} CACHE BOOL "" FORCE)
set(GGML_CUDA ${BREEZE_CUDA} CACHE BOOL "" FORCE)
# ggml leaves these off unless its built by llama.cpp
set(GGML_CUDA_GRAPHS ${BREEZE_CUDA} CACHE BOOL "" FORCE)
# ggml turns this on whenever libnccl happens to be installed, and then every binary needs it
set(GGML_CUDA_NCCL ${BREEZE_CUDA_NCCL} CACHE BOOL "" FORCE)
set(GGML_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(GGML_BUILD_TESTS OFF CACHE BOOL "" FORCE)
if(MINGW)
    # libgomp has no static build here, and it would leak into the shared library
    set(GGML_OPENMP OFF CACHE BOOL "" FORCE)
endif()

set(breeze_backends "CPU")
if(BREEZE_VULKAN)
    list(APPEND breeze_backends "Vulkan")
endif()
if(BREEZE_CUDA)
    list(APPEND breeze_backends "CUDA")
endif()
if(BREEZE_VULKAN AND BREEZE_CUDA)
    message(STATUS "breeze: Vulkan and CUDA are both built into libggml and ggml picks one at "
                   "runtime, add -DBREEZE_VULKAN=OFF to keep only CUDA")
endif()
message(STATUS "breeze: backends = ${breeze_backends}")

# msvc builds ggml as dlls, they have to land next to the exes
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
add_subdirectory(third_party/ggml)

set(BREEZE_SOURCES
    src/common.cpp
    src/audio.cpp
    src/config.cpp
    src/gguf_loader.cpp
    src/tokenizer.cpp
    src/text_encoder.cpp
    src/backbone.cpp
    src/depth_decoder.cpp
    src/codec.cpp
    src/codec_conv.cpp
    src/codec_decoder.cpp
    src/codec_transformer.cpp
    src/sampling.cpp
    src/generation.cpp
    src/scheduler.cpp
    src/text_split.cpp
    src/voice.cpp
    src/model.cpp
)

add_library(breeze_core STATIC ${BREEZE_SOURCES})
target_include_directories(breeze_core PUBLIC include)
target_link_libraries(breeze_core PUBLIC ggml)

# make mingw binaries self contained so they do not pick up stray runtime DLLs
if(MINGW)
    add_link_options(-static-libgcc -static-libstdc++ -static)
endif()

# windows hands argv and narrow paths over in the ansi code page unless the exe opts into utf-8
set(BREEZE_UTF8_RC ${CMAKE_CURRENT_BINARY_DIR}/generated/utf8.rc)
file(WRITE ${BREEZE_UTF8_RC} "1 24 \"${CMAKE_CURRENT_SOURCE_DIR}/cmake/utf8.manifest\"\n")
# newer mingw links its own default manifest under the same id and ld keeps both, an empty stand in found first via -B stops that
if(MINGW)
    set(BREEZE_NOMANIFEST ${CMAKE_CURRENT_BINARY_DIR}/generated/nomanifest)
    file(MAKE_DIRECTORY ${BREEZE_NOMANIFEST})
    file(WRITE ${BREEZE_NOMANIFEST}/empty.c "")
    execute_process(COMMAND ${CMAKE_C_COMPILER} -c ${BREEZE_NOMANIFEST}/empty.c -o ${BREEZE_NOMANIFEST}/default-manifest.o
                    RESULT_VARIABLE nomanifest_rc)
    if(nomanifest_rc EQUAL 0)
        add_link_options(-B${BREEZE_NOMANIFEST}/)
    endif()
endif()
function(breeze_utf8 target)
    if(MSVC)
        target_sources(${target} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/utf8.manifest)
    elseif(MINGW)
        target_sources(${target} PRIVATE ${BREEZE_UTF8_RC} ${CMAKE_CURRENT_SOURCE_DIR}/apps/no_wildcard.c)
    endif()
endfunction()

if(BREEZE_BUILD_TESTS)
    enable_testing()
    add_executable(breeze-test-depth tests/depth_replay.cpp)
    target_link_libraries(breeze-test-depth PRIVATE breeze_core)
    breeze_utf8(breeze-test-depth)
    add_test(NAME depth-replay-cpu COMMAND breeze-test-depth)
    add_test(NAME depth-replay-gpu COMMAND breeze-test-depth --gpu)
    set_tests_properties(depth-replay-gpu PROPERTIES SKIP_RETURN_CODE 77)
    add_executable(breeze-test-text-embedding tests/text_embedding.cpp)
    target_link_libraries(breeze-test-text-embedding PRIVATE breeze_core)
    breeze_utf8(breeze-test-text-embedding)
    add_test(NAME text-embedding-gpu COMMAND breeze-test-text-embedding)
    set_tests_properties(text-embedding-gpu PROPERTIES SKIP_RETURN_CODE 77)
    add_executable(breeze-test-vocoder tests/vocoder_stream.cpp)
    target_link_libraries(breeze-test-vocoder PRIVATE breeze_core)
    breeze_utf8(breeze-test-vocoder)
    add_test(NAME vocoder-stream-cpu COMMAND breeze-test-vocoder)
    add_test(NAME vocoder-stream-gpu COMMAND breeze-test-vocoder --gpu)
    set_tests_properties(vocoder-stream-gpu PROPERTIES SKIP_RETURN_CODE 77)
endif()

set(BREEZE_WEBUI_HEADER ${CMAKE_CURRENT_BINARY_DIR}/generated/breeze_webui_assets.h)
add_custom_command(
    OUTPUT ${BREEZE_WEBUI_HEADER}
    COMMAND ${CMAKE_COMMAND}
        -DWEBUI_DIR=${CMAKE_CURRENT_SOURCE_DIR}/webui
        -DOUT_HEADER=${BREEZE_WEBUI_HEADER}
        -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/embed_webui.cmake
    DEPENDS
        ${CMAKE_CURRENT_SOURCE_DIR}/webui/index.html
        ${CMAKE_CURRENT_SOURCE_DIR}/webui/style.css
        ${CMAKE_CURRENT_SOURCE_DIR}/webui/app.js
        ${CMAKE_CURRENT_SOURCE_DIR}/webui/talk.html
        ${CMAKE_CURRENT_SOURCE_DIR}/webui/talk.css
        ${CMAKE_CURRENT_SOURCE_DIR}/webui/talk.js
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/embed_webui.cmake
    COMMENT "Embedding WebUI assets"
    VERBATIM
)
add_custom_target(breeze_webui DEPENDS ${BREEZE_WEBUI_HEADER})

if(BREEZE_BUILD_SHARED)
    add_library(breeze SHARED src/c_api.cpp)
    target_link_libraries(breeze PRIVATE breeze_core)
    target_compile_definitions(breeze PRIVATE BREEZE_BUILD_SHARED)
    set_target_properties(breeze PROPERTIES OUTPUT_NAME breeze)
endif()

if(BREEZE_BUILD_CLI)
    add_executable(breeze-cli apps/cli/main.cpp)
    target_link_libraries(breeze-cli PRIVATE breeze_core)
    breeze_utf8(breeze-cli)

    add_executable(breeze-convert apps/convert/main.cpp)
    target_link_libraries(breeze-convert PRIVATE breeze_core)
    breeze_utf8(breeze-convert)
endif()

add_executable(breeze-quantize apps/quantize/main.cpp)
target_link_libraries(breeze-quantize PRIVATE breeze_core)
breeze_utf8(breeze-quantize)

if(BREEZE_BUILD_SERVER)
    add_executable(breeze-server apps/server/main.cpp apps/server/server.cpp apps/server/voices.cpp
                                 apps/server/ws.cpp apps/server/ws_api.cpp apps/server/openai.cpp
                                 apps/server/encode.cpp apps/server/llm_proxy.cpp)
    add_dependencies(breeze-server breeze_webui)
    target_include_directories(breeze-server PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/generated third_party)
    target_link_libraries(breeze-server PRIVATE breeze_core)
    breeze_utf8(breeze-server)
    if(WIN32)
        target_link_libraries(breeze-server PRIVATE ws2_32)
    endif()
    find_package(Threads REQUIRED)
    target_link_libraries(breeze-server PRIVATE Threads::Threads)
    if(BREEZE_MP3 AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/third_party/shine/src/lib/layer3.c")
        # shine is lgpl, it stays in its own submodule and its own static lib
        file(GLOB SHINE_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/third_party/shine/src/lib/*.c)
        add_library(shine STATIC ${SHINE_SOURCES})
        target_include_directories(shine PRIVATE third_party/shine/src/lib)
        if(MSVC)
            target_compile_options(shine PRIVATE /FI${CMAKE_CURRENT_SOURCE_DIR}/cmake/shine_msvc.h)
        endif()
        if(UNIX)
            target_link_libraries(shine PRIVATE m)
        endif()
        target_link_libraries(breeze-server PRIVATE shine)
        target_compile_definitions(breeze-server PRIVATE BREEZE_HAVE_MP3)
    elseif(BREEZE_MP3)
        message(WARNING "third_party/shine is missing so mp3 requests will get wav. Run: git submodule update --init --recursive")
    endif()
endif()
