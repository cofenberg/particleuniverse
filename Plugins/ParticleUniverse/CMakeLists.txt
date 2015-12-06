cmake_minimum_required(VERSION 3.0.0)
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS true)

project(ParticleUniverse CXX)


function(configure_precompiled_headers target_name precompiled_header precompiled_source)
	if(MSVC)
		get_filename_component(PrecompiledBasename ${precompiled_header} NAME_WE)
		get_filename_component(PrecompiledHeadername ${precompiled_header} NAME)
		set(PrecompiledBinary "${CMAKE_CURRENT_BINARY_DIR}/${PrecompiledBasename}.pch")

		get_target_property(sources ${target_name} SOURCES)
		list(REMOVE_ITEM sources "${precompiled_source}")
		
		set_property(
			SOURCE ${precompiled_source} APPEND_STRING PROPERTY
			COMPILE_FLAGS "/Yc\"${PrecompiledHeadername}\" /Fp\"${PrecompiledBinary}\""
		)
		set_property(
			SOURCE ${precompiled_source} APPEND PROPERTY
			OBJECT_OUTPUTS "${PrecompiledBinary}"
		)

		set_property(
			SOURCE ${sources} APPEND_STRING PROPERTY
			COMPILE_FLAGS "/Yu\"${PrecompiledHeadername}\" /Fp\"${PrecompiledBinary}\""
		)
		set_property(
			SOURCE ${sources} APPEND PROPERTY
			OBJECT_DEPENDS "${PrecompiledBinary}"
		)

		message("Using precompiled header: ${precompiled_header}")
	elseif(APPLE)
		set_target_properties(
			${target_name} PROPERTIES
			XCODE_ATTRIBUTE_GCC_PREFIX_HEADER "${CMAKE_CURRENT_SOURCE_DIR}/${precompiled_header}"
			XCODE_ATTRIBUTE_GCC_PRECOMPILE_PREFIX_HEADER "YES"
		)

		message("Using precompiled header: ${precompiled_header}")
	endif(MSVC)
endfunction()


# Run 'cmake -P GenFiles.cmake' to generate/update the list of sources/includes involved in the build process 
include("Files.cmake")


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")


if(WIN32)
	set(Boost_NAMESPACE libboost)
endif(WIN32)

find_package(Boost)
find_package(Ogre)


option(BUILD_SHARED_LIBS "Build shared library" ON)

set(CMAKE_DEBUG_POSTFIX "d" CACHE STRING "Debug library target filename suffix")

add_definitions(-DBOOST_ALL_NO_LIB)

if(WIN32)
	add_definitions(-DUNICODE -D_UNICODE)
endif(WIN32)

if(APPLE)
	set(CMAKE_CXX_FLAGS "-std=c++11 -stdlib=libc++ ${CMAKE_CXX_FLAGS}")
endif(APPLE)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include ${OGRE_INCLUDE_DIRS} ${Boost_INCLUDE_DIRS})

add_library(ParticleUniverse ${PU_SOURCES} ${PU_INCLUDES})

configure_precompiled_headers(ParticleUniverse include/ParticleUniversePCH.h src/ParticleUniversePCH.cpp)

# Little msvc doesn't know how to allocate a bigger heap for a bigger pch on its own :/
if(MSVC)
	target_compile_options(ParticleUniverse PRIVATE -Zm140)
endif(MSVC)

if(BUILD_SHARED_LIBS)
	add_definitions(-DPARTICLE_UNIVERSE_EXPORTS)
	target_link_libraries(ParticleUniverse ${OGRE_LIBRARIES} ${Boost_LIBRARIES})
else(BUILD_SHARED_LIBS)
	set_target_properties(ParticleUniverse PROPERTIES OUTPUT_NAME ParticleUniverse_s)
endif(BUILD_SHARED_LIBS)

set_target_properties(ParticleUniverse PROPERTIES
	COMPILE_PDB_NAME_RELWITHDEBINFO ParticleUniverse_s
	COMPILE_PDB_NAME_DEBUG ParticleUniverse_sd
)

install(
	TARGETS ParticleUniverse
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
)

if(MSVC)
	if(BUILD_SHARED_LIBS)
		install(FILES "${CMAKE_CURRENT_BINARY_DIR}/ParticleUniverse.pdb" DESTINATION bin CONFIGURATIONS RelWithDebInfo)
		install(FILES "${CMAKE_CURRENT_BINARY_DIR}/ParticleUniversed.pdb" DESTINATION bin CONFIGURATIONS Debug)
	else(BUILD_SHARED_LIBS)
		# Since COMPILE_PDB_* was added in 3.1, we don't support correctly generated pdbs for static libraries in earlier CMake versions
		if(NOT "${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}" VERSION_LESS "3.1")
			install(FILES "${CMAKE_CURRENT_BINARY_DIR}/ParticleUniverse_s.pdb" DESTINATION lib CONFIGURATIONS RelWithDebInfo)
			install(FILES "${CMAKE_CURRENT_BINARY_DIR}/ParticleUniverse_sd.pdb" DESTINATION lib CONFIGURATIONS Debug)
		endif()
	endif(BUILD_SHARED_LIBS)
endif(MSVC)

install(DIRECTORY include/ DESTINATION include/particleuniverse PATTERN "include/ParticleUniversePCH.h" EXCLUDE)
