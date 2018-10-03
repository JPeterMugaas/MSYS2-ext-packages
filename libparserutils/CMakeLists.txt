cmake_minimum_required (VERSION 2.8.4)
project (libparserutils)
include(FindPerl)
#FindIconv module was introduced in CMake 3.11
IF (${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 3.10)
  include(FindIconv)
endif()
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
enable_testing()
include( CTest )

#Obligitory version and copyright for pkg-config file and version info
#as well as possibly, automated installers.
set(PARSEUTILS_COMPONENT_NAME "parserutils")
set(PARSEUTILS_AUTHOR "John-Mark Bell <jmb@netsurf-browser.org>")
set(PARSEUTILS_COPYRIGHT "Copyright 2009-2015 John-Mark Bell <jmb@netsurf-browser.org>")
set(PARSEUTILS_DESCRIPTION "Utility library for facilitating parser development")
set(PARSEUTILS_VERSION_MAJOR 0)
set(PARSEUTILS_VERSION_MINOR 2)
set(PARSEUTILS_VERSION_PATCH 4)
set(PARSEUTILS_SOVERSION ${PARSEUTILS_VERSION_MAJOR}.${PARSEUTILS_VERSION_MINOR}.${PARSEUTILS_VERSION_PATCH})

#Do not prefix BUILD_SHARED_LIBS and BUILD_STATIC_LIBS because those are standard cmake options
#We only put them in the menu for convenience.
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(PARSEUTILS_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)
option(PARSEUTILS_BUILD_DOCS "Build Docs with Doxygen" OFF)

if(NOT PERL_FOUND)
  message(FATAL_ERROR "Perl is required")
endif()
set(PARSEUTILS_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(PARSEUTILS_ADDITIONAL_C_FILES
  ${CMAKE_SOURCE_DIR}/src/charset/aliases.h
  ${CMAKE_SOURCE_DIR}/src/charset/aliases.c  
  ${CMAKE_SOURCE_DIR}/src/charset/codec.c
  ${CMAKE_SOURCE_DIR}/src/charset/codecs/codec_impl.h
  ${CMAKE_SOURCE_DIR}/src/charset/codecs/codec_ascii.c
  ${CMAKE_SOURCE_DIR}/src/charset/codecs/codec_8859.c
  ${CMAKE_SOURCE_DIR}/src/charset/codecs/codec_ext8.c
  ${CMAKE_SOURCE_DIR}/src/charset/codecs/codec_utf8.c
  ${CMAKE_SOURCE_DIR}/src/charset/codecs/codec_utf16.c
  ${CMAKE_SOURCE_DIR}/src/charset/encodings/utf8.c
  ${CMAKE_SOURCE_DIR}/src/charset/encodings/utf16.c
  ${CMAKE_SOURCE_DIR}/src/input/filter.c
  ${CMAKE_SOURCE_DIR}/src/input/inputstream.c
  ${CMAKE_SOURCE_DIR}/src/utils/buffer.c
  ${CMAKE_SOURCE_DIR}/src/utils/errors.c
  ${CMAKE_SOURCE_DIR}/src/utils/stack.c
  ${CMAKE_SOURCE_DIR}/src/utils/vector.c
)
set(PARSEUTILS_PUBLIC_HEADER_FILES
  include/parserutils/errors.h 
  include/parserutils/functypes.h
  include/parserutils/parserutils.h
  include/parserutils/types.h
  include/parserutils/charset/codec.h
  include/parserutils/charset/mibenum.h
  include/parserutils/charset/utf16.h
  include/parserutils/charset/utf8.h
  include/parserutils/input/inputstream.h
  include/parserutils/utils/buffer.h
  include/parserutils/utils/stack.h
  include/parserutils/utils/vector.h)

if(CMAKE_COMPILER_IS_GNUCC)
  set(PARSEUTILS_WARNFLAGS -Wall -W -Wundef -Wpointer-arith -Wcast-align 
        -Wwrite-strings -Wstrict-prototypes -Wmissing-prototypes
        -Wmissing-declarations -Wnested-externs -pedantic)
endif(CMAKE_COMPILER_IS_GNUCC)

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

execute_process(WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
COMMAND ${PERL_EXECUTABLE} ${CMAKE_SOURCE_DIR}/build/make-aliases.pl ${CMAKE_SOURCE_DIR}/build/Aliases)
include_directories(${CMAKE_SOURCE_DIR}/src)
# setup any additional libs required by this.
set(PARSEUTILS_ADDITIONAL_LIBS "")
set(PARSEUTILS_ADDITIONAL_DIRS "")
set(PARSEUTILS_ADDITIONAL_DEFS "")
if (Iconv_FOUND)
   list(APPEND PARSEUTILS_ADDITIONAL_DIRS ${Iconv_INCLUDE_DIRS})
   list(APPEND PARSEUTILS_ADDITIONAL_LIBS ${Iconv_LIBRARIES})
else()
  list(APPEND PARSEUTILS_ADDITIONAL_DEFS "WITHOUT_ICONV_FILTER")
endif(Iconv_FOUND)

if(BUILD_SHARED_LIBS)
   if(CMAKE_RC_COMPILER)
  # Make .rc file  
  		set(PARSEUTILS_RC_CONTENTS "1 VERSIONINFO
FILEVERSION     ${PARSEUTILS_VERSION_MAJOR},${PARSEUTILS_VERSION_MINOR},${PARSEUTILS_VERSION_PATCH},0
PRODUCTVERSION  ${PARSEUTILS_VERSION_MAJOR},${PARSEUTILS_VERSION_MINOR},${PARSEUTILS_VERSION_PATCH},0
BEGIN
  BLOCK \"StringFileInfo\"
  BEGIN
    BLOCK \"040904E4\"
    BEGIN
      VALUE \"CompanyName\", \"${PARSEUTILS_AUTHOR}\"
      VALUE \"FileDescription\", \"${PARSEUTILS_DESCRIPTION}\"
      VALUE \"FileVersion\", \"${PARSEUTILS_VERSION_MAJOR}.${PARSEUTILS_VERSION_MINOR}.${PARSEUTILS_VERSION_PATCH}\"
      VALUE \"InternalName\", \"${CMAKE_SHARED_MODULE_PREFIX}${PARSEUTILS_COMPONENT_NAME}\"
      VALUE \"LegalCopyright\", \"${PARSEUTILS_COPYRIGHT}\"
      VALUE \"OriginalFilename\", \"${CMAKE_SHARED_MODULE_PREFIX}${PARSEUTILS_COMPONENT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}\"
      VALUE \"ProductName\", \"${PARSEUTILS_COMPONENT_NAME}\"
      VALUE \"ProductVersion\", \"${PARSEUTILS_VERSION_MAJOR}.${PARSEUTILS_VERSION_MINOR}.${PARSEUTILS_VERSION_PATCH}\"
    END
  END
  BLOCK \"VarFileInfo\"
  BEGIN
    VALUE \"Translation\", 0x409, 1252
  END
END")
    FILE(WRITE ${PARSEUTILS_BUILD_DIR}/lib${PARSEUTILS_COMPONENT_NAME}.rc "${PARSEUTILS_RC_CONTENTS}")
    add_library(libparserutils_shared SHARED ${PARSEUTILS_ADDITIONAL_C_FILES} ${PARSEUTILS_PUBLIC_HEADER_FILES} lib${PARSEUTILS_COMPONENT_NAME}.rc)
  else()
    add_library(libparserutils_shared SHARED ${PARSEUTILS_ADDITIONAL_C_FILES} ${PARSEUTILS_PUBLIC_HEADER_FILES} SOVERSION ${PARSEUTILS_SOVERSION})
  endif()
  target_include_directories(libparserutils_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libparserutils_shared PRIVATE ${PARSEUTILS_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  set_target_properties(libparserutils_shared PROPERTIES OUTPUT_NAME ${PARSEUTILS_COMPONENT_NAME})
  if(PARSEUTILS_ADDITIONAL_LIBS)
    target_link_libraries(libparserutils_shared PUBLIC ${PARSEUTILS_ADDITIONAL_LIBS}) 
  endif(PARSEUTILS_ADDITIONAL_LIBS)
  if(PARSEUTILS_ADDITIONAL_DIRS)
    target_include_directories(libparserutils_shared PUBLIC ${PARSEUTILS_ADDITIONAL_DIRS})
  endif(PARSEUTILS_ADDITIONAL_DIRS)
  if(PARSEUTILS_ADDITIONAL_DEFS)
    target_compile_definitions(libparserutils_shared PUBLIC ${PARSEUTILS_ADDITIONAL_DEFS})
  endif(PARSEUTILS_ADDITIONAL_DEFS)
  install(TARGETS libparserutils_shared
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(libparserutils_static STATIC ${PARSEUTILS_ADDITIONAL_C_FILES} ${PARSEUTILS_PUBLIC_HEADER_FILES})
  set_target_properties(libparserutils_static PROPERTIES OUTPUT_NAME ${PARSEUTILS_COMPONENT_NAME})
  target_include_directories(libparserutils_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  if(PARSEUTILS_ADDITIONAL_LIBS)
    target_link_libraries(libparserutils_static PUBLIC ${PARSEUTILS_ADDITIONAL_LIBS}) 
  endif(PARSEUTILS_ADDITIONAL_LIBS)
  if(PARSEUTILS_ADDITIONAL_DIRS)
    target_include_directories(libparserutils_static PUBLIC ${PARSEUTILS_ADDITIONAL_DIRS})
  endif(PARSEUTILS_ADDITIONAL_DIRS)
  if(PARSEUTILS_ADDITIONAL_DEFS)
    target_compile_definitions(libparserutils_static PUBLIC  ${PARSEUTILS_ADDITIONAL_DEFS})
  endif(PARSEUTILS_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libparserutils_static PRIVATE ${PARSEUTILS_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libparserutils_static
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_STATIC_LIBS)

if(PARSEUTILS_BUILD_DOCS)
  include( FindDoxygen )
  if(DOXYGEN_FOUND)
# Ideally, the documentation generated by Doxygen should be built
# inside the BUILD directories and installed from there.  But 
# the libparserutils developers had hard-coded relative paths to
# the source-directory in their Doxyfile causing Doxygen to fail 
# in building is done from outside the source-code tree.  
    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/build/docs)
    add_custom_target( doc_doxygen ALL
        COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_SOURCE_DIR}/build/Doxyfile
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM )
    install(DIRECTORY ${CMAKE_SOURCE_DIR}/build/docs/html DESTINATION ${CMAKE_INSTALL_FULL_DOCDIR})
  endif(DOXYGEN_FOUND)
endif(PARSEUTILS_BUILD_DOCS)

#from: https://cmake.org/pipermail/cmake/2010-October/040246.html
#This installs the headers into our hierarchy in the include file.
foreach(PARSEUTILS_HEADER ${PARSEUTILS_PUBLIC_HEADER_FILES})
   string(REGEX MATCH "(.*)[/\\]" PARSEUTILS_DIR ${PARSEUTILS_HEADER})
   install(FILES ${PARSEUTILS_HEADER} DESTINATION ${PARSEUTILS_DIR})
endforeach(PARSEUTILS_HEADER ${PARSEUTILS_PUBLIC_HEADER_FILES})

if(PARSEUTILS_WITH_PKGCONFIG_SUPPORT)
  set(PARSEUTILS_PC ${PARSEUTILS_BUILD_DIR}/libparserutils.pc)

  # This stuff is necessary to ensure that the dependency list
  # in our .pc file is in the proper format
#Make pkg-config
  set(PARSEUTILS_PC_EXC "${CMAKE_INSTALL_PREFIX}")
  set(PARSEUTILS_PC_INC "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(PARSEUTILS_PC_LIB "${CMAKE_INSTALL_FULL_LIBDIR}")
  if("${PARSEUTILS_PC_EXC}" STREQUAL "${CMAKE_INSTALL_PREFIX}")
    set(PARSEUTILS_PC_EXC "\${prefix}")
  endif()
  if ("${PARSEUTILS_PC_INC}" STREQUAL "${CMAKE_INSTALL_PREFIX}/include")
    set(PARSEUTILS_PC_INC "\${prefix}/include")
  endif()
  if ("${PARSEUTILS_PC_LIB}" STREQUAL "${CMAKE_INSTALL_PREFIX}/lib")
    set(PARSEUTILS_PC_LIB "\${exec_prefix}/lib")
  endif()
  set(PARSEUTILS_PKGCONF_PARSEUTILS_DEPS "")
  foreach(PARSEUTILS_LIB_DEP ${PARSEUTILS_ADDITIONAL_LIBS})
    set(PARSEUTILS_PKGCONF_PARSEUTILS_DEPS "${PARSEUTILS_PKGCONF_PARSEUTILS_DEPS} -l${PARSEUTILS_LIB_DEP}")
  endforeach(PARSEUTILS_LIB_DEP)

  set(PARSEUTILS_PKGCONFIG_CONTENTS "prefix=${CMAKE_INSTALL_PREFIX}
exec_prefix=${PARSEUTILS_PC_EXC}
libdir=${PARSEUTILS_PC_LIB}
includedir=${PARSEUTILS_PC_INC}

Name: lib${PARSEUTILS_COMPONENT_NAME}
Description: ${PARSEUTILS_DESCRIPTION}
URL: http://www.netsurf-browser.org/projects/lib${PARSEUTILS_DESCRIPTION}/
Version: ${PARSEUTILS_SOVERSION}
Cflags: -I\${includedir}
Libs: -L\${libdir} -l${PARSEUTILS_COMPONENT_NAME}
Libs.private: ${PARSEUTILS_PKGCONF_PARSEUTILS_DEPS}
")
  file(WRITE ${PARSEUTILS_PC} ${PARSEUTILS_PKGCONFIG_CONTENTS})
  install(FILES ${PARSEUTILS_PC} DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif(PARSEUTILS_WITH_PKGCONFIG_SUPPORT)

# Test stuff
if(BUILD_SHARED_LIBS)
  add_executable(aliases_shared ${CMAKE_SOURCE_DIR}/test/aliases.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(aliases_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(aliases_shared libparserutils_shared)
  add_test(NAME Encoding_alias_handling_shared COMMAND aliases_shared)

  add_executable(cscodec-utf8_shared ${CMAKE_SOURCE_DIR}/test/cscodec-utf8.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-utf8_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(cscodec-utf8_shared libparserutils_shared)
  add_test(NAME UTF-8_charset_codec_implementation_shared__Simple_tests,_designed_to_validate_testdriver
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-utf8
    COMMAND cscodec-utf8_shared simple.dat)
  add_test(NAME UTF-8_charset_codec_implementation_shared__Markus_Kuhn's_UTF-8_decoding_test_file
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-utf8
    COMMAND cscodec-utf8_shared UTF-8-test.txt)

  add_executable(cscodec-utf16_shared ${CMAKE_SOURCE_DIR}/test/cscodec-utf16.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-utf16_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  #In Windows, you MUST link with Ws2_32 for htonl and ntoll
  if(WIN32)
    target_link_libraries(cscodec-utf16_shared libparserutils_static Ws2_32)
  else(WIN32)
    target_link_libraries(cscodec-utf16_shared libparserutils_static)
  endif(WIN32)
  add_test(NAME UTF-16_charset_codec_implementation_shared__Simple_tests,_designed_to_validate_testdriver 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-utf16
    COMMAND cscodec-utf16_shared simple.dat)

  add_executable(cscodec-ext8_shared ${CMAKE_SOURCE_DIR}/test/cscodec-ext8.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-ext8_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(cscodec-ext8_shared libparserutils_shared)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1250 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1250.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1251
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1251.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1252
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1252.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1253 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1253.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1254
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1254.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1255 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1255.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1256 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1256.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1257
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1257.dat)
  add_test(NAME Extended_8bit_charset_codec_shared__Windows-1258 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_shared cp1258.dat)

  add_executable(cscodec-8859_shared ${CMAKE_SOURCE_DIR}/test/cscodec-8859.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-8859_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(cscodec-8859_shared libparserutils_shared)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-1 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 1.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-2
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 2.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-3
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 3.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-4 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 4.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-5 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 5.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-6
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 6.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-7
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 7.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-8 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 8.dat)
  add_test(NAME ISO-8859-n-_shared__ISO-8859-9
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_shared 9.dat)

  add_executable(filter_shared ${CMAKE_SOURCE_DIR}/test/filter.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(filter_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(filter_shared libparserutils_shared)
  add_test(NAME Input_stream_filtering_shared COMMAND filter_shared)

  add_executable(inputstream_shared ${CMAKE_SOURCE_DIR}/test/inputstream.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(inputstream_shared PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(inputstream_shared libparserutils_shared)
  add_test(NAME Inputstream_handling_shared__Markus_Kuhn's_UTF-8_decoding_test_file 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/input
    COMMAND inputstream_shared UTF-8-test.txt)

endif(BUILD_SHARED_LIBS)

if(BUILD_STATIC_LIBS)
  add_executable(aliases_static ${CMAKE_SOURCE_DIR}/test/aliases.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(aliases_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(aliases_static libparserutils_static)
  add_test(NAME Encoding_alias_handling_aliases_static COMMAND aliases_static)

  add_executable(cscodec-utf8_static ${CMAKE_SOURCE_DIR}/test/cscodec-utf8.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-utf8_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(cscodec-utf8_static libparserutils_static Ws2_32)
  add_test(NAME UTF-8_charset_codec_implementation_static_Simple_tests,_designed_to_validate_testdriver
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-utf8
    COMMAND cscodec-utf8_static simple.dat )
  add_test(NAME UTF-8_charset_codec_implementation_static__Markus_Kuhn's_UTF-8_decoding_test_file
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-utf8
    COMMAND cscodec-utf8_static UTF-8-test.txt)

  add_executable(cscodec-utf16_static ${CMAKE_SOURCE_DIR}/test/cscodec-utf16.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-utf16_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  #In Windows, you MUST link with Ws2_32 for htonl and ntoll
  if(WIN32)
    target_link_libraries(cscodec-utf16_static libparserutils_static Ws2_32)
  else(WIN32)
    target_link_libraries(cscodec-utf16_static libparserutils_static)
  endif(WIN32)
  add_test(NAME UTF-16_charset_codec_implementation_static__Simple_tests,_designed_to_validate_testdriver 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-utf16
    COMMAND cscodec-utf16_static simple.dat)

  add_executable(cscodec-ext8_static ${CMAKE_SOURCE_DIR}/test/cscodec-ext8.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-ext8_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(cscodec-ext8_static libparserutils_static)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1250
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1250.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1251 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1251.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1252
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1252.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1253 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1253.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1254
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1254.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1255 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1255.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1256 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1256.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1257
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1257.dat)
  add_test(NAME Extended_8bit_charset_codec_static__Windows-1258
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-ext8
    COMMAND cscodec-ext8_static cp1258.dat)

  add_executable(cscodec-8859_static ${CMAKE_SOURCE_DIR}/test/cscodec-8859.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(cscodec-8859_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(cscodec-8859_static libparserutils_static)
  add_test(NAME ISO-8859-n-_static__ISO-8859-1
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 1.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-2
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 2.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-3
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 3.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-4
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 4.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-5 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 5.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-6 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 6.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-7
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 7.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-8
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 8.dat)
  add_test(NAME ISO-8859-n-_static__ISO-8859-9
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/cscodec-8859
    COMMAND cscodec-8859_static 9.dat)

  add_executable(filter_static ${CMAKE_SOURCE_DIR}/test/filter.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(filter_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(filter_static libparserutils_static)
  add_test(NAME Input_stream_filtering_static COMMAND filter_static)

  add_executable(inputstream_static ${CMAKE_SOURCE_DIR}/test/inputstream.c ${CMAKE_SOURCE_DIR}/test/testutils.h)
  target_include_directories(inputstream_static PRIVATE ${CMAKE_SOURCE_DIR}/include)
  target_link_libraries(inputstream_static libparserutils_static)
  add_test(NAME Inputstream_handling_static__Markus_Kuhn's_UTF-8_decoding_test_file 
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/input
    COMMAND inputstream_static UTF-8-test.txt)
endif(BUILD_STATIC_LIBS)

