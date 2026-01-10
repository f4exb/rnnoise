cmake_minimum_required(VERSION 3.18)
project(rnnoise)

# Build a shared library by default (can be overridden with -DBUILD_SHARED_LIBS=OFF)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

# Ignore CRT warnings on MSVC
if(MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

set(RNNOISE_MODEL_HASH "0a8755f8e2d834eff6a54714ecc7d75f9932e845df35f8b59bc52a7cfe6e8b37")
set(RNNOISE_MODEL_URL "https://media.xiph.org/rnnoise/models/rnnoise_data-${RNNOISE_MODEL_HASH}.tar.gz")
set(RNNOISE_MODEL_ARCHIVE "${CMAKE_BINARY_DIR}/rnnoise_data.tar.gz")
set(RNNOISE_MODEL_DIR "${CMAKE_BINARY_DIR}/model")

file(DOWNLOAD "${RNNOISE_MODEL_URL}" "${RNNOISE_MODEL_ARCHIVE}"
     EXPECTED_HASH SHA256=${RNNOISE_MODEL_HASH}  # verify hash
     STATUS download_status)
if(NOT download_status EQUAL 0)
  message(FATAL_ERROR "Download failed: ${download_status}")
endif()

file(ARCHIVE_EXTRACT INPUT "${RNNOISE_MODEL_ARCHIVE}" DESTINATION "${CMAKE_SOURCE_DIR}")

add_definitions(-DRNNOISE_BUILD)

set(RNN_COMPILE_OPTIONS "-O3;-pedantic;-Wall;-Wextra;-Wno-sign-compare;-Wno-parentheses;-Wno-long-long;-fvisibility=hidden")

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
    src/rnnoise_data.h
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
	src/rnnoise_data.c
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
    set(RNN_COMPILE_OPTIONS "${RNN_COMPILE_OPTIONS};-msse4.1;-mavx;-mfma;-mavx2")
endif()

include_directories(
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/src
)

set(TARGET_NAME "rnnoise")

add_library(${TARGET_NAME} ${rnnoise_SOURCES})

target_compile_options(${TARGET_NAME} PRIVATE ${RNN_COMPILE_OPTIONS})
target_link_libraries(${TARGET_NAME} PRIVATE m)

install(TARGETS ${TARGET_NAME} DESTINATION ${INSTALL_LIB_DIR})
install(FILES ${rnnoise_HEADERS} DESTINATION ${CMAKE_INSTALL_PREFIX}/include)
