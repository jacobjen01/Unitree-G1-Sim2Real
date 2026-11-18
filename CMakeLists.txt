cmake_minimum_required(VERSION 3.10)
project(g1_sim2real 
  VERSION 1.0.0
  DESCRIPTION "Unitree G1 RL Sim2Real Deployment"
  LANGUAGES CXX
)

# ==== C++ Standard ====
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

message(STATUS "=== ${PROJECT_NAME} v${PROJECT_VERSION} ===")
message(STATUS "System Arch: ${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "C++ Standard: ${CMAKE_CXX_STANDARD}")

# ==== PATHS ====
set(UNITREE_SDK2_DIR    ${CMAKE_SOURCE_DIR}/../unitree_sdk2)

set(THIRD_PARTY_DIR     ${CMAKE_SOURCE_DIR}/thirdparty)
set(YAML_CPP_SRC_DIR    ${THIRD_PARTY_DIR}/yaml-cpp)

# ==== Unitree SDK2 ====
if (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
  set(UNITREE_LIB_DIR    ${UNITREE_SDK2_DIR}/lib/x86_64)
  set(UNITREE_3P_LIB_DIR ${UNITREE_SDK2_DIR}/thirdparty/lib/x86_64)
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
  set(UNITREE_LIB_DIR    ${UNITREE_SDK2_DIR}/lib/aarch64)
  set(UNITREE_3P_LIB_DIR ${UNITREE_SDK2_DIR}/thirdparty/lib/aarch64)
else()
  message(FATAL_ERROR "Unsupported system arch: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

# ==== YAML-CPP ====
set(YAML_CPP_BUILD_TESTS   OFF CACHE BOOL "" FORCE)
set(YAML_CPP_BUILD_TOOLS   OFF CACHE BOOL "" FORCE)
set(YAML_CPP_BUILD_CONTRIB OFF CACHE BOOL "" FORCE)

if (NOT EXISTS "${YAML_CPP_SRC_DIR}/CMakeLists.txt")
  message(FATAL_ERROR "yaml-cpp not found: ${YAML_CPP_SRC_DIR}")
endif()
add_subdirectory(${YAML_CPP_SRC_DIR})

# ==== ONNXRUNTIME (prebuilt, from NuGet) ====
set(ORT_VERSION 1.22.0)
set(ORT_DIR ${CMAKE_BINARY_DIR}/onnxruntime-${ORT_VERSION})
if (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
  set(ORT_RID linux-x64)
else()
  set(ORT_RID linux-arm64)
endif()
set(ORT_LIB_DIR ${ORT_DIR}/runtimes/${ORT_RID}/native)

if (NOT EXISTS ${ORT_LIB_DIR}/libonnxruntime.so)
  file(DOWNLOAD
    https://api.nuget.org/v3-flatcontainer/microsoft.ml.onnxruntime/${ORT_VERSION}/microsoft.ml.onnxruntime.${ORT_VERSION}.nupkg
    ${CMAKE_BINARY_DIR}/onnxruntime.nupkg SHOW_PROGRESS STATUS ORT_DL)
  list(GET ORT_DL 0 ORT_DL_CODE)
  if (NOT ORT_DL_CODE EQUAL 0)
    message(FATAL_ERROR "onnxruntime download failed: ${ORT_DL}")
  endif()
  file(ARCHIVE_EXTRACT INPUT ${CMAKE_BINARY_DIR}/onnxruntime.nupkg DESTINATION ${ORT_DIR})
  # soname is libonnxruntime.so.1; needed at runtime
  file(CREATE_LINK libonnxruntime.so ${ORT_LIB_DIR}/libonnxruntime.so.1 SYMBOLIC)
endif()

add_library(onnxruntime SHARED IMPORTED)
set_target_properties(onnxruntime PROPERTIES
  IMPORTED_LOCATION "${ORT_LIB_DIR}/libonnxruntime.so"
)
target_include_directories(onnxruntime INTERFACE "${ORT_DIR}/build/native/include")

# ==== Common Deps ====
add_library(com_deps INTERFACE)
target_include_directories(com_deps INTERFACE
  ${CMAKE_SOURCE_DIR}/include
  ${UNITREE_SDK2_DIR}/include
  ${UNITREE_SDK2_DIR}/thirdparty/include
  ${UNITREE_SDK2_DIR}/thirdparty/include/ddscxx
)
target_link_directories(com_deps INTERFACE
  ${UNITREE_LIB_DIR}
  ${UNITREE_3P_LIB_DIR}
)
target_link_libraries(com_deps INTERFACE
  unitree_sdk2
  ddsc
  ddscxx
  pthread
)

# ==== Libs ====
add_library(utils STATIC lib/utils.cpp)
target_include_directories(utils PUBLIC ${CMAKE_SOURCE_DIR}/include)

# ==== Executables ====
add_executable(g1_arm_example src/g1_arm_example.cpp)
add_executable(wholebody_rl  src/wholebody_rl.cpp)

# ==== Link ====
target_link_libraries(g1_arm_example PRIVATE com_deps)
target_link_libraries(wholebody_rl  PRIVATE utils com_deps onnxruntime yaml-cpp)
