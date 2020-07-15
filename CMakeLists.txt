cmake_minimum_required(VERSION 3.1)
include(CheckLibraryExists)

project(rnnoise LANGUAGES C)

add_definitions(-DCOMPILE_OPUS)

# Ignore CRT warnings
add_definitions(-D_CRT_SECURE_NO_WARNINGS)

# Get source files
file(GLOB SOURCES "src/*.c" "src/*.h" "include/*.h")

# Build rnnoise
add_definitions(-DRNNOISE_BUILD)

# Compile the library
add_library(rnnoise ${SOURCES})

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
if(NOT WIN32)
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

