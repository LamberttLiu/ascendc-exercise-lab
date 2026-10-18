cmake_minimum_required(VERSION 3.16)

find_package(ASC REQUIRED)
project(vector_add_cpu_debug LANGUAGES ASC CXX)

add_executable(vector_add_cpu_debug ../main.asc)

target_compile_options(vector_add_cpu_debug PRIVATE
    $<$<COMPILE_LANGUAGE:ASC>:--npu-arch=${CMAKE_ASC_ARCHITECTURES}>
)

target_link_libraries(vector_add_cpu_debug PRIVATE
    m
    dl
)
