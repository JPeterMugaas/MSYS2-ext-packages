cmake_minimum_required (VERSION 3.0.0)
project (libwapcaplet)
include(GNUInstallDirs)
include(FindPkgConfig)
enable_testing()
include( CTest )

#Obligitory version and copyright for pkg-config file and version info
#as well as possibly, automated installers.
set(WAPCAPLET_COMPONENT_NAME "wapcaplet")
set(WAPCAPLET_AUTHOR "John-Mark Bell <jmb@netsurf-browser.org>")
set(WAPCAPLET_COPYRIGHT "Copyright 2009-2015 John-Mark Bell <jmb@netsurf-browser.org>")
set(WAPCAPLET_DESCRIPTION "String internalisation dictionary")
set(WAPCAPLET_VERSION_MAJOR 0)
set(WAPCAPLET_VERSION_MINOR 4)
set(WAPCAPLET_VERSION_PATCH 1)
set(WAPCAPLET_SOVERSION ${WAPCAPLET_VERSION_MAJOR}.${WAPCAPLET_VERSION_MINOR}.${WAPCAPLET_VERSION_PATCH})

#Do not prefix BUILD_SHARED_LIBS and BUILD_STATIC_LIBS because those are standard cmake options
#We only put them in the menu for convenience.
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(WAPCAPLET_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)
if(CMAKE_COMPILER_IS_GNUCC)
set(WAPCAPLET_WARNFLAGS -Wall -W -Wundef -Wpointer-arith -Wcast-align
        -Wwrite-strings -Wstrict-prototypes -Wmissing-prototypes 
        -Wmissing-declarations -Wnested-externs)
endif(CMAKE_COMPILER_IS_GNUCC)

set(WAPCAPLET_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(WAPCAPLET_ADDITIONAL_C_FILES
   ${CMAKE_SOURCE_DIR}/src/libwapcaplet.c)
set(WAPCAPLET_PUBLIC_HEADER_FILES
   include/libwapcaplet/libwapcaplet.h)
include_directories(${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/include)
# setup any additional libs required by this.
set(WAPCAPLET_ADDITIONAL_LIBS "")
set(WAPCAPLET_ADDITIONAL_DIRS "")
set(WAPCAPLET_ADDITIONAL_DEFS "")

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
      set(WAPCAPLETRC_CONTENTS "1 VERSIONINFO
FILEVERSION     ${WAPCAPLET_VERSION_MAJOR},${WAPCAPLET_VERSION_MINOR},${WAPCAPLET_VERSION_PATCH},0
PRODUCTVERSION  ${WAPCAPLET_VERSION_MAJOR},${WAPCAPLET_VERSION_MINOR},${WAPCAPLET_VERSION_PATCH},0
BEGIN
  BLOCK \"StringFileInfo\"
  BEGIN
    BLOCK \"040904E4\"
    BEGIN
      VALUE \"CompanyName\", \"${WAPCAPLET_AUTHOR}\"
      VALUE \"FileDescription\", \"${WAPCAPLET_DESCRIPTION}\"
      VALUE \"FileVersion\", \"${WAPCAPLET_VERSION_MAJOR}.${WAPCAPLET_VERSION_MINOR}.${WAPCAPLET_VERSION_PATCH}\"
      VALUE \"InternalName\", \"${CMAKE_SHARED_MODULE_PREFIX}${WAPCAPLET_COMPONENT_NAME}\"
      VALUE \"LegalCopyright\", \"${WAPCAPLET_COPYRIGHT}\"
      VALUE \"OriginalFilename\", \"${CMAKE_SHARED_MODULE_PREFIX}${WAPCAPLET_COMPONENT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}\"
      VALUE \"ProductName\", \"${WAPCAPLET_COMPONENT_NAME}\"
      VALUE \"ProductVersion\", \"${WAPCAPLET_VERSION_MAJOR}.${WAPCAPLET_VERSION_MINOR}.${WAPCAPLET_VERSION_PATCH}\"
    END
  END
  BLOCK \"VarFileInfo\"
  BEGIN
    VALUE \"Translation\", 0x409, 1252
  END
END")
    FILE(WRITE ${WAPCAPLET_BUILD_DIR}/lib${WAPCAPLET_COMPONENT_NAME}.rc "${WAPCAPLETRC_CONTENTS}")
    add_library(libwapcatlet_shared SHARED ${WAPCAPLET_PUBLIC_HEADER_FILES} ${WAPCAPLET_ADDITIONAL_C_FILES} ${WAPCAPLET_BUILD_DIR}/lib${WAPCAPLET_COMPONENT_NAME}.rc)
  else()
    add_library(libwapcatlet_shared SHARED ${WAPCAPLET_PUBLIC_HEADER_FILES} ${WAPCAPLET_ADDITIONAL_C_FILES} SOVERSION ${WAPCAPLET_SOVERSION})
  endif()
  set_target_properties(libwapcatlet_shared PROPERTIES OUTPUT_NAME ${WAPCAPLET_COMPONENT_NAME})
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libwapcatlet_shared PRIVATE ${WAPCAPLET_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  if(WAPCAPLET_ADDITIONAL_LIBS)
    target_link_libraries(libwapcatlet_shared PUBLIC ${WAPCAPLET_ADDITIONAL_LIBS}) 
  endif(WAPCAPLET_ADDITIONAL_LIBS)
  if(WAPCAPLET_ADDITIONAL_DIRS)
    target_include_directories(libwapcatlet_shared PUBLIC ${WAPCAPLET_ADDITIONAL_DIRS})
  endif(WAPCAPLET_ADDITIONAL_DIRS)
  if(WAPCAPLET_ADDITIONAL_DEFS)
    target_include_directories(libwapcatlet_shared PUBLIC ${WAPCAPLET_ADDITIONAL_DEFS})
  endif(WAPCAPLET_ADDITIONAL_DEFS)
  install(TARGETS libwapcatlet_shared
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(libwapcatlet_static STATIC ${WAPCAPLETPUBLIC_HEADER_FILES} ${WAPCAPLET_ADDITIONAL_C_FILES})
  set_target_properties(libwapcatlet_static PROPERTIES OUTPUT_NAME ${WAPCAPLET_COMPONENT_NAME})
  if(WAPCAPLET_ADDITIONAL_LIBS)
    target_link_libraries(libwapcatlet_static PUBLIC ${WAPCAPLET_ADDITIONAL_LIBS}) 
  endif(WAPCAPLET_ADDITIONAL_LIBS)
  if(WAPCAPLET_ADDITIONAL_DIRS)
    target_include_directories(libwapcatlet_static PUBLIC ${WAPCAPLET_ADDITIONAL_DIRS})
  endif(WAPCAPLET_ADDITIONAL_DIRS)
  if(WAPCAPLET_ADDITIONAL_DEFS)
    target_include_directories(libwapcatlet_static PUBLIC ${WAPCAPLET_ADDITIONAL_DEFS})
  endif(WAPCAPLET_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libwapcatlet_static PRIVATE ${WAPCAPLET_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libwapcatlet_static
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_STATIC_LIBS)

#from: https://cmake.org/pipermail/cmake/2010-October/040246.html
#This installs the headers into our hierarchy in the include file.
foreach(WAPCAPLET_HEADER ${WAPCAPLET_PUBLIC_HEADER_FILES})
   string(REGEX MATCH "(.*)[/\\]" WAPCAPLETDIR ${WAPCAPLET_HEADER})
   install(FILES ${WAPCAPLET_HEADER} DESTINATION ${WAPCAPLETDIR})
endforeach(WAPCAPLET_HEADER ${WAPCAPLET_PUBLIC_HEADER_FILES})

if(WAPCAPLET_WITH_PKGCONFIG_SUPPORT)
  set(WAPCAPLET_PC ${WAPCAPLET_BUILD_DIR}/lib${WAPCAPLET_COMPONENT_NAME}.pc)

  # This stuff is necessary to ensure that the dependency list
  # in our .pc file is in the proper format
#Make pkg-config
  set(WAPCAPLET_PC_EXC "${CMAKE_INSTALL_PREFIX}")
  set(WAPCAPLET_PC_INC "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(WAPCAPLET_PC_LIB "${CMAKE_INSTALL_FULL_LIBDIR}")
  if("${WAPCAPLET_PC_EXC}" STREQUAL "${CMAKE_INSTALL_PREFIX}")
    set(WAPCAPLET_PC_EXC "\${prefix}")
  endif()
  if ("${WAPCAPLET_PC_INC}" STREQUAL "${CMAKE_INSTALL_PREFIX}/include")
    set(WAPCAPLET_PC_INC "\${prefix}/include")
  endif()
  if ("${WAPCAPLET_PC_LIB}" STREQUAL "${CMAKE_INSTALL_PREFIX}/lib")
    set(WAPCAPLET_PC_LIB "\${exec_prefix}/lib")
  endif()
  set(WAPCAPLET_PKGCONF_WAPCAPLETDEPS "")
  foreach(WAPCAPLETLIB_DEP ${WAPCAPLETLIB_DEPEND_LIBS})
    set(WAPCAPLET_PKGCONF_WAPCAPLETDEPS "${WAPCAPLET_PKGCONF_WAPCAPLETDEPS} -l${WAPCAPLETLIB_DEP}")
  endforeach(WAPCAPLETLIB_DEP)

  set(WAPCAPLET_PKGCONFIG_CONTENTS "prefix=${CMAKE_INSTALL_PREFIX}
exec_prefix=${WAPCAPLET_PC_EXC}
libdir=${WAPCAPLET_PC_LIB}
includedir=${WAPCAPLET_PC_INC}

Name: lib${WAPCAPLET_COMPONENT_NAME}
Description: ${WAPCAPLET_DESCRIPTION}
URL: http://www.netsurf-browser.org/projects/${WAPCAPLET_COMPONENT_NAME}/
Version: ${WAPCAPLET_SOVERSION}
Requires: libparserutils
Cflags: -I\${includedir}
Libs: -L\${libdir} -l${WAPCAPLET_COMPONENT_NAME}
Libs.private: ${WAPCAPLET_PKGCONF_WAPCAPLETDEPS}
")
  file(WRITE ${WAPCAPLET_PC} ${WAPCAPLET_PKGCONFIG_CONTENTS})
  install(FILES ${WAPCAPLET_PC} DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif(WAPCAPLET_WITH_PKGCONFIG_SUPPORT)

#
#Test section
#
#pkg_check_modules
if (BUILD_TESTING)
  if(PKG_CONFIG_FOUND)
     pkg_check_modules(check check)
  endif(PKG_CONFIG_FOUND)
  if (check_FOUND)
    if(BUILD_SHARED_LIBS)
      set(WAPCAPLET_WITH_TESTCFLAGS ${check_CFLAGS})
      set(WAPCAPLET_WITH_LIBRARIES ${check_LIBRARIES})
      add_executable(testmain_shared ${CMAKE_SOURCE_DIR}/test/testmain.c ${CMAKE_SOURCE_DIR}/test/basictests.c)
      target_link_libraries(testmain_shared libwapcatlet_shared ${check_LIBRARIES})
      add_test(NAME testrunner_shared COMMAND testmain_shared)
    endif(BUILD_SHARED_LIBS)
    if(BUILD_STATIC_LIBS)
      set(WAPCAPLET_WITH_TESTCFLAGS ${check_CFLAGS})
      set(WAPCAPLET_WITH_LIBRARIES ${check_LIBRARIES})
      add_executable(testmain_static ${CMAKE_SOURCE_DIR}/test/testmain.c ${CMAKE_SOURCE_DIR}/test/basictests.c)
      target_link_libraries(testmain_static libwapcatlet_shared ${check_LIBRARIES})
      add_test(NAME testrunner_static COMMAND testmain_shared)
    endif(BUILD_STATIC_LIBS)
  endif(check_FOUND)
endif(BUILD_TESTING)
