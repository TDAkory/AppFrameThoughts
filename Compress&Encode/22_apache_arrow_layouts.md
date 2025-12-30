# [`Apache Arrow` Layouts](https://arrow.apache.org/docs/format/Intro.html)

本文结合 [Apache Arrow Specifications](https://arrow.apache.org/docs/format/Intro.html) 和 [源代码](https://github.com/apache/arrow/tree/main/cpp/src/arrow)。学习 `Apache Arrow` 提供的数据类型，以及这些类型对应的内存分布。

> [Arrow Columnar Format](https://arrow.apache.org/docs/format/Columnar.html)

## 前置：`Arrow`的基础结构

包括 `Arrow` 最基础的类型枚举、`Buffer`、`Field`、`Array`、`DataType`

### 类型枚举

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

### `Buffer`

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

### `ArrayData`

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

### `Array`

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

### `Field`

`Field`定义了数据结构中的单个字段，包含字段名称、数据类型、可空性和元数据等信息。Field 类在 Arrow 中扮演着至关重要的角色，用于描述表和数组的结构。

```cpp
// arrow/type.h
class ARROW_EXPORT Field : public detail::Fingerprintable,
                           public util::EqualityComparable<Field> {
  std::string name_;                                    // 字段名称
  std::shared_ptr<DataType> type_;                      // 字段数据类型
  bool nullable_;                                       // 字段是否可空
  std::shared_ptr<const KeyValueMetadata> metadata_;    // 字段元数据
};
```

- `detail::Fingerprintable`：提供指纹计算功能，用于高效比较字段
- `util::EqualityComparable<Field>`：提供相等性比较的接口

`Field` 类使用 `KeyValueMetadata` 类存储元数据

```cpp
// arrow/util/key_value_metadata.h
class ARROW_EXPORT KeyValueMetadata {
  std::vector<std::string> keys_;
  std::vector<std::string> values_;
};
```

### `DataType`

`DataType` 定义了数据类型系统的核心接口和功能。它是构建 `Arrow` 类型系统的基础，提供了类型操作、比较、访问和表示的统一接口。

```cpp
// arrow/type.h
class ARROW_EXPORT DataType : public std::enable_shared_from_this<DataType>,
                              public detail::Fingerprintable,
                              public util::EqualityComparable<DataType> {
  
  virtual DataTypeLayout layout() const = 0;

  Type::type id_;          // 类型ID
  FieldVector children_;   // 子字段列表
};
```

- `detail::Fingerprintable`：提供类型指纹计算功能，用于高效比较类型

`DataType` 类使用 `DataTypeLayout` 结构体描述类型的内存布局：

```cpp
struct ARROW_EXPORT DataTypeLayout {
  enum BufferKind { FIXED_WIDTH, VARIABLE_WIDTH, BITMAP, ALWAYS_NULL };

  struct BufferSpec {
    BufferKind kind;
    int64_t byte_width;
  };

  // 类型的缓冲区布局
  std::vector<BufferSpec> buffers;
  bool has_dictionary = false;
  std::optional<BufferSpec> variadic_spec;
};
```

`TypeHolder` 是一个辅助结构体，用于安全地持有 DataType 指针：

```cpp
struct ARROW_EXPORT TypeHolder {
  const DataType* type = NULLPTR;
  std::shared_ptr<DataType> owned_type;
};
```

DataType 类有多个直接子类，构成了 Arrow 的类型系统架构：

1. **FixedWidthType**：固定宽度类型的基类，如整数、布尔值等
2. **NestedType**：嵌套类型的基类，如列表、结构体、联合等
3. **BaseBinaryType**：二进制类型的基类，如二进制、字符串等
4. **NullType**：空类型
5. **ExtensionType**：扩展类型的基类（用户自定义类型）

整体的类型体系如下：

```shell
DataType (抽象基类)
├── FixedWidthType (固定宽度类型基类)
│   ├── Int8Type, Int16Type, Int32Type, Int64Type
│   ├── UInt8Type, UInt16Type, UInt32Type, UInt64Type
│   ├── FloatType, DoubleType
│   ├── BooleanType
│   ├── Date32Type, Date64Type
│   ├── Time32Type, Time64Type
│   ├── TimestampType
│   └── Decimal128Type, Decimal256Type
├── NestedType (嵌套类型基类)
│   ├── ListType, LargeListType
│   ├── FixedSizeListType
│   ├── StructType
│   ├── UnionType (SparseUnionType, DenseUnionType)
│   └── MapType
├── BaseBinaryType (二进制类型基类)
│   ├── BinaryType, LargeBinaryType
│   └── StringType, LargeStringType
├── NullType (空类型)
└── ExtensionType (扩展类型基类)
```

## 1. 固定大小 Primitive 布局（Fixed Size Primitive）

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

`Int32 Array` 布局示例：

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

没有 `null` 元素的 `Array`，可能省略 `Validity bitmap`

```shell
# [1, 2, 3, 4, 8]
* Length 5, Null count: 0
* Validity bitmap buffer: Not required
* Value Buffer:

  | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | bytes 12-15 | bytes 16-19 | Bytes 20-63           |
  |-------------|-------------|-------------|-------------|-------------|-----------------------|
  | 1           | 2           | 3           | 4           | 8           | unspecified (padding) |
```

## 2. 变长 Binary/String 布局（Variable Length Binary/String）

`BinaryType` 是 `Arrow` 中用于表示可变大小二进制数据的核心类型类，定义在 `array_binary.h` 文件中，继承自 `BaseBinaryType`

```cpp
// arrow/array/array_binary.h
// Base class for variable-sized binary arrays, regardless of offset size
// and logical interpretation.
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

template <typename TYPE>
class BaseBinaryArray : public FlatArray {
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

``VarBinary``布局示例：

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

## 3. 变长 Binary/String View 布局（Binary/String View）

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

class ARROW_EXPORT BinaryViewArray : public FlatArray {
  const c_type* raw_values_;
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

继承关系

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
class ARROW_EXPORT BinaryViewType : public DataType {
  ……
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
};
```

布局示例:

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

## 4. 变长 List 布局（Variable Length List）

`ListArray` 是 `Arrow` 中用于表示可变大小列表数据的核心数组类型，是 `Arrow` 处理复杂数据类型的基础组件之一。

```cpp
// 变长 List 数组
// arrow/array/array_nested.h
class ARROW_EXPORT ListType : public BaseListType {
 public:
  static constexpr Type::type type_id = Type::LIST;
  using offset_type = int32_t;

  DataTypeLayout layout() const override {
    return DataTypeLayout(
        {DataTypeLayout::Bitmap(), DataTypeLayout::FixedWidth(sizeof(offset_type))});
  }
};

template <typename TYPE>
class VarLengthListLikeArray : public Array {
  const TypeClass* list_type_ = NULLPTR;
  std::shared_ptr<Array> values_;
  const offset_type* raw_value_offsets_ = NULLPTR;
};

template <typename TYPE>
class BaseListArray : public VarLengthListLikeArray<TYPE> {
};

class ARROW_EXPORT ListArray : public BaseListArray<ListType> {
 public:
  explicit ListArray(std::shared_ptr<ArrayData> data);

  ListArray(std::shared_ptr<DataType> type, int64_t length,
            std::shared_ptr<Buffer> value_offsets, std::shared_ptr<Array> values,
            std::shared_ptr<Buffer> null_bitmap = NULLPTR,
            int64_t null_count = kUnknownNullCount, int64_t offset = 0);
};
```

`ListArray` 采用偏移量+数据的经典列表存储设计：支持 `O(1)` 时间复杂度获取任意列表的起始位置和长度

```shell
# an example of List<Int8> with length 4 having values
# [[12, -7, 25], null, [0, -127, 127, 50], []]

* Length: 4, Null count: 1
* Validity bitmap buffer:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001101                 | 0 (padding)           |

* Offsets buffer (int32) # offsets[i] 表示第 i 个列表在数据区的起始位置，offsets[i+1] 表示结束位置

  | Bytes 0-3  | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63           |
  |------------|-------------|-------------|-------------|-------------|-----------------------|
  | 0          | 3           | 3           | 7           | 7           | unspecified (padding) |

* Values array (Int8Array): # 连续存储所有列表的元素数据
  * Length: 7,  Null count: 0
  * Validity bitmap buffer: Not required
  * Values buffer (int8)

    | Bytes 0-6                    | Bytes 7-63            |
    |------------------------------|-----------------------|
    | 12, -7, 25, 0, -127, 127, 50 | unspecified (padding) |
```

类似的，也有使用 64-bit offset 的派生：`class ARROW_EXPORT LargeListArray : public BaseListArray<LargeListType>`

| 特性 | ListArray | LargeListArray |
|------|-----------|----------------|
| 偏移量类型 | int32_t | int64_t |
| 最大列表长度 | 2GB | 9EB |
| 内存占用 | 较小（4字节/偏移量） | 较大（8字节/偏移量） |
| 适用场景 | 大多数列表数据 | 超大列表数据 |
| 类型ID | Type::LIST | Type::LARGE_LIST |

## 5. List View 布局（List View）

`ListViewArray` 允许创建对现有值数组中任意片段的视图，而无需复制数据

```cpp
// arrow/array/array_nested.h
class ARROW_EXPORT ListViewType : public BaseListType {
 public:
  static constexpr Type::type type_id = Type::LIST_VIEW;
  using offset_type = int32_t;
  // ...
  DataTypeLayout layout() const override {
    return DataTypeLayout({DataTypeLayout::Bitmap(),  // 空值位图
                           DataTypeLayout::FixedWidth(sizeof(offset_type)),  // 偏移量数组
                           DataTypeLayout::FixedWidth(sizeof(offset_type))});  // 大小数组
  }
};

template <typename TYPE>
class VarLengthListLikeArray : public Array {
  const TypeClass* list_type_ = NULLPTR;
  std::shared_ptr<Array> values_;
  const offset_type* raw_value_offsets_ = NULLPTR;
};

template <typename TYPE>
class BaseListViewArray : public VarLengthListLikeArray<TYPE> {
  const offset_type* raw_value_sizes_ = NULLPTR;
};

class ARROW_EXPORT ListViewArray : public BaseListViewArray<ListViewType> {
 public:
  // 构造函数
  explicit ListViewArray(std::shared_ptr<ArrayData> data);
  ListViewArray(std::shared_ptr<DataType> type, int64_t length,
                std::shared_ptr<Buffer> value_offsets,
                std::shared_ptr<Buffer> value_sizes,
                std::shared_ptr<Array> values,
                std::shared_ptr<Buffer> null_bitmap = NULLPTR,
                int64_t null_count = kUnknownNullCount,
                int64_t offset = 0);

  // 工厂方法
  static Result<std::shared_ptr<ListViewArray>> FromArrays(...);
  static Result<std::shared_ptr<ListViewArray>> FromList(const ListArray& list_array, MemoryPool* pool);
};
```

所有 `list-view `的成员, 包括 `null values`, 必须满足如下关系:

```shell
0 <= offsets[i] <= length of the child array
0 <= offsets[i] + size[i] <= length of the child array
```

布局示例：

```shell
# illustrates out of order offsets and sharing of child array values.
# [[12, -7, 25], null, [0, -127, 127, 50], [], [50, 12]]

* Length: 5, Null count: 1
* Validity bitmap buffer:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00011101                 | 0 (padding)           |

* Offsets buffer (int32)

  | Bytes 0-3  | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63           |
  |------------|-------------|-------------|-------------|-------------|-----------------------|
  | 4          | 7           | 0           | 0           | 3           | unspecified (padding) |

* Sizes buffer (int32)

  | Bytes 0-3  | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63           |
  |------------|-------------|-------------|-------------|-------------|-----------------------|
  | 3          | 0           | 4           | 0           | 2           | unspecified (padding) |

* Values array (Int8Array):
  * Length: 7,  Null count: 0
  * Validity bitmap buffer: Not required
  * Values buffer (int8)

    | Bytes 0-6                    | Bytes 7-63            |
    |------------------------------|-----------------------|
    | 0, -127, 127, 50, 12, -7, 25 | unspecified (padding) |
```

与 `ListArray` 的对比的灵活性优势：

**非连续片段引用**

```shell
值数组: [A, B, C, D, E, F, G, H, I, J]
ListArray偏移量: [0, 3, 5, 7, 10]  // 只能表示连续列表 [A,B,C], [D,E], [F,G], [H,I,J]

ListViewArray:
偏移量: [0, 2, 5, 1]  // 可以表示不连续列表 [A,B,C], [C,D], [F,G,H], [B,C,D]
大小:   [3, 2, 3, 3]
```

**重叠片段引用**

```shell
值数组: [X, Y, Z]
ListViewArray:
偏移量: [0, 0, 1]  // 允许列表重叠
大小:   [2, 3, 2]
结果列表: [X,Y], [X,Y,Z], [Y,Z]
```

**任意顺序引用**

```shell
值数组: [1, 2, 3, 4, 5]
ListViewArray:
偏移量: [3, 0, 4, 1]  // 可以按任意顺序引用值数组
大小:   [2, 2, 1, 3]
结果列表: [4,5], [1,2], [5], [2,3,4]
```

| 特性 | ListArray | ListViewArray |
|------|-----------|---------------|
| 内存效率 | 存储 `n+1` 个偏移量 | 存储 `2n` 个值（偏移量+大小） |
| 计算效率 | 需减法计算长度 | 直接获取长度 |
| 连续性要求 | 严格要求连续存储 | 无连续性要求 |
| 重叠能力 | 不支持 | 支持 |
| 灵活性 | 低 | 高 |
| 内存复用 | 有限 | 高 |

## 6. 定长列表 FixedSizeListArray

`FixedSizeListArray` 是 `Arrow` 中用于高效表示固定长度列表数据的核心数组类型。它针对所有列表元素具有相同长度的场景进行了优化，提供了更紧凑的内存布局和更高效的访问方式。

```cpp
class ARROW_EXPORT FixedSizeListType : public BaseListType {
 public:
  static constexpr Type::type type_id = Type::FIXED_SIZE_LIST;
  using offset_type = int64_t;  // 使用64位偏移量支持大数据集
  
  // 核心属性
  int32_t list_size() const { return list_size_; }  // 每个列表的固定大小
  
  // 内存布局：只有空值位图，没有偏移量数组
  DataTypeLayout layout() const override {
    return DataTypeLayout({DataTypeLayout::Bitmap()});
  }
};

class ARROW_EXPORT FixedSizeListArray : public Array {
  using TypeClass = FixedSizeListType;
  using offset_type = TypeClass::offset_type;

  int32_t list_size_;
  std::shared_ptr<Array> values_;
}
```

`FixedSizeListArray` 采用了极简的内存布局：

1. **空值位图**：标记哪些列表元素是 NULL
2. **值数组**：存储所有列表的元素数据（作为连续的子数组）

**关键优化**：由于每个列表的长度固定，不需要存储偏移量数组。偏移量可以通过简单的计算得出：

- 列表 i 的起始位置 = `i * list_size`
- 列表 i 的结束位置 = `(i + 1) * list_size`

这种设计比传统 ListArray 节省了约 25% 的内存（无需存储 n+1 个偏移量）。

```shell
# For an array of length 4 with respective values:
# [[192, 168, 0, 12], null, [192, 168, 0, 25], [192, 168, 0, 1]]

* Length: 4, Null count: 1
* Validity bitmap buffer:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001101                 | 0 (padding)           |

* Values array (byte array):
  * Length: 16,  Null count: 0
  * validity bitmap buffer: Not required

    | Bytes 0-3       | Bytes 4-7   | Bytes 8-15                      |
    |-----------------|-------------|---------------------------------|
    | 192, 168, 0, 12 | unspecified | 192, 168, 0, 25, 192, 168, 0, 1 |
```

`FixedSizeListArray` 提供了与其他列表类型的转换能力，特别是通过 `Flatten` 方法可以轻松转换为扁平数组。

```cpp
// 将固定大小列表数组扁平化为值数组
Result<std::shared_ptr<Array>> Flatten(
    MemoryPool* memory_pool = default_memory_pool()) const;

// 递归扁平化嵌套列表
Result<std::shared_ptr<Array>> FlattenRecursively(
    MemoryPool* memory_pool = default_memory_pool()) const;
```

## 7. Struct 布局（Struct）

StructArray 是 Apache Arrow 中用于表示结构化数据的核心数组类型，它允许将多个不同类型的数组组合成一个单一的复合数据结构

```cpp
// arrow/array/array_nested.h
class ARROW_EXPORT StructType : public NestedType {
 public:
  static constexpr Type::type type_id = Type::STRUCT;
  
  // 内存布局：只有空值位图
  DataTypeLayout layout() const override {
    return DataTypeLayout({DataTypeLayout::Bitmap()});
  }
};

struct StructArray : public Array {
  StructArray() { type_id = TypeID::STRUCT; }

  StructArray(const std::shared_ptr<DataType>& type, int64_t length,
              const std::vector<std::shared_ptr<Array>>& children,
              std::shared_ptr<Buffer> null_bitmap = NULLPTR,
              int64_t null_count = kUnknownNullCount, int64_t offset = 0);

  std::unique_ptr<Impl> impl_;
};

struct StructArray::Impl {
  mutable ArrayVector boxed_fields_;
};

using ArrayVector = std::vector<std::shared_ptr<Array>>;
```

StructArray 采用了高效的**列式存储**内存布局：

1. **空值位图**：标记哪些结构体元素是 NULL
2. **子数组**：每个字段对应一个独立的数组，存储该字段的所有值

**关键设计**：结构体的各个字段数据是分开存储的，而不是按行连续存储。这种列式存储方式提供了以下优势：

- 只访问需要的字段，减少 I/O 和缓存使用
- 相同类型的数据连续存储，提高压缩率
- 支持向量化操作，提高计算效率

```shell
Struct <
  name: VarBinary
  age: Int32
>

# Upper struct has two child arrays, one VarBinary array (using variable-size binary layout) 
# and one 4-byte primitive value array having Int32 logical type

# The layout for [{'joe', 1}, {null, 2}, null, {'mark', 4}], would be
* Length: 4, Null count: 1
* Validity bitmap buffer:

  | Byte 0 (validity bitmap) | Bytes 1-63            |
  |--------------------------|-----------------------|
  | 00001011                 | 0 (padding)           |

* Children arrays:
  * field-0 array (`VarBinary`):
    * Length: 4, Null count: 1
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001101                 | 0 (padding)           |

    * Offsets buffer:

      | Bytes 0-19     | Bytes 20-63           |
      |----------------|-----------------------|
      | 0, 3, 3, 8, 12 | unspecified (padding) |

     * Value buffer:

      | Bytes 0-11     | Bytes 12-63           |
      |----------------|-----------------------|
      | joealicemark   | unspecified (padding) |

  * field-1 array (int32 array):
    * Length: 4, Null count: 1
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001011                 | 0 (padding)           |

    * Value Buffer:

      | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-63           |
      |-------------|-------------|-------------|-------------|-----------------------|
      | 1           | 2           | unspecified | 4           | unspecified (padding) |
```

StructArray 的空值处理有几个重要特点：

1. **结构体级空值**：通过顶层空值位图标记整个结构体是否为 NULL
2. **字段级空值**：每个字段数组可以有自己的空值位图
3. **合并规则**：如果结构体是 NULL，那么其所有字段都被视为 NULL，无论字段本身的空值状态如何        

## 8. Map 布局（Map）

`MapArray` 继承自 `ListArray`：`MapArray` 利用 `ListArray` 的列表结构存储映射数据

```cpp
// arrow/array/array_nested.h
class ARROW_EXPORT MapType : public ListType {
  static constexpr Type::type type_id = Type::MAP;

  bool keys_sorted_;  // 键是否已排序的标志
};

class ARROW_EXPORT MapArray : public ListArray {
 public:
  using TypeClass = MapType;
  
  MapArray(const std::shared_ptr<DataType>& type, int64_t length,
         const std::shared_ptr<Buffer>& value_offsets,
         const std::shared_ptr<Array>& keys, const std::shared_ptr<Array>& items,
         const std::shared_ptr<Buffer>& null_bitmap = NULLPTR,
         int64_t null_count = kUnknownNullCount, int64_t offset = 0);
};
```

- `null_bitmap`：可选的位图，用于标记哪些映射值是 null
- `value_offsets`：整数数组，存储每个映射的键值对序列在 entries 数组中的起始和结束位置
- `entries`：一个结构体数组，包含两个子数组：
  - `keys`：所有映射的键的平面数组
  - `values`：所有映射的值的平面数组

```shell
MapArray:
┌─────────────────────┐
│ null_bitmap (可选)   │  # 映射值的空值位图
├─────────────────────┤
│ value_offsets       │  # 每个映射的键值对序列在平面数组中的偏移量
├─────────────────────┤
│ entries (结构体数组) │  # 包含所有映射的键值对的平面数组
│ ┌─────────────────┐ │
│ │ keys            │ │  # 所有映射的键的平面数组
│ ├─────────────────┤ │
│ │ values          │ │  # 所有映射的值的平面数组
│ └─────────────────┘ │
└─────────────────────┘
```

简单使用示例：

```cpp
auto key_type = arrow::string();
auto value_type = arrow::int32();

auto map_type = arrow::map(key_type, value_type, false);  // 键未排序

// 创建偏移量数组（表示两个映射：第一个有2个键值对，第二个有1个键值对）
auto offsets = arrow::ArrayFromJSON(arrow::int32(), "[0, 2, 3]");

// 创建键数组
auto keys = arrow::ArrayFromJSON(arrow::string(), R"(["key1", "key2", "key3"])");

// 创建值数组
auto values = arrow::ArrayFromJSON(arrow::int32(), "[1, 2, 3]");

// 创建 MapArray
auto map_array = std::make_shared<arrow::MapArray>(
    map_type, 2, std::static_pointer_cast<arrow::Int32Array>(offsets)->data()->buffers[1],
    keys, values);

std::cout << "Number of maps: " << map_array->length() << std::endl;
std::cout << "First map has " << map_array->value_length(0) << " key-value pairs" << std::endl;
std::cout << "Second map has " << map_array->value_length(1) << " key-value pairs" << std::endl;

// 访问第一个映射的键值对
int64_t start = map_array->value_offsets()->Value(0);
int64_t end = map_array->value_offsets()->Value(1);
std::cout << "First map key-value pairs:" << std::endl;
for (int64_t j = start; j < end; ++j) {
  auto key = std::static_pointer_cast<arrow::StringArray>(map_array->keys())->GetString(j);
  auto value = std::static_pointer_cast<arrow::Int32Array>(map_array->items())->Value(j);
  std::cout << "  " << key << ": " << value << std::endl;
}
```

输出结果：

```shell
Number of maps: 2
First map has 2 key-value pairs
Second map has 1 key-value pairs
First map key-value pairs:
  key1: 1
  key2: 2
```

## 9. Union 布局（Union）

```cpp
class ARROW_EXPORT UnionType : public NestedType {
 public:
  static constexpr int8_t kMaxTypeCode = 127;
  static constexpr int kInvalidChildId = -1;
  
  // 创建联合类型的工厂方法
  static Result<std::shared_ptr<DataType>> Make(
      const FieldVector& fields,
      const std::vector<int8_t>& type_codes,
      UnionMode::type mode = UnionMode::SPARSE);
  
  // 成员变量
  std::vector<int8_t> type_codes_;  // 类型代码列表
  std::vector<int> child_ids_;      // 子ID映射
};

class ARROW_EXPORT UnionArray : public Array {
 public:
  using TypeClass = UnionType;
  
  // 成员变量
  ArrayVector children_;                // 子数组列表
  std::shared_ptr<Array> type_codes_;   // 类型代码数组
  const int8_t* raw_type_codes_;        // 原始类型代码指针
  const UnionType* union_type_;         // 联合类型
};
```

Arrow 中的联合类型有两种主要实现模式：

- 稀疏联合（Sparse Union）：SparseUnionArray
- 密集联合（Dense Union）：DenseUnionArray

两者具有不同的内存分布

```cpp
DataTypeLayout UnionType::layout() const {
  if (mode() == UnionMode::SPARSE) {
    return DataTypeLayout(
        {DataTypeLayout::AlwaysNull(), DataTypeLayout::FixedWidth(sizeof(uint8_t))});
  } else {
    return DataTypeLayout({DataTypeLayout::AlwaysNull(),
                           DataTypeLayout::FixedWidth(sizeof(uint8_t)),
                           DataTypeLayout::FixedWidth(sizeof(int32_t))});
  }
}
```

### Sparse Union

```cpp
class ARROW_EXPORT SparseUnionArray : public UnionArray {
 public:
  using TypeClass = SparseUnionType;
  
  // 创建稀疏联合数组的工厂方法
  static Result<std::shared_ptr<SparseUnionArray>> Make(
      const ArrayVector& children,
      const std::shared_ptr<Array>& type_codes,
      const std::shared_ptr<DataType>& type);
};
```

稀疏联合类型由以下两个核心部分组成：

1. 子数组（Child Arrays）：为联合类型中的每种可能类型维护一个专用的子数组；每个子数组的长度与联合数组完全相同；存储在 `ArrayData::child_data` 中，通过 `children_` 向量访问；未使用的位置会被填充默认值（如0或空字符串）
2. 类型缓冲区（Types Buffer）：8位有符号整数（int8_t）缓冲区，存储每个联合元素的类型ID；存储在 `ArrayData::buffers[1]` 中；通过 `raw_type_codes_` 指针直接访问；与密集联合相同，类型代码最大值为127

```shell
# SparseUnion<i: Int32, f: Float32, s: VarBinary>
# [{i=5}, {f=1.2}, {s='joe'}, {f=3.4}, {i=4}, {s='mark'}]

* Length: 6, Null count: 0
* Types buffer:

 | Byte 0     | Byte 1      | Byte 2      | Byte 3      | Byte 4      | Byte 5       | Bytes  6-63           |
 |------------|-------------|-------------|-------------|-------------|--------------|-----------------------|
 | 0          | 1           | 2           | 1           | 0           | 2            | unspecified (padding) |

* Children arrays:

  * i (Int32):
    * Length: 6, Null count: 4
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00010001                 | 0 (padding)           |

    * Value buffer:

      | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-23  | Bytes 24-63           |
      |-------------|-------------|-------------|-------------|-------------|--------------|-----------------------|
      | 5           | unspecified | unspecified | unspecified | 4           |  unspecified | unspecified (padding) |

  * f (Float32):
    * Length: 6, Null count: 4
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001010                 | 0 (padding)           |

    * Value buffer:

      | Bytes 0-3    | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-23 | Bytes 24-63           |
      |--------------|-------------|-------------|-------------|-------------|-------------|-----------------------|
      | unspecified  | 1.2         | unspecified | 3.4         | unspecified | unspecified | unspecified (padding) |

  * s (`VarBinary`)
    * Length: 6, Null count: 4
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00100100                 | 0 (padding)           |

    * Offsets buffer (Int32)

      | Bytes 0-3  | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-23 | Bytes 24-27 | Bytes 28-63            |
      |------------|-------------|-------------|-------------|-------------|-------------|-------------|------------------------|
      | 0          | 0           | 0           | 3           | 3           | 3           | 7           | unspecified (padding)  |

    * Values buffer:

      | Bytes 0-6  | Bytes 7-63            |
      |------------|-----------------------|
      | joemark    | unspecified (padding) |
```

### Dense Union

```cpp
class ARROW_EXPORT DenseUnionArray : public UnionArray {
 public:
  using TypeClass = DenseUnionType;
  
  // 创建密集联合数组的工厂方法
  static Result<std::shared_ptr<Array>> Make(
      const Array& type_ids,
      const Array& value_offsets,
      ArrayVector children,
      std::vector<type_code_t> type_codes);
  
  const int32_t* raw_value_offsets_;
};
```

密集联合类型由以下三个核心部分组成：

1. 子数组（Child Arrays）：为联合类型中的每种可能类型维护一个专用的子数组；存储在 `ArrayData::child_data` 中，通过 `children_` 向量访问；只包含实际使用的元素，不包含未使用的填充值，内存效率高
2. 类型缓冲区（Types Buffer）：8位有符号整数（int8_t）缓冲区，存储每个联合元素的类型ID；存储在 `ArrayData::buffers[1]` 中；通过 `raw_type_codes_` 指针直接访问；类型代码最大值为127（`UnionType::kMaxTypeCode`），超过此限制可使用联合的联合
3. 偏移量缓冲区（Offsets Buffer）：32位有符号整数（int32_t）缓冲区，指示每个联合元素在相应子数组中的相对位置；存储在 `ArrayData::buffers[2]` 中；通过 `raw_value_offsets_` 指针直接访问；每个子数组的偏移量必须按顺序递增

```shell
# DenseUnion<f: Float32, i: Int32>
# [{f=1.2}, null, {f=3.4}, {i=5}]

* Length: 4, Null count: 0
* Types buffer:

  | Byte 0   | Byte 1      | Byte 2   | Byte 3   | Bytes 4-63            |
  |----------|-------------|----------|----------|-----------------------|
  | 0        | 0           | 0        | 1        | unspecified (padding) |

* Offset buffer:

  | Bytes 0-3 | Bytes 4-7   | Bytes 8-11 | Bytes 12-15 | Bytes 16-63           |
  |-----------|-------------|------------|-------------|-----------------------|
  | 0         | 1           | 2          | 0           | unspecified (padding) |

* Children arrays:
  * Field-0 array (f: Float32):
    * Length: 3, Null count: 1
    * Validity bitmap buffer: 00000101

    * Value Buffer:

      | Bytes 0-11     | Bytes 12-63           |
      |----------------|-----------------------|
      | 1.2, null, 3.4 | unspecified (padding) |


  * Field-1 array (i: Int32):
    * Length: 1, Null count: 0
    * Validity bitmap buffer: Not required

    * Value Buffer:

      | Bytes 0-3 | Bytes 4-63            |
      |-----------|-----------------------|
      | 5         | unspecified (padding) |
```

### 两者的对比

| **特性** | **密集联合（Dense Union）** | **稀疏联合（Sparse Union）** |
|---------|---------------------------|---------------------------|
| **核心结构** | - 子数组<br>- 类型缓冲区<br>- **偏移量缓冲区** | - 子数组<br>- 类型缓冲区<br>- **无偏移量缓冲区** |
| **子数组长度** | 仅包含实际使用的元素，长度通常小于联合数组 | 与联合数组长度完全相同 |
| **内存效率** | 高，仅存储实际使用的元素 | 低，未使用位置填充默认值，存在内存浪费 |
| **类型代码限制** | 最多127种（可通过联合的联合扩展） | 最多127种（可通过联合的联合扩展） |
| **访问复杂度** | O(1)，但需要额外的偏移量查找 | O(1)，直接索引访问，更简单 |
| **向量化支持** | 有限，需要处理偏移量映射 | 优秀，子数组与联合数组等长，支持高效向量化操作 |
| **数据转换** | 需要重新组织数据结构 | 简单，仅需定义类型数组即可 |
| **实现复杂度** | 较高，需要管理偏移量数组 | 较低，结构更简单 |
| **扩展性** | 良好，适用于类型分布不均的大数据集 | 一般，类型分布均匀时更高效 |
| **空值处理** | 支持，通过类型代码和偏移量的组合 | 支持，通过类型代码和子数组空值的组合 |
| **优势** | 1. 内存使用效率高<br>2. 适用于类型分布不均的数据集<br>3. 存储大量元素时更经济 | 1. 向量化表达式评估更友好<br>2. 数据转换简单<br>3. 实现和调试容易<br>4. 子数组可独立使用 |
| **劣势** | 1. 访问需要额外的偏移量计算<br>2. 实现更复杂<br>3. 向量化支持有限 | 1. 内存使用效率低<br>2. 类型分布不均时浪费严重<br>3. 存储大量元素时成本高 |
| **适用场景** | 1. 大数据集<br>2. 类型分布严重不均<br>3. 内存敏感型应用<br>4. 存储成本优先的场景 | 1. 向量化计算密集型任务<br>2. 简单数据转换<br>3. 开发和调试阶段<br>4. 类型分布均匀的数据集<br>5. 计算性能优先的场景 |

## 10. 字典编码布局（Dictionary Encoded）

`DictionaryArray` 是 `Arrow` 中用于实现**字典编码**（Dictionary Encoding）的数据结构。字典编码是一种内存高效的数据表示方法，通过将重复值映射到整数索引来减少内存使用。

例如，字符串数组 `["foo", "bar", "foo", "bar", "foo", "bar"]` 可以编码为：

- 字典：`["bar", "foo"]`
- 索引：`[1, 0, 1, 0, 1, 0]`

```cpp
class ARROW_EXPORT DictionaryType : public FixedWidthType {
 public:
  static constexpr Type::type type_id = Type::DICTIONARY;

  DataTypeLayout DictionaryType::layout() const {
    auto layout = index_type_->layout();
    layout.has_dictionary = true;
    return layout;
  }

  std::shared_ptr<DataType> index_type_;    // 索引类型（必须是整数类型）
  std::shared_ptr<DataType> value_type_;    // 字典值类型
  bool ordered_;                            // 字典是否有序
};

class ARROW_EXPORT DictionaryArray : public Array {
 public:
  using TypeClass = DictionaryType;
  
  // 构造函数
  DictionaryArray(const std::shared_ptr<ArrayData>& data);
  DictionaryArray(const std::shared_ptr<DataType>& type,
                  const std::shared_ptr<Array>& indices,
                  const std::shared_ptr<Array>& dictionary);

  const DictionaryType* dict_type_;        // 字典类型
  std::shared_ptr<Array> indices_;         // 索引数组
  
  // 延迟初始化的字典数组
  mutable std::shared_ptr<Array> dictionary_;
};
```

![DictionaryArray内存布局](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/AppFrameThoughts/arrow_dictionary_array.svg)

字典统一化（Dictionary Unification）是将不同字典编码的数组转换为使用相同字典的过程，以便进行比较或联合操作。

DictionaryUnifier 类提供了增量字典统一化功能：

```cpp
class ARROW_EXPORT DictionaryUnifier {
 public:
  // 创建统一器
  static Result<std::unique_ptr<DictionaryUnifier>> Make(
      std::shared_ptr<DataType> value_type, MemoryPool* pool = default_memory_pool());

  // 添加字典进行统一
  virtual Status Unify(const Array& dictionary) = 0;

  // 添加字典并获取转置映射
  virtual Status Unify(const Array& dictionary, std::shared_ptr<Buffer>* out_transpose) = 0;

  // 获取统一后的字典和类型
  virtual Status GetResult(std::shared_ptr<DataType>* out_type,
                           std::shared_ptr<Array>* out_dict) = 0;
};
```

## 11.Run-End Encoded

`RunEndEncodedArray`（简称REE数组）是 `Arrow` 中用于实现**Run-End Encoding（REE，运行结束编码）**的数据结构，是一种高效的压缩数组类型，特别适用于包含大量连续重复值的数据。

```cpp
// arrow/array/array_run_end.h
class ARROW_EXPORT RunEndEncodedType : public NestedType {
public:
  static constexpr Type::type type_id = Type::RUN_END_ENCODED;
  explicit RunEndEncodedType(std::shared_ptr<DataType> run_end_type,
                             std::shared_ptr<DataType> value_type);
  // ...
};

class ARROW_EXPORT RunEndEncodedArray : public Array {
private:
  std::shared_ptr<Array> run_ends_array_;   // 指定run结束位置的类型，必须是int16、int32或int64
  std::shared_ptr<Array> values_array_;     // 指定run值的类型，可以是任何Arrow数据类型

public:
  using TypeClass = RunEndEncodedType;
  // ...
};
```

- 不直接存储值的完整序列，而是存储"值-计数"对的压缩形式
- 使用逻辑长度和偏移量，与物理存储分离
- 支持极大的逻辑数组，即使物理存储很小

```shell
# type: Float32
# [1.0, 1.0, 1.0, 1.0, null, null, 2.0]

* Length: 7, Null count: 0
* Child Arrays:

  * run_ends (Int32):
    * Length: 3, Null count: 0 (Run Ends cannot be null)
    * Validity bitmap buffer: Not required (if it exists, it should be all 1s)
    * Values buffer

      | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-63           |
      |-------------|-------------|-------------|-----------------------|
      | 4           | 6           | 7           | unspecified (padding) |

  * values (Float32):
    * Length: 3, Null count: 1
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00000101                 | 0 (padding)           |

    * Values buffer

      | Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-63           |
      |-------------|-------------|-------------|-----------------------|
      | 1.0         | unspecified | 2.0         | unspecified (padding) |
```

## 总结

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
