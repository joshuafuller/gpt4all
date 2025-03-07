cmake_minimum_required(VERSION 3.16)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(APPLE)
  option(BUILD_UNIVERSAL "Build a Universal binary on macOS" OFF)
  if(BUILD_UNIVERSAL)
    # Build a Universal binary on macOS
    # This requires that the found Qt library is compiled as Universal binaries.
    set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "" FORCE)
  else()
    # Build for the host architecture on macOS
    set(CMAKE_OSX_ARCHITECTURES "${CMAKE_HOST_SYSTEM_PROCESSOR}" CACHE STRING "" FORCE)
  endif()
endif()

set(APP_VERSION_MAJOR 2)
set(APP_VERSION_MINOR 4)
set(APP_VERSION_PATCH 5)
set(APP_VERSION "${APP_VERSION_MAJOR}.${APP_VERSION_MINOR}.${APP_VERSION_PATCH}")

# Include the binary directory for the generated header file
include_directories("${CMAKE_CURRENT_BINARY_DIR}")

project(gpt4all VERSION ${APP_VERSION} LANGUAGES CXX C)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(GPT4ALL_LOCALHOST OFF "Build installer for localhost repo")
option(GPT4ALL_AVX_ONLY OFF "Build for avx only")
option(GPT4ALL_OFFLINE_INSTALLER "Build an offline installer" OFF)

# Generate a header file with the version number
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.in"
  "${CMAKE_CURRENT_BINARY_DIR}/config.h"
)

find_package(Qt6 6.5 COMPONENTS Core Quick QuickDialogs2 Svg HttpServer Sql Pdf REQUIRED)

# Get the Qt6Core target properties
get_target_property(Qt6Core_INCLUDE_DIRS Qt6::Core INTERFACE_INCLUDE_DIRECTORIES)
get_target_property(Qt6Core_LIBRARY_RELEASE Qt6::Core LOCATION_RELEASE)

# Find the qmake binary
find_program(QMAKE_EXECUTABLE NAMES qmake qmake6 PATHS ${Qt6Core_INCLUDE_DIRS}/../.. NO_DEFAULT_PATH)

# Get the Qt 6 root directory
get_filename_component(Qt6_ROOT_DIR "${Qt6Core_LIBRARY_RELEASE}" DIRECTORY)
get_filename_component(Qt6_ROOT_DIR "${Qt6_ROOT_DIR}/.." ABSOLUTE)

message(STATUS "qmake binary: ${QMAKE_EXECUTABLE}")
message(STATUS "Qt 6 root directory: ${Qt6_ROOT_DIR}")

set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

add_subdirectory(../gpt4all-backend llmodel)

qt_add_executable(chat
    main.cpp
    chat.h chat.cpp
    chatllm.h chatllm.cpp
    chatmodel.h chatlistmodel.h chatlistmodel.cpp
    chatgpt.h chatgpt.cpp
    database.h database.cpp
    download.h download.cpp
    network.h network.cpp
    localdocs.h localdocs.cpp localdocsmodel.h localdocsmodel.cpp
    llm.h llm.cpp
    server.h server.cpp
    sysinfo.h
)

qt_add_qml_module(chat
    URI gpt4all
    VERSION 1.0
    QML_FILES
      main.qml
      qml/ChatDrawer.qml
      qml/CollectionsDialog.qml
      qml/LocalDocs.qml
      qml/ModelDownloaderDialog.qml
      qml/NetworkDialog.qml
      qml/NewVersionDialog.qml
      qml/ThumbsDownDialog.qml
      qml/SettingsDialog.qml
      qml/StartupDialog.qml
      qml/PopupDialog.qml
      qml/AboutDialog.qml
      qml/Theme.qml
      qml/MyButton.qml
      qml/MyComboBox.qml
      qml/MyTextField.qml
      qml/MyCheckBox.qml
      qml/MyBusyIndicator.qml
      qml/MyToolButton.qml
    RESOURCES
      icons/send_message.svg
      icons/stop_generating.svg
      icons/regenerate.svg
      icons/copy.svg
      icons/db.svg
      icons/settings.svg
      icons/edit.svg
      icons/trash.svg
      icons/network.svg
      icons/thumbs_up.svg
      icons/thumbs_down.svg
      icons/logo.svg
      icons/logo-32.png
      icons/logo-48.png
      icons/favicon.ico
      icons/favicon.icns
)

set_target_properties(chat PROPERTIES
    MACOSX_BUNDLE_GUI_IDENTIFIER gpt4all
    MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
    MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    MACOSX_BUNDLE TRUE
    WIN32_EXECUTABLE TRUE
    MACOSX_BUNDLE_ICON_FILE "favicon.icns"
)

if(${CMAKE_SYSTEM_NAME} MATCHES Darwin)
    set_target_properties(chat PROPERTIES
        OUTPUT_NAME gpt4all
    )
endif()

target_compile_definitions(chat
    PRIVATE $<$<OR:$<CONFIG:Debug>,$<CONFIG:RelWithDebInfo>>:QT_QML_DEBUG>)
target_link_libraries(chat
    PRIVATE Qt6::Quick Qt6::Svg Qt6::HttpServer Qt6::Sql Qt6::Pdf)
target_link_libraries(chat
    PRIVATE llmodel)

set(COMPONENT_NAME_MAIN ${PROJECT_NAME})
set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/install)

if(NOT (CMAKE_HOST_SYSTEM_NAME STREQUAL "Darwin" AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64"))
    add_executable(test_hw test_hw.cpp)
    install(TARGETS test_hw DESTINATION bin COMPONENT ${COMPONENT_NAME_MAIN})
endif()

install(TARGETS chat DESTINATION bin COMPONENT ${COMPONENT_NAME_MAIN})
install(TARGETS llmodel DESTINATION lib COMPONENT ${COMPONENT_NAME_MAIN})

set(CPACK_GENERATOR "IFW")
set(CPACK_VERBATIM_VARIABLES YES)
set(CPACK_IFW_VERBOSE ON)

if(${CMAKE_SYSTEM_NAME} MATCHES Linux)
    set(LINUXDEPLOYQT "$ENV{HOME}/dev/linuxdeployqt/build/tools/linuxdeployqt/linuxdeployqt")
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/deploy-qt-linux.cmake.in"
                   "${CMAKE_BINARY_DIR}/cmake/deploy-qt-linux.cmake" @ONLY)
    set(CPACK_PRE_BUILD_SCRIPTS ${CMAKE_BINARY_DIR}/cmake/deploy-qt-linux.cmake)
    set(CPACK_IFW_ROOT "~/Qt/Tools/QtInstallerFramework/4.5")
    set(CPACK_PACKAGE_FILE_NAME "${COMPONENT_NAME_MAIN}-installer-linux")
    set(CPACK_IFW_TARGET_DIRECTORY "@HomeDir@/${COMPONENT_NAME_MAIN}")
elseif(${CMAKE_SYSTEM_NAME} MATCHES Windows)
    find_program(WINDEPLOYQT windeployqt HINTS ${_qt_bin_dir})
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/deploy-qt-windows.cmake.in"
                   "${CMAKE_BINARY_DIR}/cmake/deploy-qt-windows.cmake" @ONLY)
    set(CPACK_PRE_BUILD_SCRIPTS ${CMAKE_BINARY_DIR}/cmake/deploy-qt-windows.cmake)
    set(CPACK_IFW_ROOT "C:/Qt/Tools/QtInstallerFramework/4.5")
    set(CPACK_IFW_PACKAGE_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/favicon.ico")
    set(CPACK_PACKAGE_FILE_NAME "${COMPONENT_NAME_MAIN}-installer-win64")
    set(CPACK_IFW_TARGET_DIRECTORY "@HomeDir@\\${COMPONENT_NAME_MAIN}")
elseif(${CMAKE_SYSTEM_NAME} MATCHES Darwin)
    find_program(MACDEPLOYQT macdeployqt HINTS ${_qt_bin_dir})
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/deploy-qt-mac.cmake.in"
                   "${CMAKE_BINARY_DIR}/cmake/deploy-qt-mac.cmake" @ONLY)
    set(CPACK_PRE_BUILD_SCRIPTS ${CMAKE_BINARY_DIR}/cmake/deploy-qt-mac.cmake)
    set(CPACK_IFW_ROOT "~/Qt/Tools/QtInstallerFramework/4.5")
    set(CPACK_IFW_PACKAGE_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/favicon.icns")
    set(CPACK_PACKAGE_FILE_NAME "${COMPONENT_NAME_MAIN}-installer-darwin")
    set(CPACK_IFW_TARGET_DIRECTORY "@ApplicationsDir@/${COMPONENT_NAME_MAIN}")
    set(CPACK_BUNDLE_NAME ${COMPONENT_NAME_MAIN})
    set(CPACK_BUNDLE_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/favicon.icns")
endif()

set(CPACK_PACKAGE_INSTALL_DIRECTORY ${COMPONENT_NAME_MAIN})
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
SET(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(CPACK_PACKAGE_HOMEPAGE_URL "https://gpt4all.io")
set(CPACK_PACKAGE_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/logo-48.png")
set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)
set(CPACK_RESOURCE_FILE_README ${CMAKE_CURRENT_SOURCE_DIR}/README.md)
set(CPACK_PACKAGE_EXECUTABLES "GPT4All")
set(CPACK_CREATE_DESKTOP_LINKS "GPT4All")
set(CPACK_IFW_PACKAGE_NAME "GPT4All")
set(CPACK_IFW_PACKAGE_TITLE "GPT4All Installer")
set(CPACK_IFW_PACKAGE_PUBLISHER "Nomic, Inc.")
set(CPACK_IFW_PRODUCT_URL "https://gpt4all.io")
set(CPACK_IFW_PACKAGE_WIZARD_STYLE "Aero")
set(CPACK_IFW_PACKAGE_LOGO "${CMAKE_CURRENT_SOURCE_DIR}/icons/logo-48.png")
set(CPACK_IFW_PACKAGE_WINDOW_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/logo-32.png")
set(CPACK_IFW_PACKAGE_WIZARD_SHOW_PAGE_LIST OFF)

include(InstallRequiredSystemLibraries)
include(CPack)
include(CPackIFW)
cpack_add_component(${COMPONENT_NAME_MAIN} DOWNLOADED)
cpack_ifw_configure_component(${COMPONENT_NAME_MAIN} ESSENTIAL FORCED_INSTALLATION)
cpack_ifw_configure_component(${COMPONENT_NAME_MAIN} VERSION ${APP_VERSION})
cpack_ifw_configure_component(${COMPONENT_NAME_MAIN} LICENSES "MIT LICENSE" ${CPACK_RESOURCE_FILE_LICENSE})
cpack_ifw_configure_component(${COMPONENT_NAME_MAIN} SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/cmake/installerscript.qs")
cpack_ifw_configure_component(${COMPONENT_NAME_MAIN} REPLACES "gpt4all-chat") #Was used in very earliest prototypes

if (GPT4ALL_LOCALHOST)
    cpack_ifw_add_repository("GPT4AllRepository" URL "http://localhost/repository")
elseif(GPT4ALL_OFFLINE_INSTALLER)
    cpack_ifw_add_repository("GPT4AllRepository" URL "file://${CMAKE_BINARY_DIR}/packages")
else()
  if(${CMAKE_SYSTEM_NAME} MATCHES Linux)
    if (GPT4ALL_AVX_ONLY)
      cpack_ifw_add_repository("GPT4AllRepository" URL "https://gpt4all.io/installer_repos/avx_only/linux/repository")
    else()
      cpack_ifw_add_repository("GPT4AllRepository" URL "https://gpt4all.io/installer_repos/linux/repository")
    endif()
  elseif(${CMAKE_SYSTEM_NAME} MATCHES Windows)
    #To sign the target on windows have to create a batch script add use it as a custom target and then use CPACK_IFW_EXTRA_TARGETS to set this extra target
    if (GPT4ALL_AVX_ONLY)
      cpack_ifw_add_repository("GPT4AllRepository" URL "https://gpt4all.io/installer_repos/avx_only/windows/repository")
    else()
      cpack_ifw_add_repository("GPT4AllRepository" URL "https://gpt4all.io/installer_repos/windows/repository")
    endif()
  elseif(${CMAKE_SYSTEM_NAME} MATCHES Darwin)
    if (GPT4ALL_AVX_ONLY)
      cpack_ifw_add_repository("GPT4AllRepository" URL "https://gpt4all.io/installer_repos/avx_only/mac/repository")
    else()
      cpack_ifw_add_repository("GPT4AllRepository" URL "https://gpt4all.io/installer_repos/mac/repository")
    endif()
  endif()
endif()
