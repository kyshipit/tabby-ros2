cmake_minimum_required(VERSION 3.14)
project(robot_eai)

# ==========================================================================
# 1. 语言标准
# ==========================================================================
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 编译警告（与 bringup 一致）
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 目标平台（保留 robot_eai 特有逻辑）
if(NOT DEFINED TARGET_SOC)
  set(TARGET_SOC "rk3588")
  message(STATUS "robot_eai: TARGET_SOC default rk3588")
endif()
message(STATUS "robot_eai: TARGET_SOC=${TARGET_SOC}")

# ==========================================================================
# 2. 依赖查找
# ==========================================================================
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(ament_index_cpp REQUIRED)
find_package(Threads REQUIRED)

# OpenCV：板端系统安装，走标准 find_package（替代原硬编码 sysroot 路径）
find_package(OpenCV REQUIRED COMPONENTS
  core flann videoio video highgui imgcodecs imgproc dnn)

# ==========================================================================
# 3. ROS2 消息生成
# ==========================================================================
set(msg_files
  "msg/Point2D.msg"
  "msg/Box.msg"
  "msg/DetectionResult.msg"
)
rosidl_generate_interfaces(${PROJECT_NAME} ${msg_files})

# ==========================================================================
# 4. 3rdparty 和 utils 子项目
# ==========================================================================
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/ 3rdparty.out)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/utils/ utils.out)

# ==========================================================================
# 5. 平台相关库（RKNN / RKLLM）
# ==========================================================================
set(RKNN_API_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/rknpu2/Linux/aarch64)
set(LIBRKNNRT ${RKNN_API_DIR}/librknnrt.so)
set(LIBRKNNRT_INCLUDES ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/rknpu2/include)

set(RKLLM_API_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/rkllm)
set(LIBRKLLMRT ${RKLLM_API_DIR}/lib/librkllmrt.so)
set(LIBGOMP ${RKLLM_API_DIR}/lib/libgomp/libgomp.so)

# ==========================================================================
# 6. 可执行文件
# ==========================================================================
set(robot_eai_SRC
  app/main_ros.cc
  app/console_ui/console_ui.cpp
  engine/pipeline.cpp
  engine/thread_pool.cpp
  platform/model_coordinator.cpp
  platform/llm_greeting.cpp
  capture/camera_source.cpp
  capture/frame_transform.cpp
  display/opencv_display_sink.cpp
  display/result_overlay.cpp
  adapters/yolo/yolo_adapter.cpp
  adapters/yolo/yolo_postprocess.cpp
  adapters/scrfd/scrfd_adapter.cpp
  adapters/scrfd/scrfd_postprocess.cpp
  adapters/llm/rkllm_session.cpp
  adapters/llm/llm_worker.cpp
  adapters/melotts/melotts_engine.cc
  adapters/melotts/melotts_process.cc
  adapters/melotts/melotts_session.cpp
  voice/tts_worker.cpp
  voice/audio_player.cpp
  voice/tts_text_sanitizer.cpp
  voice/tts_ingress.cpp
  voice/tts_planner.cpp
  voice/voice_reply_bridge.cpp
  ros_bridge/ros_bridge.cpp
)

add_executable(robot_eai_app ${robot_eai_SRC})
target_link_options(robot_eai_app PRIVATE -no-pie)

# 原 toolchain 文件里的编译选项，板端继续保留
target_compile_options(robot_eai_app PRIVATE
  -funwind-tables -rdynamic
  -Wno-write-strings -Wno-return-type)
target_compile_options(robot_eai_app PRIVATE
  $<$<CONFIG:Debug>:-g;-O0;-ggdb;-gdwarf>)

target_include_directories(robot_eai_app PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/adapters/yolo
  ${CMAKE_CURRENT_SOURCE_DIR}/adapters/scrfd
  ${CMAKE_CURRENT_SOURCE_DIR}/adapters/llm
  ${CMAKE_CURRENT_SOURCE_DIR}/adapters/melotts
  ${CMAKE_CURRENT_SOURCE_DIR}/voice
  ${CMAKE_CURRENT_SOURCE_DIR}/utils
  ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/timer
  ${RKLLM_API_DIR}/include
  ${LIBRKNNRT_INCLUDES}
  ${OpenCV_INCLUDE_DIRS}
)

# ROS 依赖（与 bringup 同风格）
ament_target_dependencies(robot_eai_app
  rclcpp
  std_msgs
  ament_index_cpp
)

# 非 ROS 库（RKNN / RKLLM / OpenCV / 子项目 target）
target_link_libraries(robot_eai_app
  imageutils
  fileutils
  imagedrawing
  audioutils
  ${LIBRKNNRT}
  ${LIBRKLLMRT}
  ${LIBGOMP}
  ${OpenCV_LIBS}
  Threads::Threads
)

add_definitions(-DTRACETOOLS_DISABLED)

# 本包 msg 的 C++ typesupport
rosidl_get_typesupport_target(cpp_typesupport_target "${PROJECT_NAME}" "rosidl_typesupport_cpp")
target_link_libraries(robot_eai_app "${cpp_typesupport_target}")

# ==========================================================================
# 7. 测试（与 bringup 同风格）
# ==========================================================================
if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

# ==========================================================================
# 8. 安装
# ==========================================================================
install(TARGETS robot_eai_app DESTINATION lib/${PROJECT_NAME})
install(FILES ${LIBRKNNRT} DESTINATION lib)
install(FILES ${LIBRKLLMRT} ${LIBGOMP} DESTINATION lib)

install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/coco_80_labels_list.txt
  DESTINATION share/${PROJECT_NAME}/model)
install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/yolov5.rknn
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/scrfd.rknn
  DESTINATION share/${PROJECT_NAME}/model OPTIONAL)
install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/encoder-ZH_MIX_EN.rknn
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/decoder-ZH_MIX_EN.rknn
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/lexicon.txt
  ${CMAKE_CURRENT_SOURCE_DIR}/../../model/tokens.txt
  DESTINATION share/${PROJECT_NAME}/model OPTIONAL)

file(GLOB CONFIG_FILES "${CMAKE_CURRENT_SOURCE_DIR}/config/*.yaml")
install(FILES ${CONFIG_FILES} DESTINATION share/${PROJECT_NAME}/config)

ament_export_dependencies(rosidl_default_runtime)
ament_package()