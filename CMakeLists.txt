CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

# check that we use a DOS shell
if("$ENV{TERM}" MATCHES "cygwin")
    MESSAGE(FATAL_ERROR "cmake must be executed in a VisualStudio command shell for the build to work properly (see the README for details).")
endif()

project (ilastik)

# check that cmake generator and compiler path in the shell are present and consistent
execute_process(COMMAND where cl.exe
                  OUTPUT_VARIABLE CL_PATH OUTPUT_STRIP_TRAILING_WHITESPACE)

if(NOT CL_PATH)
    message(FATAL_ERROR "Compiler is not in the PATH. cmake must be executed in a VisualStudio command shell for the build to work properly (see the README for details).")
endif() 

FILE(TO_CMAKE_PATH ${CL_PATH} CL_PATH)

STRING(REGEX REPLACE "/VC/bin/.*" "" VISUAL_STUDIO_PATH ${CL_PATH})
STRING(REGEX REPLACE "/VC/bin/.*" "" CMAKE_VISUAL_STUDIO_PATH ${CMAKE_C_COMPILER})
string(TOLOWER ${VISUAL_STUDIO_PATH} VSPATH_LOWER)
string(TOLOWER ${CMAKE_VISUAL_STUDIO_PATH} CMAKE_VSPATH_LOWER)

if(NOT "${VSPATH_LOWER}" STREQUAL "${CMAKE_VSPATH_LOWER}")
   MESSAGE(FATAL_ERROR "Mismatch between generator (-G) and shell:\n  generator uses '${CMAKE_VISUAL_STUDIO_PATH}'\n  shell uses '${VISUAL_STUDIO_PATH}'")
endif()

if("${CMAKE_C_COMPILER}" MATCHES "amd64")
    set(ILASTIK_DEFAULT_BITNESS "64")
else()
    set(ILASTIK_DEFAULT_BITNESS "32")
endif()

set(ILASTIK_BITNESS ${ILASTIK_DEFAULT_BITNESS} CACHE STRING "Address model: 32 or 64.")

if(${ILASTIK_BITNESS} STREQUAL "32")
    if("${CMAKE_C_COMPILER}" MATCHES "amd64")
        MESSAGE(FATAL_ERROR "ILASTIK_BITNESS specifies a 32-bit build, but the cmake generator uses a 64-bit compiler at\n  '${CMAKE_C_COMPILER}'")
    endif()

    if("${CL_PATH}" MATCHES "amd64")
        MESSAGE(FATAL_ERROR "ILASTIK_BITNESS specifies a 32-bit build, but the shell has a 64-bit compiler in the PATH at\n  '${CL_PATH}'")
    endif()
    
    set(ILASTIK_MACHINE "/MACHINE:X86")
    
elseif(${ILASTIK_BITNESS} STREQUAL "64")
    if(NOT "${CMAKE_C_COMPILER}" MATCHES "amd64")
        MESSAGE(FATAL_ERROR "ILASTIK_BITNESS specifies a 64-bit build, but the cmake generator uses a 32-bit compiler at\n  '${CMAKE_C_COMPILER}'")
    endif()

    if(NOT "${CL_PATH}" MATCHES "amd64")
        MESSAGE(FATAL_ERROR "ILASTIK_BITNESS specifies a 64-bit build, but the shell has a 32-bit compiler in the PATH at\n  '${CL_PATH}'")
    endif()
    
    set(ILASTIK_MACHINE "/MACHINE:X64")
    
else()
    MESSAGE(FATAL_ERROR "ILASTIK_BITNESS must be 32 or 64.")
endif()    

# find compiler version
STRING(REGEX MATCH "Studio [0-9]+" VISUAL_STUDIO_VERSION ${VISUAL_STUDIO_PATH})
if (EXISTS ${VISUAL_STUDIO_VERSION})
    STRING(REGEX REPLACE "Studio " "" VISUAL_STUDIO_VERSION ${VISUAL_STUDIO_VERSION})
else()
    STRING(REGEX MATCH "[0-9]+" VISUAL_STUDIO_VERSION "$ENV{VisualStudioVersion}")
endif()
MESSAGE(STATUS "Using Visual Studio ${VISUAL_STUDIO_VERSION} at '${VISUAL_STUDIO_PATH}'")

include (ExternalProject)

find_package(git REQUIRED)

get_filename_component(GIT_PATH ${GIT_EXECUTABLE} PATH)
get_filename_component(PATCH_EXE ${GIT_PATH}/../bin/patch.exe ABSOLUTE)
    
if(NOT EXISTS ${PATCH_EXE})
    message(FATAL_ERROR "patch.exe NOT found, should be at \n${PATCH_EXE}")
endif()

find_package(perl REQUIRED)
GET_FILENAME_COMPONENT(PERL_PATH ${PERL_EXECUTABLE} ABSOLUTE)
file(TO_NATIVE_PATH ${PERL_PATH} PERL_PATH)

set (ILASTIK_DEPENDENCY_DIR "None" CACHE PATH "Install prefix for ilastik and its dependencies.")
if (${ILASTIK_DEPENDENCY_DIR} STREQUAL "None")
    message (FATAL_ERROR "ERROR: ILASTIK_DEPENDENCY_DIR build directory (for all downloads & builds) should be specified via -DILASTIK_DEPENDENCY_DIR=<path> on the cmake command line.")
endif ()
get_filename_component(ILASTIK_DEPENDENCY_DIR ${ILASTIK_DEPENDENCY_DIR} ABSOLUTE)
file(TO_NATIVE_PATH ${ILASTIK_DEPENDENCY_DIR} ILASTIK_DEPENDENCY_DIR_DOS)

OPTION(WITH_OPENSSL "Build Python with OpenSSL support ?" OFF)

OPTION(WITH_SCIPY "Build scipy (and its dependencies) ?" OFF)

if(WITH_SCIPY)
    set (MSYS_PATH "-NOTFOUND" CACHE PATH "Path to MSYS binaries (must contain make.exe).")
    set (MINGW_PATH "-NOTFOUND" CACHE PATH "Path to MinGW binaries (must contain gfortran.exe).")
    
    if (${MSYS_PATH} STREQUAL "-NOTFOUND" OR ${MINGW_PATH} STREQUAL "-NOTFOUND")
        message (FATAL_ERROR "ERROR: To build scipy, MSYS_PATH and MINGW_PATH must be specified via -DMSYS_PATH=<path> -DMINGW_PATH=<path> on the cmake command line.")
    endif ()

    get_filename_component(MSYS_PATH ${MSYS_PATH} ABSOLUTE)
    file(TO_NATIVE_PATH ${MSYS_PATH} MSYS_PATH)
    get_filename_component(MINGW_PATH ${MINGW_PATH} ABSOLUTE)
    file(TO_NATIVE_PATH ${MINGW_PATH} MINGW_PATH)
endif()

OPTION(WITH_VTK "Build VTK ?" OFF)
OPTION(WITH_PGMLINK "pgmLink requires an installed cplex to build successfully" OFF)

message ("ilastik downloads and builds will be placed here: ${ILASTIK_DEPENDENCY_DIR}")

FILE(MAKE_DIRECTORY ${ILASTIK_DEPENDENCY_DIR}/bin 
                    ${ILASTIK_DEPENDENCY_DIR}/lib 
                    ${ILASTIK_DEPENDENCY_DIR}/include
                    ${ILASTIK_DEPENDENCY_DIR}/tmp )

file(TO_NATIVE_PATH ${ILASTIK_DEPENDENCY_DIR}     ILASTIK_INSTALL_PATH)
file(TO_NATIVE_PATH ${ILASTIK_DEPENDENCY_DIR}/bin ILASTIK_BIN_PATH)

set(ADD_PATH "${ILASTIK_DEPENDENCY_DIR}/tmp/add_path.bat")
configure_file(add_path.bat ${ADD_PATH} COPYONLY)

############################################################################

SET(CMAKE_MODULE_PATH  ${CMAKE_MODULE_PATH}  ${ilastik_SOURCE_DIR})

include(openssl)
include(python)
include(python_packages)
include(numpy)
include(zlib)
include(jpeg)
include(tiff)
include(libpng)
include(fftw)
include(hdf5)
include(h5py)
include(boost)
include(vigra)
include(pil)
include(qt)
include(pyqt)
include(qimage2ndarray)
include(vigraqt)
include(freetype)
include(matplotlib)
include(lemon)
include(cylemon)
include(armadillo)
include(mlpack)
# include(spyder)

if(WITH_SCIPY)
    include(scipy)
    include(scikit_learn)
    include(scikit_image)
endif()
if(WITH_VTK)
    include(vtk)
endif()
if(WITH_PGMLINK) 
    include(ann)
    include(opengm)
    include(pgmlink)
endif()

#############################################################################

set(DEFAULT_ILASTIK_GIT_COMMIT "origin/release")
IF(NOT DEFINED ILASTIK_GIT_COMMIT)
    SET(ILASTIK_GIT_COMMIT "${DEFAULT_ILASTIK_GIT_COMMIT}")
ENDIF()
SET(ILASTIK_GIT_COMMIT ${ILASTIK_GIT_COMMIT}
    CACHE STRING "Specify ilastik branch/tag/commit to be used (default: ${DEFAULT_ILASTIK_GIT_COMMIT})"
    FORCE)

if (NOT ilastik_NAME)

set(ilastik_NAME ilastik-HEAD)

message ("Installing ${ilastik_NAME} into ilastik build area: ${ILASTIK_DEPENDENCY_DIR}/ilastik ...")

set(ILASTIK_META_DIR "${ILASTIK_DEPENDENCY_DIR_DOS}\\ilastik")

if("${ILASTIK_GIT_COMMIT}" STREQUAL "master")
    ExternalProject_Add(${ilastik_NAME}
        DEPENDS             ${python_NAME}
        PREFIX              ${ILASTIK_DEPENDENCY_DIR}
        DOWNLOAD_COMMAND    ${GIT_EXECUTABLE} clone https://github.com/ilastik/ilastik-meta ${ILASTIK_DEPENDENCY_DIR}/ilastik
                         \n cd ${ILASTIK_META_DIR}
                         \n ${GIT_EXECUTABLE} checkout ${ILASTIK_GIT_COMMIT} 
                         \n ${GIT_EXECUTABLE} submodule update --init --recursive
        DOWNLOAD_DIR        ""
        SOURCE_DIR          ""
        UPDATE_COMMAND      cd ${ILASTIK_META_DIR}\\lazyflow 
                         \n ${GIT_EXECUTABLE} checkout master 
                         \n ${GIT_EXECUTABLE} pull origin master 
                         \n cd ${ILASTIK_META_DIR}\\volumina 
                         \n ${GIT_EXECUTABLE} checkout master 
                         \n ${GIT_EXECUTABLE} pull origin master 
                         \n cd ${ILASTIK_META_DIR}\\ilastik 
                         \n ${GIT_EXECUTABLE} checkout master 
                         \n ${GIT_EXECUTABLE} pull origin master 
        CONFIGURE_COMMAND   ""
        BUILD_COMMAND       ""
        INSTALL_COMMAND     ""
        BUILD_IN_SOURCE     1
    )
else()
    ExternalProject_Add(${ilastik_NAME}
        DEPENDS             ${python_NAME}
        PREFIX              ${ILASTIK_DEPENDENCY_DIR}
        DOWNLOAD_COMMAND    ${GIT_EXECUTABLE} clone https://github.com/ilastik/ilastik-meta ${ILASTIK_DEPENDENCY_DIR}/ilastik
                         \n cd ${ILASTIK_META_DIR}
                         \n ${GIT_EXECUTABLE} checkout ${ILASTIK_GIT_COMMIT} 
                         \n ${GIT_EXECUTABLE} submodule update --init --recursive
        DOWNLOAD_DIR        ""
        SOURCE_DIR          ""
        UPDATE_COMMAND      cd ${ILASTIK_META_DIR}
                         \n ${GIT_EXECUTABLE} fetch 
                         \n ${GIT_EXECUTABLE} checkout ${ILASTIK_GIT_COMMIT} 
                         \n ${GIT_EXECUTABLE} submodule update --init --recursive
        CONFIGURE_COMMAND   ""
        BUILD_COMMAND       ""
        INSTALL_COMMAND     ""
        BUILD_IN_SOURCE     1
    )
endif()

set_target_properties(${ilastik_NAME} PROPERTIES EXCLUDE_FROM_ALL ON)
set (APP_DEPENDENCIES ${APP_DEPENDENCIES} ${ilastik_NAME})

# a file that contains something like "__version_info__ = (1, 0, 4)"
# (needed in CPackUpdateVersion.cmake below)
set(ILASTIK_VERSION_FILE "${ILASTIK_DEPENDENCY_DIR}/ilastik/ilastik/ilastik/__init__.py")

endif (NOT ilastik_NAME)

#############################################################################

if (NOT lazyflow_NAME)

set(lazyflow_NAME lazyflow-HEAD-drtile)

message ("Installing ${lazyflow_NAME} into ilastik build area: ${ILASTIK_DEPENDENCY_DIR}/ilastik ...")
ExternalProject_Add(${lazyflow_NAME}
    DEPENDS             ${python_NAME} ${vigra_NAME} ${ilastik_NAME}
    PREFIX              ${ILASTIK_DEPENDENCY_DIR}
    DOWNLOAD_COMMAND    ""
    DOWNLOAD_DIR        ""
    SOURCE_DIR          ""
    GIT_REPOSITORY      ""
    UPDATE_COMMAND      ""
    BINARY_DIR          ${ILASTIK_DEPENDENCY_DIR}/src/drtile-build
    # Read the config information from ${ILASTIK_DEPENDENCY_DIR}/lib/vigranumpy/VigranumpyConfig.cmake
    CONFIGURE_COMMAND   ${CMAKE_COMMAND} ${ILASTIK_DEPENDENCY_DIR}/ilastik/lazyflow/lazyflow/drtile
        -G ${CMAKE_GENERATOR}
        -DVigranumpy_DIR=${ILASTIK_DEPENDENCY_DIR}/lib/vigranumpy
    BUILD_COMMAND       devenv drtile.sln /build "Release" /project drtile
    INSTALL_COMMAND     ${CMAKE_COMMAND} -E copy Release/drtile.pyd ${ILASTIK_DEPENDENCY_DIR}/ilastik/lazyflow/lazyflow/drtile
)

set_target_properties(${lazyflow_NAME} PROPERTIES EXCLUDE_FROM_ALL ON)
set (APP_DEPENDENCIES ${APP_DEPENDENCIES} ${lazyflow_NAME})

endif (NOT lazyflow_NAME)

############################################################################

if(${ILASTIK_BITNESS} STREQUAL "64" AND NOT counting_NAME)

set (counting_NAME ${ilastik_NAME}-counting)

ExternalProject_Add(${counting_NAME}
    DEPENDS             ${ilastik_NAME}
    PREFIX              ${ILASTIK_DEPENDENCY_DIR}
    SOURCE_DIR          ""
    DOWNLOAD_COMMAND    ""
    UPDATE_COMMAND      ""
    PATCH_COMMAND       ""
    BINARY_DIR ${ILASTIK_DEPENDENCY_DIR}/src/cwrapper-build
    CONFIGURE_COMMAND   ${CMAKE_COMMAND} ${ILASTIK_DEPENDENCY_DIR}/ilastik/ilastik/ilastik/applets/counting/cwrapper
    -G ${CMAKE_GENERATOR} -DWITH_CPLEX=1 -DWITH_GUROBI=1
    BUILD_COMMAND       devenv cwrapper.sln /build "Release" /project ALL_BUILD
    TEST_COMMAND        ""
    INSTALL_COMMAND     ${CMAKE_COMMAND} -E copy gurobi/Release/gurobiwrapper.dll ${ILASTIK_DEPENDENCY_DIR}/ilastik/ilastik/ilastik/applets/counting/cwrapper/gurobi
                     \n ${CMAKE_COMMAND} -E copy cplex/Release/cplexwrapper.dll ${ILASTIK_DEPENDENCY_DIR}/ilastik/ilastik/ilastik/applets/counting/cwrapper/cplex
    )

set_target_properties(${counting_NAME} PROPERTIES EXCLUDE_FROM_ALL ON)
set (APP_DEPENDENCIES ${APP_DEPENDENCIES} ${counting_NAME})

endif () 

############################################################################

configure_file(ilastik.bat.in ${ILASTIK_DEPENDENCY_DIR}/ilastik.bat)
configure_file(LICENSE.txt ${ILASTIK_DEPENDENCY_DIR}/LICENSE.txt COPYONLY)
configure_file(ilastik-icon.ico ${ILASTIK_DEPENDENCY_DIR}/ilastik-icon.ico COPYONLY)

file(TO_NATIVE_PATH " ${ILASTIK_DEPENDENCY_DIR}/bin ${QT_BUILD_DIR}/bin ${PYTHON_PREFIX} ${PYTHON_PREFIX}/Scripts" ILASTIK_PATH_SETTINGS)

# APP_DEPENDENCIES is auto-generated by the above includes and holds all included targets.
ADD_CUSTOM_TARGET(ilastik DEPENDS ${APP_DEPENDENCIES}
                  COMMENT "Installed ilastik dependencies. Add directories to your PATH variable: ${ILASTIK_PATH_SETTINGS}")

install(CODE "execute_process(COMMAND ${PYTHON_EXE} ${PROJECT_SOURCE_DIR}/compile_all.py ${ILASTIK_DEPENDENCY_DIR}/ilastik)")
INSTALL(FILES ${ILASTIK_DEPENDENCY_DIR}/LICENSE.txt DESTINATION .)
INSTALL(FILES ${ILASTIK_DEPENDENCY_DIR}/ilastik-icon.ico DESTINATION .)
INSTALL(FILES ${ILASTIK_DEPENDENCY_DIR}/ilastik.bat DESTINATION .)
# INSTALL(FILES ${ILASTIK_DEPENDENCY_DIR}/spyder.bat DESTINATION .)
INSTALL(DIRECTORY ${ILASTIK_DEPENDENCY_DIR}/ilastik/ DESTINATION ilastik)
INSTALL(DIRECTORY ${ILASTIK_DEPENDENCY_DIR}/bin/ DESTINATION bin FILES_MATCHING PATTERN "*.dll")
INSTALL(DIRECTORY ${ILASTIK_DEPENDENCY_DIR}/python/ DESTINATION python PATTERN "*ui-bg_highlight-soft_75_cccccc_1x100.png" EXCLUDE)
INSTALL(DIRECTORY ${ILASTIK_DEPENDENCY_DIR}/Qt4/bin/ DESTINATION Qt4/bin FILES_MATCHING PATTERN "*.dll")
INSTALL(DIRECTORY ${ILASTIK_DEPENDENCY_DIR}/Qt4/plugins/ DESTINATION Qt4/plugins FILES_MATCHING PATTERN "*.dll")

# OpenMP runtimes are needed for cylemon
file(TO_CMAKE_PATH "$ENV{VCINSTALLDIR}" MSVC_ROOT)
string(REGEX REPLACE "\\." "" MSVC_VERSION "$ENV{VISUALSTUDIOVERSION}")

if(${ILASTIK_BITNESS} STREQUAL "32")
    INSTALL(DIRECTORY ${MSVC_ROOT}/redist/x86/Microsoft.VC${MSVC_VERSION}.OpenMP/ DESTINATION bin FILES_MATCHING PATTERN "vcomp*.dll")
else()
    INSTALL(DIRECTORY ${MSVC_ROOT}/redist/x64/Microsoft.VC${MSVC_VERSION}.OpenMP/ DESTINATION bin FILES_MATCHING PATTERN "vcomp*.dll")
endif()

SET(CPACK_PACKAGE_VENDOR "The ilastik Team")
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY
    "Interactive image labeling and segmentation toolkit."
)

# package version setup - true version number will be inserted by
# CPack using CPackUpdateVersion.cmake 
set(CPACK_PROJECT_CONFIG_FILE ${PROJECT_BINARY_DIR}/CPackUpdateVersion.cmake)
CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/CPackUpdateVersion.cmake.in 
               ${PROJECT_BINARY_DIR}/CPackUpdateVersion.cmake @ONLY)

SET(CPACK_PACKAGE_VERSION_MAJOR "0")  # will be set at runtime
SET(CPACK_PACKAGE_VERSION_MINOR "0")  # will be set at runtime
SET(CPACK_PACKAGE_VERSION_PATCH "0")  # will be set at runtime

SET(CPACK_PACKAGE_INSTALL_DIRECTORY  "${PROJECT_NAME}-0.0.0")

if(${ILASTIK_BITNESS} STREQUAL "64")
    SET(CPACK_NSIS_INSTALL_ROOT "$PROGRAMFILES64")
endif()

# cannot use SET(CPACK_PACKAGE_EXECUTABLES ...) for *.bat files because it appends .exe
SET(CPACK_NSIS_EXTRA_INSTALL_COMMANDS "
  CreateShortCut \\\"$SMPROGRAMS\\\\$STARTMENU_FOLDER\\\\ilastik.lnk\\\" \\\"$INSTDIR\\\\ilastik.bat\\\" \\\"\\\" \\\"$INSTDIR\\\\ilastik-icon.ico\\\"
")
SET(CPACK_NSIS_EXTRA_UNINSTALL_COMMANDS "
  !insertmacro MUI_STARTMENU_GETFOLDER Application $STARTMENU_FOLDER
  Delete \\\"$SMPROGRAMS\\\\$STARTMENU_FOLDER\\\\ilastik.lnk\\\"
")

# Cannot use SET(CPACK_PACKAGE_ICON  ...) because NSIS needs a Windows path here.
# The icon must be a 150x57 BMP image created with Paint (gimp's BMP doesn't work).
string(REGEX REPLACE "/" "\\\\\\\\" CPACK_PACKAGE_ICON "${PROJECT_SOURCE_DIR}/ilastik-installer.bmp")

# SET(CPACK_PACKAGE_INSTALL_REGISTRY_KEY  "${PROJECT_NAME}")
SET(CPACK_RESOURCE_FILE_LICENSE         "${PROJECT_SOURCE_DIR}/LICENSE.txt")
# SET(CPACK_RESOURCE_FILE_README          "${PROJECT_SOURCE_DIR}/README.txt")
SET(CPACK_STRIP_FILES TRUE)
SET(CPACK_PACKAGE_CONTACT "ilastik Team <team@ilastik.org>")

INCLUDE (CPack)

MESSAGE(STATUS "Using Visual Studio ${VISUAL_STUDIO_VERSION} at '${CMAKE_C_COMPILER}' for ${ILASTIK_BITNESS}-bit build.")

