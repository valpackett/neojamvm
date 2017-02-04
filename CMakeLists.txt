project(jamvm C ASM)
cmake_minimum_required(VERSION 3.0)
message("-- CMAKE_SYSTEM_NAME:      ${CMAKE_SYSTEM_NAME}")
message("-- CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")
include(CheckIncludeFiles)
include(CheckLibraryExists)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
include_directories(${CMAKE_CURRENT_BINARY_DIR})
set(version "2.0.0")
set(major "2")
set(minor "0")
set(micro "0")

check_include_files(alloca.h HAVE_ALLOCA_H)
check_include_files(endian.h HAVE_ENDIAN_H)
check_library_exists(thr pthread_create "/lib" HAVE_LIBTHR)
if(NOT HAVE_LIBTHR)
	check_library_exists(pthread pthread_create "/lib" HAVE_LIBPTHREAD)
endif()
set(CLASSPATH_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/share/classpath" CACHE STRING "Path to the GNU Classpath installation")
set(JAVA_RUNTIME_LIBRARY "openjdk7" CACHE STRING "Java runtime library to use (gnuclasspath|openjdk6|openjdk7|openjdk8|openjdk9)")
option(BUILD_SHARED_LIBS "build shared library" ON)
option(INLINING "interpreter inlining" OFF)
option(PREFETCH "interpreter uses prefetching" OFF)
option(RUNTIME_RELOC_CHECKS "compute relocatability at runtime" OFF)
option(THREADED "interpreter threaded" ON)
option(TRACEALLOC "object allocation tracing enabled for debugging" OFF)
option(TRACECOMPACT "GC compaction phase tracing enabled for debugging" OFF)
option(TRACEDIRECT "interpreter direct-mode tracing enabled for debugging" OFF)
option(TRACEDLL "library loading/unloading tracing enabled for debugging" OFF)
option(TRACEFNLZ "object finalisation tracing enabled for debugging" OFF)
option(TRACEGC "gc tracing enabled for debugging" OFF)
option(TRACEINLINING "interpreter inlining tracing enabled for debugging" OFF)
option(TRACELOCK "object locking tracing enabled for debugging" OFF)
option(TRACETHREAD "thread creation tracing enabled for debugging" OFF)
option(USE_CACHE "interpreter uses caching" OFF)
#option(USE_FFI "use FFI" OFF)
option(USE_MD_STUBS "use machine-dependent JNI stubs" OFF)
option(USE_ZIP "use zip" ON)

if(JAVA_RUNTIME_LIBRARY STREQUAL "gnuclasspath")
	set(SHARED_CHAR_BUFFERS ON)
	set(CLASSLIB_DIR "src/classlib/gnuclasspath")
	# TODO set(CLASSLIB_SRCS ) 
elseif(JAVA_RUNTIME_LIBRARY STREQUAL "openjdk6")
	set(OPENJDK_VERSION 6)
elseif(JAVA_RUNTIME_LIBRARY STREQUAL "openjdk7")
	set(JSR292 ON)
	set(OPENJDK_VERSION 7)
elseif(JAVA_RUNTIME_LIBRARY STREQUAL "openjdk8")
	set(JSR292 ON)
	set(JSR308 ON)
	set(JSR335 ON)
	set(JSR901 ON)
	set(OPENJDK_VERSION 8)
elseif(JAVA_RUNTIME_LIBRARY STREQUAL "openjdk9")
	set(JSR292 ON)
	set(JSR308 ON)
	set(JSR335 ON)
	set(JSR901 ON)
	set(OPENJDK_VERSION 9)
endif()

if(HAVE_LIBPTHREAD)
	set(THREAD_LIBRARIES pthread)
elseif(HAVE_LIBTHR)
	set(THREAD_LIBRARIES thr)
endif()

if((${CMAKE_SYSTEM_NAME} STREQUAL "Linux") OR (${CMAKE_SYSTEM_NAME} STREQUAL "Solaris")) # XXX: is Illumos "Solaris"?
	find_package(LibDL REQUIRED)
endif()

if(USE_ZIP)
	find_package(ZLIB REQUIRED)
	set(ZIP_SRC src/zip.c)
endif()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "[xX]86_64|(amd|AMD)64")
	set(ARCH_NAME x86_64)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "[xX]86|i.86")
	set(ARCH_NAME i386)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm(v.*)?")
	set(ARCH_NAME arm)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "ppc.*|powerpc.*")
	set(ARCH_NAME powerpc)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "sparc.*")
	set(ARCH_NAME sparc)
endif()

execute_process(COMMAND ln -sf "arch/${ARCH_NAME}.h" "${CMAKE_CURRENT_SOURCE_DIR}/src/arch.h")

if(${CMAKE_SYSTEM_NAME} STREQUAL "FreeBSD")
	set(OS_DIR "src/os/freebsd")
	set(OS_ARCH_NAME "src/os/freebsd/${ARCH_NAME}")
	if(ARCH_NAME STREQUAL "x86_64")
		set(OS_SRCS "${OS_DIR}/os.c" "${OS_ARCH_NAME}/callNative.S" "${OS_ARCH_NAME}/dll_md.c" "${OS_ARCH_NAME}/init.c")
	endif()
endif()

if(JAVA_RUNTIME_LIBRARY MATCHES "openjdk.*")
	set(CLASSLIB_DIR "src/classlib/openjdk")
	set(CLASSLIB_SRCS "${CLASSLIB_DIR}/thread.c" "${CLASSLIB_DIR}/class.c" "${CLASSLIB_DIR}/natives.c"
		"${CLASSLIB_DIR}/excep.c" "${CLASSLIB_DIR}/reflect.c" "${CLASSLIB_DIR}/dll.c" "${CLASSLIB_DIR}/jni.c" "${CLASSLIB_DIR}/jvm.c"
		"${CLASSLIB_DIR}/properties.c" "${CLASSLIB_DIR}/management.c" "${CLASSLIB_DIR}/access.c" "${CLASSLIB_DIR}/frame.c"
		"${CLASSLIB_DIR}/shutdown.c" "${CLASSLIB_DIR}/alloc.c" "${CLASSLIB_DIR}/perf.c" "${CLASSLIB_DIR}/mh.c")
endif()

configure_file(src/config.h.in config.h)

# XXX: add compute_relocatability

add_library(jvm
	${OS_SRCS} ${CLASSLIB_SRCS}
	src/interp/engine/interp.c src/interp/engine/interp2.c src/interp/engine/relocatability.c src/interp/direct.c src/interp/inlining.c
	src/access.c src/alloc.c src/cast.c src/class.c
	# XXX dll_ffi
	src/dll.c
	src/excep.c src/execute.c src/frame.c src/hash.c src/hooks.c src/init.c src/jam.c src/jni.c src/jni-stubs.c src/lock.c
	src/natives.c src/properties.c src/reflect.c src/resolve.c src/shutdown.c src/sig.c src/string.c src/stubs.c src/symbol.c
	src/thread.c src/time.c src/utf8.c ${ZIP_SRC})
set_target_properties(jvm PROPERTIES
	LINK_FLAGS "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/mapfile")
target_link_libraries(jvm m rt ${THREAD_LIBRARIES} ${LIBDL_LIBRARIES} ${ZLIB_LIBRARIES})
target_include_directories(jvm PRIVATE
	src src/interp/engine src/interp ${CLASSLIB_DIR} ${CMAKE_CURRENT_BINARY_DIR} ${LIBDL_INCLUDE_DIRS} ${ZLIB_INCLUDE_DIRS})
