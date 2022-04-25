
file(GLOB videoViewerSources *.cpp)
file(GLOB videoViewerIncludes *.h )

add_executable(video-viewer ${videoViewerSources})
target_link_libraries(video-viewer image-utils)

install(TARGETS video-viewer DESTINATION bin)
