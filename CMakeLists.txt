cmake_minimum_required(VERSION 3.10)

# プロジェクト名と使用する言語を設定
project(test-mp4 C CXX)

set(CMAKE_CXX_STANDARD 17)


#find_package(PkgConfig)

#pkg_check_modules(GSTREAMER REQUIRED 
#  glib-2.0
#  gstreamer-1.0 
#  gstreamer-sdp-1.0
#  gstreamer-rtp-1.0
#  gstreamer-rtsp-1.0
#  gstreamer-rtsp-server-1.0
#)

#message("GSTREAMER_INCLUDE_DIRS: ${GSTREAMER_INCLUDE_DIRS}")

#message("GSTREAMER_LIBRARY_DIRS: ${GSTREAMER_LIBRARY_DIRS}")

#link_directories(${GSTREAMER_LIBRARY_DIRS})

# コンパイルフラグを設定
#set(CMAKE_C_FLAGS "-Wall")
#set(CMAKE_C_FLAGS_DEBUG "-g3 -O0 -pg")
#set(CMAKE_C_FLAGS_RELEASE "-O2 -s -DNDEBUG -march=native")
#set(CMAKE_C_FLAGS_RELWITHDEBINFO "-g3 -Og -pg")
#set(CMAKE_C_FLAGS_MINSIZEREL "-Os -s DNDEBUG -march=native")

#set(CMAKE_CXX_FLAGS "-Wall -std=c++17")
#set(CMAKE_CXX_FLAGS_DEBUG "-g -DDEBUG_BUILD")
#set(CMAKE_CXX_FLAGS_RELEASE "-O2 -s -DNDEBUG -march=native")
#set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-g3 -Og -pg")
#set(CMAKE_CXX_FLAGS_MINSIZEREL "-Os -s DNDEBUG -march=native")

include(cmake/detect_gstreamer.cmake)


# 実行ファイルの作成
add_executable(test-mp4 
  src/main.cpp)

# gstreamer ヘッダーへのパスを設定
target_include_directories(test-mp4  PUBLIC ${GSTREAMER_INCLUDE_DIRS})

# gstreamermm ライブラリの設定
target_link_libraries(test-mp4  ${GSTREAMER_LIBRARIES} )
#target_link_libraries(test-mp4  pthread)

# gstreamer のコンパイルオプションを設定
target_compile_options(test-mp4  PUBLIC ${GSTREAMER_CFLAGS_OTHER})
