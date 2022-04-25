
# determine if nxtpipeline-image-utils is being built as a submodule inside another repo,
# or if it's being build standalone (if the later, we need to do some configuration)
get_directory_property(hasParent PARENT_DIRECTORY)

if(hasParent AND NOT nxtpipeline-image-utils-force-standalone)
	message("-- nxtpipeline-image-utils:  building as submodule, ${hasParent}")
else()
	message("-- nxtpipeline-image-utils:  building as standalone")
	if(NOT hasParent)
		cmake_minimum_required(VERSION 2.8)
	endif()


	# standalone project
	project(nxtpipeline-image-utils)
  
	# -std=gnu++11
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wno-write-strings")
  
	# setup CUDA
#	set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cuda")
	find_package(CUDA REQUIRED)
	message("-- CUDA version: ${CUDA_VERSION}")

	set(
		CUDA_NVCC_FLAGS
		${CUDA_NVCC_FLAGS};
		--keep
		-O3
#		-gencode arch=compute_37,code=sm_37   # deprecated
		-gencode arch=compute_53,code=sm_53
		-gencode arch=compute_60,code=sm_60
		-gencode arch=compute_61,code=sm_61
		-gencode arch=compute_62,code=sm_62
	)

	if(CUDA_VERSION_MAJOR GREATER 9)
		message("-- CUDA ${CUDA_VERSION_MAJOR} detected, enabling SM_72")
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_72,code=sm_72)
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_75,code=sm_75)
	endif()

	if(CUDA_VERSION_MAJOR GREATER 10)
		message("-- CUDA ${CUDA_VERSION_MAJOR} detected, enabling SM_80 and SM_86")
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_80,code=sm_80)
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_86,code=sm_86)
	endif()
	
	# setup project output paths
#	set(PROJECT_OUTPUT_DIR  ${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR})
#	set(PROJECT_INCLUDE_DIR ${PROJECT_OUTPUT_DIR}/include)
#
#	file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR})
#	file(MAKE_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)

	message("-- system arch:  ${CMAKE_SYSTEM_PROCESSOR}")
#	message("-- output path:  ${PROJECT_OUTPUT_DIR}")

#	set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)
#	set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
#	set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
	
	# build C/C++ interface
#	include_directories(${PROJECT_INCLUDE_DIR})
	#include_directories(/usr/include/gstreamer-1.0 /usr/lib/aarch64-linux-gnu/gstreamer-1.0/include /usr/include/glib-2.0 /usr/include/libxml2 /usr/lib/aarch64-linux-gnu/glib-2.0/include/)

endif()

set(CUDA_NVCC_FLAGS_DEBUG -g -G)

# option for enabling/disabling NVMM memory in multimedia stack
option(ENABLE_NVMM "Enable use of NVMM zero-copy memory in video and camera streaming" OFF)
message("-- NVMM zero-copy memory:  ENABLE_NVMM=${ENABLE_NVMM}")

if(ENABLE_NVMM)
	add_definitions(-DENABLE_NVMM)
endif()

# additional paths for includes and libraries
#include_directories(${PROJECT_INCLUDE_DIR}/nxtpipeline-image-utils)
include_directories(
	/usr/include/gstreamer-1.0
	/usr/include/glib-2.0
	/usr/include/libxml2
#	/usr/lib/aarch64-linux-gnu/gstreamer-1.0/include
	/usr/lib/x86_64-linux-gnu/gstreamer-1.0/include/
#	/usr/lib/aarch64-linux-gnu/glib-2.0/include/
	/usr/lib/x86_64-linux-gnu/glib-2.0/include/
)

#link_directories(/usr/lib/aarch64-linux-gnu/tegra)

# build library
file(GLOB jetsonUtilitySources
		*.cpp
		camera/*.cpp
		codec/*.cpp
		cuda/*.cu
		cuda/*.cpp
		display/*.cpp
		image/*.cpp
		input/*.cpp
		network/*.cpp
		threads/*.cpp
		video/*.cpp
)
file(GLOB jetsonUtilityIncludes
		*.h
		*.hpp
		camera/*.h
		codec/*.h
		cuda/*.h
		cuda/*.cuh
		display/*.h
		image/*.h
		image/*.inl
		input/*.h
		network/*.h
		threads/*.h
		threads/*.inl
		video/*.h
)

#set(CMAKE_POSITION_INDEPENDENT_CODE ON)
cuda_add_library(nxtpipeline-image-utils SHARED ${jetsonUtilitySources})
#set_property(TARGET nxtpipeline-image-utils PROPERTY POSITION_INDEPENDENT_CODE ON)

target_link_libraries(nxtpipeline-image-utils
	GL GLU GLEW gstreamer-1.0 gstapp-1.0 gstpbutils-1.0
#	nvbuf_utils
	${CUDA_nppicc_LIBRARY}
)


# transfer all headers to the include directory 
#file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR}/nxtpipeline-image-utils)
#
#foreach(include ${jetsonUtilityIncludes})
#	message("-- Copying ${include}")
#	configure_file(${include} ${PROJECT_INCLUDE_DIR}/nxtpipeline-image-utils COPYONLY)
#endforeach()

target_include_directories(
	nxtpipeline-image-utils
	PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
)
file(GLOB subdirs LIST_DIRECTORIES true *)
foreach(dir ${subdirs})
	if(IS_DIRECTORY "${dir}")
		message(STATUS "Adding ${dir}")
		target_include_directories(
			nxtpipeline-image-utils
			PUBLIC
			$<BUILD_INTERFACE:${dir}>
		)
	endif()
endforeach()


#target_compile_options(nxtpipeline-image-utils PRIVATE "-G;-src-in-ptx>")

# copy image data
#file(GLOB imageData data/*.png data/*.jpg)

#foreach(image ${imageData})
#	message("-- Copying ${image}")
#	file(COPY ${image} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
#	install(CODE "file(COPY ${image} DESTINATION ${CMAKE_INSTALL_PREFIX}/bin )" )
#endforeach()
	

# install headers
#foreach(include ${jetsonUtilityIncludes})
#	install(FILES "${include}" DESTINATION include/nxtpipeline-image-utils)
#endforeach()

# install the shared library
install(TARGETS nxtpipeline-image-utils DESTINATION lib EXPORT nxtpipeline-image-utilsConfig)

# install the cmake project, for importing
install(EXPORT nxtpipeline-image-utilsConfig DESTINATION share/nxtpipeline-image-utils/cmake)


# build tests/sample executables
#add_subdirectory(camera/camera-viewer)
#add_subdirectory(video/video-viewer)
#add_subdirectory(camera/v4l2-console)
#add_subdirectory(camera/v4l2-display)
#add_subdirectory(display/gl-display-test)
#add_subdirectory(python)
