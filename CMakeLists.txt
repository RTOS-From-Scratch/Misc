#/*
# * Description : CMake is an open-source, cross-platform family of tools
# *               designed to build, test and package software.
# * Usage       : CMake is used to control the software compilation process
# *               using simple platform and compiler independent 
# *               configuration files, and generate native makefiles
# *               and workspaces that can be used in the compiler environment
# *               of your choice.
# * Author      : Baron Leonardo <mohamedayman.fcis@zoho.com>
# * License     : Copyright (C) 2016 RTOS From Scratch
# *
# *
# * This program is free software: you can redistribute it and/or modify
# * it under the terms of the GNU General Public License as published by
# * the Free Software Foundation, either version 3 of the License, or
# * (at your option) any later version.
# *
# * This program is distributed in the hope that it will be useful,
# * but WITHOUT ANY WARRANTY; without even the implied warranty of
# * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# * GNU General Public License for more details.
# *
# * You should have received a copy of the GNU General Public License
# * along with this program.  If not, see <http://www.gnu.org/licenses/>.
# */

cmake_minimum_required(VERSION 3.2)

###################### Pre-Project configs ######################
# make the projetc name is as the same as current folder name
get_filename_component(Curr_dir ${CMAKE_CURRENT_SOURCE_DIR} NAME)
string(REPLACE " " "_" Curr_dir ${Curr_dir})

SET(PROJ_PATH ${CMAKE_CURRENT_SOURCE_DIR})
SET(CMAKE_SOURCE_DIR src)

SET(PROJECT_NAME ${Curr_dir})
#################################################################

######################## Toolchain paths ########################
SET(ARM_TOOLCHAIN_BIN_PATH /usr/bin)
SET(CMAKE_FIND_ROOT_PATH /usr/arm-none-eabi)
# Both -> the host system paths and the paths in CMAKE_FIND_ROOT_PATH will be searched
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY BOTH)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE BOTH)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE BOTH)
#################################################################

####################### Toolchain configs #######################
SET(CMAKE_SYSTEM_NAME Generic)
SET(CMAKE_SYSTEM_PROCESSOR arm)
SET(CMAKE_C_COMPILER ${ARM_TOOLCHAIN_BIN_PATH}/arm-none-eabi-gcc CACHE STRING "CC" FORCE)
SET(CMAKE_CXX_COMPILER ${ARM_TOOLCHAIN_BIN_PATH}/arm-none-eabi-g++ CACHE STRING "CXX" FORCE)
SET(CMAKE_ASM_COMPILER ${ARM_TOOLCHAIN_BIN_PATH}/arm-none-eabi-as CACHE STRING "ASM" FORCE)
#################################################################

################# check important files existance ################
if(${RFS_TESTING_${PROJECT_NAME}})
	# check `tm4c123gh6pm.lds` existance
	if( NOT EXISTS ${PROJ_PATH}/tm4c123gh6pm.lds)
		MESSAGE(FATAL_ERROR "Can't find `tm4c123gh6pm.lds` linker file in ${PROJ_PATH}.")
	endif(NOT EXISTS ${PROJ_PATH}/tm4c123gh6pm.lds)

	# check `startup.c` existance
	if( NOT EXISTS ${PROJ_PATH}/startup.c )
		MESSAGE(FATAL_ERROR "Can't find `startup.c` file in ${PROJ_PATH}.")
	endif( NOT EXISTS ${PROJ_PATH}/startup.c )
endif(${RFS_TESTING_${PROJECT_NAME}})

# check `tm4c123gh6pm.h` existance
if( NOT EXISTS ${PROJ_PATH}/tm4c123gh6pm.h )
	MESSAGE(FATAL_ERROR "Can't find `tm4c123gh6pm.h` file in ${PROJ_PATH}.")
endif( NOT EXISTS ${PROJ_PATH}/tm4c123gh6pm.h )
#################################################################

############################# Network ############################
# connect other parts to this part from RFS
if(${RFS_TESTING_${PROJECT_NAME}})
    
endif(${RFS_TESTING_${PROJECT_NAME}})
##################################################################

######################## My Sources files #######################
# `lib` path
SET( CMAKE_LIBRARY_PATH ${PROJ_PATH}/lib )
# `C` sources path
FILE(GLOB_RECURSE ${PROJECT_NAME}_CSources ${CMAKE_SOURCE_DIR}/*.c)
# `CPP` sources path
FILE(GLOB_RECURSE ${PROJECT_NAME}_CppSources ${CMAKE_SOURCE_DIR}/*.cpp)
# `ASM` sources path
FILE(GLOB_RECURSE ${PROJECT_NAME}_ASMSources ${CMAKE_SOURCE_DIR}/*.s ${CMAKE_SOURCE_DIR}/*.S)
# `h` headers path
FILE(GLOB_RECURSE ${PROJECT_NAME}_Headers ${CMAKE_SOURCE_DIR}/*.h ${CMAKE_LIBRARY_PATH}/*.h)
# other sources
SET ( Other_${PROJECT_NAME}_Sources ${PROJ_PATH}/tm4c123gh6pm.h )
# if in testing mode, add `main.c` and `startup.c`
if(${RFS_TESTING_${PROJECT_NAME}})
	SET( ${PROJECT_NAME}_main ${PROJ_PATH}/main.c )
	SET( Other_${PROJECT_NAME}_Sources ${Other_${PROJECT_NAME}_Sources} ${PROJ_PATH}/startup.c )
	FILE(GLOB_RECURSE ${PROJECT_NAME}_LIBS ${CMAKE_LIBRARY_PATH}/*.a )
endif(${RFS_TESTING_${PROJECT_NAME}})
SET (${PROJECT_NAME}_Sources ${${PROJECT_NAME}_CSources}
                             ${${PROJECT_NAME}_CppSources}
                             ${${PROJECT_NAME}_ASMSources}
                             ${${PROJECT_NAME}_Headers}
                             ${Other_${PROJECT_NAME}_Sources}
                             ${${PROJECT_NAME}_NetworkSources}
                             ${${PROJECT_NAME}_main})
#################################################################

################## Compiler and linker configs ##################
# cahce force -> used to force show in cmake-gui
#TODO: need to handle -MF -MT
set(FLAGS "-mcpu=cortex-m4 -march=armv7e-m -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 -ffunction-sections -fdata-sections -g -gdwarf-3 -gstrict-dwarf -Wall")
set(CMAKE_C_FLAGS "${FLAGS} -MD -std=c99 -MMD -MP" CACHE STRING "C flags" FORCE)
set(CMAKE_EXE_LINKER_FLAGS "-Wl,-Map,'${PROJECT_NAME}.map' -Wl,-T'${PROJ_PATH}/tm4c123gh6pm.lds' --specs=nosys.specs -Wl,--start-group -l'gcc' -l'nosys' -l'c' -Wl,--end-group" CACHE STRING "Linker flags" FORCE)
#################################################################

######################## Project configs ########################
project(${PROJECT_NAME} C CXX)
# treat `assembly` files as `C` files
set_property(SOURCE ${${PROJECT_NAME}_ASMSources} PROPERTY LANGUAGE C)

# include tm4c123gh6pm.h startup.c files
include_directories(${PROJ_PATH})

if(${RFS_TESTING_${PROJECT_NAME}})
	add_executable(${PROJECT_NAME}.elf ${${PROJECT_NAME}_Sources})
	# link the output with the libraries in `lib` folder
	target_link_libraries(${PROJECT_NAME}.elf ${${PROJECT_NAME}_LIBS})

	# add custom command for converting .elf to .hex
	# .hex file is the one which for uploading
	# .elf file is the one which used for debugging
	add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD COMMAND ${ARM_TOOLCHAIN_BIN_PATH}/arm-none-eabi-objcopy ARGS -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex)
	add_custom_target(upload lm4flash ${PROJECT_NAME}.hex DEPENDS ${PROJECT_NAME}.elf COMMENT "Uploading ...")
# else, this build without linking
else(${RFS_TESTING_${PROJECT_NAME}})
	add_library(${PROJECT_NAME} OBJECT ${${PROJECT_NAME}_Sources})
endif(${RFS_TESTING_${PROJECT_NAME}})
#################################################################

######################### CMake options #########################
# option for build and then upload
SET(RFS_BUILD_AND_UPLOAD False CACHE BOOL "Build then upload")
if(RFS_BUILD_AND_UPLOAD)
    message(STATUS "Uploading...")
    add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD COMMAND lm4flash ARGS ${PROJECT_NAME}.hex)
endif(RFS_BUILD_AND_UPLOAD)

# option for `testing parts of RFS`
SET(RFS_TESTING_${PROJECT_NAME} False CACHE BOOL "Turn on testing - include main.c")
if(${RFS_TESTING_${PROJECT_NAME}})
	message(STATUS "---------------------")
    message(STATUS "| Testing is ON ... |")
    message(STATUS "---------------------")
endif(${RFS_TESTING_${PROJECT_NAME}})
#################################################################

######################### clear config ##########################
# list of files that will be cleaned as a part of "make clean" stage
if(${RFS_TESTING_${PROJECT_NAME}})
    set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${PROJECT_NAME}.hex ${PROJECT_NAME}.map)

    file ( WRITE ${CMAKE_BINARY_DIR}/clean-cmake-files.cmake
        "# clean-cmake-files.cmake\n"
        "set(cmake_generated ${CMAKE_BINARY_DIR}/CMakeCache.txt\n"
        "                    ${CMAKE_BINARY_DIR}/cmake_install.cmake\n"
        "                    ${CMAKE_BINARY_DIR}/Makefile\n"
        "                    ${CMAKE_BINARY_DIR}/CMakeFiles\n"
        "                    ${CMAKE_BINARY_DIR}/clean-cmake-files.cmake\n"
        ")\n"
        "\n"
        "foreach(file \${cmake_generated})\n"
        "    if (EXISTS \${file})\n"
        "        file(REMOVE_RECURSE \${file})\n"
        "    endif()\n"
        "endforeach(file)\n" )

    add_custom_target(clean-cmake-files
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/clean-cmake-files.cmake
    )

    add_custom_target(clean-all
    COMMAND ${CMAKE_BUILD_TOOL} clean
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/clean-cmake-files.cmake
    )
endif(${RFS_TESTING_${PROJECT_NAME}})
#################################################################
