file(GLOB_RECURSE OPENBLACK_SOURCES "${CMAKE_CURRENT_LIST_DIR}/*.cpp")
if (NOT ANDROID)
  list(REMOVE_ITEM OPENBLACK_SOURCES "${CMAKE_CURRENT_LIST_DIR}/main.cpp")
endif ()
file(GLOB_RECURSE OPENBLACK_HEADERS "${CMAKE_CURRENT_LIST_DIR}/*.h")
if (APPLE)
  file(GLOB_RECURSE OPENBLACK_OBJC_SOURCES "${CMAKE_CURRENT_LIST_DIR}/*.mm")
  list(APPEND OPENBLACK_SOURCES "${OPENBLACK_OBJC_SOURCES}")
endif ()

set(OPENBLACK_RC ${CMAKE_SOURCE_DIR}/assets/resource/icon/openblack.rc)

file(GLOB OPENBLACK_IMGUI_VS_SHADERS
     "${CMAKE_SOURCE_DIR}/assets/shaders/imgui/vs_*.sc"
)
file(GLOB OPENBLACK_IMGUI_FS_SHADERS
     "${CMAKE_SOURCE_DIR}/assets/shaders/imgui/fs_*.sc"
)
bgfx_compile_shaders(
  AS_HEADERS
  TYPE VERTEX
  SHADERS ${OPENBLACK_IMGUI_VS_SHADERS}
  VARYING_DEF ${CMAKE_SOURCE_DIR}/assets/shaders/imgui/varying.def.sc
  OUTPUT_DIR ${CMAKE_BINARY_DIR}/include/generated/shaders/imgui
)
bgfx_compile_shaders(
  AS_HEADERS
  TYPE FRAGMENT
  SHADERS ${OPENBLACK_IMGUI_FS_SHADERS}
  VARYING_DEF ${CMAKE_SOURCE_DIR}/assets/shaders/imgui/varying.def.sc
  OUTPUT_DIR ${CMAKE_BINARY_DIR}/include/generated/shaders/imgui
)

file(GLOB OPENBLACK_VERTEX_SHADERS "${CMAKE_SOURCE_DIR}/assets/shaders/vs_*.sc")
file(GLOB OPENBLACK_FRAGMENT_SHADERS
     "${CMAKE_SOURCE_DIR}/assets/shaders/fs_*.sc"
)
file(GLOB OPENBLACK_VARYING_DEF_FILE
     "${CMAKE_SOURCE_DIR}/assets/shaders/varying.def.sc"
)

bgfx_compile_shaders(
  AS_HEADERS
  TYPE VERTEX
  SHADERS ${OPENBLACK_VERTEX_SHADERS}
  VARYING_DEF ${OPENBLACK_VARYING_DEF_FILE}
  OUTPUT_DIR ${CMAKE_BINARY_DIR}/include/generated/shaders
)
bgfx_compile_shaders(
  AS_HEADERS
  TYPE FRAGMENT
  SHADERS ${OPENBLACK_FRAGMENT_SHADERS}
  VARYING_DEF ${OPENBLACK_VARYING_DEF_FILE}
  OUTPUT_DIR ${CMAKE_BINARY_DIR}/include/generated/shaders
)

list(APPEND OPENBLACK_SHADERS ${OPENBLACK_IMGUI_VS_SHADERS})
list(APPEND OPENBLACK_SHADERS ${OPENBLACK_IMGUI_FS_SHADERS})
list(APPEND OPENBLACK_SHADERS
     ${CMAKE_SOURCE_DIR}/assets/shaders/imgui/varying.def.sc
)
list(APPEND OPENBLACK_SHADERS ${OPENBLACK_VERTEX_SHADERS})
list(APPEND OPENBLACK_SHADERS ${OPENBLACK_FRAGMENT_SHADERS})
list(APPEND OPENBLACK_SHADERS ${OPENBLACK_VARYING_DEF_FILE})

if (ANDROID)
  add_library(openblack_lib SHARED)
  set_target_properties(
    openblack_lib PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin
  )
else ()
  add_library(openblack_lib STATIC)
endif ()

target_sources(
  openblack_lib
  PRIVATE ${OPENBLACK_SOURCES}
          ${OPENBLACK_HEADERS}
          ${OPENBLACK_RC}
          ${OPENBLACK_SHADERS}
          ${OPENBLACK_VERTEX_SHADERS}
          ${OPENBLACK_FRAGMENT_SHADERS}
          ${OPENBLACK_VARYING_DEF_FILE}
)
set_target_properties(openblack_lib PROPERTIES LINKER_LANGUAGE CXX)

target_include_directories(
  openblack_lib
  PUBLIC ${CMAKE_CURRENT_LIST_DIR}
         ${CMAKE_BINARY_DIR}/include # For shaders and other bin2c
         ${BULLET_INCLUDE_DIR} ${Stb_INCLUDE_DIR} ${DRLIBS_INCLUDE_DIRS}
)
if (NOT VCPKG_TARGET_ANDROID)
  target_include_directories(
    openblack_lib PUBLIC ${BULLET_ROOT_DIR}/${BULLET_INCLUDE_DIR}
  )
endif ()
target_link_directories(
  openblack_lib PRIVATE ${BULLET_ROOT_DIR}/${BULLET_LIBRARY_DIRS}
)
target_compile_definitions(openblack_lib PRIVATE ${BULLET_DEFINITIONS})
target_link_libraries(
  openblack_lib
  PRIVATE "$<$<CXX_COMPILER_ID:MSVC>:-SAFESEH:NO>"
          l3d
          pack
          lnd
          anm
          glw
          morph
          imgui::imgui
          imgui_user
          ScriptLibrary
          glm::glm
          EnTT::EnTT
          bgfx::bgfx
          OpenAL::OpenAL
          Bullet3Common
          BulletDynamics
          BulletCollision
          BulletSoftBody
          LinearMath
          minizip::minizip
  PUBLIC spdlog::spdlog
)
target_include_directories(openblack_lib PRIVATE ${CXXOPTS_INCLUDE_DIRS})

if (OPENBLACK_CLANG_TIDY_CHECKS)
  if (CLANG_TIDY)
    if (MSVC)
      # as for now, /EHsc has to be passed manually to clang-tidy
      # see https://gitlab.kitware.com/cmake/cmake/-/issues/20512
      set_target_properties(
        openblack_lib PROPERTIES CXX_CLANG_TIDY
                                 "${CLANG_TIDY};--extra-arg=/EHsc"
      )
    else ()
      set_target_properties(
        openblack_lib PROPERTIES CXX_CLANG_TIDY ${CLANG_TIDY}
      )
    endif ()
  else ()
    message("Clang-tidy checks requested but unavailable")
  endif ()
endif ()

if (ANDROID)
  target_link_libraries(openblack_lib PRIVATE EGL)
elseif (UNIX)
  find_library(WAYLAND wayland-egl)
  if (WAYLAND)
    target_link_libraries(openblack_lib PRIVATE ${WAYLAND})
  endif ()
endif ()

if (VCPKG_TARGET_ANDROID)
  # openblack_lib need to forward the JNI exports from SDL2 so the Activity can start main
  target_link_libraries(
    openblack_lib PUBLIC -Wl,--whole-archive SDL2::SDL2 -Wl,--no-whole-archive
  )
else ()
  target_link_libraries(
    openblack_lib
    PUBLIC $<TARGET_NAME_IF_EXISTS:SDL2::SDL2main>
           $<IF:$<TARGET_EXISTS:SDL2::SDL2>,SDL2::SDL2,SDL2::SDL2-static>
  )
endif ()

target_compile_definitions(
  openblack_lib
  PRIVATE
    "GLM_FORCE_LEFT_HANDED"
    "GLM_FORCE_RADIANS"
    "GLM_ENABLE_EXPERIMENTAL"
    # Compile-time optimize out log messages (debug level if debug, else info)
    "SPDLOG_ACTIVE_LEVEL=$<IF:$<CONFIG:DEBUG>,SPDLOG_LEVEL_DEBUG,SPDLOG_LEVEL_INFO>"
    # Suppress WinMain() and main hijacking, provided by SDL
    "SDL_MAIN_HANDLED"
  PUBLIC "$<$<CONFIG:DEBUG>:OPENBLACK_DEBUG>"
)

if (MSVC)
  target_compile_definitions(
    openblack_lib
    PRIVATE
      # Get rid of useless crud from windows.h
      NOMINMAX
      WIN32_LEAN_AND_MEAN
      # disable CRT warnings on windows cause they're annoying as shit and we use C
      # functions everywhere
      _CRT_SECURE_NO_WARNINGS
  )

  target_compile_options(
    openblack_lib
    PRIVATE # Level 4 warnings
            /W4
            # Disable nonstandard extension used: nameless struct/union from glm
            /wd4201
            # Multi-threaded
            /MP
            # Enables stricter C++ standards conformance checks
            /permissive-
            # Set correct version of c++
            /Zc:__cplusplus
            # Use utf-8 for std::print and fmt/spdlog
            /utf-8
  )

  if (OPENBLACK_WARNINGS_AS_ERRORS)
    target_compile_options(openblack_lib PRIVATE /WX)
  endif ()
else ()
  target_compile_options(
    openblack_lib
    PRIVATE -Wall
            -Wextra
            -pedantic
            -Wno-unused-private-field
            -Wno-unused-variable
            -Wno-unused-parameter
            # stb_image_write.h
            -Wno-missing-field-initializers
            -Wno-deprecated-declarations
  )

  if (OPENBLACK_TRACE_TIME AND CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(openblack_lib PRIVATE -ftime-trace)
  endif ()
  if (OPENBLACK_WARNINGS_AS_ERRORS)
    target_compile_options(openblack_lib PRIVATE -Werror)
  endif ()
endif ()

set_property(
  DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT openblack
)
source_group(
  TREE ${CMAKE_CURRENT_LIST_DIR} FILES ${OPENBLACK_SOURCES}
                                       ${OPENBLACK_HEADERS}
)

foreach (SHADER ${OPENBLACK_SHADERS})
  source_group("Shaders" FILES "${SHADER}")
endforeach ()

if (ANDROID)
  add_library(openblack ALIAS openblack_lib)
else ()
  add_executable(openblack main.cpp)
  target_link_libraries(openblack PRIVATE openblack_lib)

  if (MSVC)
    # Debug builds create a debug console
    set_target_properties(
      openblack PROPERTIES LINK_FLAGS_DEBUG "/SUBSYSTEM:CONSOLE"
    )
    set_target_properties(
      openblack PROPERTIES LINK_FLAGS_RELWITHDEBINFO "/SUBSYSTEM:CONSOLE"
    )
    target_compile_definitions(openblack PRIVATE $<$<CONFIG:DEBUG>:_CONSOLE>)

    # Release builds also have a console, for now..
    set_target_properties(
      openblack PROPERTIES LINK_FLAGS_RELEASE "/SUBSYSTEM:CONSOLE"
    )
    set_target_properties(
      openblack PROPERTIES COMPILE_DEFINITIONS_RELEASE "CONSOLE"
    )
    set_target_properties(
      openblack PROPERTIES LINK_FLAGS_MINSIZEREL "/SUBSYSTEM:CONSOLE"
    )
  endif ()

  if (EMSCRIPTEN)
    # Must be run from project root using `emrun --no_browser --serve_after_close --serve_after_exit .`
    # Browser should point to http://localhost:6931/emscripten-vcpkg/bin/Debug/openblack.html?-g&relative/path/to/game/files
    set_target_properties(
      openblack
      PROPERTIES
        COMPILE_FLAGS_DEBUG "-g3 -gsource-map"
        LINK_FLAGS_DEBUG "-g3 -gsource-map"
        COMPILE_FLAGS "-s USE_SDL=2 -fexceptions"
        #COMPILE_FLAGS "-s USE_SDL=2 -s USE_PTHREADS=1"
        LINK_FLAGS
        "-s USE_SDL=2 -s FULL_ES3=1 -s GL_ASSERTIONS=1 -s USE_WEBGL2=1 -s TOTAL_MEMORY=128MB -fexceptions --emrun"
        #LINK_FLAGS "-s USE_SDL=2 -s FULL_ES3=1 -s GL_ASSERTIONS=1 -s USE_WEBGL2=1 -s USE_PTHREADS=1 -s TOTAL_MEMORY=128MB --emrun"
        SUFFIX ".html"
    )
  elseif (APPLE)
    set(OPENBLACK_ICON_SOURCE_DIR ${CMAKE_SOURCE_DIR}/assets/resource/icon)
    set(OPENBLACK_ICON_BINARY_DIR ${CMAKE_BINARY_DIR}/assets/resource/icon)
    set(OPENBLACK_ICON_SOURCE
        ${OPENBLACK_ICON_SOURCE_DIR}/openblack-400x400.png
    )
    set(OPENBLACK_ICONSET_BINARY_DIR
        ${OPENBLACK_ICON_BINARY_DIR}/openblack.iconset
    )
    set(OPENBLACK_ICONSET_BINARY ${OPENBLACK_ICON_BINARY_DIR}/openblack.icns)
    add_custom_command(
      OUTPUT ${OPENBLACK_ICONSET_BINARY}
      COMMENT "Generate MacOSX bundle icon set"
      COMMAND ${CMAKE_COMMAND} -E make_directory ${OPENBLACK_ICONSET_BINARY_DIR}
      COMMAND sips -z 16 16 ${OPENBLACK_ICON_SOURCE_DIR}/openblack-256x256.png
              --out ${OPENBLACK_ICONSET_BINARY_DIR}/icon_16x16.png
      COMMAND sips -z 32 32 ${OPENBLACK_ICON_SOURCE_DIR}/openblack-256x256.png
              --out ${OPENBLACK_ICONSET_BINARY_DIR}/icon_32x32.png
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${OPENBLACK_ICON_SOURCE_DIR}/openblack-64x64.png
        ${OPENBLACK_ICONSET_BINARY_DIR}/icon_64x64.png
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${OPENBLACK_ICON_SOURCE_DIR}/openblack-128x128.png
        ${OPENBLACK_ICONSET_BINARY_DIR}/icon_128x128.png
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${OPENBLACK_ICON_SOURCE_DIR}/openblack-256x256.png
        ${OPENBLACK_ICONSET_BINARY_DIR}/icon_256x256.png
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${OPENBLACK_ICON_SOURCE_DIR}/openblack-400x400.png
        ${OPENBLACK_ICONSET_BINARY_DIR}/icon_400x400.png
      COMMAND iconutil -c icns ${OPENBLACK_ICONSET_BINARY_DIR} -o
              ${OPENBLACK_ICONSET_BINARY}
    )
    set_source_files_properties(
      ${OPENBLACK_ICONSET_BINARY} PROPERTIES MACOSX_PACKAGE_LOCATION
                                             "Resources"
    )
    target_sources(openblack PRIVATE ${OPENBLACK_ICONSET_BINARY})
    set_target_properties(
      openblack
      PROPERTIES MACOSX_BUNDLE_GUI_IDENTIFIER org.openblack.openblack
                 MACOSX_BUNDLE_ICON_FILE openblack.icns
                 MACOSX_BUNDLE_BUNDLE_VERSION ${openblack_VERSION}
                 MACOSX_BUNDLE_SHORT_VERSION_STRING ${openblack_VERSION}
    )
  endif ()

  # Installation configuration
  install(
    TARGETS openblack
    BUNDLE DESTINATION bundle
    RUNTIME DESTINATION bin
  )
  if (UNIX)
    install(
      FILES
        "${CMAKE_SOURCE_DIR}/assets/resource/shortcut/org.openblack.openblack.desktop"
      DESTINATION "share/applications"
    )
    foreach (RES 64 128 256)
      install(
        FILES
          "${CMAKE_SOURCE_DIR}/assets/resource/icon/openblack-${RES}x${RES}.png"
        DESTINATION "share/icons/hicolor/${RES}x${RES}/apps"
        RENAME "openblack.png"
      )
    endforeach ()
  endif ()

endif ()
