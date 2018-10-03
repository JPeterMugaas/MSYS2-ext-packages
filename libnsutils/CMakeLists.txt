cmake_minimum_required (VERSION 2.8.4)
project (libnsutils)
include(GNUInstallDirs)
include(FindPkgConfig)
enable_testing()
include( CTest )

#Obligitory version and copyright for pkg-config file and version info
#as well as possibly, automated installers.
set(nsutils_COMPONENT_NAME "nsutils")
set(nsutils_AUTHOR "John-Mark Bell <jmb@netsurf-browser.org>")
set(nsutils_COPYRIGHT "Copyright 2009-2015 John-Mark Bell <jmb@netsurf-browser.org>")
set(nsutils_DESCRIPTION "NetSurf utilities")
set(nsutils_VERSION_MAJOR 0)
set(nsutils_VERSION_MINOR 2)
set(nsutils_VERSION_PATCH 1)
set(nsutils_SOVERSION ${nsutils_VERSION_MAJOR}.${nsutils_VERSION_MINOR}.${nsutils_VERSION_PATCH})

#Do not prefix BUILD_SHARED_LIBS and BUILD_STATIC_LIBS because those are standard cmake options
#We only put them in the menu for convenience.
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(nsutils_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)

if(CMAKE_COMPILER_IS_GNUCC)
set(nsutils_WARNFLAGS -Wall -W -Wundef -Wpointer-arith -Wcast-align
        -Wwrite-strings -Wstrict-prototypes -Wmissing-prototypes 
        -Wmissing-declarations -Wnested-externs)
endif(CMAKE_COMPILER_IS_GNUCC)

set(nsutils_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(nsutils_ADDITIONAL_C_FILES 
   ${CMAKE_SOURCE_DIR}/src/base64.c 
   ${CMAKE_SOURCE_DIR}/src/time.c
   ${CMAKE_SOURCE_DIR}/src/unistd.c)
set(nsutils_PUBLIC_HEADER_FILES
    include/nsutils/base64.h
    include/nsutils/endian.h  
    include/nsutils/errors.h
    include/nsutils/time.h 
    include/nsutils/unistd.h)

include_directories(${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/include)
# setup any additional libs required by this.
set(nsutils_ADDITIONAL_LIBS "")
set(nsutils_ADDITIONAL_DIRS "")
set(nsutils_ADDITIONAL_DEFS "_BSD_SOURCE" "_DEFAULT_SOURCE")

# From: https://gitlab.kitware.com/cmake/community/wikis/contrib/macros/TestInline
# Inspired from /usr/share/autoconf/autoconf/c.m4
# I put it here to replace the GCCISM inline="__inline__"
set(TEST_C "/* Test source lifted from /usr/share/autoconf/autoconf/c.m4 */
typedef int foo_t;
static inline foo_t static_foo(){return 0;}
foo_t foo(){return 0;}
int main(int argc, char *argv[]){return 0;}")
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/test_inline.c "${TEST_C}")
FOREACH(KEYWORD "inline" "__inline__" "__inline")
   IF(NOT DEFINED C_INLINE)
     TRY_COMPILE(C_HAS_${KEYWORD} "${CMAKE_CURRENT_BINARY_DIR}"
       "${CMAKE_CURRENT_BINARY_DIR}/test_inline.c"
       COMPILE_DEFINITIONS "-Dinline=${KEYWORD}")
     IF(C_HAS_${KEYWORD})
       SET(C_INLINE TRUE)
       ADD_DEFINITIONS("-Dinline=${KEYWORD}")
     ENDIF(C_HAS_${KEYWORD})
   ENDIF(NOT DEFINED C_INLINE)
ENDFOREACH(KEYWORD)
IF(NOT DEFINED C_INLINE)
   ADD_DEFINITIONS("-Dinline=")
ENDIF(NOT DEFINED C_INLINE)
file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/test_inline.c)

if(BUILD_SHARED_LIBS)
   if(CMAKE_RC_COMPILER)
      # Make .rc file  
      set(nsutils_RC_CONTENTS "1 VERSIONINFO
FILEVERSION     ${nsutils_VERSION_MAJOR},${nsutils_VERSION_MINOR},${nsutils_VERSION_PATCH},0
PRODUCTVERSION  ${nsutils_VERSION_MAJOR},${nsutils_VERSION_MINOR},${nsutils_VERSION_PATCH},0
BEGIN
  BLOCK \"StringFileInfo\"
  BEGIN
    BLOCK \"040904E4\"
    BEGIN
      VALUE \"CompanyName\", \"${nsutils_AUTHOR}\"
      VALUE \"FileDescription\", \"${nsutils_DESCRIPTION}\"
      VALUE \"FileVersion\", \"${nsutils_VERSION_MAJOR}.${nsutils_VERSION_MINOR}.${nsutils_VERSION_PATCH}\"
      VALUE \"InternalName\", \"${CMAKE_SHARED_MODULE_PREFIX}${nsutils_COMPONENT_NAME}\"
      VALUE \"LegalCopyright\", \"${nsutils_COPYRIGHT}\"
      VALUE \"OriginalFilename\", \"${CMAKE_SHARED_MODULE_PREFIX}${nsutils_COMPONENT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}\"
      VALUE \"ProductName\", \"${nsutils_COMPONENT_NAME}\"
      VALUE \"ProductVersion\", \"${nsutils_VERSION_MAJOR}.${nsutils_VERSION_MINOR}.${nsutils_VERSION_PATCH}\"
    END
  END
  BLOCK \"VarFileInfo\"
  BEGIN
    VALUE \"Translation\", 0x409, 1252
  END
END")
    FILE(WRITE ${nsutils_BUILD_DIR}/lib${nsutils_COMPONENT_NAME}.rc "${nsutils_RC_CONTENTS}")
    add_library(libnsutils_shared SHARED ${nsutils_PUBLIC_HEADER_FILES} ${nsutils_ADDITIONAL_C_FILES} ${nsutils_BUILD_DIR}/lib${nsutils_COMPONENT_NAME}.rc)
  else()
    add_library(libnsutils_shared SHARED ${nsutils_PUBLIC_HEADER_FILES} ${nsutils_ADDITIONAL_C_FILES} SOVERSION ${nsutils_SOVERSION})
  endif()
  set_target_properties(libnsutils_shared PROPERTIES OUTPUT_NAME ${nsutils_COMPONENT_NAME})
  if(nsutils_ADDITIONAL_LIBS)
    target_link_libraries(libnsutils_shared PUBLIC ${nsutils_ADDITIONAL_LIBS}) 
  endif(nsutils_ADDITIONAL_LIBS)
  if(nsutils_ADDITIONAL_DIRS)
    target_include_directories(libnsutils_shared PUBLIC ${nsutils_ADDITIONAL_DIRS})
  endif(nsutils_ADDITIONAL_DIRS)
  if(nsutils_ADDITIONAL_DEFS)
    target_include_directories(libnsutils_shared PUBLIC ${nsutils_ADDITIONAL_DEFS})
  endif(nsutils_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libnsutils_shared PRIVATE ${nsutils_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libnsutils_shared
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(libnsutils_static STATIC ${nsutils_PUBLIC_HEADER_FILES} ${nsutils_ADDITIONAL_C_FILES})
  set_target_properties(libnsutils_static PROPERTIES OUTPUT_NAME ${nsutils_COMPONENT_NAME})
  if(nsutils_ADDITIONAL_LIBS)
    target_link_libraries(libnsutils_static PUBLIC ${nsutils_ADDITIONAL_LIBS}) 
  endif(nsutils_ADDITIONAL_LIBS)
  if(nsutils_ADDITIONAL_DIRS)
    target_include_directories(libnsutils_static PUBLIC ${nsutils_ADDITIONAL_DIRS})
  endif(nsutils_ADDITIONAL_DIRS)
  if(nsutils_ADDITIONAL_DEFS)
    target_include_directories(libnsutils_static PUBLIC ${nsutils_ADDITIONAL_DEFS})
  endif(nsutils_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libnsutils_static PRIVATE ${nsutils_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libnsutils_static
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_STATIC_LIBS)

#from: https://cmake.org/pipermail/cmake/2010-October/040246.html
#This installs the headers into our hierarchy in the include file.
foreach(nsutils_HEADER ${nsutils_PUBLIC_HEADER_FILES})
   string(REGEX MATCH "(.*)[/\\]" nsutils_DIR ${nsutils_HEADER})
   install(FILES ${nsutils_HEADER} DESTINATION ${nsutils_DIR})
endforeach(nsutils_HEADER ${nsutils_PUBLIC_HEADER_FILES})

if(nsutils_WITH_PKGCONFIG_SUPPORT)
  set(nsutils_PC ${nsutils_BUILD_DIR}/lib${nsutils_COMPONENT_NAME}.pc)

  # This stuff is necessary to ensure that the dependency list
  # in our .pc file is in the proper format
#Make pkg-config
  set(nsutils_EXC "${CMAKE_INSTALL_PREFIX}")
  set(nsutils_INC "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(nsutils_LIB "${CMAKE_INSTALL_FULL_LIBDIR}")
  if("${nsutils_EXC}" STREQUAL "${CMAKE_INSTALL_PREFIX}")
    set(nsutils_EXC "\${prefix}")
  endif()
  if ("${nsutils_INC}" STREQUAL "${CMAKE_INSTALL_PREFIX}/include")
    set(nsutils_INC "\${prefix}/include")
  endif()
  if ("${nsutils_LIB}" STREQUAL "${CMAKE_INSTALL_PREFIX}/lib")
    set(nsutils_LIB "\${exec_prefix}/lib")
  endif()
  set(nsutils_DEPS "")
  foreach(nsutils_LIB_DEP ${nsutils_LIB_DEPEND_LIBS})
    set(nsutils_DEPS "${nsutils_DEPS} -l${nsutils_LIB_DEP}")
  endforeach(nsutils_LIB_DEP)

  set(nsutils_PKGCONFIG_CONTENTS "prefix=${CMAKE_INSTALL_PREFIX}
exec_prefix=${nsutils_EXC}
libdir=${nsutils_LIB}
includedir=${nsutils_INC}

Name: lib${nsutils_COMPONENT_NAME}
Description: ${nsutils_DESCRIPTION}
URL: http://www.netsurf-browser.org/projects/${nsutils_COMPONENT_NAME}/
Version: ${nsutils_SOVERSION}
Cflags: -I\${includedir}
Libs: -L\${libdir} -l${nsutils_COMPONENT_NAME}
Libs.private: ${nsutils_DEPS}
")
  file(WRITE ${nsutils_PC} ${nsutils_PKGCONFIG_CONTENTS})
  install(FILES ${nsutils_PC} DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif(nsutils_WITH_PKGCONFIG_SUPPORT)

#test section
if(BUILD_TESTING)
  if(BUILD_SHARED_LIBS)
    add_executable(test_base64_shared ${CMAKE_SOURCE_DIR}/test/base64.c)
    target_link_libraries(test_base64_shared PRIVATE libnsutils_shared) 
#    add_test(NAME 
#      COMMAND test_base64_shared
  endif(BUILD_SHARED_LIBS)
  if(BUILD_SHARED_LIBS)
    add_executable(test_base64_static ${CMAKE_SOURCE_DIR}/test/base64.c)
    target_link_libraries(test_base64_static PRIVATE libnsutils_static) 
#    add_test(NAME 
#      COMMAND test_base64_shared
  endif(BUILD_SHARED_LIBS)

endif(BUILD_TESTING)

