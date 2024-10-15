############################################################################
# CMakeLists.txt
#
#    Released under the MIT License.  See LICENSE.txt for license information.
#
############################################################################

if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

set(DECAF_HEADER_FILES
	include/constant_time.h
	include/field.h
	include/keccak_internal.h
	include/portable_endian.h
	include/word.h
)
set(DECAF_SOURCE_FILES_C
	utils.c
	shake.c
	sha512.c
	spongerng.c

)

# default target arch is arch_32 which shall be generic enough to compile mostly on anything
# target arch dirs:
# global one to get the include/arch_intrinsics.h
# availables: arch_32, arch_arm_32, arch_neon, arch_ref64, arch_x86_64
set(TARGET_ARCH_DIR arch_32)
# specific to p25519, to get f_impl.c/h in p25519
#availables: arch_32, arch_ref64, arch_x86_64
set(TARGET_ARCH_DIR_P25519 arch_32)
# specific to p448, to get f_impl.c/h in p448
# availables: arch_32, arch_arm_32, arch_neon, arch_ref64, arch_x86_64
set(TARGET_ARCH_DIR_P448 arch_32)


if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64" AND NOT MSVC)#Decaf doesn't support 64bits on MSVC yet
	message("Target architecture is x86_64")
	set(TARGET_ARCH_DIR arch_x86_64)
	set(TARGET_ARCH_DIR_P25519 arch_x86_64)
	set(TARGET_ARCH_DIR_P448 arch_x86_64)
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arch64") # shall be arm64 bits, stick to ref64.
	message("Target architecture is 64 bits general purpose(arm64 shall use this)")
	set(TARGET_ARCH_DIR arch_ref64)
	set(TARGET_ARCH_DIR_P25519 arch_ref64)
	set(TARGET_ARCH_DIR_P448 arch_ref64)
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm") # is an arm 32 bits
	if (NOT ${CMAKE_ANDROID_ARCH_ABI} STREQUAL "armeabi") # arm <= 5.0 does not support instructions from the lib, keep arch_32
		if(${ANDROID_ARM_NEON})
			message("Target architecture is arm32 NEON")
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=neon") # build with neon flag
			set(TARGET_ARCH_DIR arch_neon)
			set(TARGET_ARCH_DIR_P25519 arch_32) # nothing specific for neon on p25519
			set(TARGET_ARCH_DIR_P448 arch_neon)
		else(${ANDROID_ARM_NEON})
			message("Target architecture is arm32 no NEON")
			set(TARGET_ARCH_DIR arch_arm_32)
			set(TARGET_ARCH_DIR_P25519 arch_32) # nothing specific for arch_arm on p25519
			set(TARGET_ARCH_DIR_P448 arch_arm_32)
		endif(${ANDROID_ARM_NEON})
	endif (NOT ${CMAKE_ANDROID_ARCH_ABI} STREQUAL "armeabi")
else() # nothing picked yet, stick to the
	message("Target architecture is general purpose 32bits")
endif()

include_directories(
	${PROJECT_SOURCE_DIR}/src/include/${TARGET_ARCH_DIR}
)

set(DECAF_SOURCE_FILES_CXX
)

add_subdirectory(curve25519)
add_subdirectory(ed448goldilocks)
add_subdirectory(p25519)
add_subdirectory(p448)

add_subdirectory(generator)

if(ENABLE_STATIC)
	add_library(decaf-static STATIC ${DECAF_HEADER_FILES} ${DECAF_SOURCE_FILES_C} ${DECAF_SOURCE_FILES_CXX} $<TARGET_OBJECTS:decaf-p25519> $<TARGET_OBJECTS:decaf-p448> $<TARGET_OBJECTS:decaf-curve25519> $<TARGET_OBJECTS:decaf-curve448>)
	add_dependencies(decaf-static decaf-generated-code)
	set_target_properties(decaf-static PROPERTIES OUTPUT_NAME decaf)
	target_include_directories(decaf-static INTERFACE $<BUILD_INTERFACE:${GSOURCE_PATH}/include> $<INSTALL_INTERFACE:include/decaf>)
	target_link_libraries(decaf-static INTERFACE)
endif()
if(ENABLE_SHARED)
	add_library(decaf SHARED ${DECAF_HEADER_FILES} ${DECAF_SOURCE_FILES_C} ${DECAF_SOURCE_FILES_CXX} $<TARGET_OBJECTS:decaf-p25519> $<TARGET_OBJECTS:decaf-p448> $<TARGET_OBJECTS:decaf-curve25519> $<TARGET_OBJECTS:decaf-curve448>)
	add_dependencies(decaf decaf-generated-code)
	set_target_properties(decaf PROPERTIES VERSION ${DECAF_SO_VERSION})
	target_include_directories(decaf INTERFACE $<BUILD_INTERFACE:${GSOURCE_PATH}/include> $<INSTALL_INTERFACE:include/decaf>)
	target_link_libraries(decaf PRIVATE)
	if(MSVC)
		if(CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
			install(FILES $<TARGET_PDB_FILE:decaf>
				DESTINATION ${CMAKE_INSTALL_BINDIR}
				PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
			)
		endif()
	endif()
endif()

if(ENABLE_STATIC)
	install(TARGETS decaf-static EXPORT ${EXPORT_TARGETS_NAME}Targets
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
	)
endif()
if(ENABLE_SHARED)
	install(TARGETS decaf EXPORT ${EXPORT_TARGETS_NAME}Targets
		RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
		LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
		ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
		PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
	)
endif()

install(DIRECTORY ${GSOURCE_PATH}/include/
	DESTINATION include/decaf
	FILES_MATCHING PATTERN "*.h*"
	PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
)

# Определение директории для копирования DLL или других артефактов в зависимости от конфигурации (Debug или Release)
if(MSVC OR WIN32)
    # Для мультиконфигурационных систем (таких как MSVC) используй CMAKE_CFG_INTDIR
    set(OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/src/\$<CONFIG>")
else()
    # Для одноконфигурационных систем
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        set(OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/src/debug")
    elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
        set(OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/src/release")
    else()
        set(OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/src/${CMAKE_BUILD_TYPE}")
    endif()
endif()

# Копирование decaf.dll или других артефактов в соответствующую директорию после сборки
if(ENABLE_SHARED)
    add_custom_command(TARGET decaf POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "$<TARGET_FILE:decaf>"
        "${OUTPUT_DIRECTORY}/$<TARGET_FILE_NAME:decaf>"
        COMMENT "Copying decaf.dll to ${OUTPUT_DIRECTORY}"
    )
endif()

if(ENABLE_STATIC)
    add_custom_command(TARGET decaf-static POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "$<TARGET_FILE:decaf-static>"
        "${OUTPUT_DIRECTORY}/$<TARGET_FILE_NAME:decaf-static>"
        COMMENT "Copying decaf-static to ${OUTPUT_DIRECTORY}"
    )
endif()