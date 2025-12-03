cmake_minimum_required(VERSION 3.5)
project(QFluentExample LANGUAGES CXX)

# 支持Qt5和Qt6
find_package(QT NAMES Qt6 Qt5 REQUIRED COMPONENTS Core)
find_package(Qt${QT_VERSION_MAJOR} REQUIRED COMPONENTS Core Widgets Svg Xml)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

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
    # 对于Qt5，需要生成RCC源文件
    add_executable(QFluentExample
        WIN32 MACOSX_BUNDLE
        ${CPP_SOURCES}
        ${HEADER_SOURCES}
        version.rc
        res/res.qrc
    )
endif()



# 初始化变量
unset(QT_VERSION_STRING)

if(TARGET Qt6::Core)
    set(QT_MAJOR_VERSION "6")
    set(QT_VERSION_STRING ${Qt6_VERSION})
elseif(TARGET Qt5::Core)
    set(QT_MAJOR_VERSION "5")
    # Qt5 每个模块有自己的版本变量，Core 是基础
    set(QT_VERSION_STRING ${Qt5Core_VERSION})
else()
    message(FATAL_ERROR "Neither Qt5 nor Qt6 found! Please install Qt and set CMAKE_PREFIX_PATH.")
endif()

# 获取编译器信息
set(COMPILER_INFO ${CMAKE_CXX_COMPILER_ID})

# 拼接最终字符串
set(QT_AND_COMPILER_STRING "${QT_VERSION_STRING}_${COMPILER_INFO}")

# 输出
message(STATUS "Detected build environment: ${QT_AND_COMPILER_STRING}")


set(SDK_DIR "${CMAKE_CURRENT_SOURCE_DIR}/libs/${QT_AND_COMPILER_STRING}")

message(STATUS "SDK Path:: ${SDK_DIR}")

add_custom_command(TARGET QFluentExample POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${SDK_DIR}/bin/$<CONFIG>
    $<TARGET_FILE_DIR:QFluentExample>
    COMMENT "Copying DLLs from ${SDK_DIR}"
)

target_link_libraries(QFluentExample PRIVATE
    $<$<CONFIG:Debug>:${SDK_DIR}/lib/Debug/QFluent.lib>
    $<$<CONFIG:Release>:${SDK_DIR}/lib/Release/QFluent.lib>
)

target_link_libraries(QFluentExample
    PRIVATE
    Qt${QT_VERSION_MAJOR}::Core
    Qt${QT_VERSION_MAJOR}::Widgets
    Qt${QT_VERSION_MAJOR}::Svg
    Qt${QT_VERSION_MAJOR}::Xml
)
include(GNUInstallDirs)
install(TARGETS QFluentExample
    BUNDLE DESTINATION .
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
