cmake_minimum_required(VERSION 3.13)

set(PICO_BOARD pico_w CACHE STRING "Board type")

# No `PICO_HARD_FLOAT_ABI`: the RP2040's Cortex-M0+ has no FPU at all, so soft float is not a
# convention chosen over hardware — it is the only thing there is, and sysl's `thumbv6m-freestanding`
# and the SDK's default already agree.

include(pico_sdk_import.cmake)

project(sysl_ogol C CXX ASM)

pico_sdk_init()

set(SYSL_ARCHIVE ${CMAKE_CURRENT_BINARY_DIR}/libogol.a)

# `sh.sysl.pico` carries C, and it needs the SDK's headers to compile. `sysl build-c` is a program
# CMake knows nothing about, so what it can see has to be handed to it.
#
# The libc half comes from the toolchain rather than being written down: `-print-sysroot` answers with
# the directory whose `include` holds newlib's headers.
execute_process(
    COMMAND ${CMAKE_C_COMPILER} -print-sysroot
    OUTPUT_VARIABLE ARM_SYSROOT
    OUTPUT_STRIP_TRAILING_WHITESPACE)

# The SDK half is asked of the SDK, because it generates some of its headers into its own build
# directory. What is asked is the executable's resolved list rather than any one library's, so that
# the transitive closure CMake has already worked out for the C side is what `sysl` gets.
set(SYSL_C_INCLUDES
    ${ARM_SYSROOT}/include
    $<TARGET_PROPERTY:sysl_ogol,INCLUDE_DIRECTORIES>)

# And the macros, which finding the headers does not supply: `pico/cyw43_arch.h` `#error`s rather
# than picking an architecture variant.
set(SYSL_C_DEFINES $<TARGET_PROPERTY:sysl_ogol,COMPILE_DEFINITIONS>)

add_custom_command(
    OUTPUT ${SYSL_ARCHIVE} ${SYSL_ARCHIVE}.h
    COMMAND sysl build-c ${CMAKE_CURRENT_SOURCE_DIR}/ogol --target thumbv6m-freestanding
            "$<$<BOOL:${SYSL_C_INCLUDES}>:--include-path;$<JOIN:${SYSL_C_INCLUDES},;--include-path;>>"
            "$<$<BOOL:${SYSL_C_DEFINES}>:-D;$<JOIN:${SYSL_C_DEFINES},;-D;>>"
            -o ${SYSL_ARCHIVE}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/ogol/main.sysl
            ${CMAKE_CURRENT_SOURCE_DIR}/ogol/sysl.hocon
    COMMENT "Compiling Ogol and its console for Armv6-M"
    # Without `COMMAND_EXPAND_LISTS` the two generator expressions above are useless: a quoted
    # argument that expands to a `;`-separated list stays one argument.
    COMMAND_EXPAND_LISTS
    VERBATIM)

add_custom_target(sysl_archive DEPENDS ${SYSL_ARCHIVE})

# There is no C in this project at all: sysl exports `main`, the SDK's `crt0` branches straight into
# it, and the archive in the source list is what satisfies CMake's "a target must have sources" rule
# with no translation unit to infer a language from.
add_executable(sysl_ogol ${SYSL_ARCHIVE})

set_target_properties(sysl_ogol PROPERTIES LINKER_LANGUAGE C)

add_dependencies(sysl_ogol sysl_archive)

target_link_libraries(sysl_ogol pico_stdlib pico_cyw43_arch_none ${SYSL_ARCHIVE})

pico_enable_stdio_usb(sysl_ogol 1)
pico_enable_stdio_uart(sysl_ogol 0)

pico_add_extra_outputs(sysl_ogol)
