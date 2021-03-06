cmake_minimum_required(VERSION 3.1)

project(minnow)

if(NO_RTM)
  set(TBB_CXXFLAGS "-mno-rtm")
endif()


include(ExternalProject)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/Modules")

# We require C++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(RBF_CPP_FLAGS "-pthread -std=c++11 -W -Wall -Wextra -Wpointer-arith -Wunused -Wwrite-strings -openmp -Wno-unknown-pragmas -Wno-unused-function -O3 -fPIC")

set(DEBUG_FLAGS "-g")
set(RBF_CPP_FLAGS "${KSW_FLAGS} ${DEBUG_FLAGS} ${RBF_CPP_FLAGS} ${OPT_FLAGS}")
set(CMAKE_CXX_FLAGS ${RBF_CPP_FLAGS})
message("${CMAKE_CXX_FLAGS}")

if (NOT GAT_SOURCE_DIR)
    message("Top-level source directory variable not set externally; setting it to ${CMAKE_CURRENT_SOURCE_DIR}")
    set(GAT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
endif()

##
# Let us check the sha sum of our packages if we have the right tools
##
set(SHASUM ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_shasum.sh)

#find_package(JeMalloc)
#    if(JEMALLOC_FOUND)
#      include_directories(SYSTEM ${JEMALLOC_INCLUDE_DIRS})
#    endif()
#
#
#include(ExternalProject)

set(GAT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

#message("Build system will NOT compile libgff")
##
# Let us check the sha sum of our pacakges if we have the right tools
##
#set(SHASUM ${CMAKE_CURRENT_SOURCE_DIR}/scripts/check_shasum.sh)

#message("Build system will compile libgff")
#message("==================================================================")
#ExternalProject_Add(libgff
#    DOWNLOAD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external
#    DOWNLOAD_COMMAND curl -k -L https://github.com/COMBINE-lab/libgff/archive/v1.1.tar.gz -o libgff.tgz &&
#                     ${SHASUM} 34597ecd3718e6d6e7a1dd4350220664be2c1cd992070d1218564a20d7aa5697  libgff.tgz &&
#                     tar -xzvf libgff.tgz 
#    ##
#    #URL https://github.com/COMBINE-lab/libgff/archive/v1.1.tar.gz
#    #DOWNLOAD_NAME libff.tgz
#    #URL_HASH SHA1=37b3147d78391d1fabbe6a0df313fbf516abbc6f
#    #TLS_VERIFY FALSE
#    ##
#    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/libgff-1.1
#    #UPDATE_COMMAND sh -c "mkdir -p <SOURCE_DIR>/build"
#    INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install
#    BINARY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/libgff-1.1/build
#    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_CURRENT_SOURCE_DIR}/external/install
#)
#ExternalProject_Add_Step(libgff makedir
#  COMMAND mkdir -p <SOURCE_DIR>/build 
#  COMMENT "Make build directory"
#  DEPENDEES download 
#  DEPENDERS configure)


###
#
# Done building external dependencies.
#
###

## Try and find TBB first
find_package(TBB 2018.0 COMPONENTS tbb tbbmalloc tbbmalloc_proxy)

if (${TBB_FOUND})
  if (${TBB_VERSION} VERSION_GREATER_EQUAL 2018.0)
  message("FOUND SUITABLE TBB VERSION : ${TBB_VERSION}")
  set(TBB_TARGET_EXISTED TRUE)
  else()
    set(TBB_TARGET_EXISTED FALSE)
  endif()
else()
  set(TBB_TARGET_EXISTED FALSE)
endif()

##
#
# Fetch and build Intel's Threading Building Blocks library.
#
##
if(NOT ${TBB_TARGET_EXISTED})

set(TBB_WILL_RECONFIGURE TRUE)
# Set the appropriate compiler
if(CLANG)
  set(TBB_COMPILER "clang")
else()
  set(TBB_COMPILER "gcc")
endif()

message("Build system will fetch and build Intel Threading Building Blocks")
message("==================================================================")
# These are useful for the custom install step we'll do later
set(TBB_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/oneTBB-2019_U8)
#set(TBB_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/tbb-2019_U8)
set(TBB_INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install)

if("${TBB_COMPILER}" STREQUAL "gcc")
  ## Don't know why it's a problem yet, but if we're using
  ## GCC, get rid of the DO_ITT_NOTIFY flag
  set(TBB_CXXFLAGS "${TBB_CXXFLAGS} -UDO_ITT_NOTIFY")
endif()

set(TBB_CXXFLAGS "${TBB_CXXFLAGS} ${CXXSTDFLAG}")

externalproject_add(libtbb
  DOWNLOAD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external
  DOWNLOAD_COMMAND curl -k -L https://github.com/intel/tbb/archive/2019_U8.tar.gz -o tbb-2019_U8.tgz &&
    ${SHASUM} 6b540118cbc79f9cbc06a35033c18156c21b84ab7b6cf56d773b168ad2b68566 tbb-2019_U8.tgz &&
    tar -xzvf tbb-2019_U8.tgz
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/oneTBB-2019_U8
  INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install
  PATCH_COMMAND "${TBB_PATCH_STEP}"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND make ${QUIET_MAKE} CXXFLAGS=${TBB_CXXFLAGS} lambdas=1 compiler=${TBB_COMPILER} cfg=release tbb_build_prefix=LIBS
  INSTALL_COMMAND sh -c "mkdir -p ${TBB_INSTALL_DIR}/include && mkdir -p ${TBB_INSTALL_DIR}/lib && cp ${TBB_SOURCE_DIR}/build/LIBS_release/*.${SHARED_LIB_EXTENSION}* ${TBB_INSTALL_DIR}/lib && cp -r ${TBB_SOURCE_DIR}/include/* ${TBB_INSTALL_DIR}/include"
  BUILD_IN_SOURCE 1
)




set(RECONFIG_FLAGS ${RECONFIG_FLAGS} -DTBB_WILL_RECONFIGURE=FALSE -DTBB_RECONFIGURE=TRUE)
externalproject_add_step(libtbb reconfigure
  COMMAND ${CMAKE_COMMAND} ${CMAKE_CURRENT_SOURCE_DIR} ${RECONFIG_FLAGS}
  DEPENDEES install
)

set(FETCHED_TBB TRUE)
if(${FETCHED_BOOST})
  add_dependencies(libtbb libboost)
endif()
endif()

##
# If we're fetching tbb, we need to have dummy paths for these variables
# so that CMake won't complain
##
if(TBB_WILL_RECONFIGURE)
    set(TBB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install)
    set(TBB_INCLUDE_DIRS ${TBB_INSTALL_DIR}/include)
    set(TBB_INCLUDE_DIR ${TBB_INSTALL_DIR}/include)
    set(TBB_LIBRARY_DIRS ${TBB_INSTALL_DIR}/lib)
    set(TBB_LIBRARY ${TBB_INSTALL_DIR}/lib)
    set(TBB_LIB_DIR ${TBB_INSTALL_DIR}/lib)
    #set(TBB_LIBRARIES tbb tbbmalloc)
    set(TBB_LIBRARIES ${TBB_INSTALL_DIR}/lib/libtbb.${SHARED_LIB_EXTENSION}
    			${TBB_INSTALL_DIR}/lib/libtbbmalloc.${SHARED_LIB_EXTENSION}
	   )
    message("TBB_INCLUDE_DIRS = ${TBB_INCLUDE_DIRS}")
    message("TBB_LIBRARY_DIRS = ${TBB_LIBRARY_DIRS}")
endif()

##
#  Similar to the Boost trick above, the libtbb reconfigure should force this code
#  to be run on the second configuration pass, where it should appropriately set the
#  TBB_INSTALL_DIR variable.
##
if(TBB_RECONFIGURE)
  unset(TBB_FOUND CACHE)
  unset(TBB_INSTALL_DIR CACHE)
  unset(CMAKE_PREFIX_PATH CACHE)
  unset(TBB_INCLUDE_DIRS CACHE)
  unset(TBB_INCLUDE_DIR CACHE)
  unset(TBB_LIBRARY_DIRS CACHE)
  unset(TBB_LIBRARY CACHE)
  unset(TBB_LIBRARIES CACHE)
  set(CMAKE_PREFIX_PATH ${CMAKE_CURRENT_SOURCE_DIR}/external/install)
  set(TBB_INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install)
  set(TBB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install)
  set(TBB_INCLUDE_DIRS ${TBB_INSTALL_DIR}/include)
  set(TBB_INCLUDE_DIR ${TBB_INSTALL_DIR}/include)
  set(TBB_LIBRARY_DIRS ${TBB_INSTALL_DIR}/lib)
  set(TBB_LIBRARY ${TBB_INSTALL_DIR}/lib)
  set(TBB_LIB_DIR ${TBB_INSTALL_DIR}/lib)
  message("TBB_INSTALL_DIR = ${TBB_INSTALL_DIR}")
  find_package(TBB 2018.0 COMPONENTS tbb tbbmalloc tbbmalloc_proxy)
  message("[in TBB_RECONFIGURE] TBB_LIBRARIES = ${TBB_LIBRARIES}")
endif()

message("TBB_LIBRARIES = ${TBB_LIBRARIES}")


set(LIB_FLAGS "-L ${CMAKE_CURRENT_SOURCE_DIR}/external/lib")
set (CPACK_SOURCE_IGNORE_FILES
".git/")

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/external/twopaco)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/external/ntcard)

include_directories(include)
link_directories(lib)
add_subdirectory(src)
