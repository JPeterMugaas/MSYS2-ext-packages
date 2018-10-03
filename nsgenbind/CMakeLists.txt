cmake_minimum_required (VERSION 2.8.4)
project (nsgenbind)
include(GNUInstallDirs)
include(FindPkgConfig)
include(FindFLEX)
include(FindBISON)
enable_testing()
include( CTest )

#Obligitory version and copyright for pkg-config file and version info
#as well as possibly, automated installers.
set(nsgenbind_COMPONENT_NAME "nsgenbind")
set(nsgenbind_AUTHOR "John-Mark Bell <jmb@netsurf-browser.org>")
set(nsgenbind_COPYRIGHT "Copyright 2009-2015 John-Mark Bell <jmb@netsurf-browser.org>")
set(nsgenbind_DESCRIPTION "NetSurf JavaScript binding generator")
set(nsgenbind_VERSION_MAJOR 0)
set(nsgenbind_VERSION_MINOR 2)
set(nsgenbind_VERSION_PATCH 1)
set(nsgenbind_SOVERSION ${nsgenbind_VERSION_MAJOR}.${nsgenbind_VERSION_MINOR}.${nsgenbind_VERSION_PATCH})

if(CMAKE_COMPILER_IS_GNUCC)
set(nsgenbind_WARNFLAGS -Wall -W -Wundef -Wpointer-arith -Wcast-align
        -Wwrite-strings -Wstrict-prototypes -Wmissing-prototypes 
        -Wmissing-declarations -Wnested-externs)
endif(CMAKE_COMPILER_IS_GNUCC)


set(nsgenbind_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})

# Bison 3.0 and later require api.prefix in curly braces
# Bison 2.6 and later require api.prefix
# Bison 2.5 and earlier require name-prefix switch
  string(REPLACE "." ";" NSGENBIND_VERSION_LIST ${BISON_VERSION})
  list(GET NSGENBIND_VERSION_LIST 0 NSGENBIND_BISON_VERSION_MAJOR)
  list(GET NSGENBIND_VERSION_LIST 1 NSGENBIND_BISON_VERSION_MINOR)
  list(LENGTH NSGENBIND_VERSION_LIST NSGENBIND_VERLEN)
  if(NSGENBIND_VERLEN GREATER 2)
    list(GET NSGENBIND_VERSION_LIST 2 NSGENBIND_BISON_VERSION_PATCH)
  else()
    set( NSGENBIND_BISON_VERSION_PATCH 0)
  endif(NSGENBIND_VERLEN GREATER 2)

foreach(NSGENBIND_YACC nsgenbind webidl)
  if(NSGENBIND_BISON_VERSION_MAJOR EQUAL 1)
    set(NSGENBIND_BISON_DEFINES "--name-prefix=${NSGENBIND_YACC}_")
  else()
    if(NSGENBIND_BISON_VERSION_MAJOR EQUAL 2)
      if(NSGENBIND_BISON_VERSION_MINOR LESS 6)
        set(NSGENBIND_BISON_DEFINES "--name-prefix=${NSGENBIND_YACC}_")
      else()
        set(NSGENBIND_BISON_DEFINES "--define=api.prefix={${NSGENBIND_YACC}_}")
      endif(NSGENBIND_BISON_VERSION_MINOR LESS 6)
    else()
      set(NSGENBIND_BISON_DEFINES "--define=api.prefix={${NSGENBIND_YACC}_}")
    endif(NSGENBIND_BISON_VERSION_MAJOR EQUAL 2)
  endif(NSGENBIND_BISON_VERSION_MAJOR EQUAL 1)
  execute_process(
    COMMAND ${FLEX_EXECUTABLE} --outfile=${CMAKE_CURRENT_BINARY_DIR}/${NSGENBIND_YACC}-lexer.c --header-file=${CMAKE_CURRENT_BINARY_DIR}/${NSGENBIND_YACC}-lexer.h ${CMAKE_SOURCE_DIR}/src/${NSGENBIND_YACC}-lexer.l)
  execute_process(OUTPUT_VARIABLE OUT_VAR ERROR_VARIABLE ERR_VAR RESULT_VARIABLE RES_VAR
    COMMAND ${BISON_EXECUTABLE} -d -t ${NSGENBIND_BISON_DEFINES} --report=all --output=${CMAKE_CURRENT_BINARY_DIR}/${NSGENBIND_YACC}-parser.c --defines=${CMAKE_CURRENT_BINARY_DIR}/${NSGENBIND_YACC}-parser.h ${CMAKE_SOURCE_DIR}/src/${NSGENBIND_YACC}-parser.y)
endforeach(NSGENBIND_YACC)

set(nsgenbind_ADDITIONAL_C_FILES 
   ${CMAKE_SOURCE_DIR}/src/nsgenbind.c 
   ${CMAKE_SOURCE_DIR}/src/utils.c 
   ${CMAKE_SOURCE_DIR}/src/webidl-ast.c 
   ${CMAKE_SOURCE_DIR}/src/nsgenbind-ast.c 
   ${CMAKE_SOURCE_DIR}/src/ir.c
   ${CMAKE_SOURCE_DIR}/src/duk-libdom.c 
   ${CMAKE_SOURCE_DIR}/src/duk-libdom-interface.c 
   ${CMAKE_SOURCE_DIR}/src/duk-libdom-dictionary.c
   ${CMAKE_SOURCE_DIR}/src/duk-libdom-common.c 
   ${CMAKE_SOURCE_DIR}/src/duk-libdom-generated.c
   ${CMAKE_CURRENT_BINARY_DIR}/nsgenbind-lexer.c
   ${CMAKE_CURRENT_BINARY_DIR}/nsgenbind-lexer.h
   ${CMAKE_CURRENT_BINARY_DIR}/nsgenbind-parser.c
   ${CMAKE_CURRENT_BINARY_DIR}/nsgenbind-parser.h
   ${CMAKE_CURRENT_BINARY_DIR}/webidl-lexer.c
   ${CMAKE_CURRENT_BINARY_DIR}/webidl-lexer.h
   ${CMAKE_CURRENT_BINARY_DIR}/webidl-parser.c
   ${CMAKE_CURRENT_BINARY_DIR}/webidl-parser.h
   )

include_directories(${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/include)
# setup any additional libs required by this.
set(nsgenbind_ADDITIONAL_LIBS "")
set(nsgenbind_ADDITIONAL_DIRS "")
set(nsgenbind_ADDITIONAL_DEFS "")

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

   if(CMAKE_RC_COMPILER)
      # Make .rc file  
      set(nsgenbind_RC_CONTENTS "1 VERSIONINFO
FILEVERSION     ${nsgenbind_VERSION_MAJOR},${nsgenbind_VERSION_MINOR},${nsgenbind_VERSION_PATCH},0
PRODUCTVERSION  ${nsgenbind_VERSION_MAJOR},${nsgenbind_VERSION_MINOR},${nsgenbind_VERSION_PATCH},0
BEGIN
  BLOCK \"StringFileInfo\"
  BEGIN
    BLOCK \"040904E4\"
    BEGIN
      VALUE \"CompanyName\", \"${nsgenbind_AUTHOR}\"
      VALUE \"FileDescription\", \"${nsgenbind_DESCRIPTION}\"
      VALUE \"FileVersion\", \"${nsgenbind_VERSION_MAJOR}.${nsgenbind_VERSION_MINOR}.${nsgenbind_VERSION_PATCH}\"
      VALUE \"InternalName\", \"${nsgenbind_COMPONENT_NAME}\"
      VALUE \"LegalCopyright\", \"${nsgenbind_COPYRIGHT}\"
      VALUE \"OriginalFilename\", \"${nsgenbind_COMPONENT_NAME}${CMAKE_EXECUTABLE_SUFFIX}\"
      VALUE \"ProductName\", \"${nsgenbind_COMPONENT_NAME}\"
      VALUE \"ProductVersion\", \"${nsgenbind_VERSION_MAJOR}.${nsgenbind_VERSION_MINOR}.${nsgenbind_VERSION_PATCH}\"
    END
  END
  BLOCK \"VarFileInfo\"
  BEGIN
    VALUE \"Translation\", 0x409, 1252
  END
END")
    FILE(WRITE ${nsgenbind_BUILD_DIR}/lib${nsgenbind_COMPONENT_NAME}.rc "${nsgenbind_RC_CONTENTS}")
    add_executable(nsgenbind_exe ${nsgenbind_ADDITIONAL_C_FILES} ${nsgenbind_BUILD_DIR}/lib${nsgenbind_COMPONENT_NAME}.rc)
  else()
    add_executable(nsgenbind_exe ${nsgenbind_ADDITIONAL_C_FILES} SOVERSION ${nsgenbind_SOVERSION})
  endif()
  set_target_properties(nsgenbind_exe PROPERTIES OUTPUT_NAME ${nsgenbind_COMPONENT_NAME})
  if(nsgenbind_ADDITIONAL_LIBS)
    target_link_libraries(nsgenbind_exe PUBLIC ${nsgenbind_ADDITIONAL_LIBS}) 
  endif(nsgenbind_ADDITIONAL_LIBS)
  if(nsgenbind_ADDITIONAL_DIRS)
    target_include_directories(nsgenbind_exe PRIVATE ${nsgenbind_ADDITIONAL_DIRS})
  endif(nsgenbind_ADDITIONAL_DIRS)
  target_compile_definitions(nsgenbind_exe PRIVATE "YYENABLE_NLS=0" "_BSD_SOURCE" "_DEFAULT_SOURCE" "_POSIX_C_SOURCE=200809L")
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(nsgenbind_exe PRIVATE ${nsgenbind_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS nsgenbind_exe
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)

