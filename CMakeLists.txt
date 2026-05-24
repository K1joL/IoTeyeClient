# MIT License

# Copyright (c) 2025 Shults Bogdan aka K1joL

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.18)

project(ioteyeClient VERSION 1.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
add_compile_options(-Wall -Wextra -Wpedantic)

option(IOTEYE_USE_BOOST_ASIO "Use Boost.Asio instead of standalone Asio" OFF)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

# Conditionally add logging definitions
if(IOTEYE_CLIENT_ENABLE_LOGGING)
    message(STATUS "Client Logging enabled!")
    add_compile_definitions(IOTEYE_CLIENT_ENABLE_LOGGING)
endif()

include_directories("include/")

set(SRC_LIST
    src/client.cpp
)

# --- Find Dependencies ---
if(IOTEYE_USE_BOOST_ASIO)
    message(STATUS "${PROJECT_NAME}: Using Boost.Asio")
    find_package(Boost REQUIRED)
else()
    message(STATUS "${PROJECT_NAME}: Using standalone Asio")
    find_package(asio REQUIRED)
endif()

include(FetchContent)
find_package(ioteyeserver QUIET)
if(NOT ioteyeserver_FOUND)
    if(FETCH_LIBIOTEYESERVER)
        message(STATUS "ioteyeserver not found, fetching from GitHub")
        set(IOTEYE_USE_BOOST_ASIO ON CACHE BOOL "Use Boost.Asio instead of standalone asio" FORCE)
        FetchContent_Declare(ioteyeserver
            GIT_REPOSITORY https://github.com/K1joL/IoTeyeHttpServer.git
            GIT_TAG develop
        )
        FetchContent_MakeAvailable(ioteyeserver)
        if(TARGET ioteyeserver AND NOT TARGET ioteyeserver::ioteyeserver)
            add_library(ioteyeserver::ioteyeserver ALIAS ioteyeserver)
        endif()
    else()
        message(FATAL_ERROR "ioteyeserver not found! Install it yourself or use -DFETCH_LIBIOTEYESERVER=ON to fetch it automatically.")
    endif()
else()
    message(STATUS "Found local ioteyeserver installation.")
endif()

add_executable(${PROJECT_NAME} client_main.cpp ${SRC_LIST})

# Conditionally link and add compile definitions
if(IOTEYE_USE_BOOST_ASIO)
    # Modern CMake provides Boost::headers for header-only boost libraries
    target_link_libraries(${PROJECT_NAME} PUBLIC Boost::headers)
    target_compile_definitions(${PROJECT_NAME} PUBLIC IOTEYE_USE_BOOST_ASIO)
else()
    target_link_libraries(${PROJECT_NAME} PUBLIC asio::asio)
    target_compile_definitions(${PROJECT_NAME} PUBLIC ASIO_STANDALONE)
endif()

target_link_libraries(${PROJECT_NAME} PRIVATE ioteyeserver)

if(IOTEYE_CLIENT_ENABLE_TESTS)
    message(STATUS "Client tests compilation is ON!")
    add_executable(client_test tests/performance_test.cpp ${SRC_LIST})
    target_link_libraries(client_test PRIVATE ioteyeserver asio::asio)
endif()
