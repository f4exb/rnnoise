cmake_minimum_required(VERSION 3.18)
project(rnnoise LANGUAGES C)

include(GNUInstallDirs)

# Build a shared library by default (can be overridden with -DBUILD_SHARED_LIBS=OFF)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

set(RNNOISE_MODEL_HASH "0a8755f8e2d834eff6a54714ecc7d75f9932e845df35f8b59bc52a7cfe6e8b37")
set(RNNOISE_MODEL_URL "https://media.xiph.org/rnnoise/models/rnnoise_data-${RNNOISE_MODEL_HASH}.tar.gz")
set(RNNOISE_MODEL_ARCHIVE "${CMAKE_CURRENT_BINARY_DIR}/rnnoise_data.tar.gz" CACHE FILEPATH
    "Path to the pinned RNNoise model archive (downloaded when absent)")
set(RNNOISE_MODEL_DIR "${CMAKE_CURRENT_BINARY_DIR}/model")

if(NOT EXISTS "${RNNOISE_MODEL_ARCHIVE}")
  file(DOWNLOAD "${RNNOISE_MODEL_URL}" "${RNNOISE_MODEL_ARCHIVE}"
       EXPECTED_HASH SHA256=${RNNOISE_MODEL_HASH}
       STATUS download_status)
  list(GET download_status 0 download_code)
  if(NOT download_code EQUAL 0)
    message(FATAL_ERROR "RNNoise model download failed: ${download_status}")
  endif()
endif()
file(SHA256 "${RNNOISE_MODEL_ARCHIVE}" model_hash)
if(NOT model_hash STREQUAL RNNOISE_MODEL_HASH)
  message(FATAL_ERROR "RNNoise model archive has an unexpected SHA256: ${RNNOISE_MODEL_ARCHIVE}")
endif()

if(NOT EXISTS "${RNNOISE_MODEL_DIR}/src/rnnoise_data.c" OR
   NOT EXISTS "${RNNOISE_MODEL_DIR}/src/rnnoise_data.h")
  file(ARCHIVE_EXTRACT INPUT "${RNNOISE_MODEL_ARCHIVE}" DESTINATION "${RNNOISE_MODEL_DIR}")
endif()

if(MSVC)
    # Use CMake's configuration-specific MSVC optimization flags.
    set(RNN_COMPILE_OPTIONS "")
else()
    set(RNN_COMPILE_OPTIONS "-O3;-pedantic;-Wall;-Wextra;-Wno-sign-compare;-Wno-parentheses;-Wno-long-long;-fvisibility=hidden")
endif()

set(rnnoise_HEADERS
    include/rnnoise.h
)
set(rnnoise_PRIVATE_HEADERS
    src/arch.h
    src/celt_lpc.h
    src/cpu_support.h
    src/common.h
    src/denoise.h
    src/_kiss_fft_guts.h
    src/kiss_fft.h
    src/nnet.h
    src/nnet_arch.h
    src/opus_types.h
    src/pitch.h
    src/rnn.h
    ${RNNOISE_MODEL_DIR}/src/rnnoise_data.h
    src/vec_neon.h
    src/vec.h
    src/vec_avx.h
    src/x86/x86_arch_macros.h
    src/x86/x86cpu.h
    src/x86/dnn_x86.h
)

set(rnnoise_SOURCES
	src/denoise.c
	src/rnn.c
	src/pitch.c
	src/kiss_fft.c
	src/celt_lpc.c
	src/nnet.c
	src/nnet_default.c
	src/parse_lpcnet_weights.c
	${RNNOISE_MODEL_DIR}/src/rnnoise_data.c
	src/rnnoise_tables.c
)

set(rnnoise_SOURCES_RTCD
    src/x86/x86_dnn_map.c
    src/x86/x86cpu.c
)

set(rrnoise_SOURCES_SSE4.1
    src/x86/nnet_sse4_1.c
)

set(rnnoise_SOURCES_AVX2
    src/x86/nnet_avx2.c
)

if (RNN_ENABLE_X86_RTCD)
    set(rnnoise_SOURCES
        ${rnnoise_SOURCES}
        ${rnnoise_SOURCES_RTCD}
        ${rrnoise_SOURCES_SSE4.1}
        ${rnnoise_SOURCES_AVX2}
    )
    if (MSVC)
        set(RNN_COMPILE_OPTIONS "${RNN_COMPILE_OPTIONS};/arch:SSE2;/arch:AVX;/arch:AVX2")
    else()
        set(RNN_COMPILE_OPTIONS "${RNN_COMPILE_OPTIONS};-msse4.1;-mavx;-mfma;-mavx2")
    endif()
endif()

set(TARGET_NAME "rnnoise")

add_library(${TARGET_NAME} ${rnnoise_SOURCES})

target_compile_options(${TARGET_NAME} PRIVATE ${RNN_COMPILE_OPTIONS})
target_compile_definitions(${TARGET_NAME} PRIVATE RNNOISE_BUILD $<$<BOOL:${BUILD_SHARED_LIBS}>:DLL_EXPORT>)
target_include_directories(${TARGET_NAME}
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
           $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src ${RNNOISE_MODEL_DIR}/src)
if(MSVC)
    target_compile_definitions(${TARGET_NAME} PRIVATE _CRT_SECURE_NO_WARNINGS)
    if(CMAKE_C_COMPILER_ARCHITECTURE_ID STREQUAL "x64")
        # SSE2 is part of the x64 baseline; use the existing vectorized path.
        target_compile_definitions(${TARGET_NAME} PRIVATE OPUS_X86_MAY_HAVE_SSE2)
    endif()
endif()

if (LINUX)
    target_link_libraries(${TARGET_NAME} PRIVATE m)
endif()

install(TARGETS ${TARGET_NAME}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${rnnoise_HEADERS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
