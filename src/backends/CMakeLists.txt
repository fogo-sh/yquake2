# Backend sources (platform-specific and shared)

set(Backends-Generic-Source
	${BACKENDS_SRC_DIR}/generic/misc.c
)

set(Backends-Unix-Source
	${BACKENDS_SRC_DIR}/unix/main.c
	${BACKENDS_SRC_DIR}/unix/network.c
	${BACKENDS_SRC_DIR}/unix/signalhandler.c
	${BACKENDS_SRC_DIR}/unix/system.c
	${BACKENDS_SRC_DIR}/unix/shared/hunk.c
)

set(Backends-Windows-Source
	${BACKENDS_SRC_DIR}/windows/icon.rc
	${BACKENDS_SRC_DIR}/windows/main.c
	${BACKENDS_SRC_DIR}/windows/network.c
	${BACKENDS_SRC_DIR}/windows/system.c
	${BACKENDS_SRC_DIR}/windows/shared/hunk.c
)

set(Backends-Windows-Header
	${BACKENDS_SRC_DIR}/windows/header/resource.h
)

set(REF-Windows-Source
	${BACKENDS_SRC_DIR}/windows/shared/hunk.c
)

set(REF-Unix-Source
	${BACKENDS_SRC_DIR}/unix/shared/hunk.c
)

# Select platform-specific sources
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	set(Platform-Specific-Source ${Backends-Windows-Source} ${Backends-Windows-Header} PARENT_SCOPE)
	set(REF-Platform-Specific-Source ${REF-Windows-Source} PARENT_SCOPE)
else()
	set(Platform-Specific-Source ${Backends-Unix-Source} PARENT_SCOPE)
	set(REF-Platform-Specific-Source ${REF-Unix-Source} PARENT_SCOPE)
endif()

# Export generic sources as well
set(Backends-Generic-Source ${Backends-Generic-Source} PARENT_SCOPE)


