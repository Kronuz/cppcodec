cmake_minimum_required(VERSION 2.8)
project(cppcodec)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

set(PUBLIC_HEADERS
    cppcodec/base32_crockstr.hpp
    cppcodec/base32_default_crockstr.hpp
    cppcodec/base32_default_rfc4648.hpp
    cppcodec/base32_rfc4648.hpp
    cppcodec/parse_error.hpp
    cppcodec/data/access.hpp
    cppcodec/data/raw_result_buffer.hpp
    cppcodec/detail/base32.hpp
    cppcodec/detail/codec.hpp
    cppcodec/detail/stream_codec.hpp)

add_library(cppcodec OBJECT ${PUBLIC_HEADERS})
add_subdirectory(tool)

foreach(h ${PUBLIC_HEADERS})
    get_filename_component(HEADER_INCLUDE_DIRECTORY include/${h} PATH) # use DIRECTORY instead of PATH once requiring CMake 3.0
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/${h} DESTINATION ${HEADER_INCLUDE_DIRECTORY} COMPONENT "headers")
endforeach()