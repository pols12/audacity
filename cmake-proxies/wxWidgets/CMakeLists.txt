
add_library( ${TARGET} INTERFACE )
add_library( ${symbol} ALIAS ${TARGET} )

def_vars()

message( STATUS "========== Configuring ${name} ==========" )

option( use_system_${name} "Use ${name} system library if available" ${prefer_system_libs} )
if( use_system_${name} )
   find_package(wxWidgets)
endif()

if( wxWidgets_FOUND )
   message( STATUS "Using '${name}' system library" )

   if( wxWidgets_INCLUDE_DIRS_NO_SYSTEM )
      set( INCLUDES
         INTERFACE
            ${wxWidgets_INCLUDE_DIRS_NO_SYSTEM}
      )
   else()
      set( INCLUDES
         INTERFACE
            ${wxWidgets_INCLUDE_DIRS}
      )
   endif() 

   set( DEFINES
      INTERFACE
         ${wxWidgets_DEFINITIONS}
         $<$<CONFIG:Debug>:
            ${wxWidgets_DEFINITIONS_DEBUG}
         >
   )

   set( LINKDIRS
      INTERFACE
         $<$<PLATFORM_ID:Windows>:
            ${wxWidgets_LIB_DIR}
         >
   )

   set( LIBRARIES
      INTERFACE
         ${wxWidgets_LIBRARIES}
         $<$<NOT:$<PLATFORM_ID:Windows>>:
            z
         >
   )

   set( toolkit "${wxWidgets_LIBRARIES}" )
else()
   message( STATUS "Using local '${name}' library" )

   set( use_system_${name} OFF CACHE BOOL "Prefer ${name} system library if available" FORCE )

   set( WXWIN $ENV{WXWIN} )
   if( "${WXWIN}" STREQUAL "" )
      # XXX: Look into importing instead of adding to this project
      set( WXWIN "${_INTDIR}/wxwidgets" )
   endif()

   if( NOT EXISTS "${WXWIN}" )
      find_package( Git )
      if( NOT GIT_FOUND )
         message( FATAL_ERROR "Git is needed to clone wxWidgets" )
      endif()

      execute_process(
         COMMAND
            ${GIT_EXECUTABLE} clone
                              --depth 1
                              --single-branch
                              --recurse-submodules
                              https://github.com/audacity/wxwidgets
                              "${WXWIN}"
      )
   endif()

   if( CMAKE_SYSTEM_NAME MATCHES "Windows" )
      # Want accessibility
      set( wxUSE_ACCESSIBILITY YES )

      # Windows requires it due to missing "#include" directives
      set( wxBUILD_PRECOMP YES )
   elseif( CMAKE_SYSTEM_NAME MATCHES "Darwin" )
      # Want accessibility
      set( wxUSE_ACCESSIBILITY YES )

      # Causes problems on OSX, so turn it off
      set( wxBUILD_PRECOMP NO )
   elseif( CMAKE_SYSTEM_NAME MATCHES "Linux|FreeBSD" )
      # Doesn't yet have accessbility
      set( wxUSE_ACCESSIBILITY NO )

      # Linux can go either way, so might as well use it
      set( wxBUILD_PRECOMP YES )
   endif()

   # Just to be consistent with Audacity
   set( wxBUILD_CXX_STANDARD "14" )

   # Pull in wxWidgets
   add_subdirectory( ${WXWIN} ${WXWIN} )

   # And rearrange the folder structure
   set_dir_folder( ${WXWIN} "wxWidgets" )

   set( INCLUDES
      $<$<STREQUAL:"${wxUSE_ZLIB}","builtin">:
         ${WXWIN}/src/zlib
      >
   )

   set( DEFINES
      WXUSINGDLL
   )

   # Do NOT split the generator expressions across multiple lines here.
   # CMake appears to have a bug and doesn't see to handle it correctly
   # for target link libraries.
   set( LIBRARIES
      adv
      base
      core
      html
      net
      qa
      xml
      $<$<STREQUAL:"${wxUSE_EXPAT}","builtin">:wxexpat>
      $<$<STREQUAL:"${wxUSE_LIBJPEG}","builtin">:wxjpeg>
      $<$<STREQUAL:"${wxUSE_LIBPNG}","builtin">:wxpng>
      $<$<STREQUAL:"${wxUSE_LIBTIFF}","builtin">:wxtiff>
      $<$<STREQUAL:"${wxUSE_REGEX}","builtin">:wxregex>
      $<$<STREQUAL:"${wxUSE_ZLIB}","builtin">:wxzlib>
   )

   if( CMAKE_SYSTEM_NAME MATCHES "Darwin" )
      # When accessibility is enabled, the build will fail in "wx/chkconf.h"
      # since wxRegex compile defines do not include __WXOSX_COCOA__.  So,
      # add it here.
      target_compile_definitions( wxregex PRIVATE "__WXOSX_COCOA__" )
   endif()

   set( toolkit "${wxBUILD_TOOLKIT}" )
endif()

if( "${toolkit}" MATCHES ".*gtk2.*" )
   set( gtk gtk+-2.0 )
   set( glib glib-2.0 )
   set( wxTOOLKIT "GTK2" CACHE INTERNAL "" )
   set( wxIS_GTK YES CACHE INTERNAL "" )
elseif( "${toolkit}" MATCHES ".*gtk3.*" )
   set( gtk gtk+-3.0 )
   set( glib glib-2.0 )
   set( wxTOOLKIT "GTK3" CACHE INTERNAL "" )
   set( wxIS_GTK YES CACHE INTERNAL "" )
elseif( "${toolkit}" MATCHES ".*gtk4.*" )
   set( gtk gtk+-4.0 )
   set( glib glib-2.0 )
   set( wxTOOLKIT "GTK4" CACHE INTERNAL "" )
   set( wxIS_GTK YES CACHE INTERNAL "" )
elseif( "${toolkit}" MATCHES ".*msw.*" )
   set( wxTOOLKIT "MSW" CACHE INTERNAL "" )
   set( wxIS_WIN YES CACHE INTERNAL "" )
elseif( "${toolkit}" MATCHES ".*osx.*" )
   set( wxTOOLKIT "MAC" CACHE INTERNAL "" )
   set( wxIS_MAC YES CACHE INTERNAL "" )
else()
   message( FATAL_ERROR "Unrecognized wxWidgets toolkit" )
endif()
 
if( "${wxTOOLKIT}" MATCHES "GTK." )
   pkg_check_modules( GTK REQUIRED IMPORTED_TARGET GLOBAL ${gtk} )
   pkg_check_modules( GLIB REQUIRED IMPORTED_TARGET GLOBAL ${glib} )
endif()

target_include_directories( ${TARGET} INTERFACE ${INCLUDES} )
target_compile_definitions( ${TARGET} INTERFACE ${DEFINES} )
target_link_directories( ${TARGET} INTERFACE ${LINKDIRS} )
target_link_libraries( ${TARGET} INTERFACE ${LIBRARIES} )

INSTALL( TARGETS ${TARGET} DESTINATION ${_LIBDIR} )
