cmake_minimum_required(VERSION 3.25)
cmake_policy(SET CMP0091 NEW) # MSVC static runtime linking
include(CheckLibraryExists)
include(CheckIncludeFile)

project(rnnoise LANGUAGES C)

add_library(rnnoise
	include/rnnoise.h

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

	src/denoise.c
	src/rnn.c
	src/pitch.c
	src/kiss_fft.c
	src/celt_lpc.c
	src/nnet.c
	src/nnet_default.c
	src/parse_lpcnet_weights.c
	src/rnnoise_data.c
	src/rnnoise_tables.c)
	
target_compile_definitions(rnnoise PRIVATE RNNOISE_BUILD _CRT_SECURE_NO_WARNINGS)

# Enable SSE / AVX optimizations?
CHECK_INCLUDE_FILE("immintrin.h" HAS_X86_INTRINSICS)
if(HAS_X86_INTRINSICS)
	if(NOT EMSCRIPTEN)
		# CPUID assembly code does not compile on Emscripten
		target_compile_definitions(rnnoise PRIVATE RNN_ENABLE_X86_RTCD CPU_INFO_BY_ASM)
		target_sources(rnnoise PRIVATE
			src/x86/dnn_x86.h
			src/x86/x86_arch_macros.h
			src/x86/x86cpu.h

			src/x86/x86_dnn_map.c
			src/x86/x86cpu.c)
	endif()

	if(MSVC)
		target_sources(rnnoise PRIVATE src/x86/nnet_avx2.c src/x86/nnet_sse4_1.c)
		set_source_files_properties(src/nnet.c PROPERTIES COMPILE_DEFINITIONS "__SSE__;__SSE2__")
		set_source_files_properties(src/x86/nnet_avx2.c PROPERTIES COMPILE_DEFINITIONS "__SSE__;__SSE2__;__SSE4_1__;__AVX2__;__AVX__")
		set_source_files_properties(src/x86/nnet_sse4_1.c PROPERTIES COMPILE_DEFINITIONS "__SSE__;__SSE2__;__SSE4_1__")
	elseif(EMSCRIPTEN)
		# Emscripten does not know -mfma/-mavx2
		target_sources(rnnoise PRIVATE src/x86/nnet_sse4_1.c)
		set_source_files_properties(src/x86/nnet_sse4_1.c PROPERTIES COMPILE_FLAGS "-msse4.1 -msimd128")
	else()
		target_sources(rnnoise PRIVATE src/x86/nnet_avx2.c src/x86/nnet_sse4_1.c)
		set_source_files_properties(src/x86/nnet_avx2.c PROPERTIES COMPILE_FLAGS "-mavx -mfma -mavx2")
		set_source_files_properties(src/x86/nnet_sse4_1.c PROPERTIES COMPILE_FLAGS "-msse4.1")
	endif()
endif()

# Build DLL if needed
if(BUILD_SHARED_LIBS)
  if(WIN32)
    target_compile_definitions(rnnoise PRIVATE DLL_EXPORT)
  else()
    include(CheckCCompilerFlag)
    check_c_compiler_flag(-fvisibility=hidden COMPILER_HAS_HIDDEN_VISIBILITY)
    if(COMPILER_HAS_HIDDEN_VISIBILITY)
      set_target_properties(rnnoise PROPERTIES C_VISIBILITY_PRESET hidden)
    endif()
  endif()
endif()

# Include dirs
target_include_directories(rnnoise PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
        PRIVATE src)

# dependencies
if(WIN32)
	target_compile_definitions(rnnoise PRIVATE _USE_MATH_DEFINES)
else()
	check_library_exists(m cos "" HAVE_LIBM)
	if(HAVE_LIBM)
		#find_library(LIBM m)
		set(LIBM m)
	endif()
endif()
if(HAVE_LIBM)
	if(BUILD_SHARED_LIBS)
		target_link_libraries(rnnoise PRIVATE ${LIBM})
	else()
		target_link_libraries(rnnoise PUBLIC ${LIBM})
	endif()
endif()

install(TARGETS rnnoise
	EXPORT ${CMAKE_PROJECT_NAME}Config
	LIBRARY DESTINATION lib COMPONENT lib
	ARCHIVE DESTINATION lib COMPONENT lib
	RUNTIME DESTINATION bin COMPONENT bin)

install(DIRECTORY "include"
	DESTINATION . COMPONENT dev
	FILES_MATCHING
	PATTERN "*.h")

install(EXPORT ${CMAKE_PROJECT_NAME}Config
	DESTINATION cmake)

