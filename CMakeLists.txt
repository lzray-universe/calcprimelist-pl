cmake_minimum_required(VERSION 3.16)
project(calcprimelist LANGUAGES CXX)
# ===== Aggressive perf flags (Release only) =====
# Turn on IPO/LTO globally for Release
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)

# Silence non-essential logs in Release if你的代码支持该宏（可选）
# add_compile_definitions($<$<CONFIG:Release>:CALCPRIME_SILENT=1>)

# Global compile/link options for Release
if (MSVC)
  # Compile-time opts
  add_compile_options(
    $<$<CONFIG:Release>:/O2>
    $<$<CONFIG:Release>:/GL>           # IPO/LTO
    $<$<CONFIG:Release>:/Gw>           # Whole program ICF
    $<$<CONFIG:Release>:/Gy>           # Function-level linking
    $<$<CONFIG:Release>:/Ob3>          # Inline aggressively
    $<$<CONFIG:Release>:/Oi>           # Intrinsics
    $<$<CONFIG:Release>:/Ot>           # Favor speed
    $<$<CONFIG:Release>:/arch:AVX2>    # Vector ISA
    $<$<CONFIG:Release>:/favor:INTEL64>
    $<$<CONFIG:Release>:/fp:fast>
    $<$<CONFIG:Release>:/GS->          # disable buffer checks
    $<$<CONFIG:Release>:/DNDEBUG>
    $<$<CONFIG:Release>:/D_HAS_ITERATOR_DEBUGGING=0>
    $<$<CONFIG:Release>:/D_ITERATOR_DEBUG_LEVEL=0>
    $<$<CONFIG:Release>:/D_SECURE_SCL=0>
  )
  # Link-time opts
  add_link_options(
    $<$<CONFIG:Release>:/LTCG>         # IPO/LTO at link
    $<$<CONFIG:Release>:/OPT:ICF>
    $<$<CONFIG:Release>:/OPT:REF>
  )
else()
  # GCC/Clang
  add_compile_options(
    $<$<CONFIG:Release>:-O3>
    $<$<CONFIG:Release>:-ffp-contract=fast>
    $<$<CONFIG:Release>:-DNDEBUG>
    # x86: make sure we actually get POPCNT/BMI2 in addition to AVX2
    $<$<AND:$<CONFIG:Release>,$<STREQUAL:${CMAKE_SYSTEM_PROCESSOR},x86_64>>:-mavx2 -mpopcnt -mbmi2 -mfma>
  )
  add_link_options($<$<CONFIG:Release>:-flto>)
endif()
# ===== end aggressive flags =====
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CALCPRIME_SOURCES
    src/cpu_info.cpp
    src/wheel.cpp
    src/base_sieve.cpp
    src/bucket.cpp
    src/marker.cpp
    src/popcnt.cpp
    src/prime_count.cpp
    src/segmenter.cpp
    src/writer.cpp
)

function(calcprime_configure_library target)
    target_include_directories(${target} PUBLIC include)
    target_compile_definitions(${target} PRIVATE _USE_MATH_DEFINES)
    set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)

    if(MSVC)
        target_compile_options(${target} PRIVATE /arch:AVX2 /W4)
    else()
        target_compile_options(${target} PRIVATE -Wall -Wextra -Wpedantic)
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64|AMD64|i.86)")
            target_compile_options(${target} PRIVATE -mavx2)
        endif()
    endif()
endfunction()

add_library(calcprime STATIC ${CALCPRIME_SOURCES})
calcprime_configure_library(calcprime)

add_library(calcprime_static STATIC ${CALCPRIME_SOURCES})
calcprime_configure_library(calcprime_static)
if(MSVC)
    set_property(TARGET calcprime_static PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

add_executable(prime-sieve src/main.cpp)

target_link_libraries(prime-sieve PRIVATE calcprime)

add_executable(prime-sieve-static src/main.cpp)
target_link_libraries(prime-sieve-static PRIVATE calcprime_static)
if(MSVC)
    set_property(TARGET prime-sieve-static PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    target_compile_definitions(prime-sieve-static PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

add_library(calcprime-cli SHARED src/main.cpp src/api.cpp)
target_link_libraries(calcprime-cli PRIVATE calcprime)
target_compile_definitions(calcprime-cli PRIVATE CALCPRIME_DLL_BUILD CALCPRIME_DLL_EXPORT)
target_include_directories(calcprime-cli PUBLIC include)

enable_testing()

add_test(NAME prime_sieve_time_100k
    COMMAND $<TARGET_FILE:prime-sieve> --to 100000 --count --time)
set_tests_properties(prime_sieve_time_100k
    PROPERTIES PASS_REGULAR_EXPRESSION "Elapsed: [0-9]+ us")

add_test(NAME prime_sieve_scientific_to_100k
    COMMAND $<TARGET_FILE:prime-sieve> --to 1e5 --count --time)
set_tests_properties(prime_sieve_scientific_to_100k
    PROPERTIES PASS_REGULAR_EXPRESSION "9592;Elapsed: [0-9]+ us")

