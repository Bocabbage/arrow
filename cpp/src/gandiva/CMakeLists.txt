# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

set(GANDIVA_VERSION "${ARROW_VERSION}")

# For "make gandiva" to build everything Gandiva-related
add_custom_target(gandiva-all)
add_custom_target(gandiva)
add_custom_target(gandiva-tests)
add_custom_target(gandiva-benchmarks)

add_dependencies(gandiva-all gandiva gandiva-tests gandiva-benchmarks)

# If libgandiva.a is only built, "pkg-config --cflags --libs
# arrow-flight" outputs build flags for static linking not shared
# linking. GANDIVA_PC_* except GANDIVA_PC_*_PRIVATE are for the static
# linking case.
if(NOT ARROW_BUILD_SHARED AND ARROW_BUILD_STATIC)
  string(APPEND GANDIVA_PC_CFLAGS "${GANDIVA_PC_CFLAGS_PRIVATE}")
  set(GANDIVA_PC_CFLAGS_PRIVATE "")
endif()

include(GandivaAddBitcode)

find_package(LLVMAlt REQUIRED)
provide_find_module(LLVMAlt "Gandiva")
if(ARROW_WITH_ZSTD AND "${zstd_SOURCE}" STREQUAL "SYSTEM")
  provide_find_module(zstdAlt "Gandiva")
endif()
provide_cmake_module(GandivaAddBitcode "Gandiva")

# Set the path where the bitcode file generated, see precompiled/CMakeLists.txt
set(GANDIVA_PRECOMPILED_BC_PATH "${CMAKE_CURRENT_BINARY_DIR}/irhelpers.bc")
set(GANDIVA_PRECOMPILED_CC_PATH "${CMAKE_CURRENT_BINARY_DIR}/precompiled_bitcode.cc")
set(GANDIVA_PRECOMPILED_CC_IN_PATH
    "${CMAKE_CURRENT_SOURCE_DIR}/precompiled_bitcode.cc.in")

# add_arrow_lib will look for this not yet existing file, so flag as generated
set_source_files_properties(${GANDIVA_PRECOMPILED_CC_PATH} PROPERTIES GENERATED TRUE)

set(SRC_FILES
    annotator.cc
    bitmap_accumulator.cc
    cache.cc
    cast_time.cc
    configuration.cc
    context_helper.cc
    decimal_ir.cc
    decimal_type_util.cc
    decimal_xlarge.cc
    engine.cc
    date_utils.cc
    encrypt_utils.cc
    expr_decomposer.cc
    expr_validator.cc
    expression.cc
    expression_registry.cc
    exported_funcs_registry.cc
    exported_funcs.cc
    external_c_functions.cc
    filter.cc
    function_holder_maker_registry.cc
    function_ir_builder.cc
    function_registry.cc
    function_registry_arithmetic.cc
    function_registry_datetime.cc
    function_registry_hash.cc
    function_registry_math_ops.cc
    function_registry_string.cc
    function_registry_timestamp_arithmetic.cc
    function_signature.cc
    gandiva_object_cache.cc
    gdv_function_stubs.cc
    gdv_string_function_stubs.cc
    gdv_hash_function_stubs.cc
    hash_utils.cc
    interval_holder.cc
    llvm_generator.cc
    llvm_types.cc
    literal_holder.cc
    projector.cc
    regex_util.cc
    regex_functions_holder.cc
    selection_vector.cc
    tree_expr_builder.cc
    to_date_holder.cc
    random_generator_holder.cc
    ${GANDIVA_PRECOMPILED_CC_PATH})

set(GANDIVA_OPENSSL_LIBS ${ARROW_OPENSSL_LIBS})
if(WIN32 AND NOT CMAKE_VERSION VERSION_LESS 3.18)
  list(APPEND GANDIVA_OPENSSL_LIBS OpenSSL::applink)
endif()

set(GANDIVA_SHARED_LINK_LIBS arrow_shared LLVM::LLVM_HEADERS)
set(GANDIVA_SHARED_PRIVATE_LINK_LIBS LLVM::LLVM_LIBS ${GANDIVA_OPENSSL_LIBS}
                                     Boost::headers)
set(GANDIVA_STATIC_LINK_LIBS
    arrow_static
    LLVM::LLVM_HEADERS
    LLVM::LLVM_LIBS
    ${GANDIVA_OPENSSL_LIBS}
    Boost::headers)
if(ARROW_USE_XSIMD)
  list(APPEND GANDIVA_SHARED_PRIVATE_LINK_LIBS ${ARROW_XSIMD})
  list(APPEND GANDIVA_STATIC_LINK_LIBS ${ARROW_XSIMD})
endif()
if(ARROW_WITH_RE2)
  list(APPEND GANDIVA_SHARED_PRIVATE_LINK_LIBS re2::re2)
  list(APPEND GANDIVA_STATIC_LINK_LIBS re2::re2)
endif()
if(ARROW_WITH_UTF8PROC)
  list(APPEND GANDIVA_SHARED_PRIVATE_LINK_LIBS utf8proc::utf8proc)
  list(APPEND GANDIVA_STATIC_LINK_LIBS utf8proc::utf8proc)
endif()

if(ARROW_GANDIVA_STATIC_LIBSTDCPP AND (CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX
                                      ))
  list(APPEND GANDIVA_STATIC_LINK_LIBS -static-libstdc++ -static-libgcc)
endif()

# if (MSVC)
#   # Symbols that need to be made public in gandiva.dll for LLVM IR
#   # compilation
#   set(MSVC_SYMBOL_EXPORTS _Init_thread_header)
#   foreach(SYMBOL ${MSVC_SYMBOL_EXPORTS})
#     set(GANDIVA_SHARED_LINK_FLAGS "${GANDIVA_SHARED_LINK_FLAGS} /EXPORT:${SYMBOL}")
#   endforeach()
# endif()
if(CXX_LINKER_SUPPORTS_VERSION_SCRIPT)
  string(APPEND GANDIVA_SHARED_LINK_FLAGS
         " -Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/symbols.map")
endif()

add_arrow_lib(gandiva
              CMAKE_PACKAGE_NAME
              Gandiva
              PKG_CONFIG_NAME
              gandiva
              SOURCES
              ${SRC_FILES}
              PRECOMPILED_HEADERS
              "$<$<COMPILE_LANGUAGE:CXX>:gandiva/pch.h>"
              OUTPUTS
              GANDIVA_LIBRARIES
              DEPENDENCIES
              precompiled
              SHARED_LINK_FLAGS
              ${GANDIVA_SHARED_LINK_FLAGS}
              SHARED_LINK_LIBS
              ${GANDIVA_SHARED_LINK_LIBS}
              SHARED_PRIVATE_LINK_LIBS
              ${GANDIVA_SHARED_PRIVATE_LINK_LIBS}
              SHARED_INSTALL_INTERFACE_LIBS
              Arrow::arrow_shared
              LLVM::LLVM_HEADERS
              STATIC_LINK_LIBS
              ${GANDIVA_STATIC_LINK_LIBS}
              STATIC_INSTALL_INTERFACE_LIBS
              Arrow::arrow_static
              LLVM::LLVM_HEADERS
              LLVM::LLVM_LIBS)

foreach(LIB_TARGET ${GANDIVA_LIBRARIES})
  target_compile_definitions(${LIB_TARGET} PRIVATE GANDIVA_EXPORTING)
endforeach()

if(ARROW_BUILD_STATIC AND WIN32)
  target_compile_definitions(gandiva_static PUBLIC GANDIVA_STATIC)
endif()

add_dependencies(gandiva ${GANDIVA_LIBRARIES})

arrow_install_all_headers("gandiva")

set(GANDIVA_STATIC_TEST_LINK_LIBS gandiva_static arrow_testing_static)
set(GANDIVA_SHARED_TEST_LINK_LIBS gandiva_shared arrow_testing_shared LLVM::LLVM_LIBS)
if(ARROW_WITH_UTF8PROC)
  list(APPEND GANDIVA_SHARED_TEST_LINK_LIBS utf8proc::utf8proc)
  list(APPEND GANDIVA_STATIC_TEST_LINK_LIBS utf8proc::utf8proc)
endif()
if(WIN32)
  list(APPEND GANDIVA_STATIC_TEST_LINK_LIBS ${GANDIVA_OPENSSL_LIBS})
  list(APPEND GANDIVA_SHARED_TEST_LINK_LIBS ${GANDIVA_OPENSSL_LIBS})
endif()
list(APPEND GANDIVA_STATIC_TEST_LINK_LIBS ${ARROW_GTEST_GMOCK} ${ARROW_GTEST_GTEST_MAIN})
list(APPEND GANDIVA_SHARED_TEST_LINK_LIBS ${ARROW_GTEST_GMOCK} ${ARROW_GTEST_GTEST_MAIN})

function(ADD_GANDIVA_TEST REL_TEST_NAME)
  set(options USE_STATIC_LINKING)
  set(one_value_args)
  set(multi_value_args)
  cmake_parse_arguments(ARG
                        "${options}"
                        "${one_value_args}"
                        "${multi_value_args}"
                        ${ARGN})

  if(NO_TESTS)
    return()
  endif()

  set(TEST_ARGUMENTS
      ENABLED
      PREFIX
      "gandiva"
      LABELS
      "gandiva-tests"
      ${ARG_UNPARSED_ARGUMENTS})

  # and uses less disk space, but in some cases we need to force static
  # linking (see rationale below).
  if(ARG_USE_STATIC_LINKING OR ARROW_TEST_LINKAGE STREQUAL "static")
    add_test_case(${REL_TEST_NAME}
                  ${TEST_ARGUMENTS}
                  STATIC_LINK_LIBS
                  ${GANDIVA_STATIC_TEST_LINK_LIBS}
                  ${ARG_UNPARSED_ARGUMENTS})
  else()
    add_test_case(${REL_TEST_NAME}
                  ${TEST_ARGUMENTS}
                  STATIC_LINK_LIBS
                  ${GANDIVA_SHARED_TEST_LINK_LIBS}
                  ${ARG_UNPARSED_ARGUMENTS})
  endif()

  set(TEST_NAME gandiva-${REL_TEST_NAME})
  string(REPLACE "_" "-" TEST_NAME ${TEST_NAME})

  if(ARG_USE_STATIC_LINKING OR ARROW_TEST_LINKAGE STREQUAL "static")
    # LLVM 17 or later requires that an executable exports
    # "llvm_orc_registerEHFrameSectionWrapper()" and
    # "llvm_orc_unregisterEHFrameSectionWrapper()". We need to do
    # nothing when we use libLLVM.so. But we need to export symbols
    # explicitly when we use libLLVM*.a.
    set_target_properties(${TEST_NAME} PROPERTIES ENABLE_EXPORTS TRUE)
  endif()
endfunction()

add_gandiva_test(internals-test
                 SOURCES
                 bitmap_accumulator_test.cc
                 cache_test.cc
                 engine_llvm_test.cc
                 function_registry_test.cc
                 function_signature_test.cc
                 llvm_types_test.cc
                 llvm_generator_test.cc
                 annotator_test.cc
                 tree_expr_test.cc
                 encrypt_utils_test.cc
                 expr_decomposer_test.cc
                 exported_funcs_registry_test.cc
                 expression_registry_test.cc
                 selection_vector_test.cc
                 lru_cache_test.cc
                 to_date_holder_test.cc
                 simple_arena_test.cc
                 regex_functions_holder_test.cc
                 decimal_type_util_test.cc
                 random_generator_holder_test.cc
                 hash_utils_test.cc
                 gdv_function_stubs_test.cc
                 interval_holder_test.cc
                 tests/test_util.cc
                 EXTRA_LINK_LIBS
                 re2::re2
                 ${GANDIVA_OPENSSL_LIBS})

add_subdirectory(precompiled)
add_subdirectory(tests)
