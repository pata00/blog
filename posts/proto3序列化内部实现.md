# proto3序列化内部实现
<span id="jump00"></span>
## 序列化对照表
.proto类型 | c/c++类型 | 值的序列化编码 | wire_type<sup>[注1](#jump01)</sup>
:--: | :--: | :--: | :--: 
double | double | 小端内存原始编码8字节 | WIRETYPE_FIXED64
float | float | 小端内存原始编码4字节 | WIRETYPE_FIXED32
int32 | int32_t | 转换位uint64_t类型后再varint编码 | WIRETYPE_VARINT
int64 | int64_t | 转换位uint64_t类型后再varint编码 | WIRETYPE_VARINT
uint32 | uint32_t | 直接varint编码 | WIRETYPE_VARINT
uint64 | uint64_t | 直接varint编码 | WIRETYPE_VARINT
sint32 | int32_t | 32位zigzag编码后varint | WIRETYPE_VARINT
sint64 | int64_t | 64位zigzag编码后varint | WIRETYPE_VARINT
fixed32 | uint32_t | 小端内存原始编码(4字节) | WIRETYPE_FIXED32
fixed64 | uint64_t | 小端内存原始编码(8字节) | WIRETYPE_FIXED64
sfixed32 | int32_t | 转换为uint32_t后原始编码(4字节) | WIRETYPE_FIXED32
sfixed64 | int64_t | 转换为uint64_t后原始编码(4字节) | WIRETYPE_FIXED64
bool | bool | varint编码写入1 or 0 | WIRETYPE_VARINT
string | string | 必须是UTF-8串 <br> 用varint编码写入长度(最大2<sup>32</sup>)，再写后续内存<br> <> | WIRETYPE_LENGTH_DELIMITED
bytes | string | 同上，不过不验证是否utf-8串 | WIRETYPE_LENGTH_DELIMITED
repeat T | | 先varint写入数组长度，依次序列化对应元素 | WIRETYPE_LENGTH_DELIMITED
object | | 先varint写入对象实际序列化长度，再序列化对象本身 | WIRETYPE_LENGTH_DELIMITED

## proto 样例
- AllTypeMsg
  ```
  syntax="proto3";

  package PB_TEST;

  message AllTypeMsg {
      double a = 1;
      float b = 2;
      int32 c = 3;
      int64 d = 4;
      uint32 e = 5;
      uint64 f = 6;
      sint32 g = 7;
      sint64 h = 8;
      fixed32 i = 9;
      fixed64 j = 10;
      sfixed32 k = 11;
      sfixed64 l = 12;
      bool m = 13;
      string n = 14;
      bytes o = 15;	
  }
  ```
## 序列化实现
- protoc生成的代码会按照id从小到大开始扫描，如果存在，则按照先tag后值的方式依次写入
  1. 写tag:

     tag(uint32_t类型)值的生成按照```id << 3 | wire_type```取值，于是tag中同时包含了id和类型信息，带来的限制是id的最大值只能是536,870,911 (0x1fffffff), tag值生成后，以varint编码写入
  2. 写value:

     value值的编码按照[序列化对照表](#jump00)编码方式写入
     
# 备注
1. <span id="jump01"></span> tag type枚举列表
   ```
   enum WireType {
       WIRETYPE_VARINT = 0,
       WIRETYPE_FIXED64 = 1,
       WIRETYPE_LENGTH_DELIMITED = 2,
       WIRETYPE_START_GROUP = 3,    //
       WIRETYPE_END_GROUP = 4,      //
       WIRETYPE_FIXED32 = 5,
   };
   ```
   Type | Meaning	| Used For
   :--: | :--: | :--: 
   0	| Varint	| int32, int64, uint32, uint64, sint32, sint64, bool, enum
   1	| 64-bit	| fixed64, sfixed64, double
   2	| Length-delimited	| string, bytes, embedded messages, packed repeated fields
   3	| Start group	| groups (deprecated<sup>[注4](#jump04)</sup>)
   4	| End group	| groups (deprecated<sup>[注4](#jump04)</sup>)
   5	| 32-bit	| fixed32, sfixed32, float
2. zigzag编码

   符号位权重挪至最低，其余数值权重调高1bit
   ```
   inline uint32 WireFormatLite::ZigZagEncode32(int32 n) {
     // Note:  the right-shift must be arithmetic
     // Note:  left shift must be unsigned because of overflow
     return (static_cast<uint32>(n) << 1) ^ static_cast<uint32>(n >> 31);
   }
   
   inline int32 WireFormatLite::ZigZagDecode32(uint32 n) {
     // Note:  Using unsigned types prevent undefined behavior
     return static_cast<int32>((n >> 1) ^ (~(n & 1) + 1));
   }
   
   inline uint64 WireFormatLite::ZigZagEncode64(int64 n) {
     // Note:  the right-shift must be arithmetic
     // Note:  left shift must be unsigned because of overflow
     return (static_cast<uint64>(n) << 1) ^ static_cast<uint64>(n >> 63);
   }
   
   inline int64 WireFormatLite::ZigZagDecode64(uint64 n) {
     // Note:  Using unsigned types prevent undefined behavior
     return static_cast<int64>((n >> 1) ^ (~(n & 1) + 1));
   }
   ```
3. [varint编码](https://en.wikipedia.org/wiki/Variable-length_quantity)
   
   对于无符号数，每次取低位7bit用 8bit表示，8位中的最高bit位用1表示继续0表示结束

4. <span id="jump04"></span> 这两个值应该是goolge内部的早期协议序列化里有用到，从google开源pb的时候已经是deprecated