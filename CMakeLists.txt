cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

# 1. Globale Flags für das GESAMTE Projekt (inkl. Jolt-Subdirectory)
# Das löst die "undefined reference" Fehler beim Linken
add_definitions(-DJPH_DEBUG_RENDERER -DJPH_OBJECT_STREAM)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Nur Debug/Release Typen erlauben
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release")
endif()

project(JoltC VERSION 0.1
    DESCRIPTION "C Wrapper for Jolt Physics"
    LANGUAGES C CXX)

# Optionen
option(USE_ASSERTS "Enable asserts" OFF)
option(DOUBLE_PRECISION "Use double precision math" OFF)
option(OBJECT_LAYER_BITS "Number of bits in ObjectLayer" 16)

include(CMakeDependentOption)
cmake_dependent_option(USE_STATIC_MSVC_RUNTIME_LIBRARY "Use the static MSVC runtime library" ON "MSVC;NOT WINDOWS_STORE" OFF)

if (USE_STATIC_MSVC_RUNTIME_LIBRARY)
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

# 2. Das Library-Target erstellen (SHARED für einfacheres Linken in C/Pascal)
add_library(joltc SHARED
    JoltC/JoltC.h
    JoltC/Enums.h
    JoltC/Functions.h
    JoltCImpl/Test.cpp
    JoltCImpl/JoltC.cpp
)

# 3. Konfiguration des Targets
target_compile_definitions(joltc PUBLIC JPH_DEBUG_RENDERER JPH_OBJECT_STREAM)

# Warnungen unterdrücken, damit GCC 13 nicht bei Kleinigkeiten abbricht
target_compile_options(joltc PRIVATE 
    -Wno-unused-function 
    -Wno-unused-variable 
    -Wno-invalid-offsetof
    -Wno-missing-field-initializers
)

# System-Abhängigkeiten (Wichtig für Pascal-Linker!)
target_link_libraries(joltc PUBLIC stdc++ m pthread)

target_compile_features(joltc PUBLIC cxx_std_17)
target_include_directories(joltc PUBLIC . JoltPhysics)

# 4. Jolt Engine Unterordner hinzufügen (MUSS nach den globalen Definitionen stehen)
add_subdirectory(JoltPhysics/Build)

# Verbindung zur Jolt Engine herstellen
target_link_libraries(joltc PUBLIC Jolt)

# 5. Installation
install(TARGETS joltc DESTINATION lib)

# Header-Installation (Für /usr/local/include/JoltC/)
install(FILES 
    JoltC/JoltC.h
    JoltC/Enums.h
    JoltC/Functions.h
    DESTINATION include/JoltC)

# Compiler-spezifische Optionen
if(MSVC)
  target_compile_options(joltc PRIVATE /W4 /Zi /wd4505)
else()
  target_compile_options(joltc PRIVATE -Wall -Wextra -Wpedantic -fno-rtti)
endif()

if (DOUBLE_PRECISION)
    target_compile_definitions(joltc PUBLIC JPC_DOUBLE_PRECISION)
endif()

if (OBJECT_LAYER_BITS)
    target_compile_definitions(joltc PUBLIC JPC_OBJECT_LAYER_BITS=${OBJECT_LAYER_BITS})
endif()

# HelloWorld Beispiel
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    add_executable(HelloWorld HelloWorld/main.cpp)
    target_compile_features(HelloWorld PRIVATE cxx_std_20)
    target_include_directories(HelloWorld PUBLIC .)
    target_link_libraries(HelloWorld PUBLIC joltc)
endif()

