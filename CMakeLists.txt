# CMakeLists.txt

cmake_minimum_required(VERSION 3.15)
project(libdb LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 17)

if(MSVC)
  # Устанавливаем минимальную целевую версию Windows.
  # 0x0A00 - Windows 10.
  # 0x0601 - Windows 7.
  add_compile_definitions(_WIN32_WINNT=0x0A00)
endif()

# Подключение Conan
include(${CMAKE_BINARY_DIR}/conan_toolchain.cmake) # Генерируется CMakeToolchain

find_package(SQLite3 REQUIRED)
find_package(Catch2 QUIET CONFIG)

if(Catch2_FOUND)
    message(STATUS "Catch2 found, enabling tests. (BUILD_TESTING=ON)")
    option(BUILD_TESTING "Build the tests" ON)
    # Включаем CTest для управления тестами
    include(CTest) 
else()
    message(STATUS "Catch2 not found, disabling tests. (BUILD_TESTING=OFF)")
    option(BUILD_TESTING "Build the tests" OFF)
endif()

# Основная библиотека
add_library(libdb STATIC src/db.cpp)

target_include_directories(libdb PUBLIC 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

target_link_libraries(libdb PUBLIC SQLite::SQLite3)

option(BUILD_TESTING "Build tests" ON)  # Флаг для управления тестами

# Тесты (только если включен BUILD_TESTING)
if(BUILD_TESTING)
    find_package(Catch2 REQUIRED)
    
    add_executable(db_tests  
        test/test.cpp 
    )
    
    target_link_libraries(db_tests PRIVATE
	libdb
	Catch2::Catch2WithMain
    )

    # Подключаем модуль Catch2 для работы с тестами
    include(CTest)
    include(Catch)
    catch_discover_tests(db_tests)
endif()
