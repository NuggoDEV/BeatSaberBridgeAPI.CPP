cmake_minimum_required(VERSION 3.15)

project(BeatSaberBridgeAPI.CPP)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

# =========================================================
# Fetch Dependencies
# =========================================================

FetchContent_Declare(
    cpp_httplib
    GIT_REPOSITORY https://github.com/yhirose/cpp-httplib.git
    GIT_TAG master
)

FetchContent_Declare(
    json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG master
)

FetchContent_MakeAvailable(cpp_httplib json)

# =========================================================
# Executable
# =========================================================

add_executable(BeatSaberBridgeAPI.CPP main.cpp)

set_target_properties(BeatSaberBridgeAPI.CPP PROPERTIES
    OUTPUT_NAME "BeatSaberBridgeAPI"
)

# =========================================================
# RPATH
# =========================================================

if(APPLE)
    set(RPATH_ORIGIN "@executable_path")
else()
    set(RPATH_ORIGIN "$ORIGIN")
endif()

set_target_properties(BeatSaberBridgeAPI.CPP PROPERTIES
    BUILD_RPATH "${RPATH_ORIGIN}"
    INSTALL_RPATH "${RPATH_ORIGIN}"
)

# =========================================================
# Includes
# =========================================================

target_include_directories(BeatSaberBridgeAPI.CPP PRIVATE
    ${cpp_httplib_SOURCE_DIR}
)

# =========================================================
# OpenSSL
# =========================================================

find_package(OpenSSL REQUIRED)

# =========================================================
# libzip
# =========================================================

set(LIBZIP_TARGET "")

# Try modern CMake package first
find_package(libzip CONFIG QUIET)

if(TARGET libzip::zip)
    set(LIBZIP_TARGET libzip::zip)

elseif(TARGET zip)
    set(LIBZIP_TARGET zip)

else()

    # Fallback for Linux/macOS
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(LIBZIP REQUIRED IMPORTED_TARGET libzip)

    set(LIBZIP_TARGET PkgConfig::LIBZIP)

endif()

# =========================================================
# Link Main Libraries
# =========================================================

target_link_libraries(BeatSaberBridgeAPI.CPP PRIVATE
    nlohmann_json::nlohmann_json
    OpenSSL::SSL
    OpenSSL::Crypto
    ${LIBZIP_TARGET}
)

# =========================================================
# Platform-specific Libraries
# =========================================================

if(WIN32)

    target_compile_definitions(
        BeatSaberBridgeAPI.CPP PRIVATE
        _WIN32_WINNT=0x0A00
    )

    target_link_libraries(
        BeatSaberBridgeAPI.CPP PRIVATE
        ws2_32
        crypt32
    )

elseif(APPLE)

    target_link_libraries(
        BeatSaberBridgeAPI.CPP PRIVATE
        "-framework CoreFoundation"
        "-framework Security"
    )

endif()

# =========================================================
# Discord Social SDK
# =========================================================

set(DISCORD_SDK_ROOT "${CMAKE_SOURCE_DIR}/lib/discord_social_sdk")

set(DISCORD_SDK_LIB_DIR
    "${DISCORD_SDK_ROOT}/lib/release"
)

set(DISCORD_SDK_BIN_DIR
    "${DISCORD_SDK_ROOT}/bin/release"
)

set(DISCORD_SDK_INCLUDE_DIR
    "${DISCORD_SDK_ROOT}/include"
)

target_include_directories(
    BeatSaberBridgeAPI.CPP PRIVATE
    ${DISCORD_SDK_INCLUDE_DIR}
)

if(WIN32)

    set(DISCORD_LIB_PATH
        "${DISCORD_SDK_LIB_DIR}/discord_partner_sdk.lib"
    )

    set(DISCORD_SHARED_LIB
        "${DISCORD_SDK_BIN_DIR}/discord_partner_sdk.dll"
    )

elseif(APPLE)

    set(DISCORD_LIB_PATH
        "${DISCORD_SDK_LIB_DIR}/libdiscord_partner_sdk.dylib"
    )

    set(DISCORD_SHARED_LIB
        "${DISCORD_SDK_LIB_DIR}/libdiscord_partner_sdk.dylib"
    )

elseif(UNIX)

    set(DISCORD_LIB_PATH
        "${DISCORD_SDK_LIB_DIR}/libdiscord_partner_sdk.so"
    )

    set(DISCORD_SHARED_LIB
        "${DISCORD_SDK_LIB_DIR}/libdiscord_partner_sdk.so"
    )

endif()

target_link_libraries(
    BeatSaberBridgeAPI.CPP PRIVATE
    ${DISCORD_LIB_PATH}
)

# =========================================================
# Runtime Dependency Copy Helper
# =========================================================

function(copy_runtime_file TARGET_NAME FILE_PATH)

    if(EXISTS "${FILE_PATH}")

        get_filename_component(
            FILE_NAME
            "${FILE_PATH}"
            NAME
        )

        add_custom_command(
            TARGET ${TARGET_NAME}
            POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${FILE_PATH}"
            "$<TARGET_FILE_DIR:${TARGET_NAME}>/${FILE_NAME}"
        )

    endif()

endfunction()

# =========================================================
# Copy Discord SDK Runtime
# =========================================================

copy_runtime_file(
    BeatSaberBridgeAPI.CPP
    "${DISCORD_SHARED_LIB}"
)

# =========================================================
# Copy OpenSSL Runtime Libraries
# =========================================================

get_target_property(
    OPENSSL_SSL_LOCATION
    OpenSSL::SSL
    IMPORTED_LOCATION_RELEASE
)

get_target_property(
    OPENSSL_CRYPTO_LOCATION
    OpenSSL::Crypto
    IMPORTED_LOCATION_RELEASE
)

copy_runtime_file(
    BeatSaberBridgeAPI.CPP
    "${OPENSSL_SSL_LOCATION}"
)

copy_runtime_file(
    BeatSaberBridgeAPI.CPP
    "${OPENSSL_CRYPTO_LOCATION}"
)

# =========================================================
# Copy libzip Runtime Library
# =========================================================

if(TARGET libzip::zip)

    get_target_property(
        LIBZIP_LOCATION
        libzip::zip
        IMPORTED_LOCATION_RELEASE
    )

    copy_runtime_file(
        BeatSaberBridgeAPI.CPP
        "${LIBZIP_LOCATION}"
    )

elseif(TARGET zip)

    get_target_property(
        LIBZIP_LOCATION
        zip
        IMPORTED_LOCATION_RELEASE
    )

    copy_runtime_file(
        BeatSaberBridgeAPI.CPP
        "${LIBZIP_LOCATION}"
    )

endif()