FIND_PROGRAM(
	MAKENSIS_EXE makensis
	PATHS "[HKEY_LOCAL_MACHINE\\Software\\Microsoft\\Windows\\CurrentVersion\\Uninstall\\NSIS;InstallLocation]"
)
IF(NOT MAKENSIS_EXE)
	MESSAGE(
		FATAL_ERROR
		"makensis.exe could not be found.\n"
		"makensis.exe is a part of NSIS. Get NSIS from http://nsis.sf.net/"
	)
ENDIF(NOT MAKENSIS_EXE)
	
IF(MSVC)
	INCLUDE(InstallRequiredSystemLibraries)
	LIST(GET CMAKE_INSTALL_SYSTEM_RUNTIME_LIBS 0 CRT_REDIST_PATH)
	CMAKE_PATH(REMOVE_FILENAME CRT_REDIST_PATH)	
ENDIF()

MATH(EXPR bits "${CMAKE_SIZEOF_VOID_P} * 8")
SET(LICENSE_FILE "${CMAKE_SOURCE_DIR}/GPL3.txt")
SET(INSTALLER_FILENAME "scantailor-universal-${VERSION}-${bits}bit-install.exe")
SET(REGISTER_EXTENSION_NSH "${CMAKE_CURRENT_SOURCE_DIR}/registerExtension.nsh")

SET(
	SOURCES
	scantailor.nsi.in "${CMAKE_SOURCE_DIR}/cmake/generate_nsi_file.cmake.in"
	"${CMAKE_SOURCE_DIR}/cmake/prepare_staging_dir.cmake.in"
)

SOURCE_GROUP("Sources" FILES ${SOURCES})
SOURCE_GROUP("Generated" FILES "${CMAKE_BINARY_DIR}/scantailor.nsi")

CONFIGURE_FILE(
	"${CMAKE_SOURCE_DIR}/cmake/prepare_staging_dir.cmake.in"
	"${CMAKE_BINARY_DIR}/prepare_staging_dir.cmake" @ONLY
)

SET(MAYBE_CRASHREPORTER_EXE "")
IF(ENABLE_CRASH_REPORTER)
	SET(MAYBE_CRASHREPORTER_EXE "-DCRASHREPORTER_EXE=$<TARGET_FILE:CrashReporter>")
ENDIF()

SET(MAYBE_CRT_REDIST_PATH "")
IF(CRT_REDIST_PATH)
	SET(MAYBE_CRT_REDIST_PATH "-DCRT_REDIST_PATH=${CRT_REDIST_PATH}")
ENDIF()
ADD_CUSTOM_TARGET(
	prepare_staging_dir
	COMMAND "${CMAKE_COMMAND}" ARGS -DCFG=$<CONFIGURATION>
        "-DCONF_BUILD_DIR=$<TARGET_FILE_DIR:scantailor-universal>"
	"-DSCANTAILOR_EXE=$<TARGET_FILE:scantailor-universal>"
	"-DSCANTAILOR_CLI_EXE=$<TARGET_FILE:scantailor-universal-cli>"
	${MAYBE_CRASHREPORTER_EXE}
	${MAYBE_CRT_REDIST_PATH}
	-P "${CMAKE_BINARY_DIR}/prepare_staging_dir.cmake"
	DEPENDS "${CMAKE_SOURCE_DIR}/cmake/prepare_staging_dir.cmake.in"
	VERBATIM
)
ADD_DEPENDENCIES(
	prepare_staging_dir
	scantailor-universal scantailor-universal-cli compile_translations
)
IF(ENABLE_CRASH_REPORTER)
	ADD_DEPENDENCIES(prepare_staging_dir CrashReporter)
ENDIF()

SET(extra_deps scantailor compile_translations)

SET(DUMP_SYMBOLS_COMMANDS "")
IF(ENABLE_CRASH_REPORTER)
	LIST(APPEND extra_deps CrashReporter)
	SET(
		DUMP_SYMBOLS_COMMANDS
		COMMAND "${CMAKE_COMMAND}" ARGS -E echo "Dumping symbols..."
		
		COMMAND "${DUMP_SYMS_EXECUTABLE}" ARGS "$<TARGET_FILE:scantailor-universal>"
		">" "${SYMBOLS_PATH}/temp.sym"
		COMMAND "${CMAKE_COMMAND}" ARGS "-DSYMBOLS_PATH=${SYMBOLS_PATH}"
		-P "${CMAKE_SOURCE_DIR}/cmake/move_sym_file.cmake"
		
		COMMAND "${DUMP_SYMS_EXECUTABLE}" ARGS "$<TARGET_FILE:scantailor-universal-cli>"
		">" "${SYMBOLS_PATH}/temp.sym"
		COMMAND "${CMAKE_COMMAND}" ARGS "-DSYMBOLS_PATH=${SYMBOLS_PATH}"
		-P "${CMAKE_SOURCE_DIR}/cmake/move_sym_file.cmake"
	)
ENDIF(ENABLE_CRASH_REPORTER)

CONFIGURE_FILE(
	"${CMAKE_SOURCE_DIR}/cmake/generate_nsi_file.cmake.in"
	"${CMAKE_BINARY_DIR}/generate_nsi_file.cmake" @ONLY
)

SET(
	scantailor_nsi_command
	OUTPUT "${CMAKE_BINARY_DIR}/scantailor.nsi"
	COMMAND "${CMAKE_COMMAND}" "-DCFG=$<CONFIGURATION>"
)
IF(ENABLE_CRASH_REPORTER)
	LIST(
		APPEND scantailor_nsi_command
		"-DCRASHREPORTER_EXE=$<TARGET_FILE:CrashReporter>"
	)
ENDIF()
LIST(
	APPEND scantailor_nsi_command
	-P "${CMAKE_BINARY_DIR}/generate_nsi_file.cmake"
	DEPENDS scantailor.nsi.in
	"${CMAKE_SOURCE_DIR}/cmake/generate_nsi_file.cmake.in"
	VERBATIM
)
ADD_CUSTOM_COMMAND(${scantailor_nsi_command})

ADD_CUSTOM_COMMAND(
	OUTPUT "${INSTALLER_FILENAME}"
	${DUMP_SYMBOLS_COMMANDS}
	COMMAND "${MAKENSIS_EXE}" /V2 scantailor.nsi
	WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
	MAIN_DEPENDENCY "${CMAKE_BINARY_DIR}/scantailor.nsi"
	DEPENDS ${extra_deps}
	VERBATIM
)

OPTION(BUILD_INSTALLER_BY_DEFAULT "Whether the \"installer\" target gets built by default" ON)
SET(maybe_all "")
IF(BUILD_INSTALLER_BY_DEFAULT)
	SET(maybe_all ALL)
ENDIF()
ADD_CUSTOM_TARGET(
	installer ${maybe_all}
	SOURCES ${SOURCES}
	DEPENDS "${INSTALLER_FILENAME}"
)
ADD_DEPENDENCIES(installer prepare_staging_dir)
