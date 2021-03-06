# CMakeLists.txt for event package. It creates a library with dictionary and a main program
cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
project(pluto)

include(CTest)

include_directories(include ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR})

###########################################################
SET(PLUGIN_CLASSES "")
SET(PLUGIN_CLASSES_NAMES "")
SET(PLUGIN_COLLECTION "")

FUNCTION(addprefix var prefix)
   ##MESSAGE( STATUS "prefix: " ${prefix} )
   SET(listVar "")
   FOREACH(f ${ARGN})
      LIST(APPEND listVar "${prefix}/${f}")
   ENDFOREACH(f)
   SET(${var} "${listVar}" PARENT_SCOPE)
ENDFUNCTION(addprefix)

FUNCTION(addsuffix var suffix)
   SET(listVar "")
   FOREACH(f ${ARGN})
      LIST(APPEND listVar "${f}${suffix}")
   ENDFOREACH(f)
   SET(${var} "${listVar}" PARENT_SCOPE)
ENDFUNCTION(addsuffix)

add_subdirectory(plugins)
##MESSAGE( STATUS "PLUGIN_CLASSES: " ${PLUGIN_CLASSES} )

addsuffix(PHDRS ".h"  ${PLUGIN_CLASSES})
addsuffix(PSRC  ".cc" ${PLUGIN_CLASSES})
##MESSAGE( STATUS "PHDRS: " ${PHDRS} )
##MESSAGE( STATUS "PLUGIN_CLASSES_NAMES: " ${PLUGIN_CLASSES_NAMES} )
##MESSAGE( STATUS "PLUGIN_COLLECTION: " ${PLUGIN_COLLECTION} )

###########################################################

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/Plugins.h)
FOREACH(f ${PHDRS})
   file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/Plugins.h "\#include \"${f}\"\n")
ENDFOREACH(f)

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/Plugins.cc)
FOREACH(f ${PLUGIN_COLLECTION})
   file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/Plugins.cc "\#include \"${f}\"\n")
ENDFOREACH(f)

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/PluginLinkDef.h)
FOREACH(f ${PLUGIN_CLASSES_NAMES})
   file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/PluginLinkDef.h "\#pragma link C++ class ${f};\n")
ENDFOREACH(f) 

###########################################################


file(GLOB SOURCES "src/*.cc")
SET(SOURCES ${SOURCES} ${PLUGIN_CLASSES})

#file(GLOB INCL_FILES RELATIVE "${CMAKE_SOURCE_DIR}" "include/*.h" )
file(GLOB INCL_FILES RELATIVE "${CMAKE_CURRENT_BINARY_DIR}" "include/*.h" )
SET(INCL_FILES ${INCL_FILES} ${PHDRS})

MESSAGE( STATUS "INCL_FILES: " ${INCL_FILES} )

# You need to tell CMake where to find the ROOT installation. This can be done in a number of ways:
#   - ROOT built with classic configure/make use the provided $ROOTSYS/etc/cmake/FindROOT.cmake
#   - ROOT built with CMake. Add in CMAKE_PREFIX_PATH the installation prefix for ROOT
#list(APPEND CMAKE_PREFIX_PATH $ENV{ROOTSYS})
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} $ENV{ROOTSYS}/etc/cmake)

#---Locate the ROOT package and defines a number of variables (e.g. ROOT_INCLUDE_DIRS)
find_package(ROOT "6.00" REQUIRED COMPONENTS RIO Net)
if(ROOT_VERSION LESS 6.00)
	message(FATAL_ERROR "The required ROOT version has to be at least 6.00, found " ${ROOT_VERSION})
endif()
message(STATUS "Found ROOT version: " ${ROOT_VERSION})

#---Define useful ROOT functions and macros (e.g. ROOT_GENERATE_DICTIONARY)
include(${ROOT_USE_FILE})

# explicitly tell the linker to use the EG library to resolve linker problems with TDatabasePDG::Instance()
set(DEFAULT_LINKER_FLAGS "-lEG")

##LIST(APPEND INCL_FILES ${PHDRS})

#include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
ROOT_GENERATE_DICTIONARY(G__Pluto ${INCL_FILES} LINKDEF PlutoLinkDef.h)

#---Create a shared library with geneated dictionary
add_library(Pluto SHARED ${SOURCES} G__Pluto.cxx ${CMAKE_CURRENT_BINARY_DIR}/Compiled.h)
target_link_libraries(Pluto ${ROOT_LIBRARIES})

#---Create  a main program using the library
#add_executable(Main MainEvent.cxx)
#target_link_libraries(Main Event)

add_custom_target(Compiled.h)
add_custom_command(OUTPUT Compiled.h COMMAND echo char *date_string = \\\(char*\\\) \\\"\$\$\(date +\"%e %B %Y\"\)\\\"  > ${CMAKE_CURRENT_BINARY_DIR}/Compiled.h DEPENDS G__Pluto.cxx ${SOURCES})



#add_custom_target(PluginLinkdef.h)
