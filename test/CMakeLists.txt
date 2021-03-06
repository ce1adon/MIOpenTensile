
cmake_policy(SET CMP0057 NEW)

include(CTest)

find_package(Threads REQUIRED)
include(ProcessorCount)
ProcessorCount(N)
set(CTEST_PARALLEL_LEVEL ${N} CACHE STRING "CTest parallel level")
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure -j ${CTEST_PARALLEL_LEVEL} -C ${CMAKE_CFG_INTDIR} --timeout 1500)
add_custom_target(tests)

find_program(MIOPEN_TENSILE_GDB gdb)

if(MIOPEN_TENSILE_GDB)
    set(MIOPEN_TENSILE_TEST_GDB On CACHE BOOL "")
else()
    set(MIOPEN_TENSILE_TEST_GDB Off CACHE BOOL "")
endif()

set(SKIP_TESTS)

function(add_test_command NAME EXE)
    if(NAME IN_LIST SKIP_TESTS)
        add_test(NAME ${NAME} COMMAND echo skipped)
        set_tests_properties(${NAME} PROPERTIES DISABLED On)
    elseif(WIN32)
        set(WINPATH)
        foreach(PATH ${CMAKE_FIND_ROOT_PATH})
            list(APPEND WINPATH ${PATH}/bin)
        endforeach()
        file(GENERATE OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/test_${NAME}.cmd"
            CONTENT "set PATH=${WINPATH};%PATH%
                    %1 ${ARGN}")
        add_test(NAME ${NAME} COMMAND ${WINE_CMD} cmd /c "${CMAKE_CURRENT_BINARY_DIR}/test_${NAME}.cmd" $<TARGET_FILE:${EXE}>)
    else()
        if(MIOPEN_TENSILE_TEST_GDB)
            # add_test(NAME ${NAME} COMMAND ${MIOPEN_TENSILE_GDB} 
            #     --batch
            #     --return-child-result
            #     -ex "set disable-randomization off"
            #     -ex run
            #     -ex backtrace
            #     --args $<TARGET_FILE:${EXE}> ${ARGN})
            set(TEST_DIR ${CMAKE_CURRENT_BINARY_DIR}/gdb/test_${NAME})
            file(MAKE_DIRECTORY ${TEST_DIR})
            file(GENERATE OUTPUT "${TEST_DIR}/run.cmake"
                CONTENT "
                # Remove previous core dump
                file(REMOVE ${TEST_DIR}/core)
                execute_process(COMMAND $<TARGET_FILE:${EXE}> ${ARGN} WORKING_DIRECTORY ${TEST_DIR} RESULT_VARIABLE RESULT)
                if(NOT RESULT EQUAL 0)
                    # TODO: check for core files based on pid when setting /proc/sys/kernel/core_uses_pid
                    if(EXISTS ${TEST_DIR}/core)
                        set(\$ENV{UBSAN_OPTIONS} print_stacktrace=1)
                        set(\$ENV{ASAN_OPTIONS} print_stacktrace=1)
                        execute_process(COMMAND ${MIOPEN_TENSILE_GDB} $<TARGET_FILE:${EXE}> ${TEST_DIR}/core -batch -ex bt)
                    endif()
                    message(FATAL_ERROR \"Test failed\")
                endif()
            ")
            add_test(NAME ${NAME} COMMAND ${CMAKE_COMMAND} -P "${TEST_DIR}/run.cmake")
        else()
            add_test(NAME ${NAME} COMMAND ${EXE} ${ARGN})
        endif()
    endif()
endfunction()

function(add_test_executable TEST_NAME)
    add_executable (${TEST_NAME} EXCLUDE_FROM_ALL ${ARGN})
    target_link_libraries(${TEST_NAME} ${CMAKE_THREAD_LIBS_INIT})
    # Cmake does not add flags correctly for gcc
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU") 
        set_target_properties(${TEST_NAME} PROPERTIES COMPILE_FLAGS -pthread LINK_FLAGS -pthread)
    endif()
    add_test_command(${TEST_NAME} ${TEST_NAME})
    add_dependencies(tests ${TEST_NAME})
    add_dependencies(check ${TEST_NAME})
    set_tests_properties(${TEST_NAME} PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
    target_link_libraries(${TEST_NAME} MIOpenTensile hip::device)
endfunction(add_test_executable)

file(GLOB TESTS *.cpp)

foreach(TEST ${TESTS})
    get_filename_component(BASE_NAME ${TEST} NAME_WE)
    add_test_executable(test_${BASE_NAME} ${TEST})
endforeach()
