cmake_minimum_required(VERSION 3.8)
cmake_policy(SET CMP0048 NEW)
project(crow VERSION 0.0.6)

option(CROW_USE_CONAN "Whether to use conan package management for dependencies" ON)
if(CROW_USE_CONAN)
    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup(TARGETS)
endif()

##############################
# check for required headers #
##############################

include(CheckIncludeFiles)
check_include_files(cxxabi.h NLOHMANN_CROW_HAVE_CXXABI_H)
check_include_files(execinfo.h NLOHMANN_CROW_HAVE_EXECINFO_H)
check_include_files(dlfcn.h NLOHMANN_CROW_HAVE_DLFCN_H)

##################################
# collect additional information #
##################################

# CMake
cmake_host_system_information(RESULT NLOHMANN_CROW_HOSTNAME QUERY HOSTNAME)
cmake_host_system_information(RESULT NLOHMANN_CROW_TOTAL_PHYSICAL_MEMORY QUERY TOTAL_PHYSICAL_MEMORY)

if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.10.0")
    cmake_host_system_information(RESULT NLOHMANN_CROW_OS_RELEASE QUERY OS_RELEASE)
    string(STRIP "${NLOHMANN_CROW_OS_RELEASE}" NLOHMANN_CROW_OS_RELEASE)
    cmake_host_system_information(RESULT NLOHMANN_CROW_OS_VERSION QUERY OS_VERSION)
    string(STRIP "${NLOHMANN_CROW_OS_VERSION}" NLOHMANN_CROW_OS_VERSION)
endif()

# uname
find_program(UNAME_TOOL uname)
execute_process(COMMAND ${UNAME_TOOL} -a
        OUTPUT_VARIABLE NLOHMANN_CROW_UNAME OUTPUT_STRIP_TRAILING_WHITESPACE)

# systeminfo
find_program(SYSTEMINFO_TOOL systeminfo)
execute_process(COMMAND ${SYSTEMINFO_TOOL}
        OUTPUT_VARIABLE NLOHMANN_CROW_SYSTEMINFO OUTPUT_STRIP_TRAILING_WHITESPACE)
if(NLOHMANN_CROW_SYSTEMINFO)
    string(REGEX MATCHALL "OS [^\n]*" NLOHMANN_CROW_SYSTEMINFO ${NLOHMANN_CROW_SYSTEMINFO})
    string(REPLACE ";" "\\n" NLOHMANN_CROW_SYSTEMINFO "${NLOHMANN_CROW_SYSTEMINFO}")
endif()

# compiler version output
if(MSVC)
    execute_process(COMMAND msbuild /version
        OUTPUT_VARIABLE NLOHMANN_CROW_CXX OUTPUT_STRIP_TRAILING_WHITESPACE)
else()
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} --version
            OUTPUT_VARIABLE NLOHMANN_CROW_CXX OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()
string(REGEX REPLACE "\n" "\\\\n" NLOHMANN_CROW_CXX ${NLOHMANN_CROW_CXX})

# sysctl
find_program(SYSCTL_TOOL sysctl)
execute_process(COMMAND ${SYSCTL_TOOL} hw.model
        OUTPUT_VARIABLE NLOHMANN_CROW_SYSCTL_HW_MODEL ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
if(NLOHMANN_CROW_SYSCTL_HW_MODEL)
    string(REGEX REPLACE ".*: (.*)" "\\1" NLOHMANN_CROW_SYSCTL_HW_MODEL ${NLOHMANN_CROW_SYSCTL_HW_MODEL})
endif()

# check for MinGW which has a broken <random> header
if(MINGW)
    set(NLOHMANN_CROW_MINGW ${MINGW})
endif()

########################
# create config header #
########################

configure_file(src/crow_config.hpp.in
        ${CMAKE_CURRENT_BINARY_DIR}/src/crow_config.hpp)

################
# dependencies #
################
option(CROW_EXTERNAL_ZLIB_PROJECT "the path to zlib" OFF)
option(CROW_EXTERNAL_CURL_PROJECT "the path to curl" OFF)
if(NOT (CROW_USE_CONAN))
    if(CROW_EXTERNAL_ZLIB_PROJECT)
        add_subdirectory(${CROW_EXTERNAL_ZLIB_PROJECT})
        set(ZLIB_INCLUDE_DIRS ${CROW_EXTERNAL_ZLIB_PROJECT})
        set(ZLIB_LIBRARIES zlibstatic)
    else()
        find_package(ZLIB REQUIRED ${CROW_EXTERNAL_ZLIB_PROJECT})
    endif()

    if(CROW_EXTERNAL_CURL_PROJECT)
        set(BUILD_TESTING OFF CACHE INTERNAL "")
        set(BUILD_SHARED_LIBS OFF CACHE INTERNAL "")
        set(CURL_STATICLIB ON CACHE INTERNAL "")
        set(BUILD_CURL_EXE OFF CACHE INTERNAL "")
        set(CURL_CA_PATH_SET OFF CACHE INTERNAL "")
        set(CMAKE_USE_OPENSSL ON CACHE INTERNAL "")
        set(HTTP_ONLY ON CACHE INTERNAL "")
        add_definitions(-DCURL_STATICLIB)
        add_subdirectory(${CROW_EXTERNAL_CURL_PROJECT})
        set(CURL_INCLUDE_DIR ${CROW_EXTERNAL_CURL_PROJECT}/include)
        set(CURL_LIBRARIES libcurl)
    else()
        find_package(CURL)
    endif()
endif()

find_package(Threads)

###########
# library #
###########

add_library(crow src/crow.cpp src/crow_utilities.cpp src/crow_utilities.hpp)
set_target_properties(crow PROPERTIES CXX_STANDARD 11)
target_include_directories(crow PUBLIC include PRIVATE ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR} ${CURL_INCLUDE_DIR} ${ZLIB_INCLUDE_DIRS})
target_link_libraries(crow ${CONAN_TARGETS} ${CURL_LIBRARIES} ${CMAKE_DL_LIBS} ${CMAKE_THREAD_LIBS_INIT} ${OPENSSL_LIBRARIES} ${ZLIB_LIBRARIES})

#################
# documentation #
#################

find_package(Doxygen)
if(DOXYGEN_FOUND)
    set(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/doc/Doxyfile.in)
    set(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/doc/Doxyfile)

    configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)

    add_custom_target(crow_documentation
            COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc
            COMMENT "Generating API documentation with Doxygen"
            VERBATIM)
endif()

###########
# testing #
###########

include(CTest)
option(CROW_BUILD_TESTING "build tests" ON)
option(CROW_BUILD_LOG4CPLUS "build log4cplus example" OFF)
if(CROW_BUILD_TESTING AND (BUILD_TESTING OR CROW_EXTERNAL_CURL_PROJECT))
    enable_testing()

    add_executable(unittests tests/unittests.cpp)
    set_target_properties(unittests PROPERTIES CXX_STANDARD 11)
    target_include_directories(unittests PUBLIC ${CMAKE_CURRENT_SOURCE_DIR} tests)
    target_link_libraries(unittests crow)
    add_test(NAME unittests COMMAND unittests)

    # std::current_exception is broken with MSVC (https://developercommunity.visualstudio.com/content/problem/135332/stdcurrent-exception-returns-null-in-a-stdterminat.html)
    if (NOT MSVC)
        add_executable(uncaught_exception tests/uncaught_exception.cpp)
        set_target_properties(uncaught_exception PROPERTIES CXX_STANDARD 11)
        target_link_libraries(uncaught_exception crow)
        add_test(NAME uncaught_exception COMMAND uncaught_exception)
    endif()

    add_executable(livetest tests/livetest.cpp)
    set_target_properties(livetest PROPERTIES CXX_STANDARD 11)
    target_include_directories(livetest PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
    target_link_libraries(livetest crow)
    add_test(NAME livetest COMMAND livetest)

    if(CROW_BUILD_LOG4CPLUS)
        include(ExternalProject)
        ExternalProject_Add(log4cplus_project
                URL https://github.com/log4cplus/log4cplus/releases/download/REL_2_0_1/log4cplus-2.0.1.zip
                INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/log4cplus
                CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/log4cplus -DBUILD_SHARED_LIBS=OFF -DLOG4CPLUS_ENABLE_DECORATED_LIBRARY_NAME=OFF -DUNICODE=OFF -DWITH_UNIT_TESTS=OFF
                BUILD_BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/log4cplus/lib/${CMAKE_STATIC_LIBRARY_PREFIX}log4cplus${CMAKE_STATIC_LIBRARY_SUFFIX}
                )
        ExternalProject_Get_Property(log4cplus_project install_dir)
        add_library(log4cplus STATIC IMPORTED)
        set_target_properties(log4cplus PROPERTIES IMPORTED_LOCATION ${install_dir}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}log4cplus${CMAKE_STATIC_LIBRARY_SUFFIX})
        add_dependencies(log4cplus log4cplus_project)

        add_executable(log4cplus_example examples/log4cplus/example.cpp include/crow/integrations/loggers.hpp include/crow/integrations/log4cplus.hpp)
        set_target_properties(log4cplus_example PROPERTIES CXX_STANDARD 11)
        target_include_directories(log4cplus_example PUBLIC include ${install_dir}/include)
        target_link_libraries(log4cplus_example log4cplus crow)
        add_test(NAME log4cplus_example COMMAND log4cplus_example)
    endif()
endif()

############
# coverage #
############

option(CROW_CHECK_COVERAGE "instrument compiler to collect coverage information" OFF)
if(CROW_CHECK_COVERAGE)
    message(STATUS "Building test suite with coverage information; now build, run the unit tests and call `make crow_coverage_report`")

    # add compiler flags to collect coverage information
    set(CMAKE_CXX_FLAGS "--coverage -g -O0 -fprofile-arcs -ftest-coverage -Wno-unused-command-line-argument")

    # collect all source files that we are interested in
    file(GLOB_RECURSE SOURCE_FILES ${CMAKE_SOURCE_DIR}/include/crow/* ${CMAKE_SOURCE_DIR}/src/*)

    # define target to run lcov to collect coverage information, filter unwanted files, and create HTML report
    add_custom_target(crow_coverage_report
            COMMAND lcov --directory . --capture --output-file coverage.info --rc lcov_branch_coverage=1
            COMMAND lcov -e coverage.info ${SOURCE_FILES} --output-file coverage.info.filtered --rc lcov_branch_coverage=1
            COMMAND genhtml --title "Crow ${PROJECT_VERSION}" --legend --demangle-cpp --output-directory coverage_report --show-details --branch-coverage coverage.info.filtered
            COMMENT "Generating HTML report coverage_report/index.html")
endif()

#############
# packaging #
#############

install(TARGETS crow RUNTIME DESTINATION bin ARCHIVE DESTINATION lib LIBRARY DESTINATION lib)
install(FILES include/crow/crow.hpp DESTINATION include/crow)
install(FILES include/thirdparty/json/json.hpp DESTINATION include/thirdparty/json)

set(CPACK_GENERATOR "ZIP")
set(CPACK_STRIP_FILES TRUE)
include(InstallRequiredSystemLibraries)
include(CPack)
