cmake_minimum_required(VERSION 3.10 FATAL_ERROR)
project(sapien LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS_DEBUG "-o0 -g3 -Wall -fsized-deallocation")
set(CMAKE_CXX_FLAGS_RELEASE "-o3 -g0 -Wall -fsized-deallocation")

set(PROFILE FALSE CACHE BOOL "Whether to use Profiler")

set(CUDA_INCLUDE_PATH "" CACHE FILEPATH "Path to CUDA include directory")

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif ()

set(CMAKE_DEBUG_POSTFIX "")
set(OpenGL_GL_PREFERENCE "LEGACY")

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    message("-- DEBUG Build")
    set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_DEBUG})
else ()
    message("-- RELEASE Build")
    set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_RELEASE})
endif ()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    message("Complie source code on Mac OSX")
    set(MACOSX TRUE)
    add_definitions(-D_USE_MACOSX)
endif ()

find_package(OpenGL REQUIRED)
find_package(GLEW REQUIRED)
find_package(ASSIMP REQUIRED)
find_package(spdlog REQUIRED)

if (EXISTS ${CUDA_INCLUDE_PATH})
    include_directories(${CUDA_INCLUDE_PATH})
endif ()

include_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/include"
        "${CMAKE_SOURCE_DIR}/../PhysX/pxshared/include" "3rd_party/optifuser/include"
        "src/renderer"
        )

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    link_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/bin/linux.clang/checked"
            "3rd_party/optifuser/build/lib")
else ()
    link_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/bin/linux.clang/release"
            "3rd_party/optifuser/build/lib")
endif ()

## include headers inside the project
include_directories(
        "3rd_party/optifuser/include"
        "3rd_party/optifuser/3rd_party/imgui"
        "3rd_party/optifuser/3rd_party/imgui/examples"
        "3rd_party/optifuser/3rd_party/ImGuizmo"
        "src"
        "src/renderer")

## include headers outside the project
include_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/include"
        "${CMAKE_SOURCE_DIR}/../PhysX/pxshared/include"
        "/usr/include/eigen3")

if (MACOSX)
    include_directories("/usr/local/include" "/usr/local/opt/glew/include"
            "/usr/local/include/eigen3")
endif ()

## Link directory inside the project
link_directories("${CMAKE_CURRENT_BINARY_DIR}/3rd_party/optifuser/lib")

## Link directory outside the project
if (NOT MACOSX)
    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        link_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/bin/linux.clang/checked")
    else ()
        link_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/bin/linux.clang/release")
    endif ()
else ()
    link_directories("${CMAKE_SOURCE_DIR}/../PhysX/physx/bin/mac.x86_64/release")
    link_directories("/usr/local/opt/glm/lib" "/usr/local/opt/glew/lib"
            "/usr/local/lib")
endif ()

set(OPTIX_HOME "NOT FOUND" CACHE FILEPATH "Home to OptiX")
if (EXISTS ${OPTIX_HOME})
    include_directories("${OPTIX_HOME}/include")
    link_directories("${OPTIX_HOME}/lib64")
    set(OPTIX_LIBRARY optix)
    add_definitions(-D_USE_OPTIX)
endif ()

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-D_DEBUG)
    # add_definitions(-D_PVD)
    add_definitions(-DPVD_HOST="192.168.0.110")
else ()
    add_definitions(-DNDEBUG)
endif ()

add_subdirectory("3rd_party/optifuser" EXCLUDE_FROM_ALL)
add_subdirectory("3rd_party/pybind11")
add_subdirectory("3rd_party/tinyxml2")
include_directories("3rd_party/pybind11/include")

file(GLOB_RECURSE PROJECT_SRC "src/*.h" "src/*.cpp")
add_custom_target(shader COMMAND cp -r ${CMAKE_CURRENT_SOURCE_DIR}/glsl_shader ${CMAKE_CURRENT_BINARY_DIR})
add_library(sapien ${PROJECT_SRC})
add_dependencies(sapien shader)

if (EXISTS ${OPTIX_HOME})
    add_dependencies(sapien ptx_dir)
endif ()

if (${PROFILE})
    find_package(easy_profiler REQUIRED)
    add_definitions(-D_PROFILE)
    set(EASY_PROFILER_LIBRARY easy_profiler)
    message("-- Profiler On")
else ()
    message("-- Profiler Off")
endif ()

if (MACOSX)
    target_link_libraries(sapien
            libPhysXCharacterKinematic_static_64.a libPhysXCommon_static_64.a
            libPhysXCooking_static_64.a libPhysXExtensions_static_64.a
            libPhysXFoundation_static_64.a libPhysXPvdSDK_static_64.a
            libPhysX_static_64.a libPhysXVehicle_static_64.a
            libSnippetRender_static_64.a libSnippetUtils_static_64.a
            pthread ${OPENGL_LIBRARY} glfw GLEW ${ASSIMP_LIBRARIES} dl optifuser
            tinyxml2 ${OPTIX_LIBRARY} ${SPDLOG_LIBRARIES} ${EASY_PROFILER_LIBRARY})
else ()
    target_link_libraries(sapien -Wl,--start-group
            libPhysXCharacterKinematic_static_64.a libPhysXCommon_static_64.a
            libPhysXCooking_static_64.a libPhysXExtensions_static_64.a
            libPhysXFoundation_static_64.a libPhysXPvdSDK_static_64.a
            libPhysX_static_64.a libPhysXVehicle_static_64.a
            libSnippetRender_static_64.a libSnippetUtils_static_64.a -Wl,--end-group
            pthread ${OPENGL_LIBRARY} glfw GLEW ${ASSIMP_LIBRARIES} dl optifuser
            stdc++fs tinyxml2 ${OPTIX_LIBRARY} ${SPDLOG_LIBRARIES} ${EASY_PROFILER_LIBRARY})
endif ()

set_target_properties(optifuser PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
set_target_properties(sapien PROPERTIES POSITION_INDEPENDENT_CODE TRUE)

pybind11_add_module(pysapien "python/pysapien.cpp")
target_link_libraries(pysapien PRIVATE sapien)

file(GLOB_RECURSE UNITTEST_SRC unittest/*.cpp)

add_executable(unittest ${UNITTEST_SRC})
target_link_libraries(unittest sapien)

add_executable(manual_actor manualtest/actor.cpp python/pysapien_content.hpp)
target_link_libraries(manual_actor sapien)
add_executable(manual_articulation manualtest/articulation.cpp)
target_link_libraries(manual_articulation sapien)
add_executable(manual_urdf manualtest/urdf.cpp)
target_link_libraries(manual_urdf sapien)
add_executable(manual_kinematics manualtest/kinematics.cpp)
target_link_libraries(manual_kinematics sapien)

add_custom_target(python_test COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/test/*.py ${CMAKE_CURRENT_SOURCE_DIR}/test/*.json ${CMAKE_CURRENT_BINARY_DIR})

add_custom_target(manual_python COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/manualtest/*.py ${CMAKE_CURRENT_BINARY_DIR})


# Below are the ROS2 specified cmake
set(SUPPORTED_ROS_VERSION "dashing" "eloquent")
set(OLD_ROS_VERSION "crystal" "bouncy" "ardent" "melodic" "kinetic")
set(ROS $ENV{ROS_DISTRO})
if (ROS)
    list(FIND SUPPORTED_ROS_VERSION ${ROS} _index)
    if (_index GREATER -1)
        message("Your ROS Distribution is ${ROS}")
        if (CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            add_compile_options(-Wall -Wextra -Wpedantic)
        endif ()
        find_package(ament_cmake REQUIRED)
        find_package(rclcpp REQUIRED)
        find_package(sensor_msgs REQUIRED)
        find_package(std_msgs REQUIRED)
        find_package(sapien_ros2_communication_interface REQUIRED)
        find_package(moveit_core)
        find_package(moveit_ros_planning_interface)
        find_package(ament_index_cpp)

        include_directories(
                "ros2_src"
        )


        file(GLOB_RECURSE SAPIEN_ROBOT_SRC "ros2_src/controller/*.cpp" "ros2_src/sensor/*.cpp" "ros2_src/utils/*.hpp" "ros2_src/device/*.cpp")
        add_library(sapien_controller ${SAPIEN_ROBOT_SRC})
        target_link_libraries(sapien_controller sapien)
        ament_target_dependencies(sapien_controller rclcpp sensor_msgs sapien_ros2_communication_interface moveit_core moveit_ros_planning_interface)

        file(GLOB_RECURSE SAPIEN_ROBOT_MANAGER_SRC "ros2_src/manager/*.cpp")
        add_library(sapien_robot_handle ${SAPIEN_ROBOT_MANAGER_SRC} ros2_src/manager/robot_descriptor.cpp)
        target_link_libraries(sapien_robot_handle tinyxml2 sapien_controller)
        ament_target_dependencies(sapien_robot_handle ament_index_cpp)

        # test
        add_executable(manual_PS3 manualtest/PS3ROSTest.cpp)
        target_link_libraries(manual_PS3 sapien_controller)

        add_executable(test_ros2_robot_descriptor ros2_src/test/test_robot_descriptor.cpp)
        target_link_libraries(test_ros2_robot_descriptor sapien_robot_handle)

        # pybind for sapien-ros2
        set_target_properties(sapien_robot_handle PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
        set_target_properties(sapien_controller PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
        pybind11_add_module(pysapien_ros2 python/pysapien-ros2.cpp)
        target_link_libraries(pysapien_ros2 PRIVATE sapien_robot_handle)

        ament_package()
    elseif (${ROS} IN_LIST OLD_ROS_VERSION)
        message("Your ROS version not supported. Please update to kinetic or above.")
    endif ()
else ()
    message(WARNING "No ROS Found, will not compile source in ros2_src")
endif ()
