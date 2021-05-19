# Python_FIND_REGISTRY and CMAKE_FIND_FRAMEWORK should be set to NEVER:
# - https://cmake.org/cmake/help/v3.13/module/FindPython.html#hints
# From 3.15 use Python_FIND_VIRTUALENV = ONLY as it has fewer side effects
# - https://cmake.org/cmake/help/v3.15/module/FindPython.html#hints
cmake_minimum_required(VERSION 3.13)

project(hunter_venv)

if("${HUNTER_VENV_PYTHON_VERSION}" STREQUAL "")
  message(FATAL_ERROR "HUNTER_VENV_PYTHON_VERSION is empty")
endif()

set(HUNTER_VENV_USE_2 FALSE)
set(HUNTER_VENV_USE_3 FALSE)

if("${HUNTER_VENV_PYTHON_VERSION}" STREQUAL "2")
  set(HUNTER_VENV_USE_2 TRUE)
endif()

if("${HUNTER_VENV_PYTHON_VERSION}" MATCHES "^2\\.")
  set(HUNTER_VENV_USE_2 TRUE)
endif()

if("${HUNTER_VENV_PYTHON_VERSION}" STREQUAL "3")
  set(HUNTER_VENV_USE_3 TRUE)
endif()

if("${HUNTER_VENV_PYTHON_VERSION}" MATCHES "^3\\.")
  set(HUNTER_VENV_USE_3 TRUE)
endif()

if(HUNTER_VENV_USE_2 AND HUNTER_VENV_USE_3)
  message(FATAL_ERROR "Internal error")
endif()

if(NOT HUNTER_VENV_USE_2 AND NOT HUNTER_VENV_USE_3)
  message(
      FATAL_ERROR
      "Incorrect version: ${HUNTER_VENV_PYTHON_VERSION}\n"
      "Only 2.* and 3.* supported."
  )
endif()

find_package(Python ${HUNTER_VENV_PYTHON_VERSION} EXACT REQUIRED)

message("Python_EXECUTABLE: ${Python_EXECUTABLE}")
message("Python_SITELIB: ${Python_SITELIB}")

message("Python_VERSION: ${Python_VERSION}")
message("Python_VERSION_MAJOR: ${Python_VERSION_MAJOR}")
message("Python_VERSION_MINOR: ${Python_VERSION_MINOR}")
message("Python_VERSION_PATCH: ${Python_VERSION_PATCH}")

if(HUNTER_VENV_USE_2)
  if(NOT "${Python_VERSION_MAJOR}" STREQUAL "2")
    message(FATAL_ERROR "Internal error")
  endif()
  find_package(Python2 ${HUNTER_VENV_PYTHON_VERSION} EXACT REQUIRED)
  message("Python2_EXECUTABLE: ${Python2_EXECUTABLE}")
  message("Python2_SITELIB: ${Python2_SITELIB}")
endif()

if(HUNTER_VENV_USE_3)
  if(NOT "${Python_VERSION_MAJOR}" STREQUAL "3")
    message(FATAL_ERROR "Internal error")
  endif()
  find_package(Python3 ${HUNTER_VENV_PYTHON_VERSION} EXACT REQUIRED)
  message("Python3_EXECUTABLE: ${Python3_EXECUTABLE}")
  message("Python3_SITELIB: ${Python3_SITELIB}")
endif()

set(venv_dir "${CMAKE_CURRENT_BINARY_DIR}/venv")

if(HUNTER_VENV_USE_3)
  if(${HUNTER_VENV_PYTHON_VERSION} VERSION_GREATER_EQUAL "3.3")
    set(
        cmd
        ${Python_EXECUTABLE}
        -m
        venv
        --copies # Always copy files rather than symlinking
        ${venv_dir}
    )
    set(USE_VENV 1)
  else()
    set(
        cmd
        ${Python_EXECUTABLE}
        -m
        virtualenv
        --python=${Python_EXECUTABLE}
        --always-copy # Always copy files rather than symlinking
        ${venv_dir}
    )
    set(USE_VIRTUALENV 1)
  endif()

endif()

set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")

execute_process(COMMAND ${cmd} RESULT_VARIABLE result)

if(NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

# `virtualenv --relocatable` will not help, still not relocatable
# Update  `virtualenv --relocatable` is not supported by virtualenv anymore
set(
    unrelocatable_list
    activate
    activate.bat
    activate.csh
    activate.fish
    activate.ps1
    activate.xsh
    deactivate.bat
)

foreach(x ${unrelocatable_list})
  file(REMOVE "${venv_dir}/bin/${x}")
  file(REMOVE "${venv_dir}/Scripts/${x}")
endforeach()

# `virtualenv --relocatable` will fail if path to the script is too long.
# Create some files manually. Taken from results of running conda, e.g. for pip:
#   > conda create --name a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789
#   > conda activate a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789
#   > cat `which pip`
# Update  `virtualenv --relocatable` is not supported by virtualenv anymore

function(create_tool path from)
  set(dst "${venv_dir}/bin/${path}")

  if(NOT EXISTS "${dst}")
    return()
  endif()

  set(temp "${dst}.TEMP")

  # Use:
  # - from
  configure_file("cmake/template/tool.cmake.in" "${temp}" @ONLY)

  # Trick to keep file permissions
  file(READ "${temp}" dst_content)
  file(WRITE "${dst}" "${dst_content}")
  file(REMOVE "${temp}")
endfunction()

create_tool(easy_install "setuptools.command.easy_install")
create_tool(easy_install-${Python_VERSION_MAJOR}.${Python_VERSION_MINOR} "setuptools.command.easy_install")
create_tool(pip "pip._internal")
create_tool(pip${Python_VERSION_MAJOR} "pip._internal")
create_tool(pip${Python_VERSION_MAJOR}.${Python_VERSION_MINOR} "pip._internal")
create_tool(wheel "wheel.cli")

set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")

execute_process(COMMAND ${cmd} RESULT_VARIABLE result)

if (UNIX)
# I only tested this on linux remove the if, if it works on other OS-es

  # virtualenv v0.20 and above does not copy python-config, the venv module never did. However a lot of
  # (g*: gobject, gi, etc) packages needs it to be able to be configured. So let's do it here.
  get_filename_component(PYTHON_BIN_DIR ${Python_EXECUTABLE} DIRECTORY)
  file(GLOB PYTHON_CONFIG_EXECUTABLES
          "${PYTHON_BIN_DIR}/python*-config"
          )
  file(COPY ${PYTHON_CONFIG_EXECUTABLES} DESTINATION ${venv_dir}/bin)

  #venv does not copy the C headers, let's do it ourselves
  if (DEFINED USE_VENV)
    message("Copying C headers and libraries...")
    #headers
    file(COPY ${Python_ROOT_DIR}/include DESTINATION ${venv_dir})
    #libs
    file(GLOB PYTHON_LIBRARIES
            "${PYTHON_BIN_DIR}/lib/libpython*"
            )
    file(COPY ${PYTHON_LIBRARIES} DESTINATION ${venv_dir}/lib)
  endif()

endif()

if(NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

install(
    DIRECTORY ${venv_dir}/
    DESTINATION .
    USE_SOURCE_PERMISSIONS
)

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")

include(CMakePackageConfigHelpers)

# Use variables:
#   - PROJECT_NAME
#   - HUNTER_VENV_PYTHON_VERSION
#   - HUNTER_VENV_USE_2
#   - HUNTER_VENV_USE_3
configure_package_config_file(
    "cmake/template/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES "${project_config}"
    DESTINATION "${config_install_dir}"
)

if (UNIX)
  # Since relocatable is not supported anymore, we will try to do something similar here.
  message("venv dir: ${venv_dir}")
#  message("execute_process(COMMAND find ${CMAKE_INSTALL_PREFIX}/bin -type f -exec ex -sc \"%s,${venv_dir},${CMAKE_INSTALL_PREFIX},g|x\" \"{}\" \";\")")
  install(CODE "execute_process(COMMAND find ${CMAKE_INSTALL_PREFIX}/bin -type f -exec ex -sc \"%s,${venv_dir},${CMAKE_INSTALL_PREFIX},g|x\" \"{}\" \";\")")
  #..../Build/hunter_venv/Install/..... will become ..../Install/.....
  install(CODE "execute_process(COMMAND find ${CMAKE_INSTALL_PREFIX}/bin -type f -exec ex -sc \"%s,/Build/hunter_venv/Install,/Install,g|x\" \"{}\" \";\")")
endif()