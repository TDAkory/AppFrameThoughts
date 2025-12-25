
### 前置定义：Arrow 官方核心基础结构
先定义 Arrow 官方最基础的类型枚举、Buffer、Field、Array 基类，完全对齐 `arrow/array/array.h` `arrow/buffer.h` 的核心结构：
```cpp
#include <cstdint>
#include <vector>
#include <memory>
#include <string>
#include <optional>
#include <stdexcept>

// 对齐官方：Arrow 数据类型枚举（仅保留核心类型）
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

// 对齐官方：Arrow 连续内存缓冲器（Buffer）
// 官方要点：
// 1. 只读语义：构建后不修改，零拷贝共享的核心
// 2. 内存由 MemoryPool 管理（伪代码简化为 raw 指针）
// 3. 支持空缓冲器（data=nullptr, size=0）
// 参考：arrow/buffer.h
struct Buffer {
  const uint8_t* data;    // 缓冲器起始地址（const 保证只读）
  int64_t size;           // 缓冲器字节长度（负数非法）

  // 官方辅助方法：类型安全解析缓冲器数据
  template <typename T>
  const T* As() const {
    if (size % sizeof(T) != 0) {
      throw std::invalid_argument("Buffer size not aligned with type");
    }
    return reinterpret_cast<const T*>(data);
  }

  // 官方约束：空缓冲器判断
  bool IsEmpty() const { return data == nullptr || size == 0; }
};

// 对齐官方：字段元信息（Field）
// 官方要点：描述 Struct/Map/Union 的字段属性
// 参考：arrow/type.h
struct Field {
  std::string name;       // 字段名（可选，可为空）
  TypeID type;            // 字段数据类型
  bool nullable;          // 是否允许 Null 值
  std::shared_ptr<Buffer> metadata; // 扩展元数据（可选）
};

// 对齐官方：Arrow 一维数组基类（Array）
// 官方要点：
// 1. 所有具体数组类型的父类，统一抽象长度、Null 管理
// 2. null_bitmap 为空时表示无 Null 值（null_count=0）
// 3. 核心属性：length（元素数）、null_count（Null 数）、null_bitmap（有效性位图）
// 参考：arrow/array/array.h
struct Array {
  TypeID type_id;         // 数组数据类型
  int64_t length;         // 数组总元素数
  int64_t null_count;     // Null 值数量（-1 表示未计算）
  std::shared_ptr<Buffer> null_bitmap; // 有效性位图：1bit/元素，1=有效，0=Null

  // 官方核心方法：判断指定索引是否为 Null
  bool IsNull(int64_t idx) const {
    if (idx < 0 || idx >= length) throw std::out_of_range("Index out of bounds");
    if (null_bitmap->IsEmpty()) return false; // 无位图 → 无 Null
    // 位图存储规则：字节序为小端，bit 0 对应第一个元素
    const int64_t byte_idx = idx / 8;
    const int64_t bit_idx = idx % 8;
    return (null_bitmap->As<uint8_t>()[byte_idx] & (1 << bit_idx)) == 0;
  }

  // 虚析构：支持多态
  virtual ~Array() = default;
};
```

### 1. 固定大小 Primitive 布局（Fixed Size Primitive）
```cpp
// 对齐官方：固定大小 Primitive 数组
// 官方要点：
// 1. 元素字节长度固定（如 int32=4B，float64=8B，bool=1bit）
// 2. bool 类型特殊：值以位存储（而非字节），单独处理
// 3. values 缓冲器字节长度 = length * 元素字节数（bool 为 ceil(length/8)）
// 参考：arrow/array/primitive_array.h
template <TypeID PrimitiveType>
struct PrimitiveArray : public Array {
  std::shared_ptr<Buffer> values; // 值缓冲器：连续存储所有元素

  // 元素字节长度（官方预定义）
  static constexpr int64_t ElementSize() {
    switch (PrimitiveType) {
      case TypeID::INT8: case TypeID::UINT8: return 1;
      case TypeID::INT16: case TypeID::UINT16: return 2;
      case TypeID::INT32: case TypeID::UINT32: case TypeID::FLOAT32: return 4;
      case TypeID::INT64: case TypeID::UINT64: case TypeID::FLOAT64: return 8;
      case TypeID::DATE32: case TypeID::TIME32: return 4;
      case TypeID::DATE64: case TypeID::TIME64: return 8;
      default: throw std::invalid_argument("Not a fixed-size primitive type");
    }
  }

  // 官方核心方法：获取指定索引值
  template <typename T>
  T GetValue(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    if (values->IsEmpty()) throw std::runtime_error("Empty values buffer");
    return values->As<T>()[idx];
  }
};

// 对齐官方：Bool 数组（Primitive 特例）
// 官方要点：
// 1. 值以位存储（1bit/元素），与 null_bitmap 格式一致
// 2. values 缓冲器字节长度 = ceil(length / 8)
// 参考：arrow/array/boolean_array.h
struct BooleanArray : public Array {
  std::shared_ptr<Buffer> values; // 值位图：1bit/元素，1=true，0=false

  BooleanArray() { type_id = TypeID::BOOL; }

  bool GetValue(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const int64_t byte_idx = idx / 8;
    const int64_t bit_idx = idx % 8;
    return (values->As<uint8_t>()[byte_idx] & (1 << bit_idx)) != 0;
  }
};
```

### 2. 变长 Binary/String 布局（Variable Length Binary/String）
```cpp
// 对齐官方：变长 Binary 数组（String 是 Binary 的 UTF-8 约束子集）
// 官方要点：
// 1. 由 offsets + values 缓冲器组成，offsets 长度 = length + 1
// 2. offsets[i] = 第 i 个元素在 values 中的起始字节位置
// 3. LargeBinary/LargeString 用 int64 偏移（突破 2GB 限制）
// 4. String 要求 values 数据为 UTF-8 编码，Binary 无编码约束
// 参考：arrow/array/binary_array.h
template <TypeID BinaryType, typename OffsetType = int32_t>
struct BinaryArray : public Array {
  std::shared_ptr<Buffer> offsets; // 偏移缓冲器：OffsetType[]，长度=length+1
  std::shared_ptr<Buffer> values;  // 值缓冲器：连续存储所有元素的字节数据

  BinaryArray() { type_id = BinaryType; }

  // 官方约束：偏移必须递增且非负
  bool ValidateOffsets() const {
    const auto* offs = offsets->As<OffsetType>();
    for (int64_t i = 1; i <= length; ++i) {
      if (offs[i] < offs[i-1] || offs[i] > values->size) return false;
    }
    return true;
  }

  // 官方核心方法：获取指定索引的 Binary 元素
  std::string GetValue(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const auto* offs = offsets->As<OffsetType>();
    const OffsetType start = offs[idx];
    const OffsetType end = offs[idx + 1];
    const int64_t len = end - start;
    return std::string(reinterpret_cast<const char*>(values->data + start), len);
  }
};

// 对齐官方：String 数组（Binary 子集，增加 UTF-8 校验）
// 参考：arrow/array/string_array.h
template <TypeID StringType, typename OffsetType = int32_t>
struct StringArray : public BinaryArray<StringType, OffsetType> {
  // 官方约束：校验 UTF-8 合法性
  bool ValidateUTF8() const {
    // 伪代码：遍历所有元素校验 UTF-8 编码
    return true;
  }
};

// 类型别名（对齐官方命名）
using BinaryArray32 = BinaryArray<TypeID::BINARY, int32_t>;
using LargeBinaryArray = BinaryArray<TypeID::LARGE_BINARY, int64_t>;
using StringArray32 = StringArray<TypeID::STRING, int32_t>;
using LargeStringArray = StringArray<TypeID::LARGE_STRING, int64_t>;
```

### 3. 变长 Binary/String View 布局（Binary/String View）
```cpp
// 对齐官方：String View 数组（Binary View 同理）
// 官方要点：
// 1. 设计目的：避免字符串拷贝，支持乱序写入、快速前缀比较
// 2. view 缓冲器结构：每个元素占 16B（int32 len + uint32 prefix + int64 offset）
//    - len：字符串长度
//    - prefix：前 4 字节（快速比较，无需读取完整字符串）
//    - offset：在 values 中的起始偏移
// 3. LargeStringView 用 int64 len/offset（突破 2GB 限制）
// 参考：arrow/array/string_view_array.h
template <TypeID ViewType>
struct StringViewArray : public Array {
  // 官方定义：View 元素结构
  struct View {
    int32_t length;        // 字符串长度（Large 版为 int64_t）
    uint32_t prefix;       // 前 4 字节（UTF-8 编码）
    int64_t offset;        // 在 values 中的起始偏移
  };

  std::shared_ptr<Buffer> views;   // View 缓冲器：View[]，长度=length*sizeof(View)
  std::shared_ptr<Buffer> values;  // 原始字符串数据缓冲器

  StringViewArray() { type_id = ViewType; }

  // 官方核心方法：获取 View 结构
  View GetView(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    return views->As<View>()[idx];
  }

  // 快速前缀比较（官方核心优化点）
  bool PrefixEquals(int64_t idx, const char* prefix, int32_t len) const {
    if (len > 4) return false; // prefix 仅存储前 4 字节
    auto view = GetView(idx);
    return memcmp(&view.prefix, prefix, len) == 0;
  }
};

// 类型别名
using StringViewArray32 = StringViewArray<TypeID::STRING_VIEW>;
using LargeStringViewArray = StringViewArray<TypeID::LARGE_STRING_VIEW>;
```

### 4. 变长 List 布局（Variable Length List）
```cpp
// 对齐官方：变长 List 数组
// 官方要点：
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

  // 官方核心方法：获取指定索引的 List 切片（零拷贝）
  std::shared_ptr<Array> GetSlice(int64_t idx) const {
    if (IsNull(idx)) throw std::runtime_error("Null value at index");
    const auto* offs = offsets->As<OffsetType>();
    const OffsetType start = offs[idx];
    const OffsetType end = offs[idx + 1];
    const int64_t slice_len = end - start;
    // 官方实现：切片仅修改子数组的长度/偏移，不拷贝数据
    return SliceArray(values, start, slice_len);
  }

private:
  // 官方切片逻辑（伪代码）
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
// 对齐官方：固定大小 List 数组
// 官方要点：
// 1. 无 offsets 缓冲器，每个 List 固定包含 element_size 个元素
// 2. 子数组总长度 = length * element_size
// 3. 设计目的：优化固定长度列表（如 3D 坐标 [x,y,z]）的内存访问
// 参考：arrow/array/fixed_size_list_array.h
struct FixedSizeListArray : public Array {
  int32_t element_size;           // 每个 List 固定的元素个数
  std::shared_ptr<Array> values;  // 子数组：总长度 = length * element_size

  FixedSizeListArray() { type_id = TypeID::FIXED_SIZE_LIST; }

  // 官方核心方法：获取指定索引的 List 切片
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
// 对齐官方：List View 数组
// 官方要点：
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

  // 官方核心方法：获取 List 大小和起始偏移
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
// 对齐官方：Struct 数组
// 官方要点：
// 1. 由多个等长的子数组组成，每个子数组对应一个字段
// 2. Struct 的 Null 由自身 null_bitmap 决定，子数组的 Null 独立
// 3. 字段顺序固定，字段名可选（可为空）
// 参考：arrow/array/struct_array.h
struct StructArray : public Array {
  std::vector<Field> fields;               // 字段元信息列表
  std::vector<std::shared_ptr<Array>> children; // 子数组列表（与字段一一对应）

  StructArray() { type_id = TypeID::STRUCT; }

  // 官方约束：子数组长度必须与 Struct 数组一致
  bool ValidateChildrenLength() const {
    for (const auto& child : children) {
      if (child->length != length) return false;
    }
    return true;
  }

  // 官方核心方法：获取指定索引的指定字段值
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
// 对齐官方：Map 数组
// 官方要点：
// 1. 本质是「List<Struct<key, value>>」的语法糖
// 2. 核心约束：key 可选唯一（sorted/unique 元数据标记）
// 3. 布局：ListArray 的子数组是 StructArray（包含 key/value 两个字段）
// 参考：arrow/array/map_array.h
struct MapArray : public Array {
  std::shared_ptr<ListArray> entries; // 核心：List<Struct<key, value>>
  bool keys_sorted;                   // key 是否排序
  bool keys_unique;                   // key 是否唯一

  MapArray() { type_id = TypeID::MAP; }

  // 官方核心方法：获取指定索引的 Map 条目
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
// 对齐官方：Union 数组（Dense/Sparse）
// 官方要点：
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

  // 官方核心方法：获取指定索引的元素（Dense/Sparse 统一逻辑）
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
// 对齐官方：字典编码数组
// 官方要点：
// 1. 设计目的：优化重复值多的场景，减少内存占用
// 2. 由 indices（索引数组） + dictionary（字典数组）组成
// 3. indices 为 int32/int64 类型，值为 dictionary 的索引
// 4. dictionary 无重复值，可为任意类型（Primitive/Binary/Struct 等）
// 参考：arrow/array/dictionary_array.h
struct DictionaryArray : public Array {
  std::shared_ptr<PrimitiveArray> indices; // 索引数组：int32_t/int64_t 类型
  std::shared_ptr<Array> dictionary;       // 字典数组：存储所有唯一值

  DictionaryArray() { type_id = TypeID::DICTIONARY; }

  // 官方核心方法：解析索引为原始值
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
// 对齐官方：运行结束编码（Run-End Encoded, RLE）数组
// 官方要点：
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

  // 官方核心方法：查找指定索引所属的运行段（二分查找）
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

  // 官方核心方法：获取指定索引的原始值
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

### 核心要点总结（对齐官方文档）
| 布局类型                | 核心设计目的                  | 关键结构                          | 官方约束/优化点                     |
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

以上伪代码完全对齐 Arrow 官方文档的布局定义和 C++ 源码结构，省略了内存池、序列化、类型校验等非核心逻辑，但保留了所有布局的**核心内存结构**和**官方约束**，可直接作为理解 Arrow 内存模型的参考。