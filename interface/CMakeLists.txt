set(TARGET_NAME interface)
project(${TARGET_NAME})

# set a default root dir for each of our optional externals if it was not passed
set(OPTIONAL_EXTERNALS "Faceshift" "LibOVR" "PrioVR" "Sixense" "LeapMotion" "RtMidi" "Qxmpp" "SDL2" "RSSDK")
foreach(EXTERNAL ${OPTIONAL_EXTERNALS})
  string(TOUPPER ${EXTERNAL} ${EXTERNAL}_UPPERCASE)
  if (NOT ${${EXTERNAL}_UPPERCASE}_ROOT_DIR)
    string(TOLOWER ${EXTERNAL} ${EXTERNAL}_LOWERCASE)
    set(${${EXTERNAL}_UPPERCASE}_ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/${${EXTERNAL}_LOWERCASE}")
  endif ()
endforeach()

find_package(Qt5LinguistTools REQUIRED)
find_package(Qt5LinguistToolsMacros)

if (DEFINED ENV{JOB_ID})
  set(BUILD_SEQ $ENV{JOB_ID})
else ()
  set(BUILD_SEQ "dev")
endif ()

if (APPLE)
  set(GL_HEADERS "#include <OpenGL/glext.h>")
elseif (UNIX)
  # include the right GL headers for UNIX
  set(GL_HEADERS "#include <GL/gl.h>\n#include <GL/glext.h>")
elseif (WIN32)
  add_definitions(-D_USE_MATH_DEFINES) # apparently needed to get M_PI and other defines from cmath/math.h
  add_definitions(-DWINDOWS_LEAN_AND_MEAN) # needed to make sure windows doesn't go to crazy with its defines

  set(GL_HEADERS "#include <windowshacks.h>\n#include <GL/glew.h>\n#include <GL/wglew.h>")
endif ()

# set up the external glm library
include_glm()
include_bullet()

# create the InterfaceConfig.h file based on GL_HEADERS above
configure_file(InterfaceConfig.h.in "${PROJECT_BINARY_DIR}/includes/InterfaceConfig.h")
configure_file(InterfaceVersion.h.in "${PROJECT_BINARY_DIR}/includes/InterfaceVersion.h")

# grab the implementation and header files from src dirs
file(GLOB_RECURSE INTERFACE_SRCS "src/*.cpp" "src/*.h")

# Add SpeechRecognizer if on Windows or OS X, otherwise remove
if (WIN32)
  # Use .cpp and .h files as is.
elseif (APPLE)
  file(GLOB INTERFACE_OBJCPP_SRCS "src/SpeechRecognizer.mm")
  set(INTERFACE_SRCS ${INTERFACE_SRCS} ${INTERFACE_OBJCPP_SRCS})
  get_filename_component(SPEECHRECOGNIZER_CPP "src/SpeechRecognizer.cpp" ABSOLUTE)
  list(REMOVE_ITEM INTERFACE_SRCS ${SPEECHRECOGNIZER_CPP})
else ()
  get_filename_component(SPEECHRECOGNIZER_H "src/SpeechRecognizer.h" ABSOLUTE)
  list(REMOVE_ITEM INTERFACE_SRCS ${SPEECHRECOGNIZER_H})
  get_filename_component(SPEECHRECOGNIZER_CPP "src/SpeechRecognizer.cpp" ABSOLUTE)
  list(REMOVE_ITEM INTERFACE_SRCS ${SPEECHRECOGNIZER_CPP})
endif ()

find_package(Qt5 COMPONENTS Gui Multimedia Network OpenGL Script Svg WebKitWidgets)

# grab the ui files in resources/ui
file (GLOB_RECURSE QT_UI_FILES ui/*.ui)
# have qt5 wrap them and generate the appropriate header files
qt5_wrap_ui(QT_UI_HEADERS "${QT_UI_FILES}")

# add them to the interface source files
set(INTERFACE_SRCS ${INTERFACE_SRCS} "${QT_UI_HEADERS}" "${QT_RESOURCES}")

# translation disabled until we strip out the line numbers
# set(QM ${TARGET_NAME}_en.qm)
# set(TS ${TARGET_NAME}_en.ts)
# qt5_create_translation_custom(${QM} ${INTERFACE_SRCS} ${QT_UI_FILES} ${TS})

if (APPLE)
  # configure CMake to use a custom Info.plist
  SET_TARGET_PROPERTIES( ${this_target} PROPERTIES MACOSX_BUNDLE_INFO_PLIST MacOSXBundleInfo.plist.in )

  set(MACOSX_BUNDLE_BUNDLE_NAME Interface)
  set(MACOSX_BUNDLE_GUI_IDENTIFIER io.highfidelity.Interface)
  
  if (${CMAKE_BUILD_TYPE} MATCHES "RELEASE")
    set(ICON_FILENAME "interface.icns")
  else ()
    set(ICON_FILENAME "interface-beta.icns")
  endif ()
  
  # set how the icon shows up in the Info.plist file
  SET(MACOSX_BUNDLE_ICON_FILE "${ICON_FILENAME}")

  # set where in the bundle to put the resources file
  SET_SOURCE_FILES_PROPERTIES(${CMAKE_CURRENT_SOURCE_DIR}/icon/${ICON_FILENAME} PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
  
  # grab the directories in resources and put them in the right spot in Resources
  file(GLOB RESOURCE_SUBDIRS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/resources" "${CMAKE_CURRENT_SOURCE_DIR}/resources/*")
  foreach(DIR ${RESOURCE_SUBDIRS})
    if(IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/resources/${DIR}")
      FILE(GLOB DIR_CONTENTS "resources/${DIR}/*")
      SET_SOURCE_FILES_PROPERTIES(${DIR_CONTENTS} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources/${DIR}")

      SET(INTERFACE_SRCS ${INTERFACE_SRCS} "${DIR_CONTENTS}")
    endif()
  endforeach()

  SET(INTERFACE_SRCS ${INTERFACE_SRCS} "${CMAKE_CURRENT_SOURCE_DIR}/icon/${ICON_FILENAME}")
endif()

# create the executable, make it a bundle on OS X
add_executable(${TARGET_NAME} MACOSX_BUNDLE ${INTERFACE_SRCS} ${QM})

# link required hifi libraries
link_hifi_libraries(shared octree environment gpu model fbx metavoxels networking entities avatars 
                    audio audio-client animation script-engine physics 
                    render-utils entities-renderer)

# find any optional and required libraries
find_package(ZLIB REQUIRED)

# perform standard include and linking for found externals
foreach(EXTERNAL ${OPTIONAL_EXTERNALS})
  
  if (${${EXTERNAL}_UPPERCASE}_REQUIRED)
    find_package(${EXTERNAL} REQUIRED)
  else ()
    find_package(${EXTERNAL})
  endif ()
  
  if (${${EXTERNAL}_UPPERCASE}_FOUND AND NOT DISABLE_${${EXTERNAL}_UPPERCASE})
    add_definitions(-DHAVE_${${EXTERNAL}_UPPERCASE})
    
    # include the library directories (ignoring warnings)
    if (NOT ${${EXTERNAL}_UPPERCASE}_INCLUDE_DIRS)
      set(${${EXTERNAL}_UPPERCASE}_INCLUDE_DIRS ${${${EXTERNAL}_UPPERCASE}_INCLUDE_DIR})
    endif ()
    
    include_directories(SYSTEM ${${${EXTERNAL}_UPPERCASE}_INCLUDE_DIRS})
    
    # perform the system include hack for OS X to ignore warnings
    if (APPLE)
      foreach(EXTERNAL_INCLUDE_DIR  ${${${EXTERNAL}_UPPERCASE}_INCLUDE_DIRS})
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -isystem ${EXTERNAL_INCLUDE_DIR}")
      endforeach()
    endif ()
    
    if (NOT ${${EXTERNAL}_UPPERCASE}_LIBRARIES)
      set(${${EXTERNAL}_UPPERCASE}_LIBRARIES ${${${EXTERNAL}_UPPERCASE}_LIBRARY})
    endif ()
    
    if (NOT APPLE OR NOT ${${EXTERNAL}_UPPERCASE} MATCHES "SIXENSE")
      target_link_libraries(${TARGET_NAME} ${${${EXTERNAL}_UPPERCASE}_LIBRARIES})
    elseif (APPLE AND NOT INSTALLER_BUILD)
      add_definitions(-DSIXENSE_LIB_FILENAME=\"${${${EXTERNAL}_UPPERCASE}_LIBRARY_RELEASE}\")
    endif ()
  endif ()
endforeach()

# special APPLE modifications for Visage library
if (VISAGE_FOUND AND NOT DISABLE_VISAGE AND APPLE)
  add_definitions(-DMAC_OS_X)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-comment")
  find_library(AVFoundation AVFoundation)
  find_library(CoreMedia CoreMedia)
  target_link_libraries(${TARGET_NAME} ${AVFoundation} ${CoreMedia} ${NEW_STD_LIBRARY})
endif ()

# special OS X modifications for RtMidi library
if (RTMIDI_FOUND AND NOT DISABLE_RTMIDI AND APPLE)  
  find_library(CoreMIDI CoreMIDI)
  add_definitions(-D__MACOSX_CORE__)
  target_link_libraries(${TARGET_NAME} ${CoreMIDI})
endif ()

if (QXMPP_FOUND AND NOT DISABLE_QXMPP AND WIN32)
  # assume we're linking a static Qt on windows
  add_definitions(-DQXMPP_STATIC) 
endif ()

# include headers for interface and InterfaceConfig.
include_directories("${PROJECT_SOURCE_DIR}/src" "${PROJECT_BINARY_DIR}/includes")

target_link_libraries(
  ${TARGET_NAME} ${ZLIB_LIBRARIES}
  Qt5::Gui Qt5::Network Qt5::Multimedia Qt5::OpenGL Qt5::Script Qt5::Svg Qt5::WebKitWidgets
)

# assume we are using a Qt build without bearer management
add_definitions(-DQT_NO_BEARERMANAGEMENT)

if (APPLE)
  # link in required OS X frameworks and include the right GL headers
  find_library(OpenGL OpenGL)
  find_library(AppKit AppKit)

  target_link_libraries(${TARGET_NAME} ${OpenGL} ${AppKit})
    
  # install command for OS X bundle
  INSTALL(TARGETS ${TARGET_NAME}
    BUNDLE DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/install" COMPONENT Runtime
    RUNTIME DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/install" COMPONENT Runtime
  )
else (APPLE)
  # copy the resources files beside the executable
  add_custom_command(TARGET ${TARGET_NAME} POST_BUILD
    COMMAND "${CMAKE_COMMAND}" -E copy_directory
    "${PROJECT_SOURCE_DIR}/resources"
    $<TARGET_FILE_DIR:${TARGET_NAME}>/resources
  )
    
  find_package(OpenGL REQUIRED)
  
  if (${OPENGL_INCLUDE_DIR})
    include_directories(SYSTEM "${OPENGL_INCLUDE_DIR}")
  endif ()
  
  target_link_libraries(${TARGET_NAME} "${OPENGL_LIBRARY}")
  
  # link target to external libraries
  if (WIN32)
    find_package(GLEW REQUIRED)
    include_directories(${GLEW_INCLUDE_DIRS})

    # we're using static GLEW, so define GLEW_STATIC
    add_definitions(-DGLEW_STATIC)

    target_link_libraries(${TARGET_NAME} ${GLEW_LIBRARIES} "${NSIGHT_LIBRARIES}" wsock32.lib opengl32.lib Winmm.lib)

    # try to find the Nsight package and add it to the build if we find it
    find_package(NSIGHT)
    if (NSIGHT_FOUND)
      include_directories(${NSIGHT_INCLUDE_DIRS})
      add_definitions(-DNSIGHT_FOUND)
      target_link_libraries(${TARGET_NAME} "${NSIGHT_LIBRARIES}")
    endif ()

  endif()
endif (APPLE)

# link any dependencies bubbled up from our linked dependencies
include_dependency_includes()
