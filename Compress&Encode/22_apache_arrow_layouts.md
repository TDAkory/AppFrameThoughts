# [`Apache Arrow` Layouts](https://arrow.apache.org/docs/format/Intro.html)

本文结合 [Apache Arrow Specifications](https://arrow.apache.org/docs/format/Intro.html) 和 [源代码](https://github.com/apache/arrow/tree/main/cpp/src/arrow)。记录 `Apache Arrow` 提供的数据类型，以及这些类型对应的内存分布。

> [Arrow Columnar Format](https://arrow.apache.org/docs/format/Columnar.html)

### 前置：`Arrow`的基础结构

`Arrow` 最基础的类型枚举、`Buffer`、`Field`、`Array` 基类

```cpp
// 参考：arrow/type.h
enum class TypeID {
  // 固定大小Primitive
  INT8, INT16, INT32, INT64,
  UINT8, UINT16, UINT32, UINT64,
  FLOAT32, FLOAT64,
  BOOL,
  DATE32, DATE64, TIME32, TIME64,
  // 变长类型
  BINARY, LARGE_BINARY,
  STRING, LARGE_STRING,
  STRING_VIEW, LARGE_STRING_VIEW,
  // 列表类型
  LIST, LARGE_LIST,
  FIXED_SIZE_LIST,
  LIST_VIEW, LARGE_LIST_VIEW,
  // 复合类型
  STRUCT,
  MAP,
  DENSE_UNION, SPARSE_UNION,
  // 编码类型
  DICTIONARY,
  RUN_END_ENCODED
};
```

Arrow 连续内存缓冲器（Buffer）。Arrow 规范要求内存分配满足特定对齐要求（通常为 64 字节），Buffer 实现确保了这一点

```cpp
// 1. 通常具有只读语义：构建后不修改，零拷贝共享的核心
// 2. 内存由 MemoryPool 管理
// 3. 支持空缓冲器（data=nullptr, size=0）
// 参考：arrow/buffer.h
class Buffer {
  bool is_mutable_;     // 标志控制内存是否可写
  bool is_cpu_;         // 快速检查是否为 CPU 内存
  const uint8_t* data_; // 指向内存的指针
  int64_t size_;        // 有效数据大小
  int64_t capacity_;    // 总分配容量
  DeviceAllocationType device_type_;    // 标识内存所在设备类型
  // null by default, but may be set
  std::shared_ptr<Buffer> parent_;      // 最主要的作用是支持零拷贝切片操作(内存安全、避免悬空指针)
};
```

值得注意的事这里的 `parent_` 成员，它最主要的作用是支持零拷贝切片操作(内存安全、避免悬空指针)。

通过持有原始 Buffer 的 shared_ptr，切片 Buffer 确保：原始内存块在所有切片都被销毁前不会被释；避免了悬空指针问题；实现了内存的安全共享

`parent_` 同时还支持多层切片，形成链式引用：对切片再次切片时，新切片的 parent_ 指向直接父切片；整个链共同维护对原始内存的引用

```cpp
  Buffer(std::shared_ptr<Buffer> parent, const int64_t offset, const int64_t size)
    : Buffer(parent->data_ + offset, size) {
  parent_ = std::move(parent);  // 保存对原始Buffer的引用
  SetMemoryManager(parent_->memory_manager_);
  }

  static inline std::shared_ptr<Buffer> SliceBuffer(std::shared_ptr<Buffer> buffer,
                                                  const int64_t offset,
                                                  const int64_t length) {
  return std::make_shared<Buffer>(std::move(buffer), offset, length);
  }
```

`ArrayData` 通过**统一的内存模型、灵活的分层结构、高效的空值处理和设备无关设计**，为数据存储和分析提供了高效、灵活且安全的基础。

```cpp
// arrow/array/data.h arrow/array/array_base.h
class ArrayData{
  std::shared_ptr<DataType> type;               // 数据类型
  int64_t length = 0;                           // 数组长度
  mutable std::atomic<int64_t> null_count{0};   // // 缓存的空值计数
  // The logical start point into the physical buffers (in values, not bytes).
  // Note that, for child data, this must be *added* to the child data's own offset.
  int64_t offset = 0;
  // 基于 Buffer 的多缓冲区设计(有效性位图、值数据等分离存储, buffers[0] 通常是有效性位图)
  std::vector<std::shared_ptr<Buffer>> buffers;         
  std::vector<std::shared_ptr<ArrayData>> child_data;   // 支持嵌套数据类型
  // The dictionary for this Array, if any. Only used for dictionary type
  std::shared_ptr<ArrayData> dictionary;
  // The statistics for this Array.
  std::shared_ptr<ArrayStatistics> statistics;
};
```

`ArrayData` 支持跨设备内存管理，通过统一的内存访问接口，实现高效的跨设备数据传输

```cpp
  Result<std::shared_ptr<ArrayData>> CopyTo(const std::shared_ptr<MemoryManager>& to) const;
  Result<std::shared_ptr<ArrayData>> ViewOrCopyTo(const std::shared_ptr<MemoryManager>& to) const;
```

`Array` 类被设计为不可变数据容器，采用了数据与接口分离的设计模式

```cpp
// arrow/array.h
class Array {
  std::shared_ptr<ArrayData> data_;             // 实际数据存储
  const uint8_t* null_bitmap_data_ = NULLPTR;   // 采用位图（bitmap）表示空值状态
};
```

`Array` 与 `ArrayData`：

| 特性 | Array | ArrayData |
|------|-------|-----------|
| 性质 | 不可变接口 | 可变数据容器 |
| 用途 | 外部访问接口 | 内部数据操作 |
| 设计 | 面向用户 | 面向实现 |
| 类型安全 | 强类型 | 弱类型 |

这种分离设计使得：

- 用户可以通过类型安全的 `Array` 接口访问数据
- 内部实现可以通过 `ArrayData` 高效地操作原始数据
- 数据可以在不同类型的 `Array` 之间共享，提高内存利用率

### 1. 固定大小 Primitive 布局（Fixed Size Primitive）

`PrimitiveArray` 是 `Arrow` 中用于表示基本数据类型数组的抽象基类，专门用于处理具有固定大小元素的原始数据类型，如数值类型、布尔类型和时间间隔类型等

```cpp
// arrow/array/array_base.h arrow/array/array_primitive.h
// Base class for arrays of fixed-size logical types
class ARROW_EXPORT PrimitiveArray : public FlatArray {
    // 从父类继承的成员
    // std::shared_ptr<ArrayData> data_;
    // const uint8_t* null_bitmap_data_ = NULLPTR;

    const uint8_t* raw_values_; // // 指向数据缓冲区的直接指针
};

// 构造函数
PrimitiveArray(const std::shared_ptr<DataType>& type, int64_t length,
               const std::shared_ptr<Buffer>& data,
               const std::shared_ptr<Buffer>& null_bitmap = NULLPTR,
               int64_t null_count = kUnknownNullCount, int64_t offset = 0);

// 设置数据
void SetData(const std::shared_ptr<ArrayData>& data) {
  this->Array::SetData(data);
  raw_values_ = data->GetValuesSafe<uint8_t>(1, /*offset=*/0);
}

// 获取值缓冲区
const std::shared_ptr<Buffer>& values() const { return data_->buffers[1]; }
```

继承链

```shell
Array
└── FlatArray
    └── PrimitiveArray
        ├── BooleanArray
        ├── NumericArray<TYPE> (模板类)
        │   ├── Int8Array
        │   ├── Int16Array
        │   ├── Int32Array
        │   ├── Int64Array
        │   ├── UInt8Array
        │   ├── UInt16Array
        │   ├── UInt32Array
        │   ├── UInt64Array
        │   ├── FloatArray
        │   ├── DoubleArray
        │   ├── Date32Array
        │   ├── Date64Array
        │   ├── Time32Array
        │   ├── Time64Array
        │   ├── TimestampArray
        │   └── ...
        ├── DayTimeIntervalArray
        └── MonthDayNanoIntervalArray
```

Int32 Array

```shell
# [1, null, 2, 4, 8]
* Length: 5, Null count: 1
* Validity bitmap buffer:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00011101                 | 0 (padding)           |

* Value Buffer:

  | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63           |
  |-------------|-------------|-------------|-------------|-------------|-----------------------|
  | 1           | unspecified | 2           | 4           | 8           | unspecified (padding) |
```

Non-null int32 Array may elide bitmap

```shell
# [1, 2, 3, 4, 8]
* Length 5, Null count: 0
* Validity bitmap buffer: Not required
* Value Buffer:

  | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | bytes 12-15 | bytes 16-19 | Bytes 20-63           |
  |-------------|-------------|-------------|-------------|-------------|-----------------------|
  | 1           | 2           | 3           | 4           | 8           | unspecified (padding) |
```

### 2. 变长 Binary/String 布局（Variable Length Binary/String）

`BinaryType` 是 `Arrow` 中用于表示可变大小二进制数据的核心类型类，定义在 `array_binary.h` 文件中，继承自 `BaseBinaryType`

```cpp
// arrow/array/array_binary.h
// Base class for variable-sized binary arrays, regardless of offset size
// and logical interpretation.
template <typename TYPE>
class BaseBinaryArray : public FlatArray {
};

class ARROW_EXPORT BinaryType : public BaseBinaryType {
 public:
  static constexpr Type::type type_id = Type::BINARY;
  static constexpr bool is_utf8 = false;
  using offset_type = int32_t;
  using PhysicalType = BinaryType;

  static constexpr const char* type_name() { return "binary"; }

  BinaryType() : BinaryType(Type::BINARY) {}

  DataTypeLayout layout() const override {
    return DataTypeLayout({DataTypeLayout::Bitmap(),
                           DataTypeLayout::FixedWidth(sizeof(offset_type)),
                           DataTypeLayout::VariableWidth()});
  }
};

class ARROW_EXPORT BinaryArray : public BaseBinaryArray<BinaryType> {
 public:
  explicit BinaryArray(const std::shared_ptr<ArrayData>& data);

  BinaryArray(int64_t length, const std::shared_ptr<Buffer>& value_offsets,
              const std::shared_ptr<Buffer>& data,
              const std::shared_ptr<Buffer>& null_bitmap = NULLPTR,
              int64_t null_count = kUnknownNullCount, int64_t offset = 0);
};
```

继承结构

```shell
DataType
└── BaseBinaryType
    ├── BinaryType          # 使用 32 位偏移量的标准二进制类型
    │   └── StringType      # 用于 UTF-8 字符串
    ├── LargeBinaryType     # 用于 UTF-8 字符串
    │   └── LargeStringType
    └── BinaryViewType      # 二进制视图类型，支持内联优化
```

``VarBinary``例子：

```shell
# ['joe', null, null, 'mark']


* Length: 4, Null count: 2
* Validity bitmap buffer:   # 每个位代表一个元素是否为空

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001001                 | 0 (padding)           |

* Offsets buffer:   # offsets[j+1] >= offsets[j] for 0 <= j < length

  | Bytes 0-19     | Bytes 20-63           |
  |----------------|-----------------------|
  | 0, 3, 3, 3, 7  | unspecified (padding) |

 * Value buffer:    # 连续存储二进制数据，item[i] = [offsets[i], offsets[i+1])

  | Bytes 0-6      | Bytes 7-63            |
  |----------------|-----------------------|
  | joemark        | unspecified (padding) |
```

`StringType` 继承自 `BinaryType`，主要区别在于：

- `StringType::is_utf8 = true`
- 提供额外的 UTF-8 验证方法（如 `ValidateUTF8()`）

BinaryType 适用于以下场景：

- 存储任意二进制数据（如图像、音频、视频片段）
- 存储序列化对象
- 存储非 UTF-8 编码的文本数据
- 需要高效二进制数据处理的场景

| 类型 | 偏移量类型 | 最大大小 | 适用场景 |
|------|------------|----------|----------|
| BinaryType | int32_t | 2GB | 大多数二进制数据场景 |
| LargeBinaryType | int64_t | 9EB | 大型二进制数据 |
| BinaryViewType | 无 | 2GB | 二进制视图，支持内联优化 |
| FixedSizeBinaryType | 无 | 固定大小 | 固定长度二进制数据 |

### 3. 变长 Binary/String View 布局（Binary/String View）

`StringViewArray` 是 `Arrow` 中用于高效表示可变大小 UTF-8 字符串的数组类型，它继承自 `BinaryViewArray`，采用了**视图（View）**设计模式，结合了内联存储和引用存储的优势，旨在提供更高效的字符串处理性能。

```cpp
// arrow/array/array_binary.h
class ARROW_EXPORT StringViewType : public BinaryViewType {
 public:
  static constexpr Type::type type_id = Type::STRING_VIEW;
  static constexpr bool is_utf8 = true;
  using PhysicalType = BinaryViewType;

  static constexpr const char* type_name() { return "utf8_view"; }

  StringViewType() : BinaryViewType(Type::STRING_VIEW) {}
};

class ARROW_EXPORT StringViewArray : public BinaryViewArray {
 public:
  using TypeClass = StringViewType;

  explicit StringViewArray(std::shared_ptr<ArrayData> data);

  using BinaryViewArray::BinaryViewArray;

  /// \brief Validate that this array contains only valid UTF8 entries
  ///
  /// This check is also implied by ValidateFull()
  Status ValidateUTF8() const;
};
```

继承结构

```shell
Array
└── FlatArray
    └── BinaryViewArray
        └── StringViewArray
```

`StringViewArray` 的高效性主要来自于其底层的视图存储机制，该机制由 `BinaryViewType::c_type` 联合体实现：

```cpp
/// This union supports two states:
///
/// - Entirely inlined string data
/// \code{.unparsed}
///                |----|--------------|
///                 ^    ^
///                 |    |
///              size    in-line string data, zero padded
/// \endcode
///
/// - Reference into a buffer
/// \code{.unparsed}
///                |----|----|----|----|
///                 ^    ^    ^    ^
///                 |    |    |    |
///              size    |    |    `------.
///                  prefix   |           |
///                        buffer index   |
///                                  offset in buffer
/// \endcode
union alignas(int64_t) c_type {
  struct {
    int32_t size;
    std::array<uint8_t, kInlineSize> data;      // kInlineSize = 12：内联存储的最大数据大小
  } inlined;

  struct {
    int32_t size;
    std::array<uint8_t, kPrefixSize> prefix;    // kPrefixSize = 4：引用存储时的前缀大小
    int32_t buffer_index;
    int32_t offset;
  } ref;
};
```

layouts:

```shell
* Short strings, length <= 12
  | Bytes 0-3  | Bytes 4-15                            |
  |------------|---------------------------------------|
  | length     | data (padded with 0)                  |

* Long strings, length > 12
  | Bytes 0-3  | Bytes 4-7  | Bytes 8-11 | Bytes 12-15 |
  |------------|------------|------------|-------------|
  | length     | prefix     | buf. index | offset      |
```

example:

![Physical layout diagram for variable length string view data type](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/arrow_string_view_array.svg)

### 4. 变长 List 布局（Variable Length List）
```cpp
// 变长 List 数组

// 1. 本质是「数组的数组」，由 offsets + 子数组组成
// 2. offsets[i] = 第 i 个 List 在子数组中的起始索引
// 3. LargeList 用 int64 偏移（突破 2GB 限制）
// 4. 子数组可为任意类型（Primitive/Binary/Struct 等）
// 参考：arrow/array/list_array.h
template <TypeID ListType, typename OffsetType = int32_t>
struct ListArray : public Array {
  std::shared_ptr<Buffer> offsets; // 偏移缓冲器：OffsetType[]，长度=length+1
  std::shared_ptr<Array> values;   // 子数组：存储所有 List 的底层元素

  ListArray() { type_id = ListType; }

  // 核心方法：获取指定索引的 List 切片（零拷贝）
  std::shared_ptr<Array> GetSlice(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const auto* offs = offsets->As<OffsetType>();
    const OffsetType start = offs[idx];
    const OffsetType end = offs[idx + 1];
    const int64_t slice_len = end - start;
    // 实现：切片仅修改子数组的长度/偏移，不拷贝数据
    return SliceArray(values, start, slice_len);
  }

private:
  // 切片逻辑（伪代码）
  std::shared_ptr<Array> SliceArray(std::shared_ptr<Array> arr, int64_t start, int64_t len) const {
    // 实际仅修改数组的 length/null_count，复用原有 Buffer
    return arr;
  }
};

// 类型别名
using ListArray32 = ListArray<TypeID::LIST, int32_t>;
using LargeListArray = ListArray<TypeID::LARGE_LIST, int64_t>;
```

### 5. 固定大小 List 布局（Fixed Size List）
```cpp
// 固定大小 List 数组

// 1. 无 offsets 缓冲器，每个 List 固定包含 element_size 个元素
// 2. 子数组总长度 = length * element_size
// 3. 设计目的：优化固定长度列表（如 3D 坐标 [x,y,z]）的内存访问
// 参考：arrow/array/fixed_size_list_array.h
struct FixedSizeListArray : public Array {
  int32_t element_size;           // 每个 List 固定的元素个数
  std::shared_ptr<Array> values;  // 子数组：总长度 = length * element_size

  FixedSizeListArray() { type_id = TypeID::FIXED_SIZE_LIST; }

  // 核心方法：获取指定索引的 List 切片
  std::shared_ptr<Array> GetSlice(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const int64_t start = idx * element_size;
    const int64_t len = element_size;
    return SliceArray(values, start, len);
  }

private:
  std::shared_ptr<Array> SliceArray(std::shared_ptr<Array> arr, int64_t start, int64_t len) const {
    return arr;
  }
};
```

### 6. List View 布局（List View）
```cpp
// List View 数组

// 1. 设计目的：支持 offsets 乱序，无需从连续偏移推导 List 大小
// 2. 由 sizes + offsets + 子数组组成：
//    - sizes：每个 List 的元素个数
//    - offsets：每个 List 在子数组中的起始索引
// 3. LargeListView 用 int64 类型的 sizes/offsets
// 参考：arrow/array/list_view_array.h
template <TypeID ListViewType>
struct ListViewArray : public Array {
  using OffsetType = std::conditional_t<ListViewType == TypeID::LIST_VIEW, int32_t, int64_t>;

  std::shared_ptr<Buffer> sizes;   // 大小缓冲器：OffsetType[]，长度=length
  std::shared_ptr<Buffer> offsets; // 偏移缓冲器：OffsetType[]，长度=length
  std::shared_ptr<Array> values;   // 子数组

  ListViewArray() { type_id = ListViewType; }

  // 核心方法：获取 List 大小和起始偏移
  std::pair<OffsetType, OffsetType> GetSizeAndOffset(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const auto* sizes_data = sizes->As<OffsetType>();
    const auto* offsets_data = offsets->As<OffsetType>();
    return {sizes_data[idx], offsets_data[idx]};
  }
};

// 类型别名
using ListViewArray32 = ListViewArray<TypeID::LIST_VIEW>;
using LargeListViewArray = ListViewArray<TypeID::LARGE_LIST_VIEW>;
```

### 7. Struct 布局（Struct）
```cpp
// Struct 数组

// 1. 由多个等长的子数组组成，每个子数组对应一个字段
// 2. Struct 的 Null 由自身 null_bitmap 决定，子数组的 Null 独立
// 3. 字段顺序固定，字段名可选（可为空）
// 参考：arrow/array/struct_array.h
struct StructArray : public Array {
  std::vector<Field> fields;               // 字段元信息列表
  std::vector<std::shared_ptr<Array>> children; // 子数组列表（与字段一一对应）

  StructArray() { type_id = TypeID::STRUCT; }

  // 约束：子数组长度必须与 Struct 数组一致
  bool ValidateChildrenLength() const {
    for (const auto& child : children) {
      if (child->length != length) return false;
    }
    return true;
  }

  // 核心方法：获取指定索引的指定字段值
  std::shared_ptr<Array> GetFieldValue(int64_t struct_idx, int64_t field_idx) const {
    if (IsNull(struct_idx)) throw std::runtime_error("Null Struct element");
    auto& child = children[field_idx];
    return SliceArray(child, struct_idx, 1);
  }

private:
  std::shared_ptr<Array> SliceArray(std::shared_ptr<Array> arr, int64_t start, int64_t len) const {
    return arr;
  }
};
```

### 8. Map 布局（Map）
```cpp
// Map 数组

// 1. 本质是「List<Struct<key, value>>」的语法糖
// 2. 核心约束：key 可选唯一（sorted/unique 元数据标记）
// 3. 布局：ListArray 的子数组是 StructArray（包含 key/value 两个字段）
// 参考：arrow/array/map_array.h
struct MapArray : public Array {
  std::shared_ptr<ListArray> entries; // 核心：List<Struct<key, value>>
  bool keys_sorted;                   // key 是否排序
  bool keys_unique;                   // key 是否唯一

  MapArray() { type_id = TypeID::MAP; }

  // 核心方法：获取指定索引的 Map 条目
  std::shared_ptr<StructArray> GetEntries(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null Map element");
    auto slice = entries->GetSlice(idx);
    return std::dynamic_pointer_cast<StructArray>(slice);
  }

  // 辅助方法：获取 key/value 字段
  std::shared_ptr<Array> GetKeys(int64_t idx) const {
    auto entries = GetEntries(idx);
    return entries->children[0]; // key 是 Struct 的第一个字段
  }

  std::shared_ptr<Array> GetValues(int64_t idx) const {
    auto entries = GetEntries(idx);
    return entries->children[1]; // value 是 Struct 的第二个字段
  }
};
```

### 9. Union 布局（Union）
```cpp
// Union 数组（Dense/Sparse）

// 1. DenseUnion：type_ids + offsets 缓冲器，子数组长度适配 offsets
//    - type_ids：每个元素的子数组类型 ID
//    - offsets：每个元素在对应子数组中的索引
// 2. SparseUnion：仅 type_ids 缓冲器，子数组长度与 Union 数组一致
// 参考：arrow/array/union_array.h
struct UnionArray : public Array {
  std::vector<int8_t> type_codes;         // 子数组类型编码（与 children 一一对应）
  std::vector<std::shared_ptr<Array>> children; // 子数组列表
  std::shared_ptr<Buffer> type_ids;       // 元素类型 ID 缓冲器：int8_t[]，长度=length
  std::shared_ptr<Buffer> offsets;        // DenseUnion 专属：int32_t[]，长度=length

  UnionArray(TypeID union_type) { type_id = union_type; }

  // 核心方法：获取指定索引的元素（Dense/Sparse 统一逻辑）
  std::shared_ptr<Array> GetValue(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null Union element");
    const int8_t type_id = type_ids->As<int8_t>()[idx];
    // 查找类型编码对应的子数组
    int64_t child_idx = -1;
    for (int64_t i = 0; i < type_codes.size(); ++i) {
      if (type_codes[i] == type_id) { child_idx = i; break; }
    }
    if (child_idx == -1) throw std::invalid_argument("Unknown type ID");

    auto& child = children[child_idx];
    if (type_id == TypeID::DENSE_UNION) {
      // DenseUnion：通过 offsets 获取子数组索引
      const int32_t offset = offsets->As<int32_t>()[idx];
      return SliceArray(child, offset, 1);
    } else {
      // SparseUnion：直接用 idx 访问子数组
      return SliceArray(child, idx, 1);
    }
  }

private:
  std::shared_ptr<Array> SliceArray(std::shared_ptr<Array> arr, int64_t start, int64_t len) const {
    return arr;
  }
};

// 类型别名
using DenseUnionArray = UnionArray;
using SparseUnionArray = UnionArray;
```

### 10. 字典编码布局（Dictionary Encoded）
```cpp
// 字典编码数组

// 1. 设计目的：优化重复值多的场景，减少内存占用
// 2. 由 indices（索引数组） + dictionary（字典数组）组成
// 3. indices 为 int32/int64 类型，值为 dictionary 的索引
// 4. dictionary 无重复值，可为任意类型（Primitive/Binary/Struct 等）
// 参考：arrow/array/dictionary_array.h
struct DictionaryArray : public Array {
  std::shared_ptr<PrimitiveArray> indices; // 索引数组：int32_t/int64_t 类型
  std::shared_ptr<Array> dictionary;       // 字典数组：存储所有唯一值

  DictionaryArray() { type_id = TypeID::DICTIONARY; }

  // 核心方法：解析索引为原始值
  std::shared_ptr<Array> GetValue(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    // 1. 获取索引值
    const int32_t dict_idx = indices->GetValue<int32_t>(idx);
    // 2. 从字典中获取对应值（零拷贝切片）
    return SliceArray(dictionary, dict_idx, 1);
  }

private:
  std::shared_ptr<Array> SliceArray(std::shared_ptr<Array> arr, int64_t start, int64_t len) const {
    return arr;
  }
};
```

### 11. 运行结束编码布局（Run-End Encoded）
```cpp
// 运行结束编码（Run-End Encoded, RLE）数组

// 1. 设计目的：优化连续重复值场景（如时序/分类数据）
// 2. 由 run_ends + values 组成：
//    - run_ends：递增的结束索引，长度=运行段数
//    - values：每个运行段的对应值，长度=运行段数
// 3. run_ends 必须严格递增，且最后一个值 = 数组总长度
// 参考：arrow/array/run_end_encoded_array.h
template <typename RunEndType = int32_t>
struct RunEndEncodedArray : public Array {
  std::shared_ptr<Buffer> run_ends; // 运行结束缓冲器：RunEndType[]，长度=num_runs
  std::shared_ptr<Array> values;    // 值缓冲器：长度=num_runs

  RunEndEncodedArray() { type_id = TypeID::RUN_END_ENCODED; }

  // 核心方法：查找指定索引所属的运行段（二分查找）
  int64_t FindRunIndex(int64_t idx) const {
    if (idx < 0 || idx >= length) throw std::out_of_range("Index out of bounds");
    const auto* ends = run_ends->As<RunEndType>();
    const int64_t num_runs = run_ends->size / sizeof(RunEndType);

    // 二分查找：找到第一个 run_ends[i] > idx
    int64_t left = 0, right = num_runs - 1;
    while (left < right) {
      const int64_t mid = (left + right) / 2;
      if (ends[mid] > idx) {
        right = mid;
      } else {
        left = mid + 1;
      }
    }
    return left;
  }

  // 核心方法：获取指定索引的原始值
  std::shared_ptr<Array> GetValue(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const int64_t run_idx = FindRunIndex(idx);
    return SliceArray(values, run_idx, 1);
  }

private:
  std::shared_ptr<Array> SliceArray(std::shared_ptr<Array> arr, int64_t start, int64_t len) const {
    return arr;
  }
};

// 类型别名
using RunEndEncodedArray32 = RunEndEncodedArray<int32_t>;
using LargeRunEndEncodedArray = RunEndEncodedArray<int64_t>;
```

### 核心要点总结（对齐文档）
| 布局类型                | 核心设计目的                  | 关键结构                          | 约束/优化点                     |
|-------------------------|-------------------------------|-----------------------------------|-------------------------------------|
| 固定大小 Primitive      | 高效存储固定长度基础类型      | values 缓冲器（连续字节/位）      | bool 用位存储，减少内存占用         |
| 变长 Binary/String      | 存储变长字节/字符串           | offsets + values 缓冲器           | Large 版用 int64 偏移突破 2GB 限制  |
| Binary/String View      | 避免拷贝，快速前缀比较        | views + values 缓冲器             | prefix 存储前 4 字节，加速比较      |
| 变长 List               | 存储变长列表                  | offsets + 子数组                  | 切片零拷贝，复用底层缓冲器           |
| 固定大小 List           | 存储固定长度列表              | element_size + 子数组             | 无 offsets，减少内存开销             |
| List View               | 支持乱序 offsets              | sizes + offsets + 子数组          | 无需推导 List 大小，适配乱序写入    |
| Struct                  | 存储结构化数据                | 多等长子数组 + 字段元信息         | 子数组长度与 Struct 一致            |
| Map                     | 存储键值对                    | List<Struct<key, value>>          | key 可选唯一/排序，优化查找         |
| Union                   | 存储混合类型数据              | type_ids + offsets（Dense）       | Dense 省内存，Sparse 访问更快       |
| 字典编码                | 优化重复值多的场景            | indices + dictionary              | dictionary 无重复值，索引占空间小   |
| 运行结束编码            | 优化连续重复值                | run_ends + values                 | run_ends 递增，二分查找快速定位     |

以上伪代码完全对齐 Arrow 文档的布局定义和 C++ 源码结构，省略了内存池、序列化、类型校验等非核心逻辑，但保留了所有布局的**核心内存结构**和**约束**，可直接作为理解 Arrow 内存模型的参考。