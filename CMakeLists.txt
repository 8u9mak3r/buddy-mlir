##===- CMakeLists.txt - buddy-mlir cmake root -----------------*- cmake -*-===//
##
## Configure the buddy-mlir build.
##
##===----------------------------------------------------------------------===//

cmake_minimum_required(VERSION 3.10)

if(POLICY CMP0077)
  cmake_policy(SET CMP0077 NEW)
endif()

if(POLICY CMP0116)
  cmake_policy(SET CMP0116 OLD)
endif()

#-------------------------------------------------------------------------------
# Project setup and globals
#-------------------------------------------------------------------------------

project(buddy-mlir LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED YES)
include(ExternalProject)

#-------------------------------------------------------------------------------
# Options and settings
#-------------------------------------------------------------------------------

option(LLVM_INCLUDE_TOOLS "Generate build targets for the LLVM tools." ON)
option(LLVM_BUILD_TOOLS "Build the LLVM tools. If OFF, just generate build targets." ON)
option(BUDDY_MLIR_OUT_OF_TREE_BUILD "Specifies an out of tree build" OFF)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR OR BUDDY_MLIR_OUT_OF_TREE_BUILD)
    message(STATUS "buddy-mlir two-step build.")
    # Two-step build
    #-------------------------------------------------------------------------------
    # MLIR/LLVM Configuration
    #-------------------------------------------------------------------------------
    find_package(MLIR REQUIRED CONFIG)
    message(STATUS "Using MLIRConfig.cmake in: ${MLIR_DIR}")
    message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

    # LLVM_MAIN_SRC_DIR is a private variable for the LLVM in-tree build.
    # To provide compatibility for unifying the one-step and two-step build,
    # we set LLVM_MAIN_SRC_DIR ourselves here.
    # This could benefit users who want to specify a custom LLVM source directory,
    # but also not interfere with normal users who just want to use the buddy-mlir provided LLVM sources.
    if(NOT DEFINED LLVM_MAIN_SRC_DIR)
      get_filename_component(LLVM_MAIN_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/llvm/llvm ABSOLUTE)
    endif()
    set(MLIR_MAIN_SRC_DIR ${LLVM_MAIN_SRC_DIR}/../mlir)

    list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
    list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

    find_program(LLVM_TABLEGEN_EXE "llvm-tblgen" ${LLVM_TOOLS_BINARY_DIR}
            NO_DEFAULT_PATH)

    include(TableGen)
    include(AddLLVM)
    include(AddMLIR)
    include(HandleLLVMOptions)
else()
    message(STATUS "buddy-mlir one-step build.")
    # one-step build with LLVM_EXTERNAL_PROJECTS=buddy-mlir
    #-------------------------------------------------------------------------------
    # MLIR/LLVM Configuration
    #-------------------------------------------------------------------------------
    set(MLIR_MAIN_SRC_DIR ${LLVM_MAIN_SRC_DIR}/../mlir)
    set(MLIR_INCLUDE_DIR ${MLIR_MAIN_SRC_DIR}/include)
    set(MLIR_GENERATED_INCLUDE_DIR ${LLVM_BINARY_DIR}/tools/mlir/include)
endif()

#-------------------------------------------------------------------------------
# BUDDY configuration
#-------------------------------------------------------------------------------

# BUDDY project.
set(BUDDY_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(BUDDY_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(BUDDY_MLIR_PYTHON_PACKAGES_DIR ${BUDDY_BUILD_DIR}/python_packages)


#-------------------------------------------------------------------------------
# Initialize Python packages
#-------------------------------------------------------------------------------
if(BUDDY_MLIR_ENABLE_PYTHON_PACKAGES)
  # Find the Python interpreter and development components,
  # requiring a minimum version of 3.10
  find_package(Python3 3.10 REQUIRED COMPONENTS Interpreter Development)
  # Create directories for the BUDDY-MLIR Python packages
  file(MAKE_DIRECTORY ${BUDDY_MLIR_PYTHON_PACKAGES_DIR}/buddy)
  file(MAKE_DIRECTORY ${BUDDY_MLIR_PYTHON_PACKAGES_DIR}/buddy/compiler)
  # Create empty __init__.py files to make these directories Python packages
  file(WRITE ${BUDDY_MLIR_PYTHON_PACKAGES_DIR}/buddy/__init__.py "")
  file(WRITE ${BUDDY_MLIR_PYTHON_PACKAGES_DIR}/buddy/compiler/__init__.py "")

  install(DIRECTORY ${BUDDY_MLIR_PYTHON_PACKAGES_DIR}/buddy DESTINATION python_packages)
endif()

#-------------------------------------------------------------------------------
# Directory setup
#-------------------------------------------------------------------------------
add_subdirectory(frontend)
