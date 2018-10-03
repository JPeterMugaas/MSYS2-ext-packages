cmake_minimum_required (VERSION 2.8.4)
project (libhubbub)
include(FindPerl)
include(GNUInstallDirs)
include(FindPkgConfig)
enable_testing()
include( CTest )

#Obligitory version and copyright for pkg-config file and version info
#as well as possibly, automated installers.
set(HUBBUB_COMPONENT_NAME "hubbub")
set(HUBBUB_AUTHOR "John-Mark Bell <jmb@netsurf-browser.org>")
set(HUBBUB_COPYRIGHT "Copyright 2009-2015 John-Mark Bell <jmb@netsurf-browser.org>")
set(HUBBUB_DESCRIPTION "HTML5 parsing library")
set(HUBBUB_VERSION_MAJOR 0)
set(HUBBUB_VERSION_MINOR 3)
set(HUBBUB_VERSION_PATCH 5)
set(HUBBUB_SOVERSION ${HUBBUB_VERSION_MAJOR}.${HUBBUB_VERSION_MINOR}.${HUBBUB_VERSION_PATCH})

#Do not prefix BUILD_SHARED_LIBS and BUILD_STATIC_LIBS because those are standard cmake options
#We only put them in the menu for convenience.
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(HUBBUB_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)

if(NOT PERL_FOUND)
  message(FATAL_ERROR "Perl is required")
endif()

set(HUBBUB_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(HUBBUB_ADDITIONAL_C_FILES
   ${CMAKE_SOURCE_DIR}/src/parser.c
   ${CMAKE_SOURCE_DIR}/src/charset/detect.c
   ${CMAKE_SOURCE_DIR}/src/charset/detect.h
   ${CMAKE_SOURCE_DIR}/src/tokeniser/entities.c
   ${CMAKE_SOURCE_DIR}/src/tokeniser/entities.h
   ${CMAKE_SOURCE_DIR}/src/tokeniser/tokeniser.c
   ${CMAKE_SOURCE_DIR}/src/tokeniser/tokeniser.h
   ${CMAKE_SOURCE_DIR}/src/treebuilder/after_after_body.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/after_after_frameset.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/after_body.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/after_frameset.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/after_head.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/before_head.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/before_html.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/generic_rcdata.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_body.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_caption.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_cell.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_column_group.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_foreign_content.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_frameset.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_head.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_head_noscript.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_row.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_select.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_select_in_table.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_table.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/in_table_body.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/initial.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/internal.h
   ${CMAKE_SOURCE_DIR}/src/treebuilder/modes.h
   ${CMAKE_SOURCE_DIR}/src/treebuilder/treebuilder.c
   ${CMAKE_SOURCE_DIR}/src/treebuilder/treebuilder.h
   ${CMAKE_SOURCE_DIR}/src/utils/errors.c
   ${CMAKE_SOURCE_DIR}/src/utils/parserutilserror.h
   ${CMAKE_SOURCE_DIR}/src/utils/string.c
   ${CMAKE_SOURCE_DIR}/src/utils/string.h
   ${CMAKE_SOURCE_DIR}/src/utils/utils.h
)
set(HUBBUB_PUBLIC_HEADER_FILES
   include/hubbub/errors.h
   include/hubbub/functypes.h
   include/hubbub/hubbub.h
   include/hubbub/parser.h
   include/hubbub/tree.h
   include/hubbub/types.h)

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

if(CMAKE_COMPILER_IS_GNUCC)
  set(HUBBUB_WARNFLAGS -Wall -W -Wundef -Wpointer-arith -Wcast-align
        -Wwrite-strings -Wstrict-prototypes -Wmissing-prototypes 
        -Wmissing-declarations -Wnested-externs -pedantic)
endif(CMAKE_COMPILER_IS_GNUCC)

execute_process(WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
COMMAND ${PERL_EXECUTABLE} ${CMAKE_SOURCE_DIR}/build/make-entities.pl ${CMAKE_SOURCE_DIR}/build/Entities)
include_directories(${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/include)
# setup any additional libs required by this.
set(HUBBUB_ADDITIONAL_LIBS "")
set(HUBBUB_ADDITIONAL_DIRS "")
set(HUBBUB_ADDITIONAL_DEFS "_BSD_SOURCE" "_DEFAULT_SOURCE")

#pkg_check_modules
if(PKG_CONFIG_FOUND)
    pkg_check_modules(libparserutils REQUIRED libparserutils)
    list(APPEND HUBBUB_ADDITIONAL_DIRS ${libparserutils_INCLUDEDIR})
    list(APPEND HUBBUB_ADDITIONAL_LIBS ${libparserutils_LIBRARIES})
else()
    find_library(parserutils HUBBUB_HUBBUB_LIB)
    if (HUBBUB_HUBBUB_LIB)
      list(APPEND HUBBUB_ADDITIONAL_LIBS ${HUBBUB_HUBBUB_LIB})
    else()
      message(FATAL_ERROR "Parseutils not found")
    endif(HUBBUB_HUBBUB_LIB)
endif(PKG_CONFIG_FOUND)

if(BUILD_SHARED_LIBS)
   if(CMAKE_RC_COMPILER)
  # Make .rc file  
  		set(HUBBUB_RC_CONTENTS "1 VERSIONINFO
FILEVERSION     ${HUBBUB_VERSION_MAJOR},${HUBBUB_VERSION_MINOR},${HUBBUB_VERSION_PATCH},0
PRODUCTVERSION  ${HUBBUB_VERSION_MAJOR},${HUBBUB_VERSION_MINOR},${HUBBUB_VERSION_PATCH},0
BEGIN
  BLOCK \"StringFileInfo\"
  BEGIN
    BLOCK \"040904E4\"
    BEGIN
      VALUE \"CompanyName\", \"${HUBBUB_AUTHOR}\"
      VALUE \"FileDescription\", \"${HUBBUB_DESCRIPTION}\"
      VALUE \"FileVersion\", \"${HUBBUB_VERSION_MAJOR}.${HUBBUB_VERSION_MINOR}.${HUBBUB_VERSION_PATCH}\"
      VALUE \"InternalName\", \"${CMAKE_SHARED_MODULE_PREFIX}${HUBBUB_COMPONENT_NAME}\"
      VALUE \"LegalCopyright\", \"${HUBBUB_COPYRIGHT}\"
      VALUE \"OriginalFilename\", \"${CMAKE_SHARED_MODULE_PREFIX}${HUBBUB_COMPONENT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}\"
      VALUE \"ProductName\", \"${HUBBUB_COMPONENT_NAME}\"
      VALUE \"ProductVersion\", \"${HUBBUB_VERSION_MAJOR}.${HUBBUB_VERSION_MINOR}.${HUBBUB_VERSION_PATCH}\"
    END
  END
  BLOCK \"VarFileInfo\"
  BEGIN
    VALUE \"Translation\", 0x409, 1252
  END
END")
    FILE(WRITE ${HUBBUB_BUILD_DIR}/lib${HUBBUB_COMPONENT_NAME}.rc "${HUBBUB_RC_CONTENTS}")
    add_library(libhubbub_shared SHARED ${HUBBUB_PUBLIC_HEADER_FILES} ${HUBBUB_ADDITIONAL_C_FILES} lib${HUBBUB_COMPONENT_NAME}.rc)
  else()
    add_library(libhubbub_shared SHARED ${HUBBUB_PUBLIC_HEADER_FILES} ${HUBBUB_ADDITIONAL_C_FILES} SOVERSION ${HUBBUB_SOVERSION})
  endif()
  set_target_properties(libhubbub_shared PROPERTIES OUTPUT_NAME ${HUBBUB_COMPONENT_NAME})
  if(HUBBUB_ADDITIONAL_LIBS)
    target_link_libraries(libhubbub_shared PUBLIC ${HUBBUB_ADDITIONAL_LIBS}) 
  endif(HUBBUB_ADDITIONAL_LIBS)
  if(HUBBUB_ADDITIONAL_DIRS)
    target_include_directories(libhubbub_shared PUBLIC ${HUBBUB_ADDITIONAL_DIRS})
  endif(HUBBUB_ADDITIONAL_DIRS)
  if(HUBBUB_ADDITIONAL_DEFS)
    target_include_directories(libhubbub_shared PUBLIC ${HUBBUB_ADDITIONAL_DEFS})
  endif(HUBBUB_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libhubbub_shared PRIVATE ${WAPCAPLET_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libhubbub_shared
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(libhubbub_static STATIC ${HUBBUB_PUBLIC_HEADER_FILES} ${HUBBUB_ADDITIONAL_C_FILES})
  set_target_properties(libhubbub_static PROPERTIES OUTPUT_NAME ${HUBBUB_COMPONENT_NAME})
  if(HUBBUB_ADDITIONAL_LIBS)
    target_link_libraries(libhubbub_static PUBLIC ${HUBBUB_ADDITIONAL_LIBS}) 
  endif(HUBBUB_ADDITIONAL_LIBS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libhubbub_static PRIVATE ${WAPCAPLET_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  if(HUBBUB_ADDITIONAL_DIRS)
    target_include_directories(libhubbub_static PUBLIC ${HUBBUB_ADDITIONAL_DIRS})
  endif(HUBBUB_ADDITIONAL_DIRS)
  if(HUBBUB_ADDITIONAL_DEFS)
    target_include_directories(libhubbub_static PUBLIC ${HUBBUB_ADDITIONAL_DEFS})
  endif(HUBBUB_ADDITIONAL_DEFS)
  install(TARGETS libhubbub_static
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_STATIC_LIBS)

#from: https://cmake.org/pipermail/cmake/2010-October/040246.html
#This installs the headers into our hierarchy in the include file.
foreach(HUBBUB_HEADER ${HUBBUB_PUBLIC_HEADER_FILES})
   string(REGEX MATCH "(.*)[/\\]" HUBBUB_DIR ${HUBBUB_HEADER})
   install(FILES ${HUBBUB_HEADER} DESTINATION ${HUBBUB_DIR})
endforeach(HUBBUB_HEADER ${HUBBUB_PUBLIC_HEADER_FILES})

if(HUBBUB_WITH_PKGCONFIG_SUPPORT)
  set(HUBBUB_PC ${HUBBUB_BUILD_DIR}/lib${HUBBUB_COMPONENT_NAME}.pc)

  # This stuff is necessary to ensure that the dependency list
  # in our .pc file is in the proper format
#Make pkg-config
  set(HUBBUB_PC_EXC "${CMAKE_INSTALL_PREFIX}")
  set(HUBBUB_PC_INC "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(HUBBUB_PC_LIB "${CMAKE_INSTALL_FULL_LIBDIR}")
  if("${HUBBUB_PC_EXC}" STREQUAL "${CMAKE_INSTALL_PREFIX}")
    set(HUBBUB_PC_EXC "\${prefix}")
  endif()
  if ("${HUBBUB_PC_INC}" STREQUAL "${CMAKE_INSTALL_PREFIX}/include")
    set(HUBBUB_PC_INC "\${prefix}/include")
  endif()
  if ("${HUBBUB_PC_LIB}" STREQUAL "${CMAKE_INSTALL_PREFIX}/lib")
    set(HUBBUB_PC_LIB "\${exec_prefix}/lib")
  endif()
  set(HUBBUB_PKGCONF_HUBBUB_DEPS "")
  foreach(HUBBUB_LIB_DEP ${HUBBUB_LIB_DEPEND_LIBS})
    set(HUBBUB_PKGCONF_HUBBUB_DEPS "${HUBBUB_PKGCONF_HUBBUB_DEPS} -l${HUBBUB_LIB_DEP}")
  endforeach(HUBBUB_LIB_DEP)

  set(HUBBUB_PKGCONFIG_CONTENTS "prefix=${CMAKE_INSTALL_PREFIX}
exec_prefix=${HUBBUB_PC_EXC}
libdir=${HUBBUB_PC_LIB}
includedir=${HUBBUB_PC_INC}

Name: lib${HUBBUB_COMPONENT_NAME}
Description: ${HUBBUB_DESCRIPTION}
URL: http://www.netsurf-browser.org/projects/lib${HUBBUB_COMPONENT_NAME}/
Version: ${HUBBUB_SOVERSION}
Requires: libparserutils
Cflags: -I\${includedir}
Libs: -L\${libdir} -l${HUBBUB_COMPONENT_NAME}
Libs.private: ${HUBBUB_PKGCONF_HUBBUB_DEPS}
")
  file(WRITE ${HUBBUB_PC} ${HUBBUB_PKGCONFIG_CONTENTS})
  install(FILES ${HUBBUB_PC} DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif(HUBBUB_WITH_PKGCONFIG_SUPPORT)

#
# Test section
#
if(PKG_CONFIG_FOUND)
    pkg_check_modules(json-c json-c)
    if(NOT json-c_FOUND)
      message(WARNING "json-c (json-cpp) is required for some tests.  Download it at:
https://github.com/open-source-parsers/jsoncpp 
Disabling tests that require this library")
    endif(NOT json-c_FOUND)
endif(PKG_CONFIG_FOUND)

if(BUILD_SHARED_LIBS)

    add_executable(entities_shared ${CMAKE_SOURCE_DIR}/test/entities.c)
    target_link_libraries(entities_shared libhubbub_shared)
    add_test(NAME Named_entity_dictionary_shared COMMAND entities_shared)

    add_executable(csdetect_shared ${CMAKE_SOURCE_DIR}/test/csdetect.c)
    target_link_libraries(csdetect_shared libhubbub_shared)
    add_test(NAME Charset_detection_shared__UTF_Byte_Order_Mark_detection_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_shared bom.dat)
    add_test(NAME Charset_detection_shared__Tests_for_meta_charsets_claiming_to_be_non-ASCII
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_shared test-yahoo-jp.dat)
    add_test(NAME Charset_detection_shared__Assorted_tests,_including_edge_cases,_from_html5lib
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_shared tests1.dat)
    add_test(NAME Charset_detection_shared__Further_tests_from_html5lib
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_shared tests2.dat)
    add_test(NAME Charset_detection_shared__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_shared regression.dat)
    add_test(NAME Charset_detection_shared__Character_encoding_overrides_from_8.2.2.2.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_shared overrides.dat)

    add_executable(parser_shared ${CMAKE_SOURCE_DIR}/test/parser.c)
    target_link_libraries(parser_shared libhubbub_shared)
    add_test(NAME Public_parser_API_shared__HTML5_tree_construction_algorithm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared section-tree-construction.html)
    add_test(NAME Public_parser_API_shared__HTML5_specification
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared \#web-apps.html)
    add_test(NAME Public_parser_API_shared__Page_with_initial_</html>
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared initial-close-tag.html)
    add_test(NAME Public_parser_API_shared__HTML_document_that_breaks_libxml's_HTML_parser
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared \#phonecalls.html)
    add_test(NAME Public_parser_API_shared__Misnested_tags
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared misnested.html)
    add_test(NAME Public_parser_API_shared__Test_of_<isindex>_parsing
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared isindex.html)
    add_test(NAME Public_parser_API_shared__http://www.danmarks.net/firmaer.phtml?idxno=1418&letter=F
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared \#firmaer.phtml)
    add_test(NAME Public_parser_API_shared__http://www.secinfo.com/d11MXs.2WQ9.d.htm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared \#d11MXs.2WQ9.d.htm)
    add_test(NAME Public_parser_API_shared__Regression_test_for_script_collection
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared regression-script-collect.html)
    add_test(NAME Public_parser_API_shared__Mangleme_page_caused_assertion,_fixed_in_r5093.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared mangleme.1.html)
    add_test(NAME Public_parser_API_shared__Mangleme_page_caused_segfault,_fixed_in_r5098.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared mangleme.2.html)
    add_test(NAME Public_parser_API_shared__Mangleme_page_caused_assertion,_fixed_in_r5086.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared mangleme.3.html)
    add_test(NAME Public_parser_API_shared__Segfault_page_fixed_in_r5104.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared www.fhis.ubc.ca.html)
    add_test(NAME Public_parser_API_shared__Segfault_page_fixed_in_r5106.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared ccr.coriell.org.html)
    add_test(NAME Public_parser_API_shared__Segfault_in_treebuilder_fixed_in_r5125.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared wbh.co.uk.html)
    add_test(NAME Public_parser_API_shared__Segfault_in_current_node
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared www.directline.com.html)
    add_test(NAME Public_parser_API_shared__Abort_in_token_emitter_fixed_in_r5146.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared www.hanazonohifuku.com.html)
    add_test(NAME Public_parser_API_shared__Abort_in_generic_end_tag_handling_fixed_in_r6746.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_shared DocumentIndex.jsp)

    add_executable(tokeniser_shared ${CMAKE_SOURCE_DIR}/test/tokeniser.c)
    target_link_libraries(tokeniser_shared libhubbub_shared)
    add_test(NAME HTML_tokeniser_shared__HTML5_tree_construction_algorithm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared section-tree-construction.html)
    add_test(NAME HTML_tokeniser_shared__HTML5_specification
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared \#web-apps.html)
    add_test(NAME HTML_tokeniser_shared__Page_with_initial_</html>
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared initial-close-tag.html)
    add_test(NAME HTML_tokeniser_shared__HTML_document_that_breaks_libxml's_HTML_parser
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared \#phonecalls.html)
    add_test(NAME HTML_tokeniser_shared__Misnested_tags
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared misnested.html)
    add_test(NAME HTML_tokeniser_shared__Test_of_<isindex>_parsing
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared isindex.html)
    add_test(NAME HTML_tokeniser_shared__http://www.danmarks.net/firmaer.phtml?idxno=1418&letter=F
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared \#firmaer.phtml)
    add_test(NAME HTML_tokeniser_shared__http://www.secinfo.com/d11MXs.2WQ9.d.htm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared \#d11MXs.2WQ9.d.htm)
    add_test(NAME HTML_tokeniser_shared__Regression_test_for_script_collection
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared regression-script-collect.html)
    add_test(NAME HTML_tokeniser_shared__Mangleme_page_caused_assertion,_fixed_in_r5093.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared mangleme.1.html)
    add_test(NAME HTML_tokeniser_shared__Mangleme_page_caused_segfault,_fixed_in_r5098.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared mangleme.2.html)
    add_test(NAME HTML_tokeniser_shared__Mangleme_page_caused_assertion,_fixed_in_r5086.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared mangleme.3.html)
    add_test(NAME HTML_tokeniser_shared__Segfault_page_fixed_in_r5104.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared www.fhis.ubc.ca.html)
    add_test(NAME HTML_tokeniser_shared__Segfault_page_fixed_in_r5106.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared ccr.coriell.org.html)
    add_test(NAME HTML_tokeniser_shared__Segfault_in_treebuilder_fixed_in_r5125.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared wbh.co.uk.html)
    add_test(NAME HTML_tokeniser_shared__Segfault_in_current_node
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared www.directline.com.html)
    add_test(NAME HTML_tokeniser_shared__Abort_in_token_emitter_fixed_in_r5146.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared www.hanazonohifuku.com.html)
    add_test(NAME HTML_tokeniser_shared__Abort_in_generic_end_tag_handling_fixed_in_r6746.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_shared DocumentIndex.jsp)

  if(json-c_FOUND)
    add_executable(tokeniser2_shared ${CMAKE_SOURCE_DIR}/test/tokeniser2.c)
    target_link_libraries(tokeniser2_shared libhubbub_shared ${json-c_LIBRARIES})
    target_include_directories(tokeniser2_shared SYSTEM PUBLIC ${json-c_INCLUDE_DIRS})
    add_test(NAME HTML_tokeniser_again_shared__html5lib_tests_part_1
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared test1.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_tests_part_2
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared test2.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_tests_part_3
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared test3.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_tests_part_4
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared test4.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_content_model_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared contentModelFlags.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_entity_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared entities.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_escape_flag_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared escapeFlag.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_numeric_entities_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared numericEntities.test)
    add_test(NAME HTML_tokeniser_again_shared__html5lib_unicode_character_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared \#unicodeChars.test)
    add_test(NAME HTML_tokeniser_again_shared__CDATA_section_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared cdata.test)
    add_test(NAME HTML_tokeniser_again_shared__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_shared regression.test)

    add_executable(tokeniser3_shared ${CMAKE_SOURCE_DIR}/test/tokeniser3.c)
    target_link_libraries(tokeniser3_shared libhubbub_shared ${json-c_LIBRARIES})
    target_include_directories(tokeniser3_shared SYSTEM PUBLIC ${json-c_INCLUDE_DIRS})
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_tests_part_1
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared test1.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_tests_part_2
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared test2.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_tests_part_3
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared test3.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_tests_part_4
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared test4.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_content_model_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared contentModelFlags.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_entity_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared entities.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_escape_flag_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared escapeFlag.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_numeric_entities_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared numericEntities.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__html5lib_unicode_character_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared \#unicodeChars.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__CDATA_section_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared cdata.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_shared__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_shared regression.test)
    endif(json-c_FOUND)
#out of order
    add_executable(tree_shared ${CMAKE_SOURCE_DIR}/test/tree.c)
    target_link_libraries(tree_shared libhubbub_shared)
    add_test(NAME Treebuilding_API_shared__HTML5_tree_construction_algorithm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared section-tree-construction.html)
    add_test(NAME Treebuilding_API_shared__HTML5_specification
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared \#web-apps.html)
    add_test(NAME Treebuilding_API_shared__Page_with_initial_</html>
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared initial-close-tag.html)
    add_test(NAME Treebuilding_API_shared__HTML_document_that_breaks_libxml's_HTML_parser
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared \#phonecalls.html)
    add_test(NAME Treebuilding_API_shared__Misnested_tags
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared misnested.html)
    add_test(NAME Treebuilding_API_shared__Test_of_<isindex>_parsing
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared isindex.html)
    add_test(NAME Treebuilding_API_shared__http://www.danmarks.net/firmaer.phtml?idxno=1418&letter=F
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared \#firmaer.phtml)
    add_test(NAME Treebuilding_API_shared__http://www.secinfo.com/d11MXs.2WQ9.d.htm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared \#d11MXs.2WQ9.d.htm)
    add_test(NAME Treebuilding_API_shared__Regression_test_for_script_collection
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared regression-script-collect.html)
    add_test(NAME Treebuilding_API_shared__Mangleme_page_caused_assertion,_fixed_in_r5093.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared mangleme.1.html)
    add_test(NAME Treebuilding_API_shared__Mangleme_page_caused_segfault,_fixed_in_r5098.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared mangleme.2.html)
    add_test(NAME Treebuilding_API_shared__Mangleme_page_caused_assertion,_fixed_in_r5086.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared mangleme.3.html)
    add_test(NAME Treebuilding_API_shared__Segfault_page_fixed_in_r5104.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared www.fhis.ubc.ca.html)
    add_test(NAME Treebuilding_API_shared__Segfault_page_fixed_in_r5106.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared ccr.coriell.org.html)
    add_test(NAME Treebuilding_API_shared__Segfault_in_treebuilder_fixed_in_r5125.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared wbh.co.uk.html)
    add_test(NAME Treebuilding_API_shared__Segfault_in_current_node
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared www.directline.com.html)
    add_test(NAME Treebuilding_API_shared__Abort_in_token_emitter_fixed_in_r5146.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared www.hanazonohifuku.com.html)
    add_test(NAME Treebuilding_API_shared__Abort_in_generic_end_tag_handling_fixed_in_r6746.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_shared DocumentIndex.jsp)

    add_executable(tree2_shared ${CMAKE_SOURCE_DIR}/test/tree2.c)
    target_link_libraries(tree2_shared libhubbub_shared)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests1.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests1.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests2.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests2.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests3.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests3.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests4.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests4.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests5.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests5.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests6.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests6.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests7.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests7.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests8.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests8.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests9.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests9.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests10.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests10.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests11.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests11.dat)
    add_test(NAME Treebuilding_API_shared__html5lib__tests_tests12.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared tests12.dat)
    add_test(NAME Treebuilding_API_shared__Tests_after_after_body_mode
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared after-after-body.dat)
    add_test(NAME Treebuilding_API_shared__Tests_after_after_frameset_mode
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared after-after-frameset.dat)
    add_test(NAME Treebuilding_API_shared__Tests_after_body_mode
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared after-body.dat)
    add_test(NAME Treebuilding_API_shared__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_shared regression.dat)

    add_executable(tree-buf_shared ${CMAKE_SOURCE_DIR}/test/tree-buf.c)
    target_link_libraries(tree-buf_shared libhubbub_shared)
    add_test(NAME Treebuilder_specified_chunks_shared__Basic_test_of_test_driver
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-chunks/
       COMMAND tree-buf_shared basic.dat)
    add_test(NAME Treebuilder_specified_chunks_shared__Entity_test
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-chunks/
       COMMAND tree-buf_shared entitytest.html)

endif(BUILD_SHARED_LIBS)

if(BUILD_STATIC_LIBS)

    add_executable(entities_static ${CMAKE_SOURCE_DIR}/test/entities.c)
    target_link_libraries(entities_static libhubbub_static)
    add_test(NAME Named_entity_dictionary_static COMMAND entities_static)

    add_executable(csdetect_static ${CMAKE_SOURCE_DIR}/test/csdetect.c)
    target_link_libraries(csdetect_static libhubbub_static)
    add_test(NAME Charset_detection_static__UTF_Byte_Order_Mark_detection_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_static bom.dat)
    add_test(NAME Charset_detection_static__Tests_for_meta_charsets_claiming_to_be_non-ASCII
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_static test-yahoo-jp.dat)
    add_test(NAME Charset_detection_static__Assorted_tests,_including_edge_cases,_from_html5lib
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_static tests1.dat)
    add_test(NAME Charset_detection_static__Further_tests_from_html5lib
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_static tests2.dat)
    add_test(NAME Charset_detection_static__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_static regression.dat)
    add_test(NAME Charset_detection_static__Character_encoding_overrides_from_8.2.2.2.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/csdetect
       COMMAND csdetect_static overrides.dat)

    add_executable(parser_static ${CMAKE_SOURCE_DIR}/test/parser.c)
    target_link_libraries(parser_static libhubbub_static)
    add_test(NAME Public_parser_API_static__HTML5_tree_construction_algorithm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static section-tree-construction.html)
    add_test(NAME Public_parser_API_static__HTML5_specification
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static \#web-apps.html)
    add_test(NAME Public_parser_API_static__Page_with_initial_</html>
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static initial-close-tag.html)
    add_test(NAME Public_parser_API_static__HTML_document_that_breaks_libxml's_HTML_parser
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static \#phonecalls.html)
    add_test(NAME Public_parser_API_static__Misnested_tags
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static misnested.html)
    add_test(NAME Public_parser_API_static__Test_of_<isindex>_parsing
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static isindex.html)
    add_test(NAME Public_parser_API_static__http://www.danmarks.net/firmaer.phtml?idxno=1418&letter=F
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static \#firmaer.phtml)
    add_test(NAME Public_parser_API_static__http://www.secinfo.com/d11MXs.2WQ9.d.htm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static \#d11MXs.2WQ9.d.htm)
    add_test(NAME Public_parser_API_static__Regression_test_for_script_collection
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static regression-script-collect.html)
    add_test(NAME Public_parser_API_static__Mangleme_page_caused_assertion,_fixed_in_r5093.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static mangleme.1.html)
    add_test(NAME Public_parser_API_static__Mangleme_page_caused_segfault,_fixed_in_r5098.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static mangleme.2.html)
    add_test(NAME Public_parser_API_static__Mangleme_page_caused_assertion,_fixed_in_r5086.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static mangleme.3.html)
    add_test(NAME Public_parser_API_static__Segfault_page_fixed_in_r5104.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static www.fhis.ubc.ca.html)
    add_test(NAME Public_parser_API_static__Segfault_page_fixed_in_r5106.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static ccr.coriell.org.html)
    add_test(NAME Public_parser_API_static__Segfault_in_treebuilder_fixed_in_r5125.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static wbh.co.uk.html)
    add_test(NAME Public_parser_API_static__Segfault_in_current_node
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static www.directline.com.html)
    add_test(NAME Public_parser_API_static__Abort_in_token_emitter_fixed_in_r5146.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static www.hanazonohifuku.com.html)
    add_test(NAME Public_parser_API_static__Abort_in_generic_end_tag_handling_fixed_in_r6746.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND parser_static DocumentIndex.jsp)

    add_executable(tokeniser_static ${CMAKE_SOURCE_DIR}/test/tokeniser.c)
    target_link_libraries(tokeniser_static libhubbub_static)
    add_test(NAME HTML_tokeniser_static__HTML5_tree_construction_algorithm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static section-tree-construction.html)
    add_test(NAME HTML_tokeniser_static__HTML5_specification
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static \#web-apps.html)
    add_test(NAME HTML_tokeniser_static__Page_with_initial_</html>
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static initial-close-tag.html)
    add_test(NAME HTML_tokeniser_static__HTML_document_that_breaks_libxml's_HTML_parser
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static \#phonecalls.html)
    add_test(NAME HTML_tokeniser_static__Misnested_tags
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static misnested.html)
    add_test(NAME HTML_tokeniser_static__Test_of_<isindex>_parsing
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static isindex.html)
    add_test(NAME HTML_tokeniser_static__http://www.danmarks.net/firmaer.phtml?idxno=1418&letter=F
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static \#firmaer.phtml)
    add_test(NAME HTML_tokeniser_static__http://www.secinfo.com/d11MXs.2WQ9.d.htm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static \#d11MXs.2WQ9.d.htm)
    add_test(NAME HTML_tokeniser_static__Regression_test_for_script_collection
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static regression-script-collect.html)
    add_test(NAME HTML_tokeniser_static__Mangleme_page_caused_assertion,_fixed_in_r5093.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static mangleme.1.html)
    add_test(NAME HTML_tokeniser_static__Mangleme_page_caused_segfault,_fixed_in_r5098.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static mangleme.2.html)
    add_test(NAME HTML_tokeniser_static__Mangleme_page_caused_assertion,_fixed_in_r5086.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static mangleme.3.html)
    add_test(NAME HTML_tokeniser_static__Segfault_page_fixed_in_r5104.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static www.fhis.ubc.ca.html)
    add_test(NAME HTML_tokeniser_static__Segfault_page_fixed_in_r5106.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static ccr.coriell.org.html)
    add_test(NAME HTML_tokeniser_static__Segfault_in_treebuilder_fixed_in_r5125.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static wbh.co.uk.html)
    add_test(NAME HTML_tokeniser_static__Segfault_in_current_node
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static www.directline.com.html)
    add_test(NAME HTML_tokeniser_static__Abort_in_token_emitter_fixed_in_r5146.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static www.hanazonohifuku.com.html)
    add_test(NAME HTML_tokeniser_static__Abort_in_generic_end_tag_handling_fixed_in_r6746.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tokeniser_static DocumentIndex.jsp)

  if(json-c_FOUND)
    add_executable(tokeniser2_static ${CMAKE_SOURCE_DIR}/test/tokeniser2.c)
    target_link_libraries(tokeniser2_static libhubbub_static ${json-c_LIBRARIES})
    target_include_directories(tokeniser2_static SYSTEM PUBLIC ${json-c_INCLUDE_DIRS})
    add_test(NAME HTML_tokeniser_again_static__html5lib_tests_part_1
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static test1.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_tests_part_2
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static test2.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_tests_part_3
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static test3.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_tests_part_4
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static test4.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_content_model_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static contentModelFlags.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_entity_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static entities.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_escape_flag_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static escapeFlag.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_numeric_entities_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static numericEntities.test)
    add_test(NAME HTML_tokeniser_again_static__html5lib_unicode_character_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static \#unicodeChars.test)
    add_test(NAME HTML_tokeniser_again_static__CDATA_section_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static cdata.test)
    add_test(NAME HTML_tokeniser_again_static__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser2_static regression.test)

    add_executable(tokeniser3_static ${CMAKE_SOURCE_DIR}/test/tokeniser3.c)
    target_link_libraries(tokeniser3_static libhubbub_static ${json-c_LIBRARIES})
    target_include_directories(tokeniser3_static SYSTEM PUBLIC ${json-c_INCLUDE_DIRS})
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_tests_part_1
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static test1.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_tests_part_2
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static test2.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_tests_part_3
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static test3.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_tests_part_4
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static test4.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_content_model_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static contentModelFlags.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_entity_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static entities.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_escape_flag_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static escapeFlag.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_numeric_entities_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static numericEntities.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__html5lib_unicode_character_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static \#unicodeChars.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__CDATA_section_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static cdata.test)
    add_test(NAME HTML_tokeniser_byte-by-byte_static__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tokeniser2
       COMMAND tokeniser3_static regression.test)
    endif(json-c_FOUND)
#out of order
    add_executable(tree_static ${CMAKE_SOURCE_DIR}/test/tree.c)
    target_link_libraries(tree_static libhubbub_static)
    add_test(NAME Treebuilding_API_static__HTML5_tree_construction_algorithm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static section-tree-construction.html)
    add_test(NAME Treebuilding_API_static__HTML5_specification
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static \#web-apps.html)
    add_test(NAME Treebuilding_API_static__Page_with_initial_</html>
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static initial-close-tag.html)
    add_test(NAME Treebuilding_API_static__HTML_document_that_breaks_libxml's_HTML_parser
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static \#phonecalls.html)
    add_test(NAME Treebuilding_API_static__Misnested_tags
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static misnested.html)
    add_test(NAME Treebuilding_API_static__Test_of_<isindex>_parsing
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static isindex.html)
    add_test(NAME Treebuilding_API_static__http://www.danmarks.net/firmaer.phtml?idxno=1418&letter=F
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static \#firmaer.phtml)
    add_test(NAME Treebuilding_API_static__http://www.secinfo.com/d11MXs.2WQ9.d.htm
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static \#d11MXs.2WQ9.d.htm)
    add_test(NAME Treebuilding_API_static__Regression_test_for_script_collection
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static regression-script-collect.html)
    add_test(NAME Treebuilding_API_static__Mangleme_page_caused_assertion,_fixed_in_r5093.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static mangleme.1.html)
    add_test(NAME Treebuilding_API_static__Mangleme_page_caused_segfault,_fixed_in_r5098.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static mangleme.2.html)
    add_test(NAME Treebuilding_API_static__Mangleme_page_caused_assertion,_fixed_in_r5086.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static mangleme.3.html)
    add_test(NAME Treebuilding_API_static__Segfault_page_fixed_in_r5104.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static www.fhis.ubc.ca.html)
    add_test(NAME Treebuilding_API_static__Segfault_page_fixed_in_r5106.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static ccr.coriell.org.html)
    add_test(NAME Treebuilding_API_static__Segfault_in_treebuilder_fixed_in_r5125.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static wbh.co.uk.html)
    add_test(NAME Treebuilding_API_static__Segfault_in_current_node
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static www.directline.com.html)
    add_test(NAME Treebuilding_API_static__Abort_in_token_emitter_fixed_in_r5146.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static www.hanazonohifuku.com.html)
    add_test(NAME Treebuilding_API_static__Abort_in_generic_end_tag_handling_fixed_in_r6746.
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/html
       COMMAND tree_static DocumentIndex.jsp)

    add_executable(tree2_static ${CMAKE_SOURCE_DIR}/test/tree2.c)
    target_link_libraries(tree2_static libhubbub_static)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests1.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests1.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests2.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests2.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests3.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests3.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests4.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests4.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests5.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests5.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests6.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests6.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests7.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests7.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests8.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests8.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests9.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests9.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests10.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests10.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests11.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests11.dat)
    add_test(NAME Treebuilding_API_static__html5lib__tests_tests12.dat
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static tests12.dat)
    add_test(NAME Treebuilding_API_static__Tests_after_after_body_mode
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static after-after-body.dat)
    add_test(NAME Treebuilding_API_static__Tests_after_after_frameset_mode
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static after-after-frameset.dat)
    add_test(NAME Treebuilding_API_static__Tests_after_body_mode
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static after-body.dat)
    add_test(NAME Treebuilding_API_static__Regression_tests
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-construction/
       COMMAND tree2_static regression.dat)

    add_executable(tree-buf_static ${CMAKE_SOURCE_DIR}/test/tree-buf.c)
    target_link_libraries(tree-buf_static libhubbub_static)
    add_test(NAME Treebuilder_specified_chunks_static__Basic_test_of_test_driver
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-chunks/
       COMMAND tree-buf_static basic.dat)
    add_test(NAME Treebuilder_specified_chunks_static__Entity_test
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test/data/tree-chunks/
       COMMAND tree-buf_static entitytest.html)

endif(BUILD_STATIC_LIBS)


