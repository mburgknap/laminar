###
### Copyright 2015-2020 Oliver Giles
###
### This file is part of Laminar
###
### Laminar is free software: you can redistribute it and/or modify
### it under the terms of the GNU General Public License as published by
### the Free Software Foundation, either version 3 of the License, or
### (at your option) any later version.
###
### Laminar is distributed in the hope that it will be useful,
### but WITHOUT ANY WARRANTY; without even the implied warranty of
### MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
### GNU General Public License for more details.
###
### You should have received a copy of the GNU General Public License
### along with Laminar.  If not, see <http://www.gnu.org/licenses/>
###
project(laminar)
cmake_minimum_required(VERSION 2.8)
cmake_policy(SET CMP0058 NEW)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

add_definitions("-std=c++17 -Wall -Wextra -Wno-unused-parameter -Wno-sign-compare")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Werror -DDEBUG")

# Allow passing in the version string, for e.g. patched/packaged versions
if(NOT LAMINAR_VERSION AND EXISTS ${CMAKE_SOURCE_DIR}/.git)
    execute_process(COMMAND git describe --tags --abbrev=8 --dirty
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE LAMINAR_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()
if(NOT LAMINAR_VERSION)
    set(LAMINAR_VERSION xx-unversioned)
endif()
set_source_files_properties(src/version.cpp PROPERTIES COMPILE_DEFINITIONS
	LAMINAR_VERSION=${LAMINAR_VERSION})

# This macro takes a list of files, gzips them and converts the output into
# object files so they can be linked directly into the application.
# ld generates symbols based on the string argument given to its executable,
# so it is significant from which directory it is called. BASEDIR will be
# removed from the beginning of paths to the remaining arguments
macro(generate_compressed_bins BASEDIR)
    foreach(FILE ${ARGN})
        set(COMPRESSED_FILE "${FILE}.z")
        set(OUTPUT_FILE "${FILE}.o")
        get_filename_component(DIR ${FILE} PATH)
        if(DIR)
            file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${DIR})
        endif()
        add_custom_command(OUTPUT ${COMPRESSED_FILE}
            COMMAND gzip < ${BASEDIR}/${FILE} > ${COMPRESSED_FILE}
            DEPENDS ${BASEDIR}/${FILE}
        )
        add_custom_command(OUTPUT ${OUTPUT_FILE}
            COMMAND ${CMAKE_LINKER} -r -b binary -o ${OUTPUT_FILE} ${COMPRESSED_FILE}
            COMMAND ${CMAKE_OBJCOPY}
              --rename-section .data=.rodata.alloc,load,readonly,data,contents
              --add-section .note.GNU-stack=/dev/null
              --set-section-flags .note.GNU-stack=contents,readonly ${OUTPUT_FILE}
            DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${COMPRESSED_FILE}
        )
        list(APPEND COMPRESSED_BINS ${OUTPUT_FILE})
    endforeach()
endmacro()

# Generates Cap'n Proto interface from definition file
add_custom_command(OUTPUT laminar.capnp.c++ laminar.capnp.h
    COMMAND capnp compile -oc++:${CMAKE_BINARY_DIR}
    --src-prefix=${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/src/laminar.capnp
    DEPENDS src/laminar.capnp)

# Zip and compile statically served resources
generate_compressed_bins(${CMAKE_SOURCE_DIR}/src/resources index.html js/app.js
    style.css manifest.webmanifest favicon.ico favicon-152.png icon.png)

# The code that allows dynamic modifying of index.html requires knowing its original size
add_custom_command(OUTPUT index_html_size.h
    COMMAND sh -c '( echo -n "\\#define INDEX_HTML_UNCOMPRESSED_SIZE " && wc -c < "${CMAKE_SOURCE_DIR}/src/resources/index.html" ) > index_html_size.h'
    DEPENDS src/resources/index.html)

# Download 3rd-party frontend JS libs...
file(DOWNLOAD https://cdnjs.cloudflare.com/ajax/libs/vue/2.3.4/vue.min.js
        js/vue.min.js EXPECTED_MD5 ae2fca1cfa0e31377819b1b0ffef704c)
file(DOWNLOAD https://cdnjs.cloudflare.com/ajax/libs/vue-router/2.7.0/vue-router.min.js
        js/vue-router.min.js EXPECTED_MD5 5d3e35710dbe02de78c39e3e439b8d4e)
file(DOWNLOAD https://raw.githubusercontent.com/drudru/ansi_up/v1.3.0/ansi_up.js
        js/ansi_up.js EXPECTED_MD5 158566dc1ff8f2804de972f7e841e2f6)
file(DOWNLOAD https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.2/Chart.min.js
        js/Chart.min.js EXPECTED_MD5 f6c8efa65711e0cbbc99ba72997ecd0e)
# ...and compile them
generate_compressed_bins(${CMAKE_BINARY_DIR} js/vue-router.min.js js/vue.min.js
    js/ansi_up.js js/Chart.min.js)
# (see resources.cpp where these are fetched)

set(LAMINARD_CORE_SOURCES
    src/conf.cpp
    src/database.cpp
    src/laminar.cpp
    src/leader.cpp
    src/http.cpp
    src/resources.cpp
    src/rpc.cpp
    src/run.cpp
    src/server.cpp
    src/version.cpp
    laminar.capnp.c++
    index_html_size.h
)

## Server
add_executable(laminard ${LAMINARD_CORE_SOURCES} src/main.cpp ${COMPRESSED_BINS})
target_link_libraries(laminard capnp-rpc capnp kj-http kj-async kj pthread sqlite3 z)

## Client
add_executable(laminarc src/client.cpp src/version.cpp laminar.capnp.c++)
target_link_libraries(laminarc capnp-rpc capnp kj-async kj pthread)

## Tests
set(BUILD_TESTS FALSE CACHE BOOL "Build tests")
if(BUILD_TESTS)
    find_package(GTest REQUIRED)
    include_directories(${GTEST_INCLUDE_DIRS} src)
    add_executable(laminar-tests ${LAMINARD_CORE_SOURCES} ${COMPRESSED_BINS} test/main.cpp test/laminar-functional.cpp test/unit-conf.cpp test/unit-database.cpp)
    target_link_libraries(laminar-tests ${GTEST_LIBRARY} capnp-rpc capnp kj-http kj-async kj pthread sqlite3 z)
endif()

set(SYSTEMD_UNITDIR /lib/systemd/system CACHE PATH "Path to systemd unit files")
set(BASH_COMPLETIONS_DIR /usr/share/bash-completion/completions CACHE PATH "Path to bash completions directory")
set(ZSH_COMPLETIONS_DIR /usr/share/zsh/site-functions CACHE PATH "Path to zsh completions directory")
install(TARGETS laminard RUNTIME DESTINATION sbin)
install(TARGETS laminarc RUNTIME DESTINATION bin)
install(FILES etc/laminar.conf DESTINATION /etc)
install(FILES etc/laminarc-completion.bash DESTINATION ${BASH_COMPLETIONS_DIR} RENAME laminarc)
install(FILES etc/laminarc-completion.zsh DESTINATION ${ZSH_COMPLETIONS_DIR} RENAME _laminarc)

configure_file(etc/laminar.service.in laminar.service @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/laminar.service DESTINATION ${SYSTEMD_UNITDIR})
