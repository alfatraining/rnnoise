cmake_minimum_required(VERSION 3.1)
include(CheckLibraryExists)

project(rnnoise LANGUAGES C)

if(MSVC)
	option(MSVC_LINK_STATIC_RUNTIME "Use static linking of msvc runtime" OFF)
endif()

# switch from dynamic to static linking
if(MSVC AND MSVC_LINK_STATIC_RUNTIME)
	foreach(flag_var
		CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
		CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)

		if(${flag_var} MATCHES "/MD")
			string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
		endif(${flag_var} MATCHES "/MD")
	endforeach(flag_var)
endif()

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

