CMAKE_MINIMUM_REQUIRED(VERSION 2.8.0)

###
### FIND ADDITIONAL CMAKE MODULES
###
SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

###
### PROJECT
###
PROJECT(xolominer)

###
### CONFIG
###
IF(NOT BITNESS) # Auto-select bitness based on platform
    IF (CMAKE_SIZEOF_VOID_P EQUAL 8)
        SET(BITNESS 64)
    ELSE()
        SET(BITNESS 32)
    ENDIF()
ENDIF()

#select bitness for non-msvc platform. Can be specified as -DBITNESS=32/64 at command-line
IF(NOT MSVC)
    SET(BITNESS ${BITNESS} CACHE STRING "Specify bitness, use -DBITNESS=32/64")
    SET_PROPERTY(CACHE BITNESS PROPERTY STRINGS "64" "32")
endif()
#unset OPENCL_LIBRARIES, so that corresponding arch specific libs are found when bitness is changed
UNSET(OPENCL_LIBRARIES CACHE)

IF(BITNESS EQUAL 64)
    SET(BITNESS_SUFFIX x86_64)
ELSEIF(BITNESS EQUAL 32)
    SET(BITNESS_SUFFIX x86)
ELSE()
    MESSAGE(FATAL_ERROR "Bitness specified is invalid")
ENDIF()

#set platform
IF(NOT UNIX)
	SET(PLATFORM win)
ELSE()
	SET(PLATFORM lnx)
ENDIF()

###
### OPENCL (new)
###
#find OpenCL include and libs
FIND_PATH(OPENCL_INCLUDE_DIRS 
    NAMES OpenCL/cl.h CL/cl.h
    HINTS ../../../../include/ $ENV{AMDAPPSDKROOT}/include/
)
MARK_AS_ADVANCED(OPENCL_INCLUDE_DIRS)

FIND_LIBRARY(OPENCL_LIBRARIES
	NAMES OpenCL
	HINTS ../../../../lib/ $ENV{AMDAPPSDKROOT}/lib
	PATH_SUFFIXES ${PLATFORM}${BITNESS} ${BITNESS_SUFFIX}
)
MARK_AS_ADVANCED(OPENCL_LIBRARIES)

IF(OPENCL_INCLUDE_DIRS STREQUAL "" OR OPENCL_LIBRARIES STREQUAL "")
	MESSAGE(FATAL_ERROR "Could not locate OpenCL include & libs")
ELSE()
	MESSAGE("OpenCL information: CPP bindings found")
	MESSAGE("  OPENCL_INCLUDE_DIRS: ${OPENCL_INCLUDE_DIRS}") 
	MESSAGE("  OPENCL_LIBRARIES: ${OPENCL_LIBRARIES}") 
ENDIF()

#tweaks for cygwin makefile to work with windows-style path
IF(CYGWIN)
    SET(PATHS_TO_CONVERT
		OPENCL_INCLUDE_DIRS
		OPENCL_LIBRARIES
	)
       
    FOREACH(pathVar ${PATHS_TO_CONVERT})
        #convert windows paths to cyg linux absolute path
        EXECUTE_PROCESS(COMMAND cygpath -ua ${${pathVar}}
			OUTPUT_VARIABLE ${pathVar}
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)
    ENDFOREACH(pathVar)
ENDIF()

INCLUDE_DIRECTORIES(${OPENCL_INCLUDE_DIRS})

#gcc/g++ specific compile options
SET(COMPILER_FLAGS " ")
SET(LINKER_FLAGS " ")
SET(ADDITIONAL_LIBRARIES "")

IF(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
    SET( COMPILER_FLAGS "${COMPILER_FLAGS} -msse2 ")
    
    # Note: "rt" is not present on mingw
    IF(UNIX)
		#if(CMAKE_BUILD_TYPE STREQUAL "Debug")
		#	set( COMPILER_FLAGS " -g " )
		#endif( )
        SET(ADDITIONAL_LIBRARIES ${ADDITIONAL_LIBRARIES} "rt")
    ENDIF()
    
    IF(BITNESS EQUAL 32)
        SET(COMPILER_FLAGS "${COMPILER_FLAGS} -m32 ")
        SET(LINKER_FLAGS "${LINKER_FLAGS} -m32 ")
    ELSE()
        SET(COMPILER_FLAGS "${COMPILER_FLAGS} -m64 ")
        SET(LINKER_FLAGS "${LINKER_FLAGS} -m64 ")
    ENDIF()
    
    SET(COMPILER_FLAGS "${COMPILER_FLAGS} ${EXTRA_COMPILER_FLAGS_GXX} ")
    SET(LINKER_FLAGS "${LINKER_FLAGS} ${EXTRA_LINKER_FLAGS_GXX} ")
    SET(ADDITIONAL_LIBRARIES ${ADDITIONAL_LIBRARIES} ${EXTRA_LIBRARIES_GXX})
ENDIF()

###
### OPENCL (old)
###
#FIND_PACKAGE(OpenCL REQUIRED)
#INCLUDE_DIRECTORIES(${OPENCL_INCLUDE_DIRS})
#if(OPENCL_HAS_CPP_BINDINGS)
#	MESSAGE("OpenCL information: CPP bindings found")
#	MESSAGE("  OPENCL_INCLUDE_DIRS: ${OPENCL_INCLUDE_DIRS}") 
#	MESSAGE("  OPENCL_LIBRARIES: ${OPENCL_LIBRARIES}") 
#else(OPENCL_HAS_CPP_BINDINGS)
#	MESSAGE("OpenCL information: CPP bindings -NOT- found")
#endif(OPENCL_HAS_CPP_BINDINGS)

###
### OPENSSL
###
FIND_PACKAGE(OpenSSL REQUIRED)
IF(OPENSSL_FOUND)
	MESSAGE("OpenSSL information: found")
	MESSAGE("  OpenSSL_INCLUDE_DIR: ${OpenSSL_INCLUDE_DIR}") 
	MESSAGE("  OpenSSL_LIBRARIES: ${OpenSSL_LIBRARIES}") 
	INCLUDE_DIRECTORIES(${OpenSSL_INCLUDE_DIR})
ELSE(OPENSSL_FOUND)
	MESSAGE(FATAL_ERROR "OpenSSL -NOT- found" )
ENDIF(OPENSSL_FOUND)

###
### LIBGMP
###
FIND_PACKAGE(GMP REQUIRED)
IF(GMP_FOUND)
	MESSAGE("libGMP information: found")
	MESSAGE("  GMP_INCLUDE_DIR: ${GMP_INCLUDE_DIR}") 
	MESSAGE("  GMP(XX)_LIBRARIES: ${GMP_LIBRARIES} (${GMPXX_LIBRARIES})") 
	INCLUDE_DIRECTORIES(${GMP_INCLUDE_DIR})
ELSE()
	MESSAGE(FATAL_ERROR "libGMP -NOT- found" )
ENDIF()

###
### BOOST
###
SET(Boost_USE_MULTITHREADED ON)
SET(Boost_USE_STATIC_LIBS ON)
#SET(Boost_COMPILER "-gcc43") #set this to the compiler suffix used by Boost if FindBoost has problems
#SET(Boost_ADDITIONAL_VERSIONS "0.99" "0.99.0" "1.78" "1.78.0") #for future boost releases

FIND_PACKAGE(Boost 1.43.0 COMPONENTS system thread timer chrono REQUIRED)

MESSAGE("Boost information: found ${Boost_MAJOR_VERSION}.${Boost_MINOR_VERSION}.${Boost_SUBMINOR_VERSION}") 
MESSAGE("  Boost_INCLUDE_DIRS: ${Boost_INCLUDE_DIRS}") 
MESSAGE("  Boost_LIBRARY_DIRS: ${Boost_LIBRARY_DIRS}") 
MESSAGE("  Boost_LIBRARIES: ${Boost_LIBRARIES}") 

INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIRS})
LINK_DIRECTORIES(${Boost_LIBRARY_DIRS})

###
### YASM (not needed atm)
###
#IF(APPLE)
#    set(CMAKE_ASM_FLAGS "-f macho64 --prefix=_") #TODO: 32 BIT?
#ELSE(UNIX)
#    SET(CMAKE_ASM_FLAGS "-f elf64") #TODO: 32 BIT?
#ELSE()
#    SET(CMAKE_ASM_FLAGS "-f win64") #TODO: 32 BIT?
#ENDIF()
#
#SET(CMAKE_ASM_COMPILER "yasm")
#ENABLE_LANGUAGE(ASM)

###
### GENERAL SOURCE STUFF
###
INCLUDE_DIRECTORIES("src")

SET(SRC_FILES
	src/sha256.cpp
	src/primes.cpp
	src/prime.cpp
	src/main_config.cpp
	src/main_mining.cpp
	src/main_poolminer.cpp
	src/xpmclient.cpp
)

CONFIGURE_FILE(src/gpu/fermat.cl ${CMAKE_BINARY_DIR}/gpu/fermat.cl COPYONLY)
CONFIGURE_FILE(src/gpu/sha256.cl ${CMAKE_BINARY_DIR}/gpu/sha256.cl COPYONLY)
CONFIGURE_FILE(src/gpu/sieve.cl ${CMAKE_BINARY_DIR}/gpu/sieve.cl COPYONLY)

###
### EXECUTABLE
###
ADD_EXECUTABLE(xolominer
	${SRC_FILES}
)

SET_TARGET_PROPERTIES(xolominer PROPERTIES
	COMPILE_FLAGS ${COMPILER_FLAGS}
	LINK_FLAGS ${LINKER_FLAGS}
)

###
### LIBRARIES
###
#TARGET_LINK_LIBRARIES(xolominer ${Boost_SYSTEM_LIBRARY})
#TARGET_LINK_LIBRARIES(xolominer ${Boost_THREAD_LIBRARY})
#TARGET_LINK_LIBRARIES(xolominer ${Boost_TIMER_LIBRARY})
#TARGET_LINK_LIBRARIES(xolominer ${Boost_CHRONO_LIBRARY})
TARGET_LINK_LIBRARIES(xolominer ${Boost_LIBRARIES})
TARGET_LINK_LIBRARIES(xolominer ${GMP_LIBRARIES})
TARGET_LINK_LIBRARIES(xolominer ${GMPXX_LIBRARIES})
TARGET_LINK_LIBRARIES(xolominer ${OpenSSL_LIBRARIES})
TARGET_LINK_LIBRARIES(xolominer ${OPENCL_LIBRARIES})
TARGET_LINK_LIBRARIES(xolominer ${ADDITIONAL_LIBRARIES})
TARGET_LINK_LIBRARIES(xolominer "pthread")

IF(WIN32) #<MINGW>
	TARGET_LINK_LIBRARIES(xolominer "ws2_32")
	SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
	SET(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++") #TODO: boost -> libwinpthread?
#	TARGET_LINK_LIBRARIES(xolominer "mswsock") #needed? no.
#	TARGET_LINK_LIBRARIES(xolominer "shlwapi") #needed? no.
#	TARGET_LINK_LIBRARIES(xolominer "-DWIN32 -D_WINDOWS -DBOOST_THREAD_USE_LIB -DBOOST_SPIRIT_THREADSAFE -Wl,--dynamicbase -Wl,--nxcompat -static") #needed? no.
ELSEIF(UNIX)
	TARGET_LINK_LIBRARIES(xolominer "rt") # -NOT- APPLE !?!
ENDIF()

###
### COMMON FLAGS
###
#SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGLIBC_XX_DEBUG -Wall -ggdb -g3 -fno-rounding-math -msse -O0")
#SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3") # DEBUG/RELEASE flags omitted atm (for simplification)
#SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O2")
#SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DGLIBC_XX_DEBUG -Wall -ggdb -g3 -fno-rounding-math -msse -O0")
