
cmake_minimum_required(VERSION 2.8)
include(ExternalProject)

option(MAXWELL_USE_MPI "Set to ON to use mpi compilers. Must set PATH variable
  'MAXWELL_MPI_BASE_DIR' as well unless it is equal to /usr/local" OFF)
set(MAXWELL_MPI_BASE_DIR "/usr/local" CACHE PATH "Location of mpi directories.
  Defaults to /usr/local")

set(CMAKE_SYSTEM_PREFIX_PATH ${CMAKE_SYSTEM_PREFIX_PATH} /usr/local/mpi)


if(${MAXWELL_USE_MPI})
  #set(CMAKE_CXX_COMPILER ${MPI_CXX_COMPILER})
  #set(CMAKE_C_COMPILER ${MPI_C_COMPILER})
  #message("cxx compiler: ${MPI_CXX_COMPILER}")
  #message("c compiler: ${MPI_C_COMPILER}")

  #set(CMAKE_CXX_COMPILE_FLAGS ${CMAKE_CXX_COMPILE_FLAGS}
  #  ${MPI_CXX_COMPILE_FLAGS})
  #set(CMAKE_CXX_LINK_FLAGS ${CMAKE_CXX_LINK_FLAGS}
  #  ${MPI_CXX_LINK_FLAGS})

  set(CMAKE_CXX_COMPILER ${MAXWELL_MPI_BASE_DIR}/bin/mpicxx)
  set(CMAKE_C_COMPILER ${MAXWELL_MPI_BASE_DIR}/bin/mpicc)
endif(${MAXWELL_USE_MPI})


option(MAXWELL_USE_ParMETIS "Path to lib and include directories
  for ParMETIS. ParMETIS is used for load balancing." OFF)
set(MAXWELL_ParMETIS_BASE_DIR "/usr/local" CACHE PATH "Location of
  ParMETIS directories. Defaults to /usr/local")

set(TRILINOS_BASE_DIR "/usr/local" CACHE PATH
  "Path to lib and include directories
  for Trilinos. If not given or found automatically, the Trilinos
  libraries will be downloaded and compiled for you.")
#set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${MX_TRILINOS_BASE_DIR}/lib/cmake/Trilinos")

find_package(Trilinos PATHS ${TRILINOS_BASE_DIR})

if(Trilinos_FOUND)
  message("  Trilinos was found!")
  set(MxTrilinosIncludeDirs ${Trilinos_INCLUDE_DIRS})
  set(MxTrilinosLinkDirs ${Trilinos_LIBRARY_DIRS})
  set(MxTrilinosLinkLibs ${Trilinos_LIBRARIES})
else(Trilinos_FOUND)
  # build trilinos using cmake ExternalProject module
  # set flags that should always be set
  set(MxTrilinosFlags
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DTrilinos_ENABLE_ALL_PACKAGES:BOOL=OFF
    -DTrilinos_ENABLE_TESTS:BOOL=OFF
    -DTrilinos_ENABLE_Anasazi:BOOL=ON
    -DTrilinos_ENABLE_ML:BOOL=ON
    -DTrilinos_ENABLE_AztecOO:BOOL=ON
    -DTrilinos_ENABLE_Ifpack:BOOL=ON
    -DTrilinos_ENABLE_Amesos:BOOL=ON
    -DTrilinos_ENABLE_Komplex:BOOL=ON
    -DTrilinos_ENABLE_EpetraExt:BOOL=ON
    -DTrilinos_ENABLE_Epetra:BOOL=ON
    -DTrilinos_ENABLE_Zoltan:BOOL=ON
    -DTrilinos_ENABLE_Teuchos:BOOL=ON
    -DTeuchos_ENABLE_COMPLEX:BOOL=ON
    -DTeuchos_ENABLE_EXTENDED:BOOL=ON
    -DTrilinos_ENABLE_Triutils:BOOL=ON
    -DTrilinos_ENABLE_Fortran:BOOL=OFF
  )
  
  if(${CMAKE_BUILD_TYPE} STREQUAL "DEBUG")
    set(MxTrilinosFlags ${MxTrilinosFlags}
      -DTeuchos_ENABLE_DEBUG:BOOL=ON
      -DTeuchos_ENABLE_DEBUG_RCP_NODE_TRACING:BOOL=ON
    )
  endif(${CMAKE_BUILD_TYPE} STREQUAL "DEBUG")
  
  # trilinos library names for linking
  set(MxTrilinosLinkLibs
    anasazi
    ml
    aztecoo
    ifpack
    amesos
    komplex
    tpetra
    kokkos
    kokkosnodeapi
    tpi
    epetraext
    epetra
    zoltan
    teuchos
    triutils
    bfd
  )
    
  set(MxTrilinosIncludeDirs "${CMAKE_BINARY_DIR}/extproj/trilinos/include")
  set(MxTrilinosLinkDirs "${CMAKE_BINARY_DIR}/extproj/trilinos/lib")
  
  if(${MAXWELL_USE_MPI})
    set(MxTrilinosFlags ${MxTrilinosFlags}
      -DTPL_ENABLE_MPI:BOOL=ON
      -DMPI_BASE_DIR:PATH=${MAXWELL_MPI_BASE_DIR}
    )
    if(${MAXWELL_USE_ParMETIS})
      set(MxTrilinosFlags ${MxTrilinosFlags}
        -DTPL_ENABLE_ParMETIS:BOOL=ON
        -DTPL_ParMETIS_LIBRARIES:PATH=${MAXWELL_ParMETIS_BASE_DIR}/lib/libparmetis.a:${MAXWELL_ParMETIS_BASE_DIR}/lib/metis.a
        -DTPL_ParMETIS_INCLUDE_DIRS:PATH=${MAXWELL_ParMETIS_BASE_DIR}/include
        #-DParMETIS_LIBRARY_DIRS:PATH=${MAXWELL_ParMETIS_BASE_DIR}/lib
      )
    endif(${MAXWELL_USE_ParMETIS})
  endif(${MAXWELL_USE_MPI})
  
  ExternalProject_Add(MxTrilinos
    PREFIX extproj/trilinos
    #URL "http://trilinos.sandia.gov/download/files/trilinos-10.8.5-Source.tar.gz"
    URL "http://trilinos.sandia.gov/download/files/trilinos-10.12.2-Source.tar.gz"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
      ${MxTrilinosFlags} 
    #PATCH_COMMAND patch -p4 -t -N < ${CMAKE_SOURCE_DIR}/patch/klp.patch
  )

endif(Trilinos_FOUND)

option(USE_PREBUILT_HDF5 OFF)
set(HDF5_BASE_DIR "/usr/local" CACHE PATH
  "Path to lib and include directories
  for HDF5. If not given or found automatically, the Trilinos
  libraries will be downloaded and compiled for you.")

#find_package(HDF5 PATHS ${HDF5_BASE_DIR})

#find_library(Hdf5Lib hdf5 PATHS ${HDF5_BASE_DIR})
#if(Hdf5Lib)
#  set(HDF5_FOUND ON)
#endif(Hdf5Lib)

if(USE_PREBUILT_HDF5)
  message("-- Using pre-built HDF5")
  set(MxH5IncludeDirs "${HDF5_BASE_DIR}/include")
  set(MxH5LinkDirs "${HDF5_BASE_DIR}/lib")
  #set(MxH5LinkLibs ${HDF5_LIBRARIES})
  #add_library(hdf5 STATIC)
  set(MxH5LinkLibs hdf5)
else(USE_PREBUILT_HDF5)
  # build hdf5 as external project
  set(MxHDF5Flags
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DHDF5_ENABLE_Z_LIB_SUPPORT:BOOL=ON
  )
  
  # hdf5 library names for linking
  if(${CMAKE_BUILD_TYPE} STREQUAL "DEBUG")
    set(MxH5LinkLibs
      hdf5_debug
    )
  else(${CMAKE_BUILD_TYPE} STREQUAL "DEBUG")
    set(MxH5LinkLibs
      hdf5
    )
  endif(${CMAKE_BUILD_TYPE} STREQUAL "DEBUG")
  
  if(${MAXWELL_USE_MPI})
    set(MxHDF5Flags ${MxHDF5Flags}
      -DHDF5_ENABLE_PARALLEL:BOOL=ON
      -DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}
      -DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}
    )
  endif(${MAXWELL_USE_MPI})
  
  ExternalProject_Add(MxHDF5
    PREFIX extproj/hdf5
    URL "http://www.hdfgroup.org/ftp/HDF5/prev-releases/hdf5-1.8.8/src/hdf5-1.8.8.tar.gz"
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
      ${MxHDF5Flags} 
  )
  
  set(MxH5IncludeDirs "${CMAKE_BINARY_DIR}/extproj/hdf5/include")
  set(MxH5LinkDirs "${CMAKE_BINARY_DIR}/extproj/hdf5/lib")
endif(USE_PREBUILT_HDF5)

##find_package(Trilinos PATHS /usr/local ${TRILINOS_DIRS})
#find_package(Trilinos REQUIRED)
#
## Echo trilinos build info just for fun
#if(Trilinos_FOUND)
#  message("\nFound Trilinos!  Here are the details: ")
#  message("   Trilinos_DIR = ${Trilinos_DIR}")
#  message("   Trilinos_VERSION = ${Trilinos_VERSION}")
#  message("   Trilinos_PACKAGE_LIST = ${Trilinos_PACKAGE_LIST}")
#  message("   Trilinos_LIBRARIES = ${Trilinos_LIBRARIES}")
#  message("   Trilinos_INCLUDE_DIRS = ${Trilinos_INCLUDE_DIRS}")
#  message("   Trilinos_LIBRARY_DIRS = ${Trilinos_LIBRARY_DIRS}")
#  message("   Trilinos_TPL_LIST = ${Trilinos_TPL_LIST}")
#  message("   Trilinos_TPL_INCLUDE_DIRS = ${Trilinos_TPL_INCLUDE_DIRS}")
#  message("   Trilinos_TPL_LIBRARIES = ${Trilinos_TPL_LIBRARIES}")
#  message("   Trilinos_TPL_LIBRARY_DIRS = ${Trilinos_TPL_LIBRARY_DIRS}")
#  message("   Trilinos_BUILD_SHARED_LIBS = ${Trilinos_BUILD_SHARED_LIBS}")
#  message("End of Trilinos details\n")
#endif(Trilinos_FOUND)
#
## Make sure to use same compilers and flags as Trilinos
#set(CMAKE_CXX_COMPILER ${Trilinos_CXX_COMPILER} )
#set(CMAKE_C_COMPILER ${Trilinos_C_COMPILER} )
#set(CMAKE_Fortran_COMPILER ${Trilinos_Fortran_COMPILER} )
#
#set(CMAKE_CXX_FLAGS  "${Trilinos_CXX_COMPILER_FLAGS} ${CMAKE_CXX_FLAGS}")
#set(CMAKE_C_FLAGS  "${Trilinos_C_COMPILER_FLAGS} ${CMAKE_C_FLAGS}")
#set(CMAKE_Fortran_FLAGS  "${Trilinos_Fortran_COMPILER_FLAGS} ${CMAKE_Fortran_FLAGS}")
#
#message("${Trilinos_CXX_COMPILER_FLAGS}")
#message("${Trilinos_C_COMPILER_FLAGS}")
#message("${Trilinos_Fortran_COMPILER_FLAGS}")

#
#set(HDF5_BASE_DIR "/usr/local" CACHE PATH "Base directory for HDF5 installation")
#
#
#
#find_library(HDF5_LIBRARY hdf5 "${HDF5_BASE_DIR}/lib")
#if(HDF5_LIBRARY)
#  message("-- Found hdf5: ${HDF5_LIBRARY}")
#endif(HDF5_LIBRARY)
#
##set(HDF5_USE_STATIC_LIBRARIES OFF)
##find_package(HDF5 REQUIRED HINTS ${HDF5_BASE_DIR})
##find_package(HDF5 COMPONENTS C CXX HINTS ${HDF5_BASE_DIR})
##find_package(HDF5 REQUIRED HINTS /scr_verus/bauerca/prog)
##if(HDF5_FOUND)
##  message("-- Found HDF5: ${HDF5_DIR}")
##  message("-- HDF5 Library Dirs: ${HDF5_INCLUDE_DIRS}")
##endif(HDF5_FOUND)
#
#
#set(EXTERN_LIB_DIRS ${HDF5_LIBRARY_DIRS} ${Trilinos_LIBRARY_DIRS})
#message("external library dirs: ${EXTERN_LIB_DIRS}")
#set(EXTERN_INC_DIRS ${HDF5_INCLUDE_DIRS} ${Trilinos_INCLUDE_DIRS})
#set(LIBS ${LIBS} ${HDF5_LIBRARIES} ${Trilinos_LIBRARIES})
#
#set(dummy "slugturd" CACHE "" STRING)
#message("dummy is set to ${dummy}")
#message("dummy2 is set to ${dummy2}")

#
# End of setup and error checking
#  NOTE: PROJECT command checks for compilers, so this statement
#        is moved AFTER setting CMAKE_CXX_COMPILER from Trilinos

project(Maxwell)


set(maxwell_VERSION_MAJOR 0)
set(maxwell_VERSION_MINOR 1)

include_directories(${MxTrilinosIncludeDirs})
include_directories(${MxH5IncludeDirs})

link_directories(${MxTrilinosLinkDirs})
link_directories(${MxH5LinkDirs})

set(MAXWELL_EXTPROJ_LINK_LIBS ${MxTrilinosLinkLibs} ${MxH5LinkLibs})
if(MAXWELL_USE_MPI)
  set(MAXWELL_EXTPROJ_LINK_LIBS ${MAXWELL_EXTPROJ_LINK_LIBS} mpi)
endif(MAXWELL_USE_MPI)

# configure a header file to pass some of the CMake settings
# to the source code
#configure_file (
#  "${Maxwell_SOURCE_DIR}/src/MxConfig.h.in"
#  "${Maxwell_BINARY_DIR}/src/MxConfig.h"
#)

add_subdirectory(src)
add_subdirectory(test)
