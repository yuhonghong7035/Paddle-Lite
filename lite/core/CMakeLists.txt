if (WITH_TESTING)
    lite_cc_library(lite_gtest_main SRCS lite_gtest_main.cc DEPS gtest gflags)
endif()
lite_cc_library(target_wrapper SRCS target_wrapper.cc
  DEPS target_wrapper_host place
  X86_DEPS target_wrapper_x86
  CUDA_DEPS target_wrapper_cuda
  CL_DEPS cl_target_wrapper
  FPGA_DEPS fpga_target_wrapper
  BM_DEPS target_wrapper_bm)

lite_cc_library(memory SRCS memory.cc DEPS target_wrapper CL_DEPS cl_target_wrapper)

set(tensor_extra_deps "")
if (LITE_WITH_FPGA)
    set(tensor_extra_deps lite_tensor_fpga)
endif()
lite_cc_library(tensor SRCS tensor.cc DEPS memory ${tensor_extra_deps})


if (NOT LITE_ON_TINY_PUBLISH)
    proto_library(framework_proto SRCS framework.proto)
endif()

if (LITE_WITH_X86)
lite_cc_library(variable SRCS variable.cc DEPS tensor)
lite_cc_library(types SRCS types.cc)
else()
lite_cc_library(variable SRCS variable.cc DEPS tensor)
lite_cc_library(types SRCS types.cc)
endif()
lite_cc_library(op_registry SRCS op_registry.cc DEPS kernel)
lite_cc_library(scope SRCS scope.cc DEPS tensor)
lite_cc_library(device_info SRCS device_info.cc DEPS tensor)

if (LITE_WITH_ARM)
lite_cc_library(context SRCS context.cc DEPS tensor any device_info CL_DEPS cl_context gflags)
else()
lite_cc_library(context SRCS context.cc DEPS tensor any device_info eigen3 CL_DEPS cl_context gflags)
endif()

#-------------------------------------------- GET CODE META INFO ------------------------------------------
execute_process(
  COMMAND git describe --tags --exact-match
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE PADDLE_LITE_TAG
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

execute_process(
  COMMAND git rev-parse --abbrev-ref HEAD
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE PADDLE_LITE_BRANCH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

execute_process(
  COMMAND git log -1 --format=%h
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE PADDLE_LITE_COMMIT
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

message(STATUS "tag: ${PADDLE_LITE_TAG}")
message(STATUS "branch: ${PADDLE_LITE_BRANCH}")
message(STATUS "commit: ${PADDLE_LITE_COMMIT}")

configure_file(version.h.in version.h)
#----------------------------------------------- NOT CHANGE -----------------------------------------------
# A trick to generate the paddle_use_kernels.h
add_custom_command(
  COMMAND python ${CMAKE_SOURCE_DIR}/lite/tools/cmake_tools/parse_kernel_registry.py
  ${kernels_src_list}
  ${CMAKE_SOURCE_DIR}/lite/api/paddle_use_kernels.h
  "${LITE_OPTMODEL_DIR}/.tailored_kernels_list"
  LITE_BUILD_TAILOR
  OUTPUT kernels.h # not a real path to the output to force it execute every time.
  )
# A trick to generate the paddle_use_ops.h
add_custom_command(
  COMMAND python ${CMAKE_SOURCE_DIR}/lite/tools/cmake_tools/parse_op_registry.py
  ${ops_src_list}
  ${CMAKE_SOURCE_DIR}/lite/api/paddle_use_ops.h
  "${LITE_OPTMODEL_DIR}/.tailored_ops_list"
  LITE_BUILD_TAILOR
  OUTPUT ops.h # not a real path to the output to force it execute every time.
  )
# generate fake kernels for memory_optimize_tool
add_custom_command(
  COMMAND python ${CMAKE_SOURCE_DIR}/lite/tools/cmake_tools/create_fake_kernel_registry.py
  ${kernels_src_list}
  ${CMAKE_BINARY_DIR}/all_kernel_faked.cc
  ${CMAKE_BINARY_DIR}/kernel_src_map.h
  OUTPUT all_kernel_faked.cc # not a real path to the output to force it execute every time.
  )
add_custom_target(op_list_h DEPENDS ops.h)
add_custom_target(kernel_list_h DEPENDS kernels.h)
add_custom_target(all_kernel_faked_cc DEPENDS all_kernel_faked.cc)
# create headfile to restore ops info sorted by suppported platforms
add_custom_command(
  COMMAND python ${CMAKE_SOURCE_DIR}/lite/tools/cmake_tools/record_supported_kernel_op.py
  ${kernels_src_list}
  ${ops_src_list}
  ${CMAKE_BINARY_DIR}/supported_kernel_op_info.h
  OUTPUT supported_kernel_op_info.h # not a real path to the output to force it execute every time.
  )
  add_custom_target(supported_kernel_op_info_h DEPENDS supported_kernel_op_info.h)
#----------------------------------------------- NOT CHANGE -----------------------------------------------
lite_cc_library(kernel SRCS kernel.cc
        DEPS context type_system target_wrapper any op_params tensor
        PROFILE_DEPS lite_profiler
  )
lite_cc_library(op SRCS op_lite.cc DEPS scope op_registry target_wrapper kernel
  cpp_op_desc tensor
  )

add_dependencies(kernel kernel_list_h)
add_dependencies(op op_list_h)


lite_cc_library(type_system SRCS type_system.cc DEPS tensor target_wrapper)

lite_cc_library(program SRCS program.cc
    DEPS op kernel model_parser ${ops} ${cpp_wrapper}
    PROFILE_DEPS lite_profiler)

if (NOT LITE_ON_TINY_PUBLISH)
  lite_cc_library(optimizer SRCS optimizer.cc DEPS mir_pass_manager model_parser program)
  add_subdirectory(mir)
  add_subdirectory(profile)
  add_subdirectory(arena)
endif()

# for mobile, unnecessary to compile the following testings.
if (LITE_WITH_LIGHT_WEIGHT_FRAMEWORK)
    return()
endif()

# lite_cc_library(program_fake_utils SRCS program_fake_utils.cc DEPS mir_ssa_graph
#         scope op_registry proto_desc op
#         ${ops}
#         ${host_kernels}
#         )

lite_cc_test(test_scope SRCS scope_test.cc DEPS scope)
lite_cc_test(test_kernel SRCS kernel_test.cc DEPS kernel target_wrapper any)
lite_cc_test(test_op SRCS op_lite_test.cc DEPS op)
lite_cc_test(test_tensor SRCS lite_tensor_test.cc DEPS tensor)
lite_cc_test(test_type_system SRCS type_system_test.cc DEPS type_system utils)
#lite_cc_test(test_optimizer SRCS optimizer_test.cc DEPS mir_pass_manager program_fake_utils mir_passes optimizer fc_op)
lite_cc_test(test_types SRCS types_test.cc DEPS types)
lite_cc_test(test_memory SRCS memory_test.cc DEPS memory)
lite_cc_test(test_context SRCS context_test.cc DEPS context)


# # A trick to generate the paddle_use_kernels.h
# execute_process(
#   COMMAND python ${CMAKE_SOURCE_DIR}/lite/tools/cmake_tools/parse_kernel_registry.py
#   ${kernels_src_list}
#   ${CMAKE_SOURCE_DIR}/lite/api/paddle_use_kernels.h
#   )
# # A trick to generate the paddle_use_ops.h
# execute_process(
#   COMMAND python ${CMAKE_SOURCE_DIR}/lite/tools/cmake_tools/parse_op_registry.py
#   ${ops_src_list}
#   ${CMAKE_SOURCE_DIR}/lite/api/paddle_use_ops.h
#   )
