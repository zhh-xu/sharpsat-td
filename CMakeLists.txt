cmake_minimum_required (VERSION 3.5)
project (SHARPSAT C CXX)

option(STATIC "STATIC" OFF)
option(SSTD_BUILD_EXECUTABLES "Build the sharpSAT and flow_cutter_pace17 executables" ON)
set(CMAKE_VERBOSE_MAKEFILE on)

if(STATIC)
set(CMAKE_EXE_LINKER_FLAGS -static)
set(CMAKE_FIND_LIBRARY_SUFFIXES .a)
set(CMAKE_EXE_LINK_DYNAMIC_C_FLAGS)
set(CMAKE_EXE_LINK_DYNAMIC_CXX_FLAGS)
set(CMAKE_SHARED_LIBRARY_C_FLAGS)
set(CMAKE_SHARED_LIBRARY_CXX_FLAGS)
set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)
set(BUILD_SHARED_LIBS OFF)
endif(STATIC)

set(CMAKE_CXX_FLAGS "-std=c++17 -O3 -Wall -Wextra -Wshadow -g -Wfatal-errors -Wno-sign-compare -Wno-literal-suffix")

SET(CMAKE_C_FLAGS "-std=c99 -Wall")
# clhash needs SSE4.1/PCLMUL on x86; it has its own code path on ARM64.
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64|amd64|i[3-6]86)$")
SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse4.1 -mpclmul")
endif()

find_library(GMP_LIB gmp)
find_library(GMPXX_LIB gmpxx)
find_library(MPFR_LIB mpfr)
find_path(GMP_INCLUDE_DIR gmp.h)
find_path(MPFR_INCLUDE_DIR mpfr.h)

add_definitions( -DMPFR_USE_NO_MACRO )
add_subdirectory(flowcutter)

SET(SOURCES
	src/preprocessor/graph.cpp
	src/preprocessor/instance.cpp
	src/preprocessor/oracle.cpp
	src/preprocessor/preprocessor.cpp
	src/preprocessor/subsumer.cpp
	src/preprocessor/treewidth.cpp
	src/preprocessor/twpp.cpp
	src/decomposition.cpp
	src/clhash/clhash.c
)

# The counter itself is header-only templates; this library holds the compiled
# sources so that other projects can embed the solver.
add_library(sstd STATIC ${SOURCES})
target_include_directories(sstd PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src)
target_include_directories(sstd SYSTEM PUBLIC ${GMP_INCLUDE_DIR} ${MPFR_INCLUDE_DIR})
target_link_libraries(sstd PUBLIC ${MPFR_LIB} ${GMPXX_LIB} ${GMP_LIB})

if(SSTD_BUILD_EXECUTABLES)
add_executable (sharpSAT src/main.cpp)
if(STATIC)
SET_TARGET_PROPERTIES (sharpSAT PROPERTIES LINK_SEARCH_START_STATIC 1)
endif(STATIC)
target_link_libraries (sharpSAT sstd)

add_executable (flow_cutter_pace17 flowcutter/pace.cpp)
if(STATIC)
SET_TARGET_PROPERTIES (flow_cutter_pace17 PROPERTIES LINK_SEARCH_START_STATIC 1)
endif(STATIC)
target_link_libraries (flow_cutter_pace17 libflowcutter)
endif(SSTD_BUILD_EXECUTABLES)
