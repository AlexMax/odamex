# use unquoted #defines
if(COMMAND cmake_policy)
  cmake_policy(SET CMP0005 NEW)
endif(COMMAND cmake_policy)

# Client flags
if(NOT MSVC)
  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.2)
      if(target_arch STREQUAL "i386" OR target_arch STREQUAL "amd64")        
        set(CMAKE_CXX_FLAGS_RELEASE "-O3 -march=native")
      else()
        set(CMAKE_CXX_FLAGS_RELEASE "-O3")
      endif()
    else(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.2)
      if(APPLE)
        set(CMAKE_CXX_FLAGS_RELEASE "-O3 -faltivec")
      endif()
    endif()
  endif()
else()
  set(CMAKE_CXX_FLAGS_RELEASE "/Ox /Oi /arch:SSE2") 
endif()


global_compile_options()

# #define CLIENT_APP for use in the source code
add_definitions(-DCLIENT_APP)

# Common
set(COMMON_DIR ../common)
file(GLOB COMMON_HEADERS ${COMMON_DIR}/*.h)
file(GLOB COMMON_SOURCES ${COMMON_DIR}/*.cpp)

# Client
set(CLIENT_DIR sdl src)
file(GLOB CLIENT_HEADERS sdl/*.h src/*.h)
file(GLOB CLIENT_SOURCES sdl/*.cpp src/*.cpp)
if(WIN32)
  set(CLIENT_WIN32_RESOURCES sdl/client.rc)
endif()

# git describe
set_source_files_properties(${COMMON_DIR}/version.cpp PROPERTIES COMPILE_FLAGS -DGIT_DESCRIBE=\\"${GIT_DESCRIBE}\\")

# Platform definitions
define_platform()

# Client definitions
include_directories(${COMMON_DIR} ${CLIENT_DIR})

# Textscreen
set(TEXTSCREEN_LIBRARY "textscreen")
set(TEXTSCREEN_DIR ../libraries/textscreen)
set(TEXTSCREEN_HEADERS ${TEXTSCREEN_DIR}/txt_sdl.h ${TEXTSCREEN_DIR}/txt_main.h)
set(TEXTSCREEN_SOURCES ${TEXTSCREEN_DIR}/txt_sdl.c)

include_directories(${TEXTSCREEN_DIR})
add_library(${TEXTSCREEN_LIBRARY} STATIC ${TEXTSCREEN_SOURCES} ${TEXTSCREEN_HEADERS})
message(STATUS "Textscreen will be statically linked.")

# Make sure we don't find libraries that are in Program Files/Odamex
if(WIN32)
  list(REMOVE_ITEM CMAKE_SYSTEM_PREFIX_PATH ${CMAKE_INSTALL_PREFIX})
endif()

# SDL configuration
# try to use SDL 2.0 first but fall back on SDL 1.2 if necessary
# ignore SDL 2.0 if -DUSE_SDL12=1 is specified
if(NOT USE_SDL12)
  find_package(SDL2)
  if(NOT SDL2_FOUND AND WIN32)
    message(STATUS "Downloading SDL2 Development Library...")
    if(MSVC)
      file(DOWNLOAD
        "https://www.libsdl.org/release/SDL2-devel-2.0.12-VC.zip"
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2-VC.zip"
        EXPECTED_HASH SHA256=00c55a597cebdb9a4eb2723f2ad2387a4d7fd605e222c69b46099b15d5d8b32d)
      execute_process(COMMAND "${CMAKE_COMMAND}" -E tar xf
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2-VC.zip"
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
      set(SDL2_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/SDL2-2.0.12/include")
      set(SDL2_LIBRARY "${CMAKE_CURRENT_BINARY_DIR}/SDL2-2.0.12/lib/x64/SDL2.lib")
      set(SDL2MAIN_LIBRARY "${CMAKE_CURRENT_BINARY_DIR}/SDL2-2.0.12/lib/x64/SDL2main.lib")
    else()
      file(DOWNLOAD
        "https://www.libsdl.org/release/SDL2-devel-2.0.12-mingw.tar.gz"
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2-mingw.tar.gz"
        EXPECTED_HASH SHA256=e614a60f797e35ef9f3f96aef3dc6a1d786de3cc7ca6216f97e435c0b6aafc46)
      execute_process(COMMAND "${CMAKE_COMMAND}" -E tar xf
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2-mingw.tar.gz"
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
    endif()
    find_package(SDL2)
  endif()

  find_package(SDL2_mixer)
  if(NOT SDL2_MIXER_FOUND AND WIN32)
    message(STATUS "Downloading SDL2_mixer Development Library...")
    if(MSVC)
      file(DOWNLOAD
        "https://www.libsdl.org/projects/SDL_mixer/release/SDL2_mixer-devel-2.0.4-VC.zip"
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2_mixer-VC.zip"
        EXPECTED_HASH SHA256=258788438b7e0c8abb386de01d1d77efe79287d9967ec92fbb3f89175120f0b0)
      execute_process(COMMAND "${CMAKE_COMMAND}" -E tar xf
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2_mixer-VC.zip"
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
      set(SDL2_MIXER_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/SDL2_mixer-2.0.4/include")
      set(SDL2_MIXER_LIBRARY "${CMAKE_CURRENT_BINARY_DIR}/SDL2_mixer-2.0.4/lib/x64/SDL2_mixer.lib")
    else()
      file(DOWNLOAD
        "https://www.libsdl.org/projects/SDL_mixer/release/SDL2_mixer-devel-2.0.4-mingw.tar.gz"
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2_mixer-mingw.tar.gz"
        EXPECTED_HASH SHA256=14250b2ade20866c7b17cf1a5a5e2c6f3920c443fa3744f45658c8af405c09f1)
      execute_process(COMMAND "${CMAKE_COMMAND}" -E tar xf
        "${CMAKE_CURRENT_BINARY_DIR}/SDL2_mixer-mingw.tar.gz"
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
    endif()
    find_package(SDL2_mixer)
  endif()

  if(SDL2_FOUND AND SDL2_MIXER_FOUND)
    set(SDL_VERSION "2")
    message(STATUS "Using SDL library version 2.0.")
    include_directories(${SDL2_INCLUDE_DIR})
    include_directories(${SDL2_MIXER_INCLUDE_DIR})
    set(SDL_LINK_LIBRARIES ${SDL2_LIBRARY} ${SDL2_MIXER_LIBRARY})
  endif()
endif()

if(NOT SDL2_FOUND OR NOT SDL2_MIXER_FOUND)
  find_package(SDL)
  find_package(SDL_mixer)
  if(SDL_FOUND AND SDLMIXER_FOUND)
    set(SDL_VERSION "1")
    message(STATUS "Using SDL library version 1.2.")
    include_directories(${SDL_INCLUDE_DIR})
    include_directories(${SDL_MIXER_INCLUDE_DIR})
    set(SDL_LINK_LIBRARIES ${SDL_LIBRARY} ${SDL_MIXER_LIBRARY})
  else()
    if(NOT SDL_FOUND)
      message(WARNING "SDL not found, client target not generated.")
	endif()
    if(NOT SDLMIXER_FOUND)
      message(WARNING "SDL_mixer not found, client target not generated.")
    endif()
    SET(BUILD_CLIENT OFF CACHE BOOL "" FORCE)
  endif()
endif()

# X11 configuration
if(UNIX AND NOT APPLE)
  if(X11_FOUND)
    include_directories(${X11_INCLUDE_DIRS})
  endif()
endif()

# zlib configuration
find_package(ZLIB)
if(NOT ZLIB_FOUND)
  set(USE_INTREE_ZLIB)
  set(ZLIB_LIBRARY "z")
  set(ZLIB_LIBRARIES ${ZLIB_LIBRARY})
  set(ZLIB_DIR ../libraries/zlib/)
  set(ZLIB_INCLUDE_DIR ${ZLIB_DIR})
  set(ZLIB_INCLUDE_DIRS ${ZLIB_DIR})
  file(GLOB ZLIB_HEADERS ${ZLIB_DIR}/*.h)
  file(GLOB ZLIB_SOURCES ${ZLIB_DIR}/*.c)
  include_directories(${ZLIB_DIR})
  add_library(${ZLIB_LIBRARY} STATIC ${ZLIB_SOURCES} ${ZLIB_HEADERS})
  message(STATUS "zlib will be built and staticaly linked when compiling the client application.")
endif()

include_directories(${ZLIB_INCLUDE_DIRS})

# libpng configuration
find_package(PNG)
if(NOT PNG_FOUND)
  set(USE_INTREE_LIBPNG)
  set(PNG_LIBRARY "png")
  set(PNG_LIBRARIES ${PNG_LIBRARY})
  set(PNG_DIR ../libraries/libpng/)
  set(PNG_INCLUDE_DIR ${PNG_DIR})
  set(PNG_INCLUDE_DIRS ${PNG_DIR})
  file(GLOB PNG_HEADERS ${PNG_DIR}/*.h)
  file(GLOB PNG_SOURCES ${PNG_DIR}/*.c)
  include_directories(${LIBPNG_DIR})
  add_library(${PNG_LIBRARY} STATIC ${PNG_SOURCES} ${PNG_HEADERS})
  message(STATUS "libpng will be built and staticaly linked when compiling the client application.")
  add_definitions(-DUSE_PNG)
endif()

include_directories(${PNG_INCLUDE_DIRS})
add_definitions(-DUSE_PNG)

# PortMidi configuration

# PortMidi
if(ENABLE_PORTMIDI)
  find_package(PortMidi)
  if(NOT PORTMIDI_FOUND)
    # build the PortMidi library
    set(PORTMIDI_LIBRARY "portmidi")
    set(PORTMIDI_LIBRARIES ${PORTMIDI_LIBRARY})
    set(PORTMIDI_DIR ../libraries/portmidi)
    set(PORTMIDI_INCLUDE_DIR ${PORTMIDI_DIR}/pm_common/ ${PORTMIDI_DIR}/porttime/)  

    if(APPLE)
      find_library(COREFOUNDATION_LIBRARY CoreFoundation)
      find_library(COREAUDIO_LIB CoreAudio)
      find_library(COREMIDI_LIB CoreMIDI)
      find_library(CORESERVICES_LIB CoreServices)

      set(PM_NEEDED_LIBS ${COREAUDIO_LIB} ${COREFOUNDATION_LIB} ${COREMIDI_LIB} ${CORESERVICES_LIB} CACHE INTERNAL "")
      file(GLOB PORTMIDI_PLATFORM_HEADERS ${PORTMIDI_DIR}/pm_mac/*.h)
      file(GLOB PORTMIDI_PLATFORM_SOURCES ${PORTMIDI_DIR}/pm_mac/*.c)
      list(APPEND PORTMIDI_PLATFORM_SOURCES ${PORTMIDI_DIR}/porttime/ptmacosx_mach.c)
    elseif(WIN32)
      set(PM_NEEDED_LIBS winmm.lib)
      file(GLOB PORTMIDI_PLATFORM_HEADERS ${PORTMIDI_DIR}/pm_win/*.h)
      file(GLOB PORTMIDI_PLATFORM_SOURCES ${PORTMIDI_DIR}/pm_win/*.c)
      list(APPEND PORTMIDI_PLATFORM_SOURCES ${PORTMIDI_DIR}/porttime/ptwinmm.c)
    elseif(UNIX)
      set(LINUX_FLAGS "-DPMALSA")
      set(PM_NEEDED_LIBS pthread asound)
      file(GLOB PORTMIDI_PLATFORM_HEADERS ${PORTMIDI_DIR}/pm_linux/*.h)
      file(GLOB PORTMIDI_PLATFORM_SOURCES ${PORTMIDI_DIR}/pm_linux/*.c)
      list(APPEND PORTMIDI_PLATFORM_SOURCES ${PORTMIDI_DIR}/porttime/ptlinux.c)
    endif()

    file(GLOB PORTMIDI_HEADERS ${PORTMIDI_DIR}/pm_common/*.h)
    list(APPEND PORTMIDI_HEADERS ${PORTMIDI_DIR}/porttime/porttime.h)
    list(APPEND PORTMIDI_HEADERS ${PORTMIDI_PLATFORM_HEADERS})

    file(GLOB PORTMIDI_SOURCES ${PORTMIDI_DIR}/pm_common/*.c)
    list(APPEND PORTMIDI_SOURCES ${PORTMIDI_DIR}/porttime/porttime.c)
    list(APPEND PORTMIDI_SOURCES ${PORTMIDI_PLATFORM_SOURCES})

    add_library(${PORTMIDI_LIBRARY} STATIC ${PORTMIDI_SOURCES} ${PORTMIDI_HEADERS})
    target_link_libraries(${PORTMIDI_LIBRARY} ${PM_NEEDED_LIBS})
    message(STATUS "portmidi will be built and staticaly linked when compiling the client application.")
  endif()

  add_definitions(-DPORTMIDI)
  include_directories(${PORTMIDI_INCLUDE_DIR})
endif()

# Find Mac frameworks
if(APPLE)
  find_library(CARBON_LIBRARY Carbon)
  find_library(AUDIOTBOX_LIBRARY AudioToolbox)
  find_library(AUDIOUNIT_LIBRARY AudioUnit)
  set(APPLE_FRAMEWORKS ${CARBON_LIBRARY} ${AUDIOTBOX_LIBRARY}
    ${AUDIOUNIT_LIBRARY})
endif()

# Set up FHS installation path
if(NOT APPLE AND NOT WIN32)
  add_definitions(-DINSTALL_PREFIX="${CMAKE_INSTALL_PREFIX}")
endif()

# Client target
if(SDL_VERSION)
  add_executable(odamex MACOSX_BUNDLE WIN32
    ${COMMON_SOURCES} ${COMMON_HEADERS}
    ${CLIENT_SOURCES} ${CLIENT_HEADERS} ${CLIENT_WIN32_RESOURCES})
  target_link_libraries(odamex ${TEXTSCREEN_LIBRARY})
  target_link_libraries(odamex ${SDL_LINK_LIBRARIES})

  target_link_libraries(odamex ${ZLIB_LIBRARY})
  target_link_libraries(odamex ${PNG_LIBRARY} ${ZLIB_LIBRARY})

  if(ENABLE_PORTMIDI)
    target_link_libraries(odamex ${PORTMIDI_LIBRARY})
  endif()

  if(WIN32)
    target_link_libraries(odamex winmm wsock32 Shlwapi)
  elseif(APPLE)
    target_link_libraries(odamex ${APPLE_FRAMEWORKS})
  elseif(SOLARIS)
    target_link_libraries(odamex socket nsl)
  endif()

  if(UNIX AND NOT APPLE)
    target_link_libraries(odamex rt)
    if(X11_FOUND)
      target_link_libraries(odamex X11)
    endif()
  endif()

  set(MACOSX_BUNDLE_BUNDLE_NAME ${CMAKE_PROJECT_NAME})
  set(MACOSX_BUNDLE_INFO_STRING "ODAMEX and its likeness © ${PROJECT_COPYRIGHT} Odamex team.")
  set(MACOSX_BUNDLE_LONG_VERSION_STRING "${PROJECT_VERSION}")
  set(MACOSX_BUNDLE_SHORT_VERSION_STRING "${PROJECT_VERSION}")
  set(MACOSX_BUNDLE_BUNDLE_VERSION "${PROJECT_VERSION}")
  set(MACOSX_BUNDLE_ICON_FILE odamex.icns)

  if(APPLE)
    add_custom_command(
      TARGET odamex
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/Resources
      COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/media/${MACOSX_BUNDLE_ICON_FILE} ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/Resources
      COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wad/odamex.wad ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/MacOS
    )

    set_directory_properties(
      PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES
      "${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/Resources/${MACOSX_BUNDLE_ICON_FILE};${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/MacOS/odamex.wad"
    )

    #****
    # The remaining commands are issued on 'make install'. They make the app
    # bundle "stand-alone" by packaging dependencies inside.
    #****

    # Install into bundle under client directory
    set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR})

    if(SDL2_FOUND AND SDL2_MIXER_FOUND)
        string(REPLACE "-framework Cocoa" "" SDL_DIR ${SDL2_LIBRARY})
        string(REPLACE "-framework Cocoa" "" SDLMIXER_DIR ${SDL2_MIXER_LIBRARY})
    else(SDL2_FOUND AND SDL2_MIXER_FOUND)
        string(REPLACE "-framework Cocoa" "" SDL_DIR ${SDL_LIBRARY})
        string(REPLACE "-framework Cocoa" "" SDLMIXER_DIR ${SDLMIXER_LIBRARY})
    endif()

    # Install the SDL framework into the bundle
    if(IS_DIRECTORY ${SDL_DIR})
      install(DIRECTORY ${SDL_DIR} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/Frameworks)
      install(CODE "execute_process(COMMAND
        install_name_tool -change @rpath/SDL.framework/Versions/A/SDL
        @executable_path/../Frameworks/SDL.framework/Versions/A/SDL
        ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/MacOS/odamex)
        " )
    endif()

    # Install the SDL_mixer framework into the bundle
    if(IS_DIRECTORY ${SDLMIXER_DIR})
      install(DIRECTORY ${SDLMIXER_DIR} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/Frameworks)
    endif()

    # If PortMidi was built in install the PortMidi library into the bundle
    if(PortMidi_FOUND)
      install(FILES ${PORTMIDI_LIBRARIES} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/Frameworks/)
      install(CODE "execute_process(COMMAND
        install_name_tool -change libportmidi.dylib
        @executable_path/../Frameworks/libportmidi.dylib
        ${CMAKE_CURRENT_BINARY_DIR}/odamex.app/Contents/MacOS/odamex)
        " )
    endif()

  elseif(WIN32)
    install(TARGETS odamex
      RUNTIME DESTINATION .
      COMPONENT client)
  else()
    install(TARGETS odamex
      RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
      COMPONENT client)
  endif()
endif()
