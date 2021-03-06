include_directories("${GenTC_SOURCE_DIR}/codec")
include_directories("${GenTC_BINARY_DIR}/codec")

include_directories("${GLFW_SOURCE_DIR}/include")
include_directories("${GenTC_SOURCE_DIR}/gpu")
include_directories("${GenTC_SOURCE_DIR}/ans")
include_directories("${GenTC_BINARY_DIR}/ans")
include_directories("${GenTC_SOURCE_DIR}/lib/include")
include_directories("${GenTC_SOURCE_DIR}/lib/vptree/include")
include_directories("${GenTC_SOURCE_DIR}/lib")
INCLUDE_DIRECTORIES( ${OPENCL_INCLUDE_DIRS} )

SET( INVERSE_WAVELET_KERNEL_PATH ${GenTC_SOURCE_DIR}/codec/inverse_wavelet.cl )
SET( ASSEMBLE_KERNEL_PATH ${GenTC_SOURCE_DIR}/codec/assemble.cl )
SET( DECODE_INDICES_KERNEL_PATH ${GenTC_SOURCE_DIR}/codec/decode_indices.cl )

CONFIGURE_FILE(
  "decoder_config.h.in"
  "decoder_config.h"
)

SET( KERNELS
  ${INVERSE_WAVELET_KERNEL_PATH}
  ${ASSEMBLE_KERNEL_PATH}
  ${DECODE_INDICES_KERNEL_PATH}
)

SET( HEADERS
  "codec_base.h"
  "dxt_image.h"
  "image.h"
  "pixel_traits.h"
)

SET( SOURCES
  "codec_base.cpp"
  "dxt_image.cpp"
  "image.cpp"
)

ADD_LIBRARY(gentc_codec_base ${HEADERS} ${SOURCES})
TARGET_LINK_LIBRARIES( gentc_codec_base vptree)

SET( HEADERS
  "encoder.h"
  "data_stream.h"
  "entropy.h"
  "fast_dct.h"
  "image_utils.h"
  "image_processing.h"
  "pipeline.h"
  "wavelet.h"
)  

SET( SOURCES
  "encoder.cpp"
  "data_stream.cpp"
  "entropy.cpp"
  "image_processing.cpp"
  "image_utils.cpp"
  "wavelet.cpp"
)

ADD_LIBRARY(gentc_encoder ${HEADERS} ${SOURCES})
TARGET_LINK_LIBRARIES( gentc_encoder ans)
TARGET_LINK_LIBRARIES( gentc_encoder gentc_codec_base)

SET( HEADERS
  "decoder.h"
  "decoder_config.h.in"
)

SET( SOURCES
  "decoder.cpp"
)

SOURCE_GROUP(OpenCL FILES ${KERNELS})
ADD_LIBRARY(gentc_decoder ${HEADERS} ${SOURCES} ${KERNELS})
TARGET_LINK_LIBRARIES( gentc_decoder ans_ocl)
TARGET_LINK_LIBRARIES( gentc_decoder gentc_gpu )
TARGET_LINK_LIBRARIES( gentc_decoder ${OPENCL_LIBRARIES} )
TARGET_LINK_LIBRARIES( gentc_decoder gentc_codec_base)

ADD_EXECUTABLE(gentenc command_line.cpp)
TARGET_LINK_LIBRARIES( gentenc gentc_encoder )

################################################################################
##
## Test
##
################################################################################

INCLUDE_DIRECTORIES(${GenTC_SOURCE_DIR}/googletest/include)

# Summary of tuple support for Microsoft Visual Studio:
# Compiler    version(MS)  version(cmake)  Support
# ----------  -----------  --------------  -----------------------------
# <= VS 2010  <= 10        <= 1600         Use Google Tests's own tuple.
# VS 2012     11           1700            std::tr1::tuple + _VARIADIC_MAX=10
# VS 2013     12           1800            std::tr1::tuple
if (MSVC AND MSVC_VERSION EQUAL 1700)
  add_definitions(/D _VARIADIC_MAX=10)
endif()

if (MSVC AND MSVC_VERSION EQUAL 1900)
  add_definitions(/D _SCL_SECURE_NO_WARNINGS)
endif()

SET(CODEC_TEST_DIR "${GenTC_SOURCE_DIR}/codec/test")
CONFIGURE_FILE(
  "test/test_config.h.in"
  "test/test_config.h"
)
include_directories("${GenTC_SOURCE_DIR}/codec")
INCLUDE_DIRECTORIES(${GenTC_BINARY_DIR}/codec/test)

FOREACH(TEST image wavelet codec)
  ADD_EXECUTABLE(${TEST}_test "test/${TEST}_test.cpp")

  TARGET_LINK_LIBRARIES(${TEST}_test gentc_encoder)
  TARGET_LINK_LIBRARIES(${TEST}_test gentc_decoder)
  IF ("${TEST}" STREQUAL "codec")
    TARGET_LINK_LIBRARIES(${TEST}_test gentc_gpu)
  ENDIF()
  TARGET_LINK_LIBRARIES(${TEST}_test gtest_main)

  ADD_TEST(Test_${TEST} ${TEST}_test)
ENDFOREACH()