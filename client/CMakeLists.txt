## LinuxDeX Client - Core library
add_library(linuxdex_core STATIC
    src/transport/adb_message.cpp
    src/transport/adb_auth.cpp
    src/transport/adb_client.cpp
    src/transport/usb_device.cpp
    src/transport/connection.cpp
    src/protocol/message.cpp
    src/protocol/channel.cpp
    src/video/decoder.cpp
    src/audio/decoder.cpp
    src/input/keymap.cpp
    src/input/keyboard.cpp
    src/input/mouse.cpp
    src/features/clipboard.cpp
    src/features/file_transfer.cpp
    src/features/notifications.cpp
)

target_include_directories(linuxdex_core PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${LIBUSB_INCLUDE_DIRS}
    ${AVCODEC_INCLUDE_DIRS}
    ${AVUTIL_INCLUDE_DIRS}
    ${SWSCALE_INCLUDE_DIRS}
    ${SDL2_INCLUDE_DIRS}
    ${PIPEWIRE_INCLUDE_DIRS}
)

target_link_libraries(linuxdex_core PUBLIC
    ${LIBUSB_LIBRARIES}
    ${AVCODEC_LIBRARIES}
    ${AVFORMAT_LIBRARIES}
    ${AVUTIL_LIBRARIES}
    ${SWSCALE_LIBRARIES}
    ${SDL2_LIBRARIES}
    ${PIPEWIRE_LIBRARIES}
    OpenSSL::Crypto
)

## Main executable
add_executable(linuxdex src/main.cpp)
target_link_libraries(linuxdex PRIVATE
    linuxdex_core
    ${GTK4_LIBRARIES}
    ${ADWAITA_LIBRARIES}
)
target_include_directories(linuxdex PRIVATE
    ${GTK4_INCLUDE_DIRS}
    ${ADWAITA_INCLUDE_DIRS}
)
