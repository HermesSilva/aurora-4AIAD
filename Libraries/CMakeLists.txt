add_subdirectory(Foundation)
add_subdirectory(Aurora)
add_subdirectory(HdAurora)
# 4AIAD: the image processing resolver is a usdview helper and requires OpenImageIO.
option(ENABLE_IMAGE_PROCESSING_RESOLVER "Build the ImageProcessingResolver USD plugin (requires OpenImageIO)." ON)
if(ENABLE_IMAGE_PROCESSING_RESOLVER)
    add_subdirectory(ImageProcessingResolver)
endif()
