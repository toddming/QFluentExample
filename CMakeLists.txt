cmake_minimum_required(VERSION 3.5)
project(QFluentExample LANGUAGES CXX)

# 支持Qt5和Qt6
find_package(QT NAMES Qt6 Qt5 REQUIRED COMPONENTS Core)
find_package(Qt${QT_VERSION_MAJOR} REQUIRED COMPONENTS Core Widgets Svg Xml)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# 是否采用QWK无边框窗口方案
set(BUILD_QWINDOWKIT ON)

add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/libs/include)


file(GLOB_RECURSE SOURCES
    CONFIGURE_DEPENDS
    RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
    src/*.cpp
    src/*.h
)
set(CPP_SOURCES "")
set(HEADER_SOURCES "")
foreach(file ${SOURCES})
    if(file MATCHES "\\.cpp$")
        list(APPEND CPP_SOURCES ${file})
    elseif(file MATCHES "\\.h$")
        list(APPEND HEADER_SOURCES ${file})
    endif()
endforeach()

file(GLOB_RECURSE AUTO_RESOURCE_FILES
    RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
    res/*.png res/*.jpg res/*.jpeg res/*.gif res/*.svg res/*.ico res/*.ttf res/*.qss
)

# 资源处理：兼容Qt5和Qt6
if(${QT_VERSION_MAJOR} EQUAL 6)
    add_executable(QFluentExample
        WIN32 MACOSX_BUNDLE
        ${CPP_SOURCES}
        ${HEADER_SOURCES}
        version.rc
    )
qt_add_resources(QFluentExample "QFluentExample"
    PREFIX "/"
    FILES ${AUTO_RESOURCE_FILES}
)
else()
    add_executable(QFluentExample
        WIN32 MACOSX_BUNDLE
        ${CPP_SOURCES}
        ${HEADER_SOURCES}
        version.rc
        res/res.qrc
    )
endif()

# ============================================================================
# 获取构建环境信息
# ============================================================================
# 获取 Qt 版本
if(TARGET Qt6::Core)
    set(QT_MAJOR_VERSION "6")
    set(QT_VERSION_STRING ${Qt6_VERSION})
elseif(TARGET Qt5::Core)
    set(QT_MAJOR_VERSION "5")
    set(QT_VERSION_STRING ${Qt5Core_VERSION})
else()
    message(FATAL_ERROR "Neither Qt5 nor Qt6 found! Please install Qt and set CMAKE_PREFIX_PATH.")
endif()

# 标准化架构名称
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|amd64")
    set(ARCH "x64")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "i686|i386|x86")
    set(ARCH "x86")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64")
    set(ARCH "arm64")
else()
    set(ARCH ${CMAKE_SYSTEM_PROCESSOR})
endif()

# 构建环境字符串
set(BUILD_ENV_STRING "${QT_VERSION_STRING}_${CMAKE_SYSTEM_NAME}_${CMAKE_CXX_COMPILER_ID}_${ARCH}")
message(STATUS "Detected build environment: ${BUILD_ENV_STRING}")

# SDK 路径
set(SDK_DIR "${CMAKE_CURRENT_SOURCE_DIR}/libs/${BUILD_ENV_STRING}")
message(STATUS "SDK Path: ${SDK_DIR}")

# ============================================================================
# 查找 QFluent 库
# ============================================================================
find_library(QFLUENT_LIB_DEBUG
    NAMES QFluent libQFluent
    PATHS ${SDK_DIR}/Debug
    NO_DEFAULT_PATH
)

find_library(QFLUENT_LIB_RELEASE
    NAMES QFluent libQFluent
    PATHS ${SDK_DIR}/Release
    NO_DEFAULT_PATH
)

# 检查库文件
if(NOT QFLUENT_LIB_DEBUG)
    message(WARNING "QFluent Debug library not found in ${SDK_DIR}/Debug")
endif()

if(NOT QFLUENT_LIB_RELEASE)
    message(WARNING "QFluent Release library not found in ${SDK_DIR}/Release")
endif()

message(STATUS "QFluent Debug library: ${QFLUENT_LIB_DEBUG}")
message(STATUS "QFluent Release library: ${QFLUENT_LIB_RELEASE}")

# ============================================================================
# 链接库
# ============================================================================
# 链接 QFluent 库
target_link_libraries(QFluentExample PRIVATE
    $<$<CONFIG:Debug>:${QFLUENT_LIB_DEBUG}>
    $<$<CONFIG:Release>:${QFLUENT_LIB_RELEASE}>
    $<$<CONFIG:RelWithDebInfo>:${QFLUENT_LIB_RELEASE}>
    $<$<CONFIG:MinSizeRel>:${QFLUENT_LIB_RELEASE}>
)

# 链接 Qt 库
target_link_libraries(QFluentExample PRIVATE
    Qt${QT_VERSION_MAJOR}::Core
    Qt${QT_VERSION_MAJOR}::Widgets
    Qt${QT_VERSION_MAJOR}::Svg
    Qt${QT_VERSION_MAJOR}::Xml
)

if(WIN32 AND NOT BUILD_QWINDOWKIT)
    target_link_libraries(QFluentExample PRIVATE dwmapi)
endif()

if (BUILD_QWINDOWKIT)
    add_subdirectory(libs/qwindowkit)
    target_link_libraries(QFluentExample PRIVATE QWindowKit::Widgets)
    target_compile_definitions(QFluentExample PRIVATE USE_QWINDOWKIT=1)
else()
    target_compile_definitions(QFluentExample PRIVATE USE_QWINDOWKIT=0)
endif()

# ============================================================================
# 运行时库处理（平台特定）
# ============================================================================
if(WIN32)
    # Windows: 复制 DLL 到可执行文件目录
    if(WIN32)
        add_custom_command(TARGET QFluentExample POST_BUILD
            COMMAND powershell -ExecutionPolicy Bypass -Command "Copy-Item -Path '${SDK_DIR}/$<CONFIG>/*.dll' -Destination '$<TARGET_FILE_DIR:QFluentExample>' -Force"
            COMMENT "Copying runtime DLLs (Windows)"
        )
endif()
elseif(UNIX AND NOT APPLE)
    # Linux: 设置 RPATH
    set_target_properties(QFluentExample PROPERTIES
        INSTALL_RPATH "${SDK_DIR}/$<CONFIG>"
        BUILD_RPATH "${SDK_DIR}/$<CONFIG>"
        INSTALL_RPATH_USE_LINK_PATH TRUE
    )
message(STATUS "RPATH set to: ${SDK_DIR}/$<CONFIG>")
elseif(APPLE)
    # macOS: 设置 RPATH
    set_target_properties(QFluentExample PROPERTIES
        INSTALL_RPATH "@executable_path/../Frameworks;${SDK_DIR}/$<CONFIG>"
        BUILD_RPATH "${SDK_DIR}/$<CONFIG>"
        MACOSX_RPATH ON
    )
endif()
