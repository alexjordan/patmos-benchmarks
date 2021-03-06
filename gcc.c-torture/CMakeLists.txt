file(GLOB SOURCES "${GCC_C_TORTURE_EXECUTE_PATH}/*.c" "${GCC_C_TORTURE_EXECUTE_PATH}/ieee/*.c")

# Sources that fail on host
# TODO: Update when changing compiler version
file(STRINGS FailHost.txt SOURCES_FAIL_HOST)

# Sources that fail building for Patmos
# TODO: Try to get rid of the build errors
file(STRINGS FailBuildTarget.txt SOURCES_FAIL_BUILDTARGET)

# Sources that cannot be run on HW due to size constraints
file(STRINGS NoHwTest.txt SOURCES_NO_HW_TEST)

# executable for each .c file
foreach(SRC ${SOURCES})

  get_filename_component(SRCPATH ${SRC} PATH)
  get_filename_component(SRCNAME ${SRC} NAME_WE)

  if (${SRCPATH} STREQUAL ${GCC_C_TORTURE_EXECUTE_PATH})
    set(EXEC_NAME "${SRCNAME}")
  else()
    get_filename_component(SRCDIR ${SRCPATH} NAME_WE)
    set(EXEC_NAME "${SRCDIR}_${SRCNAME}")
  endif()

  list(FIND SOURCES_FAIL_HOST ${EXEC_NAME} FAIL_BUILD)
  list(FIND SOURCES_FAIL_BUILDTARGET ${EXEC_NAME} FAIL_BUILDTARGET)
  list(FIND SOURCES_NO_HW_TEST ${EXEC_NAME} NO_HW_TEST)

  if((${FAIL_BUILD} EQUAL -1) AND (${FAIL_BUILDTARGET} EQUAL -1))

    add_executable(${EXEC_NAME} ${SRC})
    if(${NO_HW_TEST} EQUAL -1)
        set(torture-${EXEC_NAME}-run-hw-test true)
    endif()

    run_io(torture-${EXEC_NAME}
      "${CMAKE_CURRENT_BINARY_DIR}/${EXEC_NAME}" "" "" "")

    # A little bit of optimization and calm down warnings
    set(COMPILE_FLAGS "-O1 -Wno-implicit-int -Wno-return-type -fbuiltin")
    set(LINK_FLAGS "")

    # On Patmos, compile for RAM if HW test is enabled
    if("${CMAKE_SYSTEM_NAME}" MATCHES "patmos")
      set(TARGET_COMPILE_FLAGS "-DSTACK_SIZE=1024 -DNO_TRAMPOLINES")
      if (${NO_HW_TEST} EQUAL -1)
          set(TARGET_LINK_FLAGS "-Xgold -T${PROJECT_SOURCE_DIR}/scripts/patmos-ram.t -mpatmos-method-cache-size=0x800 -mpatmos-stack-base=0x1f0000 -mpatmos-shadow-stack-base=0x200000")
      else()
          set(TARGET_LINK_FLAGS "-mpatmos-method-cache-size=0x800")
      endif()
	endif()

	set_target_properties(${EXEC_NAME} PROPERTIES COMPILE_FLAGS
	  "${COMPILE_FLAGS} ${TARGET_COMPILE_FLAGS}")
	set_target_properties(${EXEC_NAME} PROPERTIES LINK_FLAGS
	  "${LINK_FLAGS} ${TARGET_LINK_FLAGS}")

    TARGET_LINK_LIBRARIES(${EXEC_NAME} "m")

  endif()

endforeach(SRC)
