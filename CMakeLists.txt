######################################################################
#
# This will make the following outputs :
#
# cudaDeconv.exe
# otfviewer.exe
# radialft.exe
# libcudaDeconv.dll
#
######################################################################


cmake_minimum_required (VERSION 2.6)
cmake_policy(SET CMP0054 OLD) #Set IF statements to dereference variables like in CMAKE version < 3.1

project (cudaDeconv)

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

set(VERSION_MAJOR "0")
set(VERSION_MINOR "0")
set(VERSION_PATCH "1")
set(PROJECT_VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH})

# set -fPIC for all targets (for shared libraries)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# set (CMAKE_GENERATOR "Visual Studio 12 2013 Win64" CACHE INTERNAL "" FORCE)
######################################################################
#
# Set permissions before adding subdirectories
#
######################################################################
set(SCI_GROUP_WRITE GROUP_WRITE)
set(SCI_WORLD_FILE_PERMS WORLD_READ)
set(SCI_WORLD_PROGRAM_PERMS WORLD_READ WORLD_EXECUTE)


# If no CMAKE_BUILD_TYPE is specified on command line, Windows and Unix
# give different treatments. On Windows, CMAKE_BUILD_TYPE is defaulted to
# "Debug"; on Unix, it's some empty string that cannot be legally placed in
# a IF statement to do string compare like "STREQUAL". Hence the following
# 2-step verification. The goal is that if no Release is specified in
# the command line, we want the compiler flag to be almost identical to the
# ones used for Release except for -DNDEBUG

######################################################################
#
# Set compilier
#
######################################################################

if ( NOT CMAKE_BUILD_TYPE )
  set(CMAKE_BUILD_TYPE "Debug")
endif()

if (WIN32)
	message(STATUS "Windows build environment.")
endif()

if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
  set(CMAKE_BUILD_TYPE "Release")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS}")
  if (WIN32)
	  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MD")
  endif()
endif()

# message(STATUS "2 Build type is '${CMAKE_BUILD_TYPE}'")

if (NOT APPLE)
  find_package(OpenMP)
  set( CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${OpenMP_CXX_FLAGS}" )
endif()

if (NOT WIN32)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Wall")
endif()

if (CMAKE_COMPILER_IS_GNUCXX)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -std=c++11")
endif()


message(STATUS "CXX_FLAGS_Release is '${CMAKE_CXX_FLAGS_RELEASE}'")
message(STATUS "CXX_FLAGS is '${CMAKE_CXX_FLAGS}'")
message(STATUS "CXX_FLAGS_DEBUG is '${CMAKE_CXX_FLAGS_DEBUG}'")
message(STATUS "CXX_FLAGS_RELWITHDEBINFO is '${CMAKE_CXX_FLAGS_RELWITHDEBINFO}'")


######################################################################
#
# Set packages
#
######################################################################

set(CUDA_64_BIT_DEVICE_CODE ON)
find_package(CUDA REQUIRED)


if (WIN32)
  set (FFTW3_ROOT_DIR C:/fftw3)
  set (FFTW3_INCLUDE_DIR C:/fftw3)
  set (FFTW3F_LIBRARY libfftw3f-3)
  find_package(FFTW3 COMPONENTS single)

  if (EXISTS C:/libtiff/libtiff)
	set (TIFF_LIBRARY C:/libtiff/libtiff)
	set (TIFF_INCLUDE_DIR C:/libtiff/libtiff)
  else()
	set (TIFF_LIBRARY C:/libtiff)
	set (TIFF_INCLUDE_DIR C:/libtiff)
  endif()

else()
  find_package(FFTW3 COMPONENTS single threads)
endif()

find_package(TIFF REQUIRED)

if (WIN32)
  set (TIFF_LIBRARIES libtiff)
endif()

if (NOT WIN32)
  find_package(X11)
endif()

set(BOOST_LIBRARY_DIR C:/boost/stage/lib)
set(BOOST_INCLUDE_DIR C:/boost)

if(WIN32)
  find_package( Boost )
else()
  find_package( Boost REQUIRED COMPONENTS program_options filesystem system)
endif(WIN32)


#find_package(Boost COMPONENTS program_options filesystem system REQUIRED)


message(STATUS "CUDA version : ${CUDA_VERSION}")
if(NOT(CUDA_VERSION LESS 8.0))  # if CUDA version is > or = 8.0
	set(CUDA_NVCC_FLAGS "-O2;\
		-gencode=arch=compute_30,code=sm_30;\
		-gencode=arch=compute_35,code=sm_35;\
		-gencode=arch=compute_37,code=sm_37;\
		-gencode=arch=compute_50,code=sm_50;\
		-gencode=arch=compute_52,code=sm_52;\
		-gencode=arch=compute_61,code=sm_61") # compute capability 6.1 only avaiable in CUDA 8.0 or greater
	else()
		set(CUDA_NVCC_FLAGS "-O2;\
		-gencode=arch=compute_30,code=sm_30;\
		-gencode=arch=compute_35,code=sm_35;\
		-gencode=arch=compute_37,code=sm_37;\
		-gencode=arch=compute_50,code=sm_50;\
		-gencode=arch=compute_52,code=sm_52")
											  # compute capability 6.1 only avaiable in CUDA 8.0 or greater
endif()


# Specifies which compute capabilities are supported by the built .exe http://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#virtual-architecture-feature-list http://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#gpu-feature-list
# Need to add to these if different GPUs are targeted.
#--ptxas-options=-v;

######################################################################
#
# Add subdirectories
#
######################################################################

add_subdirectory(Buffers)

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}
  "${CMAKE_CURRENT_SOURCE_DIR}/Buffers"
  ${CUDA_INCLUDE_DIRS}
  ${FFTW3_INCLUDE_DIR}
  ${X11_INCLUDE_DIR}
   )


if(WIN32)
  set(CUDA_NVCC_FLAGS
    ${CUDA_NVCC_FLAGS};--use-local-env;--cl-version=2013)
    # ${CUDA_NVCC_FLAGS};--use-local-env)
  link_directories ( ${Boost_LIBRARY_DIRS} ${FFTW3_INCLUDE_DIR} ${TIFF_LIBRARY} )
  include_directories( ${Boost_INCLUDE_DIRS} ${FFTW3_INCLUDE_DIR}  ${TIFF_INCLUDE_DIR}
    "$ENV{NVCUDASAMPLES_ROOT}/common/inc" )
else()
  set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -w -Xcompiler -fPIC" )
  include_directories( ${CUDA_TOOLKIT_ROOT_DIR}/samples/common/inc )
endif()




##### Make this everything time so that linearDecon.cpp is remade each time.  This will ensure that the build date is correct #####
#add_custom_target(invalidate_files ALL
#                  COMMAND ${CMAKE_COMMAND} -E touch "${CMAKE_CURRENT_SOURCE_DIR}/linearDecon.cpp")


######################################################################
#
# Add build outputs and their dependencies
#
######################################################################



#####  libcudaDeconv.dll ###########################################

 CUDA_ADD_LIBRARY(
   libcudaDeconv
   SHARED
   RL-Biggs-Andrews.cpp
   RLgpuImpl.cu
   geometryTransform.cu
   camcor.cu
   )

add_dependencies(
   libcudaDeconv
   Buffer
   )

if (WIN32)
  set_property(TARGET libcudaDeconv PROPERTY POSITION_INDEPENDENT_CODE ON)
else()
  set_target_properties(libcudaDeconv
                        PROPERTIES
                        OUTPUT_NAME cudaDeconv
                        MACOSX_RPATH 1
                        POSITION_INDEPENDENT_CODE ON)
endif()



#####  libradialft.dll ###########################################

 add_library(
   libradialft
   SHARED
   radialft_interface.cpp
   )

add_dependencies(
   libradialft
   Buffer
   )

if (WIN32)
  set_property(TARGET libradialft PROPERTY POSITION_INDEPENDENT_CODE ON)
else()
  set_target_properties(libradialft
                        PROPERTIES
                        OUTPUT_NAME radialft
                        MACOSX_RPATH 1
                        POSITION_INDEPENDENT_CODE ON)
endif()


#####  cudaDeconv.exe ###########################################

CUDA_ADD_EXECUTABLE(
  cudaDeconv
  linearDecon.cpp
  RL-Biggs-Andrews.cpp
  boostfs.cpp
  RLgpuImpl.cu
  geometryTransform.cu
  camcor.cu
  )

add_dependencies(
  cudaDeconv
  Buffer
  )


#####  RL_interface_driver.exe ###########################################

# CUDA_ADD_EXECUTABLE(
#   RL_interface_driver
#   RL_interface_driver.cpp
#   )

# target_link_libraries(
#   RL_interface_driver
#   libcudaDeconv
#   ${TIFF_LIBRARIES}
#   )

#####  radialft.exe ###########################################
add_executable(radialft radialft-nonSIM.cpp)



######################################################################
#
# Link libraries
#
######################################################################

target_link_libraries(
  cudaDeconv
  Buffer
  ${FFTW3F_LIBRARY}
  ${FFTW3F_THREADS_LIBRARY}
  ${X11_X11_LIB}
  ${Boost_PROGRAM_OPTIONS_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_PREDICATE_LIBRARY}
#  ${Boost_REGEX_LIBRARY}
  ${Boost_SYSTEM_LIBRARY}
  ${TIFF_LIBRARIES}
  )

target_link_libraries(
  radialft
  ${FFTW3F_LIBRARY}
  ${FFTW3F_THREADS_LIBRARY}
  ${X11_X11_LIB}
  ${Boost_PROGRAM_OPTIONS_LIBRARY}
  ${TIFF_LIBRARIES}
  )



if(WIN32)

else()
  #####  otfviewer.exe ###########################################
  add_executable(otfviewer OTF_TIFF_viewer.cpp)
  target_link_libraries(
    otfviewer
    ${TIFF_LIBRARIES}
    ${X11_X11_LIB}
    )
endif(WIN32)


target_link_libraries(
  libcudaDeconv
  Buffer
  ${FFTW3F_LIBRARY}
  ${FFTW3F_THREADS_LIBRARY}
  ${X11_X11_LIB}
  ${Boost_PROGRAM_OPTIONS_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_PREDICATE_LIBRARY}
#  ${Boost_REGEX_LIBRARY}
  ${Boost_SYSTEM_LIBRARY}
  ${TIFF_LIBRARIES}
  )

target_link_libraries(
  libradialft
  Buffer
  ${FFTW3F_LIBRARY}
  ${FFTW3F_THREADS_LIBRARY}
  ${X11_X11_LIB}
  ${Boost_PROGRAM_OPTIONS_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_PREDICATE_LIBRARY}
#  ${Boost_REGEX_LIBRARY}
  ${Boost_SYSTEM_LIBRARY}
  ${TIFF_LIBRARIES}
  )


CUDA_ADD_CUFFT_TO_TARGET(libcudaDeconv)
CUDA_ADD_CUFFT_TO_TARGET(cudaDeconv)


