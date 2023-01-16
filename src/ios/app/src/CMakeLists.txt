add_library(citra-ios SHARED
    emu_window/emu_window.cpp
    emu_window/emu_window.h
    emu_window/emu_window_gl.cpp
    emu_window/emu_window_gl.h
    emu_window/emu_window_vk.cpp
    emu_window/emu_window_vk.h
)

target_link_libraries(citra-ios PRIVATE audio_core common core input_common network)
target_link_libraries(citra-ios PRIVATE android camera2ndk EGL glad inih jnigraphics lodepng log mediandk yuv)

set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} citra-android)
