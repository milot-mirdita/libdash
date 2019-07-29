cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)
project(dash C)

option(ENABLE_FNMATCH "Use fnmatch(3) from libc" OFF)
option(ENABLE_GLOB "Use glob(3) from libc" OFF)

if(ENABLE_FNMATCH)
	check_symbol_exists(fnmatch "fnmatch.h" HAVE_FNMATCH)
	if(HAVE_FNMATCH AND ENABLE_GLOB)
		check_symbol_exists(glob "glob.h" HAVE_GLOB)	
	endif()
endif()

include(CheckIncludeFile)
check_include_file(alloca.h HAVE_ALLOCA_H)
check_include_file(paths.h HAVE_PATHS_H)
if(NOT HAVE_PATHS_H)
	set(_PATH_BSHELL "/bin/sh" CACHE STRING "Define to system shell path")
	set(_PATH_DEVNULL "/dev/null" CACHE STRING "Define to devnull device node path")
	set(_PATH_TTY "/dev/tty" CACHE STRING "Define to tty device node path")
endif()

include(CheckCSourceCompiles)
check_c_source_compiles(
    "void t() {};
     void a() __attribute__((__alias__(\"t\")));
     int main() { a(); return 0; }"
    HAVE_ALIAS_ATTRIBUTE)

include(CheckSymbolExists)
check_symbol_exists(isblank "ctype.h" HAVE_DECL_ISBLANK)

include(CheckTypeSize)
check_type_size("intmax_t" SIZEOF_INTMAX_T)
check_type_size("long long int" SIZEOF_LONG_LONG_INT)
check_symbol_exists(PRIdMAX "inttypes.h" PRIdMAX)
if(NOT PRIdMAX)
	if(SIZEOF_INTMAX_T STREQUAL SIZEOF_LONG_LONG_INT)
		set(PRIdMAX lld)
	else()
		set(PRIdMAX jd)
	endif()
endif()

check_symbol_exists(bsearch "stdlib.h" HAVE_BSEARCH) 
check_symbol_exists(faccessat "unistd.h" HAVE_FACCESSAT) 
check_symbol_exists(getpwnam "pwd.h" HAVE_GETPWNAM) 
check_symbol_exists(getrlimit "sys/resource.h" HAVE_GETRLIMIT) 
check_symbol_exists(isalpha "ctype.h" HAVE_ISALPHA) 
check_symbol_exists(killpg "signal.h" HAVE_KILLPG) 
check_symbol_exists(mempcpy "string.h" HAVE_MEMPCPY) 
check_symbol_exists(sigsetmask "signal.h" HAVE_SIGSETMASK) 
check_symbol_exists(stpcpy "string.h" HAVE_STPCPY) 
check_symbol_exists(strchrnul "string.h" HAVE_STRCHRNUL) 
check_symbol_exists(strsignal "string.h" HAVE_STRSIGNAL) 
check_symbol_exists(strtod "stdlib.h" HAVE_STRTOD) 
check_symbol_exists(strtoimax "inttypes.h" HAVE_STRTOIMAX) 
check_symbol_exists(strtoumax "inttypes.h" HAVE_STRTOUMAX) 
check_symbol_exists(sysconf "unistd.h" HAVE_SYSCONF)

check_symbol_exists(signal "signal.h" HAVE_SIGNAL)
if(NOT HAVE_SIGNAL)
	check_symbol_exists(bsd_signal "signal.h" HAVE_BSD_SIGNAL)
	if(HAVE_BSD_SIGNAL)
		set(signal bsd_signal)
	endif()
endif()

include(CheckFunctionExists)
check_function_exists(stat64 HAVE_STAT64)
if(NOT HAVE_STAT64)
	set(fstat64 fstat)
	set(lstat64 lstat)
	set(stat64 stat)
endif()
check_function_exists(open64 HAVE_OPEN64)
if(NOT HAVE_OPEN64)
	set(open64 open)
endif()

check_c_source_compiles(
    "#include <time.h>
	 #include <sys/time.h>
	 #include <sys/stat.h>
     int main() { struct stat foo; return sizeof(foo.st_mtim.tv_sec); }"
    HAVE_ST_MTIM)

option(WITH_LIBEDIT "Compile with libedit support" OFF)
if(WITH_LIBEDIT)
	include(FindLibEdit)
	find_package(LibEdit REQUIRED)
else()
	set(SMALL 1)
endif()

option(DISABLE_LINENO "Disable LINENO support" OFF)
if(NOT DISABLE_LINENO)
	set(WITH_LINENO 1)
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

set(DASH_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../src")
add_definitions(-std=gnu89 -DBSD=1 -DSHELL -include ${CMAKE_CURRENT_BINARY_DIR}/config.h)

add_executable(mksignames ${DASH_SOURCE_DIR}/mksignames.c)
add_custom_command(
    OUTPUT  ${CMAKE_CURRENT_BINARY_DIR}/signames.c
    COMMAND mksignames 
    DEPENDS mksignames
)

add_executable(mksyntax ${DASH_SOURCE_DIR}/mksyntax.c)
target_include_directories(mksyntax PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
add_custom_command(
	OUTPUT
		${CMAKE_CURRENT_BINARY_DIR}/token.h
		${CMAKE_CURRENT_BINARY_DIR}/token_vars.h
		${CMAKE_CURRENT_BINARY_DIR}/syntax.h
		${CMAKE_CURRENT_BINARY_DIR}/syntax.c
    COMMAND sh ${DASH_SOURCE_DIR}/mktokens
	COMMAND mksyntax
	DEPENDS 
		${DASH_SOURCE_DIR}/parser.h
		${DASH_SOURCE_DIR}/mktokens
		mksyntax
)

add_executable(mknodes ${DASH_SOURCE_DIR}/mknodes.c)
add_custom_command(
    OUTPUT 
        ${CMAKE_CURRENT_BINARY_DIR}/nodes.h
        ${CMAKE_CURRENT_BINARY_DIR}/nodes.c
    COMMAND mknodes ${DASH_SOURCE_DIR}/nodetypes ${DASH_SOURCE_DIR}/nodes.c.pat
    DEPENDS mknodes ${DASH_SOURCE_DIR}/nodetypes ${DASH_SOURCE_DIR}/nodes.c.pat
)

add_custom_command(
    OUTPUT 
        ${CMAKE_CURRENT_BINARY_DIR}/builtins.def
    COMMAND ${CMAKE_C_COMPILER} -E -x c -o ${CMAKE_CURRENT_BINARY_DIR}/builtins.def ${DASH_SOURCE_DIR}/builtins.def.in -include ${CMAKE_CURRENT_BINARY_DIR}/config.h
    DEPENDS ${DASH_SOURCE_DIR}/builtins.def.in ${DASH_SOURCE_DIR}/config.h
)
add_custom_command(
    OUTPUT 
        ${CMAKE_CURRENT_BINARY_DIR}/builtins.h
        ${CMAKE_CURRENT_BINARY_DIR}/builtins.c
    COMMAND sh ${DASH_SOURCE_DIR}/mkbuiltins ${CMAKE_CURRENT_BINARY_DIR}/builtins.def
    DEPENDS ${DASH_SOURCE_DIR}/mkbuiltins ${CMAKE_CURRENT_BINARY_DIR}/builtins.def
)

set(bltin_h ${DASH_SOURCE_DIR}/bltin/bltin.h)
set(bltin_c
    ${DASH_SOURCE_DIR}/bltin/printf.c
    ${DASH_SOURCE_DIR}/bltin/test.c
    ${DASH_SOURCE_DIR}/bltin/times.c
)
add_library(bltin ${bltin_h} ${bltin_c})
target_include_directories(bltin PUBLIC ${DASH_SOURCE_DIR})

set(dash_c
    ${DASH_SOURCE_DIR}/alias.c
    ${DASH_SOURCE_DIR}/arith_yacc.c
    ${DASH_SOURCE_DIR}/arith_yylex.c
    ${DASH_SOURCE_DIR}/cd.c
    ${DASH_SOURCE_DIR}/error.c
    ${DASH_SOURCE_DIR}/eval.c
    ${DASH_SOURCE_DIR}/exec.c
    ${DASH_SOURCE_DIR}/expand.c
    ${DASH_SOURCE_DIR}/histedit.c
    ${DASH_SOURCE_DIR}/input.c
    ${DASH_SOURCE_DIR}/jobs.c
    ${DASH_SOURCE_DIR}/mail.c
    ${DASH_SOURCE_DIR}/main.c
    ${DASH_SOURCE_DIR}/memalloc.c
    ${DASH_SOURCE_DIR}/miscbltin.c
    ${DASH_SOURCE_DIR}/mystring.c
    ${DASH_SOURCE_DIR}/options.c
    ${DASH_SOURCE_DIR}/output.c
    ${DASH_SOURCE_DIR}/parser.c
    ${DASH_SOURCE_DIR}/redir.c
    ${DASH_SOURCE_DIR}/show.c
    ${DASH_SOURCE_DIR}/system.c
    ${DASH_SOURCE_DIR}/trap.c
    ${DASH_SOURCE_DIR}/var.c
)

set(dash_h
    ${DASH_SOURCE_DIR}/alias.h
    ${DASH_SOURCE_DIR}/arith_yacc.h
    ${DASH_SOURCE_DIR}/cd.h
    ${DASH_SOURCE_DIR}/error.h
    ${DASH_SOURCE_DIR}/eval.h
    ${DASH_SOURCE_DIR}/exec.h
    ${DASH_SOURCE_DIR}/expand.h
    ${DASH_SOURCE_DIR}/init.h
    ${DASH_SOURCE_DIR}/input.h
    ${DASH_SOURCE_DIR}/jobs.h
    ${DASH_SOURCE_DIR}/machdep.h
    ${DASH_SOURCE_DIR}/mail.h
    ${DASH_SOURCE_DIR}/main.h
    ${DASH_SOURCE_DIR}/memalloc.h
    ${DASH_SOURCE_DIR}/miscbltin.h
    ${DASH_SOURCE_DIR}/myhistedit.h
    ${DASH_SOURCE_DIR}/mystring.h
    ${DASH_SOURCE_DIR}/options.h
    ${DASH_SOURCE_DIR}/output.h
    ${DASH_SOURCE_DIR}/parser.h
    ${DASH_SOURCE_DIR}/redir.h
    ${DASH_SOURCE_DIR}/shell.h
    ${DASH_SOURCE_DIR}/show.h
    ${DASH_SOURCE_DIR}/system.h
    ${DASH_SOURCE_DIR}/trap.h
    ${DASH_SOURCE_DIR}/var.h
)

add_executable(mkinit ${DASH_SOURCE_DIR}/mkinit.c)
add_custom_command(
    OUTPUT  ${CMAKE_CURRENT_BINARY_DIR}/init.c
    COMMAND mkinit ${dash_c} ${bltin_c}
    DEPENDS mkinit ${dash_c} ${bltin_c}
)

set(generated
	${CMAKE_CURRENT_BINARY_DIR}/builtins.c
	${CMAKE_CURRENT_BINARY_DIR}/builtins.h
	${CMAKE_CURRENT_BINARY_DIR}/nodes.c
	${CMAKE_CURRENT_BINARY_DIR}/nodes.h
	${CMAKE_CURRENT_BINARY_DIR}/init.c
	${CMAKE_CURRENT_BINARY_DIR}/signames.c
	${CMAKE_CURRENT_BINARY_DIR}/token.h
	${CMAKE_CURRENT_BINARY_DIR}/token_vars.h
	${CMAKE_CURRENT_BINARY_DIR}/syntax.c
	${CMAKE_CURRENT_BINARY_DIR}/syntax.h
)
add_library(libdash ${dash_h} ${dash_c} ${generated})
target_include_directories(libdash PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(libdash bltin)
if(WITH_LIBEDIT)
	target_include_directories(libdash PRIVATE ${libedit_INCLUDE_DIRS})
	target_link_libraries(libdash ${libedit_LIBRARIES})
endif()

option(ENABLE_BINARY "Build dash binary" OFF)
if(ENABLE_BINARY)
	add_executable(dash ${dash_h} ${dash_c} ${generated})
	target_compile_definitions(dash PRIVATE MAIN)
	target_include_directories(dash PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
	target_link_libraries(dash bltin)
	if(WITH_LIBEDIT)
		target_include_directories(dash PRIVATE ${libedit_INCLUDE_DIRS})
		target_link_libraries(dash ${libedit_LIBRARIES})
	endif()
endif()

