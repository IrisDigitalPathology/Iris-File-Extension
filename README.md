# Iris File Extension

This is the official implementation of the Iris File Extension specification, part of the Iris Digital Pathology project. This repository has a very limited scope; it provies the byte-offset vtables and enumerations referenced by the Iris Codec specification and validates files against the published IFE specification. This is an advanced repository. **If this is your first foray into Iris, the [Iris Codec Community Module](https://github.com/IrisDigitalPathology/Iris-Codec.git) is a much better choice**. 

Example Iris slide files are hosted to test decoding are hosted at [the Iris-Example-Files repository](https://github.com/IrisDigitalPathology/Iris-Example-Files). 

> [!CAUTION]
> **This repository is primarily for scanner device manufacturers and programmers wishing to write custom encoders and decoders. If this does not describe your goals, you should instead incorporate the [Iris Codec Community Module](https://github.com/IrisDigitalPathology/Iris-Codec.git) into your project**. This repository allows for low-level manipulation of the Iris File Extension file structure in a very narrow scope: it just provides byte offsets and validation checks against the current IFE standard. Most programmers (particularly for research) attempting to access Iris files **should not use this repository** and use Iris Codec instead. 

> [!NOTE]
> The scope of this repository is only serializing or deserializing Iris slide files. Compression and decompression are **NOT** components of this repository. The WSI tile byte arrays will be referenced in their on-disk compressed forms and it is up to your implementation to compress or decompress tiles. If you would like a system that performs image compression and decompression, you should instead incorporate the [Iris Codec Community Module](https://github.com/IrisDigitalPathology/Iris-Codec.git), which incorporates this repository for Iris slide file serialization.

This repository builds tools to access Iris files as C++ headers/library or as modules with Python or JavaScript (32-bit) bindings. The repository uses the CMake build system.

<p xmlns:cc="http://creativecommons.org/ns#" >This repository is licensed under the MIT software license. The Iris File Extension is licensed under <a href="https://creativecommons.org/licenses/by-nd/4.0/?ref=chooser-v1" target="_blank" rel="license noopener noreferrer" style="display:inline-block;">CC BY-ND 4.0 <img style="height:22px!important;margin-left:3px;vertical-align:text-bottom;" src="https://mirrors.creativecommons.org/presskit/icons/cc.svg?ref=chooser-v1" alt=""><img style="height:22px!important;margin-left:3px;vertical-align:text-bottom;" src="https://mirrors.creativecommons.org/presskit/icons/by.svg?ref=chooser-v1" alt=""><img style="height:22px!important;margin-left:3px;vertical-align:text-bottom;" src="https://mirrors.creativecommons.org/presskit/icons/nd.svg?ref=chooser-v1" alt=""></a></p>

# Installation
Incorporating the Iris File Extension into your code base is simple; Additional [Iris headers](https://github.com/IrisDigitalPathology/Iris-Headers) are required but are automatically included when this repository is built or included in a CMake project.

In addition to building from source, we provide pre-compiled binaries for all major systems under the **releases tab**, as well as additional language bindings for [Python](README.md#python-interface) and [JavaScript](README.md#javascript-interface).

### Non-CMake Project
If you are **NOT** using CMake to build your project, you should still use CMake to generate the Iris File Extension library.
```shell
git clone --depth 1 https://github.com/IrisDigitalPathology/Iris-File-Extension.git
# Optional cmake flags to consider: 
#   -DCMAKE_INSTALL_PREFIX='' for custom install directory
#   -DBUILD_EXAMPLES=ON to test build the included examples
#   -DBUILD_PYTHON=ON to build the Python interface
cmake -B ./Iris-File-Extension/build ./Iris-File-Extension 
cmake --build ./Iris-File-Extension/build --config Release
cmake --install ./Iris-File-Extension/build
```

### CMake Project
If you **are** using CMake for your build, You may directly incorporate this repository into your code base using the following about **10 lines of code** in your project's CMakeLists.txt:
```CMake
FetchContent_Declare (
    IrisFileExtension
    GIT_REPOSITORY https://github.com/IrisDigitalPathology/Iris-File-Extension.git
    GIT_TAG "origin/main"
    GIT_SHALLOW ON
)
FetchContent_MakeAvailable(IrisFileExtension)
```
and then link against this repository as you would any other library
```CMake
target_link_libraries (
    YOUR_TARGET PRIVATE
    IrisFileExtension #Use 'IrisFileExtensionStatic' for static linkage
)
```
# Implementation
> [!CAUTION]
> **This API is not still early in development and liable to change. As we apply new updates some of the exposed calls may change.** If dynamically linked, always check new headers against your code base when updating your version of the Iris File Extension API.

## C++ Interface

### Always Validate a Slide
> [!WARNING]
> When reading Iris slide files, you should **always validate** a slide before attempting to read data from it. 

Validation will return an `Iris::Result` structure. In the event of failure `Iris::Result::message` will provide information about the failure. Validation requires the operating system's returned file size as part of the validation process and will fail if inaccurate. Validation *can be* performed by calling the [`IrisCodec::validate_file_structure`](./src/IrisCodecExtension.hpp#L101) method.
```cpp
size_t  size = GET_FILE_SIZE(file_handle);
uint8_t* ptr = FILE_MAP(file_handle, size);

auto  result = IrisCodec::validate_file_structure(ptr, size);

if (result != IRIS_SUCCESS) {
    printf(result.message);
    ...handle the validation error
}
```
This method performs a chain of `validate_full(uint8_t*)` methods on the component parts of slides. If you prefer to validate individual data blocks, you may individually call the `validate_offset(uint8_t*)` and `validate_full(uint8_t*)` methods that are defined in all data blocks. See the more in-depth [README](./src/README.md) associated with the source directory. 


### Using Slide Abstraction
The easiest way to access slide information is via the [`IrisCodec::Abstraction::File`](https://github.com/IrisDigitalPathology/Iris-File-Extension/blob/2646ee4e986f90247e447000c035490d3114d98f/src/IrisCodecExtension.hpp#L206-L212), which abstracts representations of the data elements still residing on disk (and providing byte-offset locations within the mapped WSI file to access these elements in an optionally **zero-copy manner**). [An example implementation reading using file abstraction is available](./examples/slide_info_abstraction.cpp). 
> [!WARNING]
> If you did not validate prior to abstraction, uncaught runtime exceptions will be thrown if the slide violates the standard. We leave how to deal with validation exceptions to your implementation, should they arise.  
```cpp
struct IrisCodec::Abstraction::File {
    Header          header;      // File Header information
    TileTable       tileTable;   // Table of slide extent and WSI 256 pixel tiles
    Images          images;      // Set of ancillary images (label, thumbnail, etc...)
    Annotations     annotations; // Set of on-slide annotation objects
    Metadata        metadata;    // Slide metadata (patient info, acquisition. etc...)
};
```
```cpp
try {
    using namespace IrisCodec::Abstraction;
    File file = abstract_file_structure ((uint8_t*)ptr, size);
    std::cout   << "Encoded using IFE Spec v"
                << (file.header.extVersion >> 16) << "."
                << (file.header.extVersion & 0xFFFF) << std::endl;
    
    // We can retrieve data easily from the slide
    // Get the encoding type (IrisCodec::Encoding)
    auto compression_format = file.tileTable.encoding;
    // Get the location and offset of the tile (layer, tile_index)
    auto& layer_0_1_bytes = file.tileTable.layers[0][1];
    // And 'decompress' based upon whatever JPEG, AVIF, etc... library you use
    char* some_buffer = decompress (ptr + layer_0_1_bytes.offset,layer_0_1_bytes.size);
    // Or copy it from disk
    memcpy(some_buffer, ptr + layer_0_1_bytes.offset,layer_0_1_bytes.size);

    // Don't worry about clean up. You're just referencing on-disk locations.
} catch (std::runtime_error &error) {
    ...handle the read error
}
```

### Manually *without* File Abstraction 
Instead of using the file abstraction routine, you may manually access data block elements. All data blocks within the slide file are derived from the `Serialization::DATA_BLOCK` structure defined below. These data blocks reside within the `Serialization:: namespace` and are **always** fully capitalized. They are accessed by retrieval from parent data blocks using methods that begin with "get" followed by the name of any derived data blocks. Data block information is read using the "read" methods. These methods generate structures within the `Abstraction:: namespace`. This method for manually reading data from within the IFE will be covered in greater detail within the [README](./src/README.md) associated with the source directory. 
```cpp
struct DATA_BLOCK {
    // Each datablock has an vtable
    enum vtable_sizes   {
        VALIDATION_S                = TYPE_SIZE_UINT64,
        RECOVERY_S                  = TYPE_SIZE_UINT16,
        ///... Other elements (see IFE Specification)
    };
    enum vtable_offsets {
        VALIDATION                  = 0,
        RECOVERY                    = VALIDATION + VALIDATION_S,
        ///... Other elements (see IFE Specification)
    };
    // And only stores 3 pieces of information
    // 1) The datablock offset on disk
    // 2) The file size for validation
    // 3) The IFE version
    Offset      __offset            = NULL_OFFSET;
    Size        __size              = 0;
    uint32_t    __version           = 0;
    explicit    DATA_BLOCK          (Offset, Size file_size, uint32_t IFE_version);
    Result      validate_offset     (BYTE* const __base) const noexcept;
};
```

### File Data Mapping
**File data mapping is more advanced functionality**. The IFE provides a powerful tool to assess the location of data blocks within a serialized slide file. This is critical when performing file recovery or file updates, as you may overwrite already used regions (eg. when expanding arrays). A file map allows for finding all data-blocks before or after a byte offset location (using std::map binary search tree internally). A file map entry, shown below, describes the location, size, and type of data block within the file at the given location. You may recast the datablock as it's internally defined type and use it per the API, though we recommend validating it first before attempting to do so. 
```cpp
// File Map Entry contains information about the datablock (what type)
// and the offset location
struct FileMapEntry {
    using Datablock                 = Serialization::DATA_BLOCK;
    MapEntryType        type        = MAP_ENTRY_UNDEFINED;
    Datablock           datablock;
    Size                size        = 0;
};
```
```cpp
try {
    // Always validate the slide file first
    IrisCodec::validate_file_structure(ptr, size);
    // Then generate the slide map. 
    // See IrisCodecExtension.cpp for implementation of its construction
    auto file_map = IrisCodec::generate_file_map((uint8_t*)ptr, size);

    Offset write_location = //...some location you will write at;
    auto data_blocks_after = file_map.upper_bound(write_location);
    for (;data_blocks_after!=file_map.cend();++data_blocks_after) {

        //...do something (copy into memory for writing later, etc...)

        Offset offset_location = data_block->first;
        Size   block_byte_size = data_block->second.size;
        switch (data_block->second.type) {
            ...
            case MAP_ENTRY_TILE_TABLE: 
            static_cast<TILE_TABLE&>(data_block->second.datablock).validate_offset(ptr);
            static_cast<TILE_TABLE&>(data_block->second.datablock).read_tile_table(ptr);
            ...
        }
        
    }    
} catch (std::runtime_error &error) {
    ...handle the validation error
}
```

## Python Interface

## JavaScript Interface

# Publications
