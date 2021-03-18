# Copyright (C) 2007-2015 LuaDist.
# Created by Peter Drahoš, Peter Kapec
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.
cmake_minimum_required(VERSION 3.10)

project ( lua )
include ( cmake/dist.cmake )

## CONFIGURATION
# Default configuration (we assume POSIX by default)
set ( LUA_PATH "LUA_PATH" CACHE STRING "Environment variable to use as package.path." )
set ( LUA_CPATH "LUA_CPATH" CACHE STRING "Environment variable to use as package.cpath." )
set ( LUA_INIT "LUA_INIT" CACHE STRING "Environment variable for initial script." )

option ( LUA_USE_C89 "Use only C89 features." OFF )
option ( LUA_USE_RELATIVE_LOADLIB "Use modified loadlib.c with support for relative paths on posix systems." ON )

option ( LUA_COMPAT_5_1 "Enable backwards compatibility options with lua-5.1." ON )
option ( LUA_COMPAT_5_2 "Enable backwards compatibility options with lua-5.2." ON )

#2DO: LUAI_* and LUAL_* settings, for now defaults are used.
set ( LUA_DIRSEP "/" )
set ( LUA_MODULE_SUFFIX ${CMAKE_SHARED_MODULE_SUFFIX} )
set ( LUA_LDIR ${INSTALL_LMOD} )
set ( LUA_CDIR ${INSTALL_CMOD} )

if ( LUA_USE_RELATIVE_LOADLIB )
  # This will set up relative paths to lib
  string ( REGEX REPLACE "[^!/]+" ".." LUA_DIR "!/${INSTALL_BIN}/" )
else ( )
  # Direct path to installation
  set ( LUA_DIR ${CMAKE_INSTALL_PREFIX} CACHE STRING "Destination from which modules will be resolved. See INSTALL_LMOD and INSTALL_CMOD." )
endif ( )

set ( LUA_PATH_DEFAULT "./?.lua;${LUA_DIR}${LUA_LDIR}/?.lua;${LUA_DIR}${LUA_LDIR}/?/init.lua" )
set ( LUA_CPATH_DEFAULT "./?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/loadall${LUA_MODULE_SUFFIX}" )

if ( WIN32 AND NOT CYGWIN )
  # Windows systems
  option ( LUA_USE_WINDOWS "Windows specific build." ON )
  option ( LUA_BUILD_WLUA "Build wLua interpretter without console output." ON )
  option ( LUA_BUILD_AS_DLL "Build Lua library as Dll." ${BUILD_SHARED_LIBS} )

  # Paths (Double escapes needed)
  set ( LUA_DIRSEP "\\\\" )
  string ( REPLACE " /" ${LUA_DIRSEP} LUA_DIR "${LUA_DIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_LDIR "${LUA_LDIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_CDIR "${LUA_CDIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_PATH_DEFAULT "${LUA_PATH_DEFAULT}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_CPATH_DEFAULT "${LUA_CPATH_DEFAULT}" )
else ( )
  # Posix systems (incl. Cygwin)
  option ( LUA_USE_POSIX "Use POSIX features." ON )
  option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )
  # Apple and Linux specific
  if ( LINUX OR APPLE )
    option ( LUA_USE_AFORMAT "Assume 'printf' handles 'aA' specifiers" ON )
  endif ( )

  if (APPLE)
    option(LUA_USE_MACOSX "" ON)
  endif(APPLE)
endif ( )
option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )

# SETUP
# Optional libraries
find_package ( Readline )
message("READLINE_FOUND:${READLINE_FOUND} LUA_USE_READLINE:${LUA_USE_READLINE} ${READLINE_LIBRARY} ${READLINE_INCLUDE_DIR}")
if ( READLINE_FOUND )
  option ( LUA_USE_READLINE "Use readline in the Lua CLI." ON )
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DLUA_USE_READLINE")  
endif ( )

# Setup needed variables and libraries
if ( LUA_USE_POSIX )
  # On POSIX Lua links to standard math library "m"
  list ( APPEND LIBS m )
endif ( )

if(APPLE)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fembed-bitcode -DLUA_USE_MACOSX -DMAKE_LIB")  
  set(CMAKE_MACOSX_RPATH ON)
  SET(CMAKE_SKIP_BUILD_RPATH TRUE)
  SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
  SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
  set(CMAKE_INSTALL_RPATH "@loader_path")
endif(APPLE)

# if ( LUA_USE_DLOPEN )
#   # Link to dynamic linker library "dl"
# 	find_library ( DL_LIBRARY NAMES dl )
# 	if ( DL_LIBRARY )
#     list ( APPEND LIBS ${DL_LIBRARY} )
# 	endif ( )
# endif ( )

if ( READLINE_FOUND )
  # Add readline
  include_directories ( ${READLINE_INCLUDE_DIR} )
  list ( APPEND LIBS ${READLINE_LIBRARY} )
endif ( )

# ## SOURCES
# # Generate luaconf.h
# configure_file ( ./luaconf.h.in ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h )

# Sources and headers
include_directories ( src ${CMAKE_CURRENT_BINARY_DIR} )

file(GLOB_RECURSE SRC_CORE 
    ${CMAKE_CURRENT_SOURCE_DIR}/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/*.c
)

# file(GLOB_RECURSE SRC_CORE_EXCLUDE
#     ${CMAKE_CURRENT_SOURCE_DIR}/testes/*
#     ${CMAKE_CURRENT_SOURCE_DIR}/luac.c
# )
# list(REMOVE_ITEM SRC_CORE ${SRC_CORE_EXCLUDE})
# or
list(FILTER SRC_CORE EXCLUDE REGEX ".*testes.*")
list(FILTER SRC_CORE EXCLUDE REGEX ".*luac.c")

source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} SRC_CORE)

set ( SRC_LUA  ./luaexe.c )
set ( SRC_LUAC ./luac.c )

# if ( LUA_USE_RELATIVE_LOADLIB )
#   # Use modified loadlib
#   list ( APPEND SRC_LIB ./loadlib_rel.c )
# else ( )
#   list ( APPEND SRC_LIB ./loadlib.c )
# endif ( )

## BUILD
# Create lua library

# add_library ( liblua_static STATIC ${SRC_CORE} ${SRC_LIB} ${LUA_DLL_RC} )
add_library ( liblua ${SRC_CORE} ${SRC_LIB} ${LUA_DLL_RC} )
target_link_libraries ( liblua ${LIBS})

set_target_properties ( liblua PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )
if ( LUA_BUILD_AS_DLL )
  set_target_properties ( liblua PROPERTIES COMPILE_DEFINITIONS LUA_BUILD_AS_DLL )
endif ()

add_executable ( lua ${SRC_LUA})
target_link_libraries ( lua liblua  ${LIBS})

add_executable ( luac ${SRC_LUAC})
target_link_libraries ( luac liblua ${LIBS})

# readline
if(READLINE_FOUND)
  target_link_libraries ( liblua readline)
  target_link_libraries ( lua readline)
endif(READLINE_FOUND)

if(APPLE)
  set_target_properties(liblua PROPERTIES 
    INSTALL_RPATH "@rpath;@rpath/libs;@executable_path;@executable_path/libs;@executable_path/Frameworks;@loader_path;@loader_path/libs;@loader_path/Frameworks"
  )

  set_target_properties(lua PROPERTIES
    MACOSX_BUNDLE_GUI_IDENTIFIER "com.cn.lua"
    MACOSX_BUNDLE_BUNDLE_VERSION "1.0"
    INSTALL_RPATH "@rpath;@rpath/libs;@executable_path;@executable_path/libs;@executable_path/Frameworks;@loader_path;@loader_path/libs;@loader_path/Frameworks"
  )
  # # 复制文件 https://www.cnblogs.com/JoyPoint/p/11629521.html
  # add_custom_command(TARGET lua
  #   POST_BUILD
  #   COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/res 
  #   COMMAND ${CMAKE_COMMAND} -E make_directory "Heads"
  # )
endif(APPLE)

# add_library ( lua_static STATIC ${SRC_CORE})

# On windows a variant of the lua interpreter without console output needs to be built
if ( LUA_BUILD_WLUA )
  add_executable ( wlua WIN32 ./wmain.c ${SRC_LUA} ./lua.rc )
  target_link_libraries ( wlua liblua )
  install_executable ( wlua )
endif ( )

if(NOT IOS AND NOT ANDROID)
  install_executable ( lua )
  install_library ( liblua )
  install_data ( README.md )
  #install_lua_module ( strict etc/strict.lua )
  install_header ( lua.h lualib.h lauxlib.h lua.hpp ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h )
  install_doc ( doc/ )
  install_foo ( etc/ )
  #install_test ( test/ )

endif(NOT IOS AND NOT ANDROID)
