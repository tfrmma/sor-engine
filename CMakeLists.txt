cmake_minimum_required(VERSION 3.22)
project(sor_engine LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ── Compiler flags ────────────────────────────────────────────────────────────
add_library(sor_compile_flags INTERFACE)
target_compile_options(sor_compile_flags INTERFACE
    # Optimization
    $<$<CONFIG:Release>:-O3 -march=native -funroll-loops -fno-semantic-interposition>
    $<$<CONFIG:RelWithDebInfo>:-O2 -march=native -g>
    $<$<CONFIG:Debug>:-O0 -g3 -fno-omit-frame-pointer>

    # Diagnostics
    -Wall -Wextra -Wpedantic
    -Wconversion -Wshadow -Wnull-dereference
    -Wno-unused-parameter

    # No exceptions, no RTTI — standard in HFT C++
    -fno-exceptions
    -fno-rtti

    # Silence GCC pragma target warnings when not on x86
    -Wno-unknown-pragmas
)

# ── Core library ──────────────────────────────────────────────────────────────
add_library(sor_lib STATIC
    sor/routing_engine.cpp
)
target_include_directories(sor_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(sor_lib PUBLIC sor_compile_flags)
target_compile_features(sor_lib PUBLIC cxx_std_20)

# ── Example binary ────────────────────────────────────────────────────────────
add_executable(sor_example main_example.cpp)
target_link_libraries(sor_example PRIVATE sor_lib)

# ── ASan + UBSan debug target ─────────────────────────────────────────────────
add_executable(sor_example_san main_example.cpp sor/routing_engine.cpp)
target_include_directories(sor_example_san PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_options(sor_example_san PRIVATE
    -O1 -g3
    -fsanitize=address,undefined
    -fno-omit-frame-pointer
    -fno-exceptions -fno-rtti
)
target_link_options(sor_example_san PRIVATE -fsanitize=address,undefined)

# ── Installation ──────────────────────────────────────────────────────────────
install(TARGETS sor_example DESTINATION bin)
install(DIRECTORY sor/ DESTINATION include/sor FILES_MATCHING PATTERN "*.hpp")
