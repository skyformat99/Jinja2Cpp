cmake_minimum_required(VERSION 3.0)
project(Jinja2Cpp VERSION 0.5.0)

list (APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

if(CMAKE_COMPILER_IS_GNUCXX AND COVERAGE_ENABLED)
    message (STATUS "This is DEBUG build with enabled Code Coverage")
    set (CMAKE_BUILD_TYPE Debug)
    include(code_coverage)
    setup_target_for_coverage(${PROJECT_NAME}_coverage jinja2cpp_tests coverage)
endif()

set(GTEST_ROOT $ENV{GTEST_DIR} CACHE PATH "Path to GTest/GMock library root")
if (NOT UNIX)
    set(BOOST_ROOT $ENV{BOOST_DIR} CACHE PATH "Path to boost library root")
    set(LIBRARY_TYPE STATIC CACHE PATH "Library link type")
else ()
    set(BOOST_ROOT "/usr" CACHE PATH "Path to boost library root")
    set(LIBRARY_TYPE SHARED CACHE PATH "Library link type")
endif ()

if (NOT DEFINED WITH_TESTS)
    set (WITH_TESTS TRUE)
endif ()

set (EXTRA_TEST_LIBS "")
set (CMAKE_CXX_STANDARD 14)
set (CMAKE_CXX_STANDARD_REQUIRED ON)
if (${CMAKE_CXX_COMPILER_ID} MATCHES "Clang" OR ${CMAKE_CXX_COMPILER_ID} MATCHES "GNU")
    if (NOT UNIX)
        set (EXTRA_TEST_LIBS "stdc++")
        set (CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} "-Wa,-mbig-obj")
    else ()
        include(CMakeFindDependencyMacro)
        find_dependency(Threads)
        set (EXTRA_TEST_LIBS Threads::Threads)
    endif ()
else ()
    # MSVC
    if (NOT DEFINED Boost_USE_STATIC_LIBS)
        set (Boost_USE_STATIC_LIBS ON)
    endif ()
    if (NOT DEFINED MSVC_RUNTIME_TYPE)
        set (MSVC_RUNTIME_TYPE "/MD")
    endif ()
    if (CMAKE_BUILD_TYPE MATCHES "Debug")
        message("#######>>>>>>>>>>!!!!!!!!!!!!!! AAAAAAAAAAAAAAAAAAAAAAAAAAAA")
        set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${MSVC_RUNTIME_TYPE}d")
        set (Boost_USE_DEBUG_RUNTIME ON)
    else ()
        set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${MSVC_RUNTIME_TYPE}")
        set (CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} ${MSVC_RUNTIME_TYPE}")
        set (CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "/PROFILE")
        set (Boost_USE_DEBUG_RUNTIME OFF)
    endif ()
endif()

if("${GTEST_ROOT}" STREQUAL "" AND WITH_TESTS)
    set (THIRDPARTY_TARGETS ${THIRDPARTY_TARGETS} gtest)
endif()

if(NOT "${BOOST_ROOT}" STREQUAL "")
    list (APPEND CMAKE_PREFIX_PATH ${BOOST_ROOT})
    set (Boost_DIR ${BOOST_ROOT})
endif()

make_directory (${CMAKE_CURRENT_BINARY_DIR}/thirdparty)
execute_process (
    COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER} -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER} -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/thirdparty
    RESULT_VARIABLE THIRDPARTY_BUILD_RESULT
    )

if (THIRDPARTY_BUILD_RESULT EQUAL 0 AND THIRDPARTY_TARGETS)
    execute_process (
        COMMAND ${CMAKE_COMMAND} --build . --target ${THIRDPARTY_TARGETS}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/thirdparty
        RESULT_VARIABLE THIRDPARTY_BUILD_RESULT
        )
endif ()

if (NOT THIRDPARTY_BUILD_RESULT EQUAL 0)
    message (FATAL_ERROR "Can't build thirdparty libraries")
endif ()

if("${GTEST_ROOT}" STREQUAL "" AND WITH_TESTS)
    set (GTEST_ROOT ${CMAKE_CURRENT_BINARY_DIR}/thirdparty/gtest/install)
    set (GTEST_SELF_BUILD ON)
endif ()

if(NOT "${GTEST_ROOT}" STREQUAL "" AND WITH_TESTS)
    list (APPEND CMAKE_PREFIX_PATH ${GTEST_ROOT})
    set (Gtest_DIR ${GTEST_ROOT})
    message(STATUS "GTest library search path: ${Gtest_DIR}")
    if (NOT GTEST_SELF_BUILD)
        find_package(GTest)
    else ()
        set (Gtest_DIR ${GTEST_ROOT})
        find_package(GTest)
    endif ()
    if (NOT GTEST_INCLUDE_DIRS)
        if (MSVC AND NOT GTEST_INCLUDE_DIRS)
            set (GTEST_MSVC_SEARCH "MT")
            find_package(GTest)
        else ()
            set (GTEST_BOTH_LIBRARIES "optimized;${GTEST_ROOT}/lib/libgtest.a;debug;${GTEST_ROOT}/lib/libgtestd.a;optimized;${GTEST_ROOT}/lib/libgtest_main.a;debug;${GTEST_ROOT}/lib/libgtest_maind.a")
            set (GTEST_INCLUDE_DIRS ${GTEST_ROOT}/include)
        endif ()
    endif ()
endif()

if (WITH_TESTS)
    find_package(Boost COMPONENTS system filesystem REQUIRED)
else ()
    find_package(Boost)
endif ()

add_subdirectory (thirdparty/nonstd)

include(collect_sources)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    )

set (LIB_TARGET_NAME jinja2cpp)

CollectSources(Sources Headers ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/src)
CollectSources(PublicSources PublicHeaders ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/include)

if(CMAKE_COMPILER_IS_GNUCXX AND COVERAGE_ENABLED)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
endif()

add_library(${LIB_TARGET_NAME} STATIC
    ${Sources}
    ${Headers}
    ${PublicHeaders}
    )

target_link_libraries(${LIB_TARGET_NAME} PUBLIC ThirdParty::nonstd Boost::boost) # Boost::system Boost::filesystem)

target_include_directories(${LIB_TARGET_NAME}
    INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)

if (WITH_TESTS)
    enable_testing()

    include_directories(
        ${GTEST_INCLUDE_DIRS}
        )

    CollectSources(TestSources TestHeaders ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/test)
    add_executable(jinja2cpp_tests ${TestSources} ${TestHeaders})
    target_link_libraries(jinja2cpp_tests ${GTEST_BOTH_LIBRARIES} ${LIB_TARGET_NAME} ${EXTRA_TEST_LIBS} ${Boost_LIBRARIES})

    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/test_data/simple_template1.j2tpl
        COMMAND ${CMAKE_COMMAND} ARGS -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/test/test_data ${CMAKE_CURRENT_BINARY_DIR}/test_data
        MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/test/test_data/simple_template1.j2tpl
        COMMENT "Copy test data to the destination dir"
        )

    add_custom_target(CopyTestData ALL
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/test_data/simple_template1.j2tpl
        )
endif ()

install(TARGETS ${LIB_TARGET_NAME}
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib/static)

install (DIRECTORY include/ DESTINATION include)
install (DIRECTORY thirdparty/nonstd/expected-light/include/ DESTINATION include)
install (FILES cmake/public/FindJinja2Cpp.cmake DESTINATION cmake)

add_test(NAME jinja2cpp_tests COMMAND jinja2cpp_tests)
