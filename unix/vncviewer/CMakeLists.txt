if(APPLE AND TVNC_BUILDJAVA)

find_package(JNI REQUIRED)

configure_file(vncviewer-mac.in ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vncviewer
	@ONLY)
execute_process(COMMAND chmod +x vncviewer
	WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

include_directories(${JAVA_INCLUDE_PATH} ${JAVA_INCLUDE_PATH2})

set(HELPER_SOURCES turbovnchelper.m JRSwizzle.m)
add_library(turbovnchelper SHARED ${HELPER_SOURCES})
set_source_files_properties(${HELPER_SOURCES} PROPERTIES LANGUAGE C)
set_target_properties(turbovnchelper PROPERTIES
	LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/java)
target_link_libraries(turbovnchelper "-framework Cocoa")

add_executable(JavaAppLauncher JavaAppLauncher.m)
set_source_files_properties(JavaAppLauncher.m PROPERTIES LANGUAGE C)
target_link_libraries(JavaAppLauncher "-framework Cocoa")

install(TARGETS turbovnchelper DESTINATION ${CMAKE_INSTALL_JAVADIR})

else() # APPLE AND TVNC_BUILDJAVA

include(CheckCSourceCompiles)

include_directories(${X11_INCLUDE_DIR})

add_executable(checkshmpixmaps
	checkshmpixmaps.c)

target_link_libraries(checkshmpixmaps ${X11_LIBRARIES})

install(TARGETS checkshmpixmaps DESTINATION ${CMAKE_INSTALL_BINDIR})

configure_file(vncviewer.in ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vncviewer @ONLY)
execute_process(COMMAND chmod +x vncviewer
	WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

if(TVNC_BUILDJAVA)

if(BITS EQUAL 64)
	set(DEFAULT_TJPEG_JNILIBRARY /opt/libjpeg-turbo/lib64/libturbojpeg.so)
else()
	set(DEFAULT_TJPEG_JNILIBRARY /opt/libjpeg-turbo/lib32/libturbojpeg.so)
endif()

set(TJPEG_JNILIBRARY ${DEFAULT_TJPEG_JNILIBRARY} CACHE FILEPATH
	"Shared library containing TurboJPEG JNI functions (default: ${DEFAULT_TJPEG_JNILIBRARY})")

get_filename_component(TJPEG_JNILIBRARY_RP ${TJPEG_JNILIBRARY} REALPATH)

if(NOT EXISTS ${TJPEG_JNILIBRARY_RP})
	message(FATAL_ERROR "Could not find TurboJPEG JNI library at ${TJPEG_JNILIBRARY}.  Set TJPEG_JNILIBRARY to the full pathname of libturbojpeg.so or install the libjpeg-turbo SDK.")
endif()

set(CMAKE_REQUIRED_LIBRARIES ${TJPEG_JNILIBRARY})
check_c_source_compiles("\n
	struct JNINativeInterface_;\n
	typedef const struct JNINativeInterface_ *JNIEnv;\n
	typedef struct _jobject *jobject;\n
	typedef jobject jclass;\n
	int Java_org_libjpegturbo_turbojpeg_TJ_bufSize\n
		(JNIEnv *, jclass, int, int, int);\n
	int main(void) {\n
		Java_org_libjpegturbo_turbojpeg_TJ_bufSize(0, 0, 1, 1, 0);\n
		return 0;\n
	}" TURBOJPEG_JNI_WORKS)
set(CMAKE_REQUIRED_LIBRARIES)
if(NOT TURBOJPEG_JNI_WORKS)
	message(FATAL_ERROR "Could not link with TurboJPEG JNI library ${TJPEG_JNILIBRARY}.  This may be because it was not built with JNI support.")
endif()

message(STATUS "TJPEG_JNILIBRARY = ${TJPEG_JNILIBRARY}")

find_package(JNI REQUIRED)

install(PROGRAMS ${TJPEG_JNILIBRARY_RP} DESTINATION ${CMAKE_INSTALL_JAVADIR}
	RENAME libturbojpeg.so)

include_directories(${JAVA_INCLUDE_PATH} ${JAVA_INCLUDE_PATH2}
	${CMAKE_SOURCE_DIR}/common/rfb)
add_library(turbovnchelper SHARED turbovnchelper.c)
if(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
	set_target_properties(turbovnchelper PROPERTIES LINK_FLAGS "-lc -z defs")
else()
	set_target_properties(turbovnchelper PROPERTIES LINK_FLAGS "-z defs")
endif()
set_target_properties(turbovnchelper PROPERTIES
	LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/java)
if(NOT CMAKE_SYSTEM_NAME MATCHES "(OpenBSD|FreeBSD|NetBSD|DragonFly)")
	set(LIBDL dl)
endif()
target_link_libraries(turbovnchelper ${X11_LIBRARIES} ${X11_Xi_LIB} ${LIBDL})

install(TARGETS turbovnchelper DESTINATION ${CMAKE_INSTALL_JAVADIR})

endif()

endif() # APPLE AND TVNC_BUILDJAVA

install(PROGRAMS ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/vncviewer
	DESTINATION ${CMAKE_INSTALL_BINDIR})

install(FILES vncviewer.man DESTINATION ${CMAKE_INSTALL_MANDIR}/man1
	RENAME vncviewer.1)
