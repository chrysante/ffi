cmake_minimum_required(VERSION 3.9)
project(libffi C ASM)

set(CMAKE_SHARED_LIBRARY_PREFIX)
set(CMAKE_STATIC_LIBRARY_PREFIX)

if(NOT FFI_ROOT)
    set(FFI_ROOT ${CMAKE_CURRENT_LIST_DIR})
else()
    message(STATUS "FFI_ROOT = ${FFI_ROOT}")
endif()

if(NOT CMAKE_SYSTEM_PROCESSOR)
    set(CMAKE_SYSTEM_PROCESSOR "${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(FFI_SKIP_HEADERS ON)
else()
    set(FFI_CONFIG_FILE ${FFI_ROOT}/fficonfig.h)
endif()

if(NOT INSTALL_DIRECTORY)
    set(INSTALL_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/out)
endif()

if(WIN32 OR MINGW)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-deprecated-declarations -Wno-pointer-to-int-cast")
endif()


# Create pkgconfig file
set(PACKAGE_VERSION ${VERSION})
set(prefix "${CMAKE_CURRENT_BINARY_DIR}")
set(exec_prefix "\${prefix}")
set(libdir "\${prefix}/lib")
set(toolexeclibdir "\${libdir}")
set(includedir "\${prefix}/include")
configure_file("${FFI_ROOT}/libffi.pc.in" "${CMAKE_CURRENT_BINARY_DIR}/conf/pkgconfig/libffi.pc" @ONLY)

message(STATUS "Building to ${prefix}")
set(CMAKE_INSTALL_PREFIX ${INSTALL_DIRECTORY})

# debug
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(prefix "${CURRENT_INSTALLED_DIR}/debug")
    set(exec_prefix "\${prefix}")
    set(libdir "\${prefix}/lib")
    set(toolexeclibdir "\${libdir}")
    set(includedir "\${prefix}/../include")
    configure_file("${FFI_ROOT}/libffi.pc.in" "${CMAKE_CURRENT_BINARY_DIR}/debug/conf/pkgconfig/libffi.pc" @ONLY)
endif()

# config variables for ffi.h.in
set(VERSION 3.4.2)

set(KNOWN_PROCESSORS x86 x86_64 amd64 arm arm64 i386 i686 armv7l armv7-a aarch64)

string(TOLOWER "${CMAKE_SYSTEM_PROCESSOR}" lower_system_processor)

if(NOT lower_system_processor IN_LIST KNOWN_PROCESSORS)
    message(FATAL_ERROR "Unknown processor: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

if(CMAKE_SYSTEM_NAME MATCHES "Windows" AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm")
    set(TARGET ARM_WIN32)
elseif(CMAKE_SYSTEM_NAME MATCHES "Windows" AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64")
    set(TARGET ARM_WIN64)
elseif(CMAKE_SYSTEM_NAME MATCHES "Windows" AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "x86")
    set(TARGET X86_WIN32)
elseif(CMAKE_SYSTEM_NAME MATCHES "Windows" AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "x64")
    set(TARGET X86_WIN64)
elseif(lower_system_processor MATCHES "arm64|aarch64")
    set(TARGET ARM64)
elseif(lower_system_processor MATCHES "arm")
    set(TARGET ARM)
elseif(CMAKE_SYSTEM_NAME MATCHES "BSD" AND CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(TARGET X86_FREEBSD)
elseif(CMAKE_SYSTEM_NAME MATCHES "Darwin"  AND CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(TARGET X86_DARWIN)
elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(TARGET X86_64)
elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(TARGET X86)
else()
    message(FATAL_ERROR "Cannot determine target. Please consult ${CMAKE_CURRENT_SOURCE_DIR}/configure.ac and add your platform to this CMake file.")
endif()

if("${TARGET}" STREQUAL "X86_64")
    set(HAVE_LONG_DOUBLE 1)
else()
    set(HAVE_LONG_DOUBLE 0)
endif()
set(FFI_EXEC_TRAMPOLINE_TABLE 0)

# mimic layout of original buildsystem
configure_file(${FFI_ROOT}/include/ffi.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/ffi.h)
file(COPY ${FFI_CONFIG_FILE} DESTINATION ${CMAKE_CURRENT_BINARY_DIR})

if ("${TARGET}" STREQUAL "ARM_WIN64" OR "${TARGET}" STREQUAL "ARM64")
    file(COPY ${FFI_ROOT}/src/aarch64/ffitarget.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/include)
elseif ("${TARGET}" STREQUAL "ARM_WIN32" OR "${TARGET}" STREQUAL "ARM")
    file(COPY ${FFI_ROOT}/src/arm/ffitarget.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/include)
else()
    file(COPY ${FFI_ROOT}/src/x86/ffitarget.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/include)
endif()

set(FFI_INCLUDE_DIRS
    PATHS ${CMAKE_CURRENT_BINARY_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}
    ${FFI_ROOT}/include
    ${FFI_ROOT})

if (NOT FFI_USE_SHARED_LIBS)
    add_definitions(-DFFI_BUILDING)
endif()
if((BUILD_SHARED_LIBS OR FFI_USE_SHARED_LIBS) AND WIN32)
    add_definitions(-DFFI_BUILDING_DLL)
endif()

set(FFI_SOURCES
    ${FFI_ROOT}/src/closures.c
    ${FFI_ROOT}/src/prep_cif.c
    ${FFI_ROOT}/src/types.c
    ${FFI_ROOT}/src/tramp.c)

if ("${TARGET}" STREQUAL "ARM_WIN64" OR "${TARGET}" STREQUAL "ARM64")
    set(FFI_SOURCES
        ${FFI_SOURCES}
        ${FFI_ROOT}/src/aarch64/ffi.c)
elseif("${TARGET}" STREQUAL "ARM_WIN32" OR "${TARGET}" STREQUAL "ARM")
    set(FFI_SOURCES
        ${FFI_SOURCES}
        ${FFI_ROOT}/src/arm/ffi.c)
else()
    set(FFI_SOURCES
        ${FFI_SOURCES}
        ${FFI_ROOT}/src/java_raw_api.c
        ${FFI_ROOT}/src/raw_api.c)
    if("${TARGET}" STREQUAL "X86_WIN32" OR "${TARGET}" STREQUAL "X86_DARWIN" OR "${TARGET}" STREQUAL "X86")
        set(FFI_SOURCES
            ${FFI_SOURCES}
            ${FFI_ROOT}/src/x86/ffi.c)
    elseif("${TARGET}" STREQUAL "X86_WIN64")
        set(FFI_SOURCES
            ${FFI_SOURCES}
            ${FFI_ROOT}/src/x86/ffiw64.c)
    elseif("${TARGET}" STREQUAL "X86_64")
        set(FFI_SOURCES
            ${FFI_SOURCES}
            ${FFI_ROOT}/src/x86/ffi64.c
            ${FFI_ROOT}/src/x86/ffiw64.c)
    endif()
endif()

macro(add_assembly ASMFILE)
    get_filename_component(ASMFILE_FULL "${ASMFILE}" ABSOLUTE)
    if(MSVC)
        if ("${TARGET}" STREQUAL "ARM_WIN64")
            set(ARCH_ASSEMBLER armasm64)
        elseif ("${TARGET}" STREQUAL "ARM_WIN32")
            set(ARCH_ASSEMBLER armasm)
        elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
            set(ARCH_ASSEMBLER ml /safeseh /c /Zi)
        else()
            set(ARCH_ASSEMBLER ml64 /c /Zi)
        endif()

        get_filename_component(ARCH_ASM_NAME "${ASMFILE_FULL}" NAME_WE)

        execute_process(
            COMMAND ${CMAKE_C_COMPILER} /nologo /EP /I. /Iinclude /I${CMAKE_CURRENT_SOURCE_DIR}/include "${ASMFILE_FULL}"
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            OUTPUT_FILE ${ARCH_ASM_NAME}.asm
            RESULT_VARIABLE retcode
        )

        if(NOT ${retcode} STREQUAL "0")
            message(FATAL_ERROR "Unable to assemble, exit code: '${retcode}'.")
        endif()

        # Produced *.asm file could be just added to sources.
        # It works in x64 mode, but for some strange reason MASM returns error code when in x86,
        # (even though it didn't report any errors and correctly generated object file)
        # which in turn causes MSBUILD to stop.
        execute_process(
            COMMAND ${ARCH_ASSEMBLER} ${ARCH_ASM_NAME}.asm
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            RESULT_VARIABLE retcode
        )

        if(NOT ${retcode} STREQUAL "0")
            message(FATAL_ERROR "Unable to compile assembly, exit code: '${retcode}'.")
        endif()

        list(APPEND FFI_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/${ARCH_ASM_NAME}.obj)
    else()
        list(APPEND FFI_SOURCES ${ASMFILE})
    endif()
endmacro()

if("${TARGET}" STREQUAL "X86")
    set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} -m32")
endif()

if("${TARGET}" STREQUAL "X86" OR "${TARGET}" STREQUAL "X86_DARWIN")
    add_assembly(${FFI_ROOT}/src/x86/sysv.S)
elseif("${TARGET}" STREQUAL "X86_64")
#    if(NOT MSVC AND NOT MINGW)
        add_assembly(${FFI_ROOT}/src/x86/unix64.S)
#    endif()
    add_assembly(${FFI_ROOT}/src/x86/win64.S)
elseif("${TARGET}" STREQUAL "X86_WIN32")
    if(MSVC)
        add_assembly(${FFI_ROOT}/src/x86/sysv_intel.S)
    else()
        add_assembly(${FFI_ROOT}/src/x86/sysv.S)
    endif()
elseif("${TARGET}" STREQUAL "X86_WIN64")
    if(MSVC)
        add_assembly(${FFI_ROOT}/src/x86/win64_intel.S)
    else()
        add_assembly(${FFI_ROOT}/src/x86/win64.S)
    endif()
elseif("${TARGET}" STREQUAL "ARM_WIN32")
    if(MSVC)
        add_assembly(${FFI_ROOT}/src/arm/sysv_msvc_arm32.S)
    else()
        add_assembly(${FFI_ROOT}/src/arm/sysv.S)
    endif()
elseif("${TARGET}" STREQUAL "ARM")
    add_assembly(${FFI_ROOT}/src/arm/sysv.S)
elseif("${TARGET}" STREQUAL "ARM_WIN64")
    if(MSVC)
        add_assembly(${FFI_ROOT}/src/aarch64/win64_armasm.S)
    else()
        add_assembly(${FFI_ROOT}/src/aarch64/sysv.S)
    endif()
elseif("${TARGET}" STREQUAL "ARM64")
    add_assembly(${FFI_ROOT}/src/aarch64/sysv.S)
else()
    message(FATAL_ERROR "Target not implemented")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    list(APPEND FFI_SOURCES ${FFI_ROOT}/src/debug.c)
    add_definitions(-DFFI_DEBUG)
endif()

function(string_replace source search replacement)
    file(READ ${source} FILE_CONTENTS)
    string(REPLACE ${search} ${replacement} NEW_CONTENT "${FILE_CONTENTS}")
    file(WRITE ${source} "${NEW_CONTENT}")
endfunction()

if(WIN32 OR MINGW)
    if(NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "Release")
        string_replace("${CMAKE_CURRENT_BINARY_DIR}/conf/pkgconfig/libffi.pc"
            "-lffi" "-llibffi")
    endif()
    if(NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "Debug")
        string_replace("${CMAKE_CURRENT_BINARY_DIR}/debug/conf/pkgconfig/libffi.pc"
            "-lffi" "-llibffi")
    endif()
endif()

add_library(libffi ${FFI_SOURCES})

target_include_directories(libffi PUBLIC ${FFI_INCLUDE_DIRS})

install(TARGETS libffi
    EXPORT ${PROJECT_NAME}Targets
    RUNTIME DESTINATION lib
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib)

include(CMakePackageConfigHelpers)

configure_package_config_file(${FFI_ROOT}/${PROJECT_NAME}Config.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION share/${PROJECT_NAME})
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
    VERSION ${VERSION}
    COMPATIBILITY AnyNewerVersion)

if(NOT FFI_SKIP_HEADERS)
    install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/include/ffi.h
        ${CMAKE_CURRENT_BINARY_DIR}/include/ffitarget.h
        DESTINATION include)
endif()

set(FFI_VERSION ${VERSION})
set(FFI_TARGET ${TARGET})
set(FFI_LIBRARIES $<TARGET_FILE:libffi>)
