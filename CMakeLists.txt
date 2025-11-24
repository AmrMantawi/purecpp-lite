cmake_minimum_required(VERSION 3.22)
project(RagPUREAI VERSION 1.0)

# General build settings
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_BUILD_TYPE "Release")
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Basic compiler flags
set(CMAKE_CXX_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")

option(CURL_STATIC_LINKING "Set to ON to build libcurl with static linking." OFF)
option(BUILD_APPS "Build apps" OFF)

# Loader feature toggles
option(ENABLE_LOADER_TXT  "Enable TXT loader"  ON) # On by default
option(ENABLE_LOADER_PDF  "Enable PDF loader"  OFF)
option(ENABLE_LOADER_DOCX "Enable DOCX loader" OFF)
option(ENABLE_LOADER_WEB  "Enable Web/HTML loader" OFF)

# External Dependencies

# Core text processing (used by TXT loader and others)
find_package(ICU REQUIRED)
find_package(re2 REQUIRED)

# Loader: PDF
if(ENABLE_LOADER_PDF)
    find_package(pdfium REQUIRED)
endif()

# Loader: DOCX
if(ENABLE_LOADER_DOCX)
    find_package(miniz REQUIRED)
    find_package(rapidxml REQUIRED)
endif()

# Loader: Web / HTML
if(ENABLE_LOADER_WEB)
    find_package(lexbor REQUIRED)
    find_package(CURL REQUIRED)
    find_package(beauty REQUIRED)
endif()

# ML runtimes
find_package(onnxruntime REQUIRED)

# Caching / vector database backend
find_package(redis++ REQUIRED)

# Parallelism
find_package(OpenMP REQUIRED)

# Protobuf
find_package(Protobuf REQUIRED)
include_directories(${Protobuf_INCLUDE_DIRS})

# Torch
set(Torch_DIR "${CMAKE_SOURCE_DIR}/libs/libtorch/cpu/share/cmake/Torch")
find_package(Torch REQUIRED)
include_directories("${CMAKE_SOURCE_DIR}/libs/libtorch/cpu/include")
link_directories("${CMAKE_SOURCE_DIR}/libs/libtorch/cpu/lib")

# Tokenizers
set(TOKENIZERS_PATH "${CMAKE_SOURCE_DIR}/libs/tokenizers-cpp")
add_subdirectory(${TOKENIZERS_PATH} tokenizers EXCLUDE_FROM_ALL)

# OpenAI
set(OPEANAI_CPP_PATH "${CMAKE_SOURCE_DIR}/libs/openai-cpp")

# RPATH
set(CMAKE_BUILD_RPATH
    "\$ORIGIN/purecpp.libs"
    "\$ORIGIN/d_libs/libtorch/cpu/lib"
    "${CMAKE_SOURCE_DIR}/libs/libtorch/cpu/lib"
    "\$ORIGIN/libs/libtorch/cpu/lib"
    "\$ORIGIN:/usr/bin/protoc"
    "/usr/lib/x86_64-linux-gnu"
    "/usr/lib64"
    "\$ORIGIN/"
    "\$ORIGIN/purecpp/d_libs/libtorch/cpu/lib"
)
set(CMAKE_INSTALL_RPATH
    "\$ORIGIN/purecpp.libs"
    "\$ORIGIN/d_libs/libtorch/cpu/lib"
    "${CMAKE_SOURCE_DIR}/libs/libtorch/cpu/lib"
    "\$ORIGIN/libs/libtorch/cpu/lib"
    "\$ORIGIN:/usr/bin/protoc"
    "/usr/lib/x86_64-linux-gnu"
    "/usr/lib64"
    "\$ORIGIN/"
    "\$ORIGIN/purecpp/d_libs/libtorch/cpu/lib"
)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--enable-new-dtags")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Sources
file(GLOB_RECURSE VDB_SRCS ${CMAKE_SOURCE_DIR}/components/VectorDatabase/src/*.cpp)

set(RagPUREAI_IMPL_SRCS
    ${VDB_SRCS}
    ${CMAKE_SOURCE_DIR}/libs/StringUtils/StringUtils.cpp
    ${CMAKE_SOURCE_DIR}/libs/CommonStructs/CommonStructs.cpp

    # Data loaders
    ${CMAKE_SOURCE_DIR}/components/DataLoader/BaseLoader.cpp

    ${CMAKE_SOURCE_DIR}/components/MetadataExtractor/MetadataExtractor.cpp
    ${CMAKE_SOURCE_DIR}/components/MetadataExtractor/MetadataRegexExtractor/MetadataRegexExtractor.cpp
    ${CMAKE_SOURCE_DIR}/components/MetadataExtractor/MetadataHFExtractor/MetadataHFExtractor.cpp

    ${CMAKE_SOURCE_DIR}/components/Embedding/EmbeddingOpenAI/EmbeddingOpenAI.cpp
    ${CMAKE_SOURCE_DIR}/components/Embedding/EmbeddingModel/EmbeddingModel.cpp

    ${CMAKE_SOURCE_DIR}/components/Chunk/ChunkCommons/ChunkCommons.cpp
    ${CMAKE_SOURCE_DIR}/components/Chunk/ChunkCount/ChunkCount.cpp
    ${CMAKE_SOURCE_DIR}/components/Chunk/ChunkDefault/ChunkDefault.cpp
    ${CMAKE_SOURCE_DIR}/components/Chunk/ChunkSimilarity/ChunkSimilarity.cpp
    ${CMAKE_SOURCE_DIR}/components/Chunk/ChunkQuery/ChunkQuery.cpp

    ${CMAKE_SOURCE_DIR}/components/CleanData/ContentCleaner/ContentCleaner.cpp

    ${CMAKE_SOURCE_DIR}/components/Chat/Message/HumanMessage.cpp
    ${CMAKE_SOURCE_DIR}/components/Chat/Message/AIMessage.cpp
    ${CMAKE_SOURCE_DIR}/components/Chat/Message/SystemMessage.cpp
    ${CMAKE_SOURCE_DIR}/components/Chat/ChatHistory/ChatHistory.cpp
)

if(ENABLE_LOADER_PDF)
    list(APPEND RagPUREAI_IMPL_SRCS
        ${CMAKE_SOURCE_DIR}/components/DataLoader/PDFLoader/PDFLoader.cpp
    )
endif()

if(ENABLE_LOADER_DOCX)
    list(APPEND RagPUREAI_IMPL_SRCS
        ${CMAKE_SOURCE_DIR}/components/DataLoader/DOCXLoader/DOCXLoader.cpp
    )
endif()

if(ENABLE_LOADER_WEB)
    list(APPEND RagPUREAI_IMPL_SRCS
        ${CMAKE_SOURCE_DIR}/components/DataLoader/WebLoader/WebLoader.cpp
    )
endif()

if(ENABLE_LOADER_TXT)
    list(APPEND RagPUREAI_IMPL_SRCS
        ${CMAKE_SOURCE_DIR}/components/DataLoader/TXTLoader/TXTLoader.cpp
    )
endif()

# RagPUREAILib
add_library(RagPUREAILib STATIC ${RagPUREAI_IMPL_SRCS})
target_include_directories(RagPUREAILib PUBLIC
    ${CMAKE_SOURCE_DIR}/components
    ${CMAKE_SOURCE_DIR}/components/DataLoader
    ${CMAKE_SOURCE_DIR}/components/MetadataExtractor
    ${CMAKE_SOURCE_DIR}/components/Chunk
    ${CMAKE_SOURCE_DIR}/components/CleanData
    ${CMAKE_SOURCE_DIR}/components/Embedding
    ${CMAKE_SOURCE_DIR}/components/Embedding/EmbeddingOpenAI
    ${CMAKE_SOURCE_DIR}/components/Embedding/EmbeddingModel

    ${CMAKE_SOURCE_DIR}/components/Chat
    ${CMAKE_SOURCE_DIR}/components/Chat/ChatHistory
    ${CMAKE_SOURCE_DIR}/components/Chat/Message

    ${CMAKE_SOURCE_DIR}/components/VectorDatabase/include

    ${CMAKE_SOURCE_DIR}/libs/RagException
    ${CMAKE_SOURCE_DIR}/libs/ThreadSafeQueue
    ${CMAKE_SOURCE_DIR}/libs/CommonStructs
    ${CMAKE_SOURCE_DIR}/libs/StringUtils
    ${CMAKE_SOURCE_DIR}/libs/FileUtils
    ${CMAKE_SOURCE_DIR}/libs/MemoryUtils
    ${TOKENIZERS_PATH}/include
    ${OPEANAI_CPP_PATH}/include
    ${CMAKE_SOURCE_DIR}/libs/libtorch/cpu/include
    ${CURL_INCLUDE_DIRS}
    ${TORCH_INCLUDE_DIRS}
)

target_link_libraries(RagPUREAILib PUBLIC
    # Core text processing (used by TXT loader and others)
    icu::icu
    re2::re2

    # Caching / vector database backend
    redis++::redis++_static
    hiredis::hiredis

    # ML runtimes
    onnxruntime::onnxruntime
    ${TORCH_LIBRARIES}

    # Tokenization / model formats
    tokenizers_cpp
    protobuf::libprotobuf
)

if(ENABLE_LOADER_PDF)
    target_link_libraries(RagPUREAILib PUBLIC
        # Loader: PDF
        pdfium::pdfium
    )
endif()

if(ENABLE_LOADER_DOCX)
    target_link_libraries(RagPUREAILib PUBLIC
        # Loader: DOCX
        miniz::miniz
        rapidxml::rapidxml
    )
endif()

if(ENABLE_LOADER_WEB)
    target_link_libraries(RagPUREAILib PUBLIC
        # Loader: Web / HTML
        lexbor::lexbor_static
        CURL::libcurl
        beauty::beauty
    )
endif()
