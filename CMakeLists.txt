cmake_minimum_required(VERSION 3.5.1)
set(
    CMAKE_TOOLCHAIN_FILE
    "${CMAKE_SOURCE_DIR}/cmake/toolchain/cxx17.cmake"
    CACHE
    FILEPATH
    "Default toolchain"
)
# Project
project(ipfs-lite-cpp C CXX)

#if (NOT EXISTS "${CMAKE_TOOLCHAIN_FILE}")
#  # https://cgold.readthedocs.io/en/latest/tutorials/toolchain/globals/cxx-standard.html#summary
#  set(CMAKE_CXX_STANDARD 17)
#  set(CMAKE_CXX_STANDARD_REQUIRED ON)
#  set(CMAKE_CXX_EXTENSIONS OFF)
#  set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)
#endif ()

# set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
# set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
# set(CMAKE_C_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
# set(CMAKE_C_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")

# set(CompilerFlags
#         CMAKE_CXX_FLAGS
#         CMAKE_CXX_FLAGS_DEBUG
#         CMAKE_CXX_FLAGS_RELEASE
#         CMAKE_C_FLAGS
#         CMAKE_C_FLAGS_DEBUG
#         CMAKE_C_FLAGS_RELEASE
#         )
# foreach(CompilerFlag ${CompilerFlags})
#   string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
# endforeach()

include(cmake/functions.cmake)
include(cmake/print.cmake)
#print("C flags: ${CMAKE_C_FLAGS}")
#print("CXX flags: ${CMAKE_CXX_FLAGS}")
#print("C Debug flags: ${CMAKE_C_FLAGS_DEBUG}")
#print("CXX Debug flags: ${CMAKE_CXX_FLAGS_DEBUG}")
#print("C Release flags: ${CMAKE_C_FLAGS_RELEASE}")
#print("CXX Release flags: ${CMAKE_CXX_FLAGS_RELEASE}")

# Include
set(GV_other_dir_name z_other)
set(GV_other_dir ${PROJECT_SOURCE_DIR}/${GV_other_dir_name})
list(PREPEND CMAKE_MODULE_PATH "${GV_other_dir}/cmake")
include(common_var)
include(helper)
include(compiler)

macro(setup_platform_environment)
    ## Setup compilation options
    compiler_set_compilation_options()
endmacro()

macro(setup_directories)
    set(GV_external_dir ${GV_other_dir_name}/external/${CMAKE_CXX_COMPILER_ID})
    message("External library path: ${GV_external_dir}")
    get_filename_component(GV_external_dir_a ${GV_external_dir} ABSOLUTE)

    set(GV_dir_gen ${GV_other_dir}/gen)
    set(GV_dir_gen_lib ${GV_dir_gen}/lib)
    set(GV_dir_gen_grpc_cpp ${GV_dir_gen}/grpc_cpp)
    dir_target(GT_dir_gen "${GV_dir_gen}" GV_other_dir)
    dir_target(GT_dir_gen_lib "${GV_dir_gen_lib}" GT_dir_gen)
    dir_target(GT_dir_gen_grpc_cpp "${GV_dir_gen_grpc_cpp}" GT_dir_gen)
endmacro()

macro(setup_conan)
    find_program(V_conan_cmd conan)
    if (NOT V_conan_cmd)
        message(WARNING "conan is not found!" )
    else()
        message("Found conan: ${V_conan_cmd}")
        set(V_conan_gen_files
            "${CMAKE_BINARY_DIR}/conanbuildinfo.cmake")
        set(V_conanfile
            "${PROJECT_SOURCE_DIR}/${GV_other_dir_name}/conanfile.txt")
        if (NOT EXISTS ${V_conanfile})
            message(WARNING "${V_conanfile} is not found!" )
        else()
            if (${V_conanfile} IS_NEWER_THAN ${V_conan_gen_files})
                file(REMOVE ${V_conan_gen_files})

                ## set up compiler settings
                set(V_conan_s_compiler "")
                set(V_conan_s_compiler_ver "")
                set(V_conan_s_compiler_cppstd "")
                set(V_conan_s_compiler_rt "")
                set(V_conan_s_build_type "")
                if (NOT V_conan_compiler STREQUAL "")
                    set(V_conan_s_compiler "-scompiler=${V_conan_compiler}")
                    set(V_conan_s_compiler_ver "-scompiler.version=${CMAKE_CXX_COMPILER_VERSION_MAJOR}")
                    if (V_conan_compiler STREQUAL "msvc")
                        string(SUBSTRING "${CMAKE_CXX_COMPILER_VERSION_MINOR}" 0 1 V_tmp_number)
                        set(V_conan_s_compiler_ver "${V_conan_s_compiler_ver}${V_tmp_number}")
                        set(V_conan_s_compiler_cppstd "-scompiler.cppstd=14")
                        set(V_conan_s_compiler_rt "-scompiler.runtime=dynamic")
                    else()
                        if ("${CMAKE_CXX_COMPILER_VERSION_MINOR}" STREQUAL "")
                        elseif("${CMAKE_CXX_COMPILER_VERSION_MINOR}" STREQUAL "0")
                        else()
                            #                            set(V_conan_s_compiler_ver "${V_conan_s_compiler_ver}.${CMAKE_CXX_COMPILER_VERSION_MINOR}")
                        endif ()
                    endif()
                endif()

                ## set up build_type
                if (CMAKE_BUILD_TYPE STREQUAL "Debug")
                    set(V_conan_s_build_type "-sbuild_type=Debug")
                else()
                    set(V_conan_s_build_type "-sbuild_type=Release")
                endif()

                ## conan install command
                set(V_conan_cmd ${V_conan_cmd} install ${V_conanfile} ${V_conan_s_compiler} ${V_conan_s_compiler_ver} ${V_conan_s_compiler_cppstd} ${V_conan_s_compiler_rt} ${V_conan_s_build_type} --build=missing)
                string(REPLACE ";" " " V_conan_cmd_str "${V_conan_cmd}")
                message("Executing: ${V_conan_cmd_str}")
                execute_process(COMMAND ${V_conan_cmd})
            endif()
        endif()
        include(${V_conan_gen_files})
    endif()
endmacro()

macro(setup_dependency)
    list(PREPEND CMAKE_MODULE_PATH ${CMAKE_BINARY_DIR})
    list(PREPEND CMAKE_PREFIX_PATH ${CMAKE_BINARY_DIR})

    find_package(Boost REQUIRED) # header only libraries must not be added here

    find_package(GTest REQUIRED)
    find_package(OpenSSL REQUIRED)
    find_package(spdlog REQUIRED)
    find_package(tsl-hat-trie REQUIRED)
    #find_package(libp2p REQUIRED)
    find_package(c-ares REQUIRED)
    find_package(fmt REQUIRED)
    #find_package(soralog REQUIRED)
    find_package(yaml-cpp REQUIRED)
    find_package(Protobuf REQUIRED)
    find_package(RocksDB REQUIRED)
    find_package(Microsoft.GSL REQUIRED)
endmacro()

macro(main_func)
    setup_platform_environment()
    setup_directories()
    setup_conan()
    setup_dependency()
endmacro()

main_func()

# BOOST library
# set(Boost_USE_STATIC_LIBS ON)
# set(Boost_USE_MULTITHREADED ON)
# set(Boost_USE_STATIC_RUNTIME ON)
#find_package(Boost REQUIRED COMPONENTS date_time filesystem random regex) # header only libraries must not be added here

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}
    ${Microsoft.GSL_INCLUDE_DIRS}
  deps/tinycbor/src
    ${GV_external_dir_a}/include
)

link_directories(
    ${GV_dir_gen_lib}
    ${CONAN_LIB_DIRS}
    ${GV_external_dir_a}/lib
)

# # find packages for common
# message("libp2p_DIR=" ${libp2p_DIR})
#find_package(GTest CONFIG REQUIRED)
#find_package(OpenSSL REQUIRED)
#find_package(spdlog CONFIG REQUIRED)
#find_package(tsl-hat-trie CONFIG REQUIRED)
##find_package(libp2p CONFIG REQUIRED)
##find_package(c-ares CONFIG REQUIRED)
#find_package(fmt CONFIG REQUIRED)
##find_package(soralog CONFIG REQUIRED)
#find_package(yaml-cpp CONFIG REQUIRED)
#find_package(Protobuf CONFIG REQUIRED)
#find_package(RocksDB CONFIG REQUIRED)

option(TESTING "Build tests" ON)

add_subdirectory(deps)
add_subdirectory(primitives)
add_subdirectory(clock)
add_subdirectory(codec)
add_subdirectory(common)
add_subdirectory(crypto)
add_subdirectory(data_transfer)
add_subdirectory(ipfs_lite)

      #message("${CMAKE_CURRENT_LIST_DIR}")
      #message("${CMAKE_INSTALL_INCLUDEDIR}")
    install_hfile(ipfs-lite)

if (TESTING)
    SET(WIN_CRYPT_LIBRARY)
    if (NOT UNIX)
        SET(WIN_CRYPT_LIBRARY crypt32)
    endif()
  enable_testing()
  add_subdirectory(test)
  add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} DEPENDS rocksdb_integration_test)
endif ()

set(CONFIG_NAME "ipfs-lite-cppConfig")
set(CONFIG_DESTINATION_DIR "${CMAKE_INSTALL_PREFIX}/cmake/${PROJECT_NAME}")
#make_directory(${CMAKE_INSTALL_PREFIX}/include/protobuf)

install(TARGETS ipfs_blockservice
              ipfs_datastore_rocksdb
              ipfs_datastore_in_memory
              ipfs_merkledag_service
              graphsync
              buffer  rocksdb_wrapper  cid ipld_node cbor ipfs_lite_hasher logger
              hexutil
              blake2
              ipld_link
              outcome
              tinycbor
              ipld_block
              blob
              graphsync_proto
              ipld_node_protobuf
      EXPORT ${CONFIG_NAME}
      LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
      ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
      RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
      INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
      PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
      PUBLIC_PROTO_HEADER DESTINATION "${CMAKE_INSTALL_PREFIX}/include/protobuf"
      FRAMEWORK DESTINATION ${CMAKE_INSTALL_PREFIX}
      )
message("--install --- ${CMAKE_INSTALL_PREFIX}/include/protobuf")
### install header files ###
install_hfile(codec)
install_hfile(common)
install_hfile(crypto)
install_hfile(data_transfer)
install_hfile(deps)
install_hfile(platform)
install_hfile(primitives)
install_hfile(ipfs_lite)
### install protobuf header files ###
message("-gen---${CMAKE_BINARY_DIR}/generated")
#install(
#        DIRECTORY ${CMAKE_BINARY_DIR}/generated/ipfs_lite
#        DESTINATION ${CMAKE_INSTALL_PREFIX}/include
#        FILES_MATCHING # install only matched files
#        PATTERN "*.pb.h" # select header files hpp or h file
#    )
#message("---proto-----${proto_h_files}")

#INSTALL(FILES ${proto_h_files} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/protobuf)

install(
  EXPORT ${CONFIG_NAME}
  NAMESPACE ipfs-lite-cpp::
  DESTINATION ${CONFIG_DESTINATION_DIR}
)
