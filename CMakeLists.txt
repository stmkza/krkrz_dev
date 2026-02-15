cmake_minimum_required(VERSION 3.20)

set(PROJECT_NAME krkrz_all)
set(PROJECT_VERSION 1.0)

project(${PROJECT_NAME} VERSION ${PROJECT_VERSION})

if(WIN32 AND (CMAKE_SIZEOF_VOID_P EQUAL 8))
set(WIN64 1)
endif()

list(APPEND TVP_PLUGIN_FOLDERS ${CMAKE_CURRENT_SOURCE_DIR}/src/plugins)

# 基本プラグイン
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/src/plugins/tp_stub ${CMAKE_CURRENT_BINARY_DIR}/tp_stub)

# プラグイン一覧
list(APPEND TVP_PLUGINS
KAGParserEx
binaryStream
csvParser
fpslimit
getSample
json
layerExAreaAverage
layerExBTOA
layerExDraw
layerExImage
layerExLongExposure
layerExRaster
layerExSave
lineParser
#minizip
#psbfile
saveStruct
scriptsEx
shrinkCopy
#sigcheck
systemEx
tftSave
#krkrlua
)


if (WIN32)
list(APPEND TVP_PLUGINS
# WIN32固有機能をつかっていて現時点でWIN専用。要改修
addFont
memfile
#psdfile
varfile
fstat
gamepad
menu
msgreceiver
stdio
#resourceRW
win32dialog
windowEx
httprequest
shellExecute
#httpserv
process
messenger
windowExProgress
win32ole
)
endif()

# 特殊プラグイン
# STEAMプラグインはSDKパスが環境変数で提供された場合のみビルド
if(DEFINED ENV{STEAMWORKS_SDK})
list(APPEND PLUGINS 
#	steam
)
endif()

if (WIN64)
set(PLUGIN_DIR "plugin64")
set(EXENAME krkrz64)
else()
set(PLUGIN_DIR "plugin")
set(EXENAME krkrz)
endif()

# StaticPlugins.cpp 生成用の変数を初期化
set(STATIC_PLUGIN_EXTERNS "")
set(STATIC_PLUGIN_ATSTARTS "")

foreach(PLUGIN ${TVP_PLUGINS})
    if(${PLUGIN} IN_LIST TVP_PLUGINS_STATIC)
        set(TVP_STATIC_PLUGIN ON)
    else()
        set(TVP_STATIC_PLUGIN OFF)
    endif()

    # TVP_PLUGIN_FOLDERS から該当プラグインのフォルダを検索
    set(PLUGIN_SOURCE_DIR "")
    foreach(FOLDER ${TVP_PLUGIN_FOLDERS})
        if(EXISTS "${FOLDER}/${PLUGIN}")
            set(PLUGIN_SOURCE_DIR "${FOLDER}/${PLUGIN}")
            break()
        endif()
    endforeach()

    if(NOT PLUGIN_SOURCE_DIR)
        message(FATAL_ERROR "Plugin '${PLUGIN}' not found in any of: ${PLUGIN_FOLDERS}")
    endif()

    add_subdirectory(${PLUGIN_SOURCE_DIR} ${CMAKE_CURRENT_BINARY_DIR}/plugins/${PLUGIN})

    if(TVP_STATIC_PLUGIN)
        message(STATUS "static  plugin: ${PLUGIN}")
        list(APPEND KRKRZ_LIB_ADD ${PLUGIN})
        # StaticPlugins.cpp 用のデータを生成
        string(APPEND STATIC_PLUGIN_EXTERNS "extern \"C\" void krkrz_plugin_${PLUGIN}();\n")
        string(APPEND STATIC_PLUGIN_ATSTARTS "static tTVPAtStart AtStart_${PLUGIN}(TVP_ATSTART_PRI_PREPARE, [](){ krkrz_plugin_${PLUGIN}(); });\n")
    else()
        message(STATUS "dynamic plugin: ${PLUGIN}")
        install(TARGETS ${PLUGIN} 
            LIBRARY DESTINATION ${PLUGIN_DIR}
            RUNTIME DESTINATION ${PLUGIN_DIR}
        )
    endif()
endforeach()

# Static Pluginが存在する場合のみ StaticPlugins.cpp を生成してソースに追加
if(STATIC_PLUGIN_EXTERNS)
    set(STATIC_PLUGINS_CPP_CONTENT "#include \"tjsCommHead.h\"\n#include \"SysInitIntf.h\"\n${STATIC_PLUGIN_EXTERNS}${STATIC_PLUGIN_ATSTARTS}")
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/StaticPlugins.cpp" "${STATIC_PLUGINS_CPP_CONTENT}")
    list(APPEND KRKRZ_SRC_ADD "${CMAKE_CURRENT_BINARY_DIR}/StaticPlugins.cpp")
    message(STATUS "Generated StaticPlugins.cpp with static plugins")
endif()

# KRKRZ 本体
add_subdirectory(src/core ${CMAKE_CURRENT_BINARY_DIR}/core)
