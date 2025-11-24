# PureCPP Lite

**PureCPP Lite** is a minimal C++‑only fork of the original [purecpp](https://github.com/pureai-ecosystem/purecpp) backend.

- No Python bindings or Python package.
- No Conan toolchain.
- Focused on a **pure C++ RAG core**: loaders, chunking, embeddings, and a Redis‑based vector database.



## Project Structure

```text
.
├── components/              # Core RAG components (loaders, chunking, metadata, chat, vector DB)
├── libs/                    # Third‑party C++ libraries (libtorch, tokenizers-cpp, etc.)
├── scripts/                 # Helper scripts (e.g. install_torch.sh, install_libs.sh)
├── src/                     # Core library sources (if present)
├── CMakeLists.txt           # Main CMake build configuration
├── Dockerfile               # Optional build environment
└── README.md
```

## Features

- **C++‑only RAG core**
  - TXT/PDF/DOCX/Web loaders (configurable at build time).
  - Chunking, cleaning, and metadata extraction.
  - Embedding via OpenAI or ONNX models (HuggingFace‑style).
  - Vector store backed by Redis/RediSearch.

- **No Python bindings**
  - All pybind11 modules, `src/binding.cpp`, and Python package plumbing have been removed.

- **Configurable loaders**
  - CMake options control which loaders and their heavy dependencies are included:
    - `ENABLE_LOADER_TXT`  – TXT loader (ON by default).
    - `ENABLE_LOADER_PDF`  – PDF loader (requires `pdfium`, `ICU`).
    - `ENABLE_LOADER_DOCX` – DOCX loader (requires `miniz`, `rapidxml`).
    - `ENABLE_LOADER_WEB`  – Web/HTML loader (requires `lexbor`, `beauty`, `CURL`).

## Building

### Prerequisites

- A recent C++ compiler (tested with GCC 13+ or Clang with C++23 support).
- **CMake** ≥ 3.22.
- System development packages for the libraries you plan to use, for example:
  - `pdfium`, `icu`, `miniz`, `rapidxml`, `lexbor`, `re2`, `nlohmann_json`,
    `CURL`, `onnxruntime`, `redis++`, `hiredis`, and libtorch (CPU).
- Submodules checked out:

```bash
git submodule update --init --recursive
```

Some bundled dependencies (e.g. libtorch, tokenizers-cpp) are expected under `libs/`.  
You can use the provided helper scripts (where applicable), such as:

```bash
chmod +x scripts/install_torch.sh
chmod +x scripts/install_libs.sh
# then run them as needed to populate ./libs
```

### Configure and Build

From the repository root:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DENABLE_LOADER_TXT=ON \
  -DENABLE_LOADER_PDF=OFF \
  -DENABLE_LOADER_DOCX=OFF \
  -DENABLE_LOADER_WEB=OFF

cmake --build build --target RagPUREAILib -j"$(nproc)"
```

This builds the static library `RagPUREAILib` containing the core RAG functionality.

You can then link it into your own C++ application:

```cmake
add_executable(my_rag_app main.cpp)
target_link_libraries(my_rag_app PRIVATE RagPUREAILib)
```

Adjust the loader options and system dependencies according to the formats you need.

## Notes on Dependencies

- **Redis (redis++ / hiredis)**  
  Used by the VectorDatabase backend in `components/VectorDatabase/src/backends/redis_backend.cpp`
  to provide persistent K‑NN search over embeddings via Redis/RediSearch.

- **Torch (libtorch)**  
  Used in chunking code for vector math (tensors, norms, dot products).  
  If you want to remove Torch, you must first rewrite those usages to plain C++ math and then
  drop `find_package(Torch)` and `${TORCH_LIBRARIES}` from `CMakeLists.txt`.

- **ONNX Runtime + tokenizers-cpp + protobuf**  
  Used by ONNX‑based embedding and metadata extractors (e.g. `MetadataHFExtractor` and parts of
  `ChunkCommons`). You can remove these if you only use OpenAI embeddings or a custom pipeline.

## License

This project is licensed under the MIT License. See [`LICENSE`](./LICENSE) for details.