cmake_minimum_required(VERSION 2.6)
project(toxvpn)

set(BOOTSTRAP_PATH "${CMAKE_INSTALL_PREFIX}/share/toxvpn/bootstrap.json")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBOOTSTRAP_FILE=\\\"${BOOTSTRAP_PATH}\\\"")

if(WIN32)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DWIN32")
else(WIN32)
endif(WIN32)

if(STATIC)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSTATIC")
set(LIBMODE STATIC)
set(PTHERAD_LIBS "-static-libgcc -static-libstdc++ -Wl,-Bstatic,--whole-archive -lpthread -Wl,-Bdynamic,--no-whole-archive")
else()
set(PTHERAD_LIBS "-lpthread")
endif(STATIC)

if(ANDROID)
#set(PTHERAD_LIBS "-Wl,-Bstatic,--whole-archive -pthread")
set(PTHERAD_LIBS "")
add_definitions(-D__ANDROID__)
endif(ANDROID)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wconversion -Wshadow -Wextra -std=c++14")

if(SYSTEMD)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSYSTEMD")
set(SYSTEMD_LIBRARIES "systemd")
endif(SYSTEMD)

if(ZMQ)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DZMQ")
set(ZMQ_LIBS "zmq")
endif(ZMQ)

find_package(nlohmann_json)

if(STATIC OR ANDROID)
    find_library(TOXCORE_LIBRARIES libtoxcore.a REQUIRED HINTS "${TOX_PREFIX}/lib")
#    find_library(TOXENC_LIBRARIES libtoxencryptsave.a  HINTS "${TOX_PREFIX}/lib")
else()
    find_library(TOXCORE_LIBRARIES toxcore REQUIRED HINTS "${TOX_PREFIX}/lib")
endif()
find_path(TOXCORE_INCLUDE_DIRS tox/tox.h REQUIRED HINTS "${TOX_PREFIX}/include")

if(STATIC OR ANDROID)
    find_library(SODIUM_LIBRARIES libsodium.a REQUIRED "${SODIUM_PREFIX}/lib")
else()
    find_library(SODIUM_LIBRARIES sodium REQUIRED "${SODIUM_PREFIX}/lib")
endif()
find_path(SODIUM_INCLUDE_DIRS sodium.h REQUIRED "${SODIUM_PREFIX}/include")


if(ANDROID)
    set(GCC_LIBS "-static-libgcc -pie -lc -ldl")
else()
endif()


if(ANDROID)
else()
    find_library(CAP_LIBS cap)
endif()

if(WIN32)
  set(extra_files src/interface_windows.cpp src/route_windows.cpp)
elseif(CYGWIN)
  set(extra_files src/interface_windows.cpp src/route_windows.cpp)
elseif(APPLE)
  set(extra_files src/interface_mac.cpp src/route_mac.cpp)
else()
  set(extra_files src/interface_linux.cpp src/route_linux.cpp)
endif()

#string(STRIP ${CMAKE_EXE_LINKER_FLAGS_INIT} CMAKE_EXE_LINKER_FLAGS_INIT)

add_executable(toxvpn src/main.cpp src/control.cpp src/interface.cpp src/listener.cpp ${extra_files})
target_link_libraries(toxvpn ${GCC_LIBS} ${JSONCPP_LIBRARIES} ${TOXCORE_LIBRARIES} ${TOXENC_LIBRARIES} ${PTHERAD_LIBS} ${SODIUM_LIBRARIES})
include_directories(${JSONCPP_INCLUDE_DIRS} ${TOXCORE_INCLUDE_DIRS})

add_executable(toxvpn-remote src/toxvpn-remote.cpp)
target_link_libraries(toxvpn-remote ${ZMQ_LIBS})

if(WIN32)
  target_link_libraries(toxvpn ${TOXCORE_LIBRARIES} ws2_32 ${SODIUM_LIBRARIES} pthread iphlpapi ${JSONCPP_LIBRARIES})
elseif(CYGWIN)
  target_link_libraries(toxvpn ${TOXCORE_LIBRARIES} ws2_32 ${SODIUM_LIBRARIES} pthread iphlpapi)
elseif(APPLE)
  target_link_libraries(toxvpn ${ZMQ_LIBS})
else()
  if(STATIC)
    target_link_libraries(toxvpn ${CAP_LIBS} ${ZMQ_LIBS} pthread ${SODIUM_LIBRARIES})
  else(STATIC)
    target_link_libraries(toxvpn ${CAP_LIBS} ${ZMQ_LIBS} ${SYSTEMD_LIBRARIES})
  endif(STATIC)
endif()

install(TARGETS toxvpn toxvpn-remote DESTINATION bin)
install(FILES res/bootstrap.json DESTINATION share/toxvpn)
