
cmake_minimum_required(VERSION 3.20)

# Specify default build type if none provided (before project() command)
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build (Debug or Release)" FORCE)
endif()

# For generators with multiple configurations (like VS), only allow Debug and Release
if(CMAKE_CONFIGURATION_TYPES)
	set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "Limited configurations" FORCE)
	mark_as_advanced(CMAKE_CONFIGURATION_TYPES)
endif()

# Project name
project(sfml-book)

# Configuration options
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)

# Specific compiler options - set C++11 flag for g++ and clang
if(CMAKE_COMPILER_IS_GNUCXX)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
	
	# Mac OS X: clang uses libc++ standard library
	if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
	endif()
endif()

#####################################################################################################################################

# Download SFML
include(FetchContent)
FetchContent_Declare(SFML
    GIT_REPOSITORY https://github.com/SFML/SFML.git
    GIT_TAG 2.6.x
    GIT_SHALLOW ON
    EXCLUDE_FROM_ALL
    SYSTEM)
FetchContent_MakeAvailable(SFML)

set(SFML_LIBRARIES sfml-graphics sfml-window sfml-system sfml-audio sfml-network)

######################################################################################################################################


# Interprets an argument list separated with keywords.
# Example: 
#  parse_argument_list("MY" "FIRST;SECOND" "FIRST;a;b;c;SECOND;x;y") defines the following variables:
#  MY_FIRST  = a;b;c
#  MY_SECOND = x;y
macro(parse_argument_list PREFIX KEYWORDS ARGUMENTS)
	# Indirection over KEYWORD_LIST necessary because macro parameters are no real variables.
	set(KEYWORD_LIST "${KEYWORDS}")
	set(CURRENT_KEYWORD "")

	# Parse argument list
	foreach(ARG ${ARGUMENTS})
		# See if current argument is in the list of keywords
		list(FIND KEYWORD_LIST "${ARG}" KEYWORD_FOUND)
		if(NOT KEYWORD_FOUND EQUAL -1)
			# If current element is a keyword, store it
			set(CURRENT_KEYWORD ${ARG})
		else()
			# Append current argument to last stored keyword variable
			set(${PREFIX}_${CURRENT_KEYWORD} ${${PREFIX}_${CURRENT_KEYWORD}};${ARG})
		endif()
	endforeach()
endmacro()

######################################################################################################################################

# Macro for chapter building
# Usage:
#  build_chapter(Audio SOURCES First.cpp Second.cpp)
macro(build_chapter CHAPTER_NAME)

	# Parse additional arguments (fills variables CHAPTER_SOURCES and CHAPTER_DEPENDS)
	parse_argument_list("CHAPTER" "SOURCES" "${ARGN}")

	# Status output
	message(STATUS "-> Chapter ${CHAPTER_NAME}")


	#############################################################################
	# Build executable
	project(${CHAPTER_NAME})

	set(CHAPTER_DIR "${PROJECT_SOURCE_DIR}/..")
	include_directories("${CHAPTER_DIR}//Include")

	# Executable: Add only Main.cpp, set name without _EXE postfix
	add_executable(${PROJECT_NAME} Main.cpp ${CHAPTER_SOURCES})
	set_target_properties(${PROJECT_NAME} PROPERTIES OUTPUT_NAME ${CHAPTER_NAME})

	# Build static or shared libraries? Set chapter-specific DLL import macro
	if(SFML_STATIC_LIBRARIES)
		set_target_properties(${PROJECT_NAME} PROPERTIES COMPILE_DEFINITIONS "SFML_STATIC")
	endif()

	# Link SFML. For versions < 2.2, SFML_DEPENDENCIES is not defined, and we do not need to link dependencies
	if(SFML_VERSION_MINOR LESS 2)
        set(SFML_DEPENDENCIES "")
	endif()

	target_link_libraries(${PROJECT_NAME} PRIVATE ${SFML_LIBRARIES})
	target_compile_features(${PROJECT_NAME} PRIVATE cxx_std_17)

	install(TARGETS ${PROJECT_NAME}	RUNTIME DESTINATION ${CHAPTER_NAME})

	install(DIRECTORY ${CHAPTER_DIR}/Media
			DESTINATION ${PROJECT_NAME}
			PATTERN "CMakeLists.txt" EXCLUDE)

	# Copy OpenAL DLL if on Windows
	if(WIN32)
		add_custom_command(
			TARGET ${PROJECT_NAME}
			COMMENT "Copy OpenAL DLL"
			PRE_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${SFML_SOURCE_DIR}/extlibs/bin/$<IF:$<EQUAL:${CMAKE_SIZEOF_VOID_P},8>,x64,x86>/openal32.dll $<TARGET_FILE_DIR:${PROJECT_NAME}>
			VERBATIM)
	endif()

	# Copy Media files to output directory
	add_custom_command(
		TARGET ${PROJECT_NAME}
		COMMAND ${CMAKE_COMMAND} -E copy_directory
			${CHAPTER_DIR}/Media
			$<TARGET_FILE_DIR:${PROJECT_NAME}>/Media
		COMMENT "Copy Media files"
		VERBATIM)
endmacro()

# C++ source code, list of all subdirectories
# Must appear after macros, otherwise they are not visible in subdirectories
add_subdirectory(01_Intro/Source)
add_subdirectory(02_Resources/Source)
add_subdirectory(03_World/Source)
add_subdirectory(04_Input/Source)
add_subdirectory(05_States/Source)
add_subdirectory(06_Menus/Source)
add_subdirectory(07_Gameplay/Source)
add_subdirectory(08_Graphics/Source)
add_subdirectory(09_Audio/Source)
add_subdirectory(10_Network/Source)
