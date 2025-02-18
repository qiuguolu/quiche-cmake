cmake_minimum_required(VERSION 3.16)

project(study-webrtc LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)

set(QUICHE_LIB quiche)

add_subdirectory(third_party/googleurl)
add_subdirectory(third_party/boringssl)

set(ABSL_PROPAGATE_CXX_STD ON)
add_subdirectory(third_party/abseil-cpp)

set(protobuf_BUILD_TESTS OFF)
add_subdirectory(third_party/protobuf)

set(Protobuf_PROTOC_EXECUTABLE ${CMAKE_BINARY_DIR}/third_party/protobuf/protoc)
set(Protobuf_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/third_party/protobuf/src)
set(Protobuf_LIBRARIES protobuf::libprotobuf)

find_package(Protobuf REQUIRED)
set(proto_files 
	${PROJECT_SOURCE_DIR}/quiche/quic/core/proto/cached_network_parameters.proto
	${PROJECT_SOURCE_DIR}/quiche/quic/core/proto/crypto_server_config.proto
	${PROJECT_SOURCE_DIR}/quiche/quic/core/proto/source_address_token.proto
)

protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${proto_files} IMPORT_DIRS ${PROJECT_SOURCE_DIR})

add_custom_target(copy_protobuf_files
    DEPENDS ${PROTO_SRCS} ${PROTO_HDRS} 
    COMMENT "Copying Protobuf generated files ..."
)

foreach(SRC ${PROTO_HDRS})
    get_filename_component(FILENAME ${SRC} NAME)
    add_custom_command(
        TARGET copy_protobuf_files
        COMMAND ${CMAKE_COMMAND} -E copy ${SRC} ${PROJECT_SOURCE_DIR}/quiche/quic/core/proto/${FILENAME}
        DEPENDS ${SRC}
        COMMENT "Copying ${FILENAME}"
    )
endforeach()

include(ExternalProject)
set(ZLIB_SOURCE_DIR ${CMAKE_SOURCE_DIR}/third_party/zlib)
set(ZLIB_BUILD_DIR ${CMAKE_BINARY_DIR}/zlib_build)
ExternalProject_Add(
    zlib
    SOURCE_DIR ${ZLIB_SOURCE_DIR}
    CONFIGURE_COMMAND ${ZLIB_SOURCE_DIR}/configure --prefix=${ZLIB_BUILD_DIR}
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    BUILD_IN_SOURCE 1
)

include(quiche.cmake)
add_library(${QUICHE_LIB} STATIC ${quiche_sources} ${PROTO_SRCS})
add_dependencies(${QUICHE_LIB} zlib copy_protobuf_files)

target_include_directories(${QUICHE_LIB} PRIVATE 
	${PROJECT_SOURCE_DIR}
	${PROJECT_SOURCE_DIR}/quiche/common/platform/default/
	${PROJECT_SOURCE_DIR}/third_party/abseil-cpp
	${PROJECT_SOURCE_DIR}/third_party/boringssl/src/include
	${PROJECT_SOURCE_DIR}/third_party/googleurl
	${PROJECT_SOURCE_DIR}/third_party/googleurl
	${ZLIB_BUILD_DIR}/include
	${Protobuf_INCLUDE_DIR}
)

target_compile_options(${QUICHE_LIB} PRIVATE "-w" "-g")

target_link_libraries(${QUICHE_LIB} PRIVATE
	${Protobuf_LIBRARIES}
	${ZLIB_BUILD_DIR}/lib/libz.a
	absl_flags_parse
	absl_log_initialize
	absl_log_internal_check_op
	absl_log_internal_conditions
	absl_log_internal_message
	absl_raw_logging_internal
	absl_status
	absl_statusor
	gurl
	ssl
)


set(CLIENT client)
set(SERVER server)

add_executable(${CLIENT}
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_client_bin.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_epoll_client_factory.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_simple_client_session.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_simple_client_stream.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_spdy_client_base.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_toy_client.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_url.cc
)

add_executable(${SERVER}
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_server_bin.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_backend_response.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_backend_response.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_memory_cache_backend.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_server.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_server_factory.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_server_factory.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_simple_crypto_server_stream_helper.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_simple_dispatcher.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_simple_server_session.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_simple_server_stream.cc
	${PROJECT_SOURCE_DIR}/quiche/quic/tools/quic_toy_server.cc
)

set(TARGETS ${CLIENT} ${SERVER})
foreach(TARGET ${TARGETS})
	target_include_directories(${TARGET} PRIVATE 
		${PROJECT_SOURCE_DIR}
		${PROJECT_SOURCE_DIR}/quiche/common/platform/default/
		${PROJECT_SOURCE_DIR}/third_party/abseil-cpp
		${PROJECT_SOURCE_DIR}/third_party/boringssl/src/include
		${PROJECT_SOURCE_DIR}/third_party/googleurl
		${PROJECT_SOURCE_DIR}/third_party/googleurl
		${ZLIB_BUILD_DIR}/include
		${Protobuf_INCLUDE_DIR}
	)

	target_compile_options(${TARGET} PRIVATE "-w" "-g")
	target_link_libraries(${TARGET}  PRIVATE ${QUICHE_LIB} )
endforeach()