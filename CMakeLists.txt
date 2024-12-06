cmake_minimum_required(VERSION 3.23)

set(CMAKE_TOOLCHAIN_FILE "${CMAKE_SOURCE_DIR}/vcpkg/scripts/buildsystems/vcpkg.cmake")

project(test-buf CXX)

find_package(Protobuf REQUIRED)
find_package(unofficial-bufbuild-protovalidate REQUIRED)

find_program(BUF_EXECUTABLE NAMES buf REQUIRED)
message(STATUS "Found Buf CLI: ${BUF_EXECUTABLE}")

file(CONFIGURE
    OUTPUT "buf.gen.yaml"
    CONTENT [[
version: v2
plugins:
  - protoc_builtin: cpp
    out: gen
    protoc_path: @Protobuf_PROTOC_EXECUTABLE@
]]
    @ONLY
)

unset(PB_CC_FILES)
unset(PB_H_FILES)

file(GLOB_RECURSE PROTO_FILES "${CMAKE_SOURCE_DIR}/protocol/proto/server/*.proto")

foreach(file ${PROTO_FILES})
    file(RELATIVE_PATH relative_file "${CMAKE_SOURCE_DIR}/protocol/proto" "${file}")
    get_filename_component(directory "${relative_file}" DIRECTORY)
    get_filename_component(name_wle "${relative_file}" NAME_WLE)
    list(APPEND PB_CC_FILES "${CMAKE_CURRENT_BINARY_DIR}/gen/${directory}/${name_wle}.pb.cc")
    list(APPEND PB_H_FILES "${CMAKE_CURRENT_BINARY_DIR}/gen/${directory}/${name_wle}.pb.h")
endforeach()

add_custom_command(
    OUTPUT ${PB_CC_FILES} ${PB_H_FILES}
    COMMAND ${BUF_EXECUTABLE} generate "${CMAKE_SOURCE_DIR}/protocol"
    DEPENDS ${PROTO_FILES} ${BUF_EXECUTABLE}
    COMMENT "Generating C++ code from Protobuf files"
    VERBATIM
)

add_executable(${PROJECT_NAME} main.cpp ${PB_CC_FILES})

target_include_directories(${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/gen")

target_link_libraries(${PROJECT_NAME} PRIVATE protobuf::libprotobuf)
target_link_libraries(${PROJECT_NAME} PRIVATE unofficial::bufbuild-protovalidate::bufbuild-protovalidate)
