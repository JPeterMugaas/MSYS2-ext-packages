cmake_minimum_required (VERSION 2.8.4)
project (libdom)
include(GNUInstallDirs)
include(FindPkgConfig)
enable_testing()
include( CTest )

#Obligitory version and copyright for pkg-config file and version info
#as well as possibly, automated installers.
set(DOM_COMPONENT_NAME "dom")
set(DOM_AUTHOR "John-Mark Bell <jmb@netsurf-browser.org>")
set(DOM_COPYRIGHT "Copyright 2009-2015 John-Mark Bell <jmb@netsurf-browser.org>")
set(DOM_DESCRIPTION "Implementation of the W3C DOM")
set(DOM_VERSION_MAJOR 0)
set(DOM_VERSION_MINOR 3)
set(DOM_VERSION_PATCH 3)
set(DOM_SOVERSION ${DOM_VERSION_MAJOR}.${DOM_VERSION_MINOR}.${DOM_VERSION_PATCH})

#Do not prefix BUILD_SHARED_LIBS and BUILD_STATIC_LIBS because those are standard cmake options
#We only put them in the menu for convenience.
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(DOM_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)
#XML Binding should be a combo-box GUI item.
set(DOM_XML_BINDING "Expat" CACHE STRING "With XML Binding")
set_property(CACHE DOM_XML_BINDING PROPERTY STRINGS None Libxml2 Expat)
option(DOM_WITH_LIBHUBBUB_BINDING "With ibhubbub bindings" ON)

if(CMAKE_COMPILER_IS_GNUCC)
set(DOM_WARNFLAGS -Wall -W -Wundef -Wpointer-arith -Wcast-align
        -Wwrite-strings -Wstrict-prototypes -Wmissing-prototypes 
        -Wmissing-declarations -Wnested-externs)
endif(CMAKE_COMPILER_IS_GNUCC)

set(DOM_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(DOM_ADDITIONAL_C_FILES 
   ${CMAKE_SOURCE_DIR}/src/core/string.c
   ${CMAKE_SOURCE_DIR}/src/core/node.c
   ${CMAKE_SOURCE_DIR}/src/core/attr.c 
   ${CMAKE_SOURCE_DIR}/src/core/characterdata.c
   ${CMAKE_SOURCE_DIR}/src/core/element.c
   ${CMAKE_SOURCE_DIR}/src/core/implementation.c
   ${CMAKE_SOURCE_DIR}/src/core/text.c 
   ${CMAKE_SOURCE_DIR}/src/core/typeinfo.c
   ${CMAKE_SOURCE_DIR}/src/core/comment.c
   ${CMAKE_SOURCE_DIR}/src/core/namednodemap.c
   ${CMAKE_SOURCE_DIR}/src/core/nodelist.c
   ${CMAKE_SOURCE_DIR}/src/core/cdatasection.c 
   ${CMAKE_SOURCE_DIR}/src/core/document_type.c
   ${CMAKE_SOURCE_DIR}/src/core/entity_ref.c 
   ${CMAKE_SOURCE_DIR}/src/core/pi.c
   ${CMAKE_SOURCE_DIR}/src/core/doc_fragment.c 
   ${CMAKE_SOURCE_DIR}/src/core/document.c
   ${CMAKE_SOURCE_DIR}/src/events/event.c
   ${CMAKE_SOURCE_DIR}/src/events/dispatch.c
   ${CMAKE_SOURCE_DIR}/src/events/event_target.c
   ${CMAKE_SOURCE_DIR}/src/events/document_event.c
   ${CMAKE_SOURCE_DIR}/src/events/custom_event.c
   ${CMAKE_SOURCE_DIR}/src/events/keyboard_event.c
   ${CMAKE_SOURCE_DIR}/src/events/mouse_wheel_event.c
   ${CMAKE_SOURCE_DIR}/src/events/text_event.c
   ${CMAKE_SOURCE_DIR}/src/events/event_listener.c
   ${CMAKE_SOURCE_DIR}/src/events/mouse_event.c
   ${CMAKE_SOURCE_DIR}/src/events/mutation_event.c
   ${CMAKE_SOURCE_DIR}/src/events/ui_event.c
   ${CMAKE_SOURCE_DIR}/src/events/mouse_multi_wheel_event.c
   ${CMAKE_SOURCE_DIR}/src/events/mutation_name_event.c
   ${CMAKE_SOURCE_DIR}/src/html/html_document.c
   ${CMAKE_SOURCE_DIR}/src/html/html_collection.c
   ${CMAKE_SOURCE_DIR}/src/html/html_options_collection.c
   ${CMAKE_SOURCE_DIR}/src/html/html_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_html_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_head_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_link_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_title_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_meta_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_base_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_style_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_body_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_form_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_select_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_button_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_input_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_text_area_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_opt_group_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_option_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_hr_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_dlist_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_directory_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_menu_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_fieldset_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_legend_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_div_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_paragraph_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_heading_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_quote_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_pre_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_br_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_label_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_ulist_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_olist_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_li_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_font_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_mod_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_anchor_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_basefont_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_image_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_object_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_param_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_applet_element.c 
   ${CMAKE_SOURCE_DIR}/src/html/html_area_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_map_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_script_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_tablecaption_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_tablecell_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_tablecol_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_tablesection_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_table_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_tablerow_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_frameset_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_frame_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_iframe_element.c
   ${CMAKE_SOURCE_DIR}/src/html/html_isindex_element.c
   ${CMAKE_SOURCE_DIR}/src/utils/namespace.c
   ${CMAKE_SOURCE_DIR}/src/utils/hashtable.c
   ${CMAKE_SOURCE_DIR}/src/utils/character_valid.c
   ${CMAKE_SOURCE_DIR}/src/utils/validate.c)
set(DOM_PUBLIC_HEADER_FILES
    include/dom/dom.h
    include/dom/functypes.h
    include/dom/inttypes.h
    include/dom/core/attr.h
    include/dom/core/doc_fragment.h
    include/dom/core/entity_ref.h
    include/dom/core/node.h
    include/dom/core/text.h
    include/dom/core/cdatasection.h
    include/dom/core/document.h
    include/dom/core/exceptions.h
    include/dom/core/nodelist.h
    include/dom/core/typeinfo.h
    include/dom/core/characterdata.h
    include/dom/core/document_type.h
    include/dom/core/implementation.h
    include/dom/core/pi.h
    include/dom/core/comment.h
    include/dom/core/element.h
    include/dom/core/namednodemap.h
    include/dom/core/string.h
    include/dom/events/custom_event.h
    include/dom/events/document_event.h
    include/dom/events/event.h
    include/dom/events/event_listener.h
    include/dom/events/event_target.h
    include/dom/events/events.h
    include/dom/events/keyboard_event.h
    include/dom/events/mouse_event.h
    include/dom/events/mouse_multi_wheel_event.h
    include/dom/events/mouse_wheel_event.h
    include/dom/events/mutation_event.h
    include/dom/events/mutation_name_event.h
    include/dom/events/text_event.h
    include/dom/events/ui_event.h
    include/dom/html/html_anchor_element.h
    include/dom/html/html_applet_element.h
    include/dom/html/html_area_element.h
    include/dom/html/html_base_element.h
    include/dom/html/html_basefont_element.h
    include/dom/html/html_body_element.h
    include/dom/html/html_br_element.h
    include/dom/html/html_button_element.h
    include/dom/html/html_collection.h
    include/dom/html/html_directory_element.h
    include/dom/html/html_div_element.h
    include/dom/html/html_dlist_element.h
    include/dom/html/html_document.h
    include/dom/html/html_element.h
    include/dom/html/html_elements.h
    include/dom/html/html_fieldset_element.h
    include/dom/html/html_font_element.h
    include/dom/html/html_form_element.h
    include/dom/html/html_frame_element.h
    include/dom/html/html_frameset_element.h
    include/dom/html/html_head_element.h
    include/dom/html/html_heading_element.h
    include/dom/html/html_hr_element.h
    include/dom/html/html_html_element.h
    include/dom/html/html_iframe_element.h
    include/dom/html/html_image_element.h
    include/dom/html/html_input_element.h
    include/dom/html/html_isindex_element.h
    include/dom/html/html_label_element.h
    include/dom/html/html_legend_element.h
    include/dom/html/html_li_element.h
    include/dom/html/html_link_element.h
    include/dom/html/html_map_element.h
    include/dom/html/html_menu_element.h
    include/dom/html/html_meta_element.h
    include/dom/html/html_mod_element.h
    include/dom/html/html_object_element.h
    include/dom/html/html_olist_element.h
    include/dom/html/html_opt_group_element.h
    include/dom/html/html_option_element.h
    include/dom/html/html_options_collection.h
    include/dom/html/html_paragraph_element.h
    include/dom/html/html_param_element.h
    include/dom/html/html_pre_element.h
    include/dom/html/html_quote_element.h
    include/dom/html/html_script_element.h
    include/dom/html/html_select_element.h
    include/dom/html/html_style_element.h
    include/dom/html/html_table_element.h
    include/dom/html/html_tablecaption_element.h
    include/dom/html/html_tablecell_element.h
    include/dom/html/html_tablecol_element.h
    include/dom/html/html_tablerow_element.h
    include/dom/html/html_tablesection_element.h
    include/dom/html/html_text_area_element.h
    include/dom/html/html_title_element.h
    include/dom/html/html_ulist_element.h)

include_directories(${CMAKE_SOURCE_DIR}/src ${CMAKE_SOURCE_DIR}/include)
# setup any additional libs required by this.
set(DOM_ADDITIONAL_LIBS "")
set(DOM_ADDITIONAL_DIRS "")
set(DOM_ADDITIONAL_DEFS "_BSD_SOURCE" "_DEFAULT_SOURCE")

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

if(PKG_CONFIG_FOUND)
    pkg_check_modules(libparserutils REQUIRED libparserutils)
    list(APPEND DOM_ADDITIONAL_DIRS ${libparserutils_INCLUDEDIR})
    list(APPEND DOM_ADDITIONAL_LIBS ${libparserutils_LIBRARIES})
else()
    find_library(parserutils DOM_LIBPARSEERUTILS_LIB)
    if (DOM_LIBPARSEERUTILS_LIB)
      list(APPEND DOM_ADDITIONAL_LIBS ${DOM_LIBPARSEERUTILS})
    else()
      message(FATAL_ERROR "Parseutils not found")
    endif(DOM_LIBPARSEERUTILS_LIB)
endif(PKG_CONFIG_FOUND)
if(PKG_CONFIG_FOUND)
    pkg_check_modules(libwapcaplet REQUIRED libwapcaplet)
    list(APPEND DOM_ADDITIONAL_DIRS ${libwapcaplet_INCLUDEDIR})
    list(APPEND DOM_ADDITIONAL_LIBS ${libwapcaplet_LIBRARIES})
else()
    find_library(wapcaplet DOM_libwapcaplet_LIB)
    if (DOM_libwapcaplet_LIB)
      list(APPEND DOM_ADDITIONAL_LIBS ${DOM_libwapcaplet_LIB})
    else()
      message(FATAL_ERROR "wapcaple not found")
    endif(HUBBUB_HUBBUB_LIB)
endif(PKG_CONFIG_FOUND)

message("DOM_XML_BINDING = ${DOM_XML_BINDING}")
if(${DOM_XML_BINDING} STREQUAL "Libxml2")
  include(FindLibXml2)
  if(LIBXML2_FOUND)
    #${LIBXML2_INCLUDE_DIR}/libxml2 is a just in case the compiler can't find the header
    list(APPEND DOM_ADDITIONAL_DIRS ${LIBXML2_INCLUDE_DIR} ${LIBXML2_INCLUDE_DIR}/libxml2)
    list(APPEND DOM_ADDITIONAL_LIBS ${LIBXML2_LIBRARIES})
    list(APPEND DOM_ADDITIONAL_DEFS ${LIBXML2_DEFINITIONS})
    set(DOM_ADDITIONAL_C_FILES ${DOM_ADDITIONAL_C_FILES} 
      ${CMAKE_SOURCE_DIR}/bindings/xml/libxml_xmlparser.c
      ${CMAKE_SOURCE_DIR}/bindings/xml/xmlerror.h
      ${CMAKE_SOURCE_DIR}/bindings/xml/xmlparser.h)
    set(DOM_INSTALL_XML "TRUE")
  endif(LIBXML2_FOUND)
endif(${DOM_XML_BINDING} STREQUAL "Libxml2")

if(${DOM_XML_BINDING} STREQUAL "Expat")
  include(FindEXPAT)
  if(EXPAT_FOUND)
    list(APPEND DOM_ADDITIONAL_DIRS ${EXPAT_INCLUDE_DIRS})
    list(APPEND DOM_ADDITIONAL_LIBS ${EXPAT_LIBRARIES})
    set(DOM_ADDITIONAL_C_FILES ${DOM_ADDITIONAL_C_FILES} ${CMAKE_SOURCE_DIR}/bindings/xml/expat_xmlparser.c)
    set(DOM_INSTALL_XML "TRUE")
  endif(EXPAT_FOUND)
endif(${DOM_XML_BINDING} STREQUAL "Expat")

if(DOM_INSTALL_XML)
  install(FILES ${CMAKE_SOURCE_DIR}/bindings/xml/xmlerror.h ${CMAKE_SOURCE_DIR}/bindings/xml/xmlparser.h
     DESTINATION include/dom/bindings/xml) 
endif(DOM_INSTALL_XML)

if(DOM_WITH_LIBHUBBUB_BINDING)
  if(PKG_CONFIG_FOUND)
    pkg_check_modules(libhubbub REQUIRED libhubbub)
    if(libhubbub_FOUND)
      list(APPEND DOM_ADDITIONAL_DIRS ${libhubbub_INCLUDEDIR})
      list(APPEND DOM_ADDITIONAL_LIBS ${libhubbub_LIBRARIES})
      set(DOM_ADDITIONAL_C_FILES ${DOM_ADDITIONAL_C_FILES} ${CMAKE_SOURCE_DIR}/bindings/hubbub/parser.c)
    else()
      message(FATAL_ERROR "libhubbub is required if DOM_WITH_LIBHUBBUB_BINDING is ON")
    endif(libhubbub_FOUND)
  else()
    find_library(libhubbub DOM_libhubbub_LIB)
    if (DOM_libhubbub_LIB)
      list(APPEND DOM_ADDITIONAL_LIBS ${DOM_libhubbub_LIB})
      set(DOM_ADDITIONAL_C_FILES ${DOM_ADDITIONAL_C_FILES} ${CMAKE_SOURCE_DIR}/bindings/hubbub/parser.c)
    else()
      message(FATAL_ERROR "libhubbub is required if DOM_WITH_LIBHUBBUB_BINDING is ON")
    endif(DOM_libhubbub_LIB)
  endif(PKG_CONFIG_FOUND)
  install(FILES ${CMAKE_SOURCE_DIR}/bindings/hubbub/errors.h
      ${CMAKE_SOURCE_DIR}/bindings/hubbub/parser.h
      DESTINATION include/dom/bindings/hubbub) 
endif(DOM_WITH_LIBHUBBUB_BINDING)

if(BUILD_SHARED_LIBS)
   if(CMAKE_RC_COMPILER)
      # Make .rc file  
      set(DOM_RC_CONTENTS "1 VERSIONINFO
FILEVERSION     ${DOM_VERSION_MAJOR},${DOM_VERSION_MINOR},${DOM_VERSION_PATCH},0
PRODUCTVERSION  ${DOM_VERSION_MAJOR},${DOM_VERSION_MINOR},${DOM_VERSION_PATCH},0
BEGIN
  BLOCK \"StringFileInfo\"
  BEGIN
    BLOCK \"040904E4\"
    BEGIN
      VALUE \"CompanyName\", \"${DOM_AUTHOR}\"
      VALUE \"FileDescription\", \"${DOM_DESCRIPTION}\"
      VALUE \"FileVersion\", \"${DOM_VERSION_MAJOR}.${DOM_VERSION_MINOR}.${DOM_VERSION_PATCH}\"
      VALUE \"InternalName\", \"${CMAKE_SHARED_MODULE_PREFIX}${DOM_COMPONENT_NAME}\"
      VALUE \"LegalCopyright\", \"${DOM_COPYRIGHT}\"
      VALUE \"OriginalFilename\", \"${CMAKE_SHARED_MODULE_PREFIX}${DOM_COMPONENT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}\"
      VALUE \"ProductName\", \"${DOM_COMPONENT_NAME}\"
      VALUE \"ProductVersion\", \"${DOM_VERSION_MAJOR}.${DOM_VERSION_MINOR}.${DOM_VERSION_PATCH}\"
    END
  END
  BLOCK \"VarFileInfo\"
  BEGIN
    VALUE \"Translation\", 0x409, 1252
  END
END")
    FILE(WRITE ${DOM_BUILD_DIR}/lib${DOM_COMPONENT_NAME}.rc "${DOM_RC_CONTENTS}")
    add_library(libdom_shared SHARED ${DOM_PUBLIC_HEADER_FILES} ${DOM_ADDITIONAL_C_FILES} ${DOM_BUILD_DIR}/lib${DOM_COMPONENT_NAME}.rc)
  else()
    add_library(libdom_shared SHARED ${DOM_PUBLIC_HEADER_FILES} ${DOM_ADDITIONAL_C_FILES} SOVERSION ${DOM_SOVERSION})
  endif()
  set_target_properties(libdom_shared PROPERTIES OUTPUT_NAME ${DOM_COMPONENT_NAME})
  if(DOM_ADDITIONAL_LIBS)
    target_link_libraries(libdom_shared PUBLIC ${DOM_ADDITIONAL_LIBS}) 
  endif(DOM_ADDITIONAL_LIBS)
  if(DOM_ADDITIONAL_DIRS)
    target_include_directories(libdom_shared PUBLIC ${DOM_ADDITIONAL_DIRS})
  endif(DOM_ADDITIONAL_DIRS)
  if(DOM_ADDITIONAL_DEFS)
    target_include_directories(libdom_shared PUBLIC ${DOM_ADDITIONAL_DEFS})
  endif(DOM_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libdom_shared PRIVATE ${DOM_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libdom_shared
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_SHARED_LIBS)
if(BUILD_STATIC_LIBS)
  add_library(libdom_static STATIC ${DOM_PUBLIC_HEADER_FILES} ${DOM_ADDITIONAL_C_FILES})
  set_target_properties(libdom_static PROPERTIES OUTPUT_NAME ${DOM_COMPONENT_NAME})
  if(DOM_ADDITIONAL_LIBS)
    target_link_libraries(libdom_static PUBLIC ${DOM_ADDITIONAL_LIBS}) 
  endif(DOM_ADDITIONAL_LIBS)
  if(DOM_ADDITIONAL_DIRS)
    target_include_directories(libdom_static PUBLIC ${DOM_ADDITIONAL_DIRS})
  endif(DOM_ADDITIONAL_DIRS)
  if(DOM_ADDITIONAL_DEFS)
    target_include_directories(libdom_static PUBLIC ${DOM_ADDITIONAL_DEFS})
  endif(DOM_ADDITIONAL_DEFS)
  if(CMAKE_COMPILER_IS_GNUCC)
    target_compile_options(libdom_static PRIVATE ${DOM_WARNFLAGS})
  endif(CMAKE_COMPILER_IS_GNUCC)
  install(TARGETS libdom_static
      RUNTIME DESTINATION bin
      LIBRARY DESTINATION lib
      ARCHIVE DESTINATION lib)
endif(BUILD_STATIC_LIBS)

#from: https://cmake.org/pipermail/cmake/2010-October/040246.html
#This installs the headers into our hierarchy in the include file.
foreach(DOM_HEADER ${DOM_PUBLIC_HEADER_FILES})
   string(REGEX MATCH "(.*)[/\\]" DOM_DIR ${DOM_HEADER})
   install(FILES ${DOM_HEADER} DESTINATION ${DOM_DIR})
endforeach(DOM_HEADER ${DOM_PUBLIC_HEADER_FILES})

if(DOM_WITH_PKGCONFIG_SUPPORT)
  set(DOM_PC ${DOM_BUILD_DIR}/lib${DOM_COMPONENT_NAME}.pc)

  # This stuff is necessary to ensure that the dependency list
  # in our .pc file is in the proper format
#Make pkg-config
  set(DOM_EXC "${CMAKE_INSTALL_PREFIX}")
  set(DOM_INC "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(DOM_LIB "${CMAKE_INSTALL_FULL_LIBDIR}")
  if("${DOM_EXC}" STREQUAL "${CMAKE_INSTALL_PREFIX}")
    set(DOM_EXC "\${prefix}")
  endif()
  if ("${DOM_INC}" STREQUAL "${CMAKE_INSTALL_PREFIX}/include")
    set(DOM_INC "\${prefix}/include")
  endif()
  if ("${DOM_LIB}" STREQUAL "${CMAKE_INSTALL_PREFIX}/lib")
    set(DOM_LIB "\${exec_prefix}/lib")
  endif()
  set(DOM_DEPS "")
  foreach(DOM_LIB_DEP ${DOM_LIB_DEPEND_LIBS})
    set(DOM_DEPS "${DOM_DEPS} -l${DOM_LIB_DEP}")
  endforeach(DOM_LIB_DEP)

  set(DOM_PKGCONFIG_CONTENTS "prefix=${CMAKE_INSTALL_PREFIX}
exec_prefix=${DOM_EXC}
libdir=${DOM_LIB}
includedir=${DOM_INC}

Name: lib${DOM_COMPONENT_NAME}
Description: ${DOM_DESCRIPTION}
URL: http://www.netsurf-browser.org/projects/${DOM_COMPONENT_NAME}/
Version: ${DOM_SOVERSION}
Requires: libparserutils, libwapcaplet
Cflags: -I\${includedir}
Libs: -L\${libdir} -l${DOM_COMPONENT_NAME}
Libs.private: ${DOM_DEPS}
")
  file(WRITE ${DOM_PC} ${DOM_PKGCONFIG_CONTENTS})
  install(FILES ${DOM_PC} DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif(DOM_WITH_PKGCONFIG_SUPPORT)

