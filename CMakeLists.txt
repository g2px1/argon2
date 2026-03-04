cmake_minimum_required(VERSION 3.27)
cmake_policy(SET CMP0144 NEW)
project(argon2 VERSION 1.0.0 LANGUAGES C)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(CMAKE_C_STANDARD 90)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_FLAGS_DEBUG "-g -O0" CACHE STRING "Debug flags" FORCE)

option(ARGON2_NO_THREADS "Disable threading support" OFF)
option(ARGON2_BUILD_CLI "Build argon2 CLI executable" ON)
option(ARGON2_BUILD_BENCH "Build argon2 bench executable" OFF)
option(ARGON2_BUILD_GENKAT "Build argon2 genkat executable" OFF)
option(ARGON2_ENABLE_OPT "Try to use optimized implementation (SSE/AVX)" ON)

set(ABI_VERSION 1)

# --- Sources ---

set(ARGON2_HEADERS
        ${CMAKE_CURRENT_SOURCE_DIR}/include/argon2.h
)

set(ARGON2_SOURCES
        src/argon2.c
        src/core.c
        src/blake2/blake2b.c
        src/thread.c
        src/encoding.c
)

# --- Detect optimized implementation support ---

if (ARGON2_ENABLE_OPT)
    include(CheckCSourceCompiles)
    set(CMAKE_REQUIRED_FLAGS "-march=native")
    set(CMAKE_REQUIRED_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/include" "${CMAKE_CURRENT_SOURCE_DIR}/src")
    check_c_source_compiles("
        #include \"core.h\"
        #include \"opt.h\"
        int main(void) { return 0; }
    " ARGON2_OPT_SUPPORTED)
    unset(CMAKE_REQUIRED_FLAGS)
    unset(CMAKE_REQUIRED_INCLUDES)
endif()

if (ARGON2_ENABLE_OPT AND ARGON2_OPT_SUPPORTED)
    message(STATUS "Argon2: building with optimizations (-march=native)")
    list(APPEND ARGON2_SOURCES src/opt.c)
    set(ARGON2_OPT_FLAG "-march=native")
else()
    message(STATUS "Argon2: building without optimizations (reference)")
    list(APPEND ARGON2_SOURCES src/ref.c)
    set(ARGON2_OPT_FLAG "")
endif()

# --- Library target ---

add_library(argon2 ${ARGON2_SOURCES} ${ARGON2_HEADERS})
add_library(argon2::argon2 ALIAS argon2)

target_include_directories(argon2
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

target_compile_options(argon2 PRIVATE -Wall -O3)

if (ARGON2_OPT_FLAG)
    target_compile_options(argon2 PRIVATE ${ARGON2_OPT_FLAG})
endif()

if (ARGON2_NO_THREADS)
    target_compile_definitions(argon2 PUBLIC ARGON2_NO_THREADS)
else()
    find_package(Threads REQUIRED)
    target_link_libraries(argon2 PUBLIC Threads::Threads)
endif()

# Visibility control on Linux shared builds
if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND BUILD_SHARED_LIBS)
    target_compile_definitions(argon2 PRIVATE A2_VISCTL=1)
    target_compile_options(argon2 PRIVATE -fvisibility=hidden)
endif()

set_target_properties(argon2 PROPERTIES
        EXPORT_NAME argon2
        VERSION ${PROJECT_VERSION}
        SOVERSION ${ABI_VERSION}
        PUBLIC_HEADER "${ARGON2_HEADERS}"
)

# --- CLI executable ---

if (ARGON2_BUILD_CLI)
    add_executable(argon2-cli src/run.c)
    set_target_properties(argon2-cli PROPERTIES OUTPUT_NAME argon2)
    target_link_libraries(argon2-cli PRIVATE argon2)
endif()

# --- Bench executable ---

if (ARGON2_BUILD_BENCH)
    add_executable(argon2-bench src/bench.c)
    target_link_libraries(argon2-bench PRIVATE argon2)
endif()

# --- Genkat executable ---

if (ARGON2_BUILD_GENKAT)
    add_executable(argon2-genkat src/genkat.c)
    target_compile_definitions(argon2-genkat PRIVATE GENKAT)
    target_link_libraries(argon2-genkat PRIVATE argon2)
endif()

# --- Tests ---

enable_testing()
add_executable(argon2-testcase EXCLUDE_FROM_ALL src/test.c)
target_link_libraries(argon2-testcase PRIVATE argon2)
target_compile_options(argon2-testcase PRIVATE -Wextra -Wno-type-limits)
add_test(NAME argon2-test COMMAND argon2-testcase)

# --- Install ---

install(TARGETS argon2
        EXPORT argon2Targets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

if (ARGON2_BUILD_CLI)
    install(TARGETS argon2-cli
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

# --- Package config ---

configure_package_config_file(
        cmake/argon2Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/argon2Config.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/argon2
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/argon2ConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/argon2Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/argon2ConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/argon2
)

install(EXPORT argon2Targets
        NAMESPACE argon2::
        FILE argon2Targets.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/argon2
)
