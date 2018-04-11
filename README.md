# Protobuf3 语言指南

- [定义一个消息类型](#定义一个消息类型)
  - [指定字段类型](#指定字段类型)
  - [分配标识号](#分配标识号)
  - [指定字段规则](#指定字段规则)
  - [添加更多消息类型](#添加更多消息类型)
  - [添加注释](#添加注释)
  - [保留标识符（Reserved）](#保留标识符reserved)
  - [从.proto文件生成了什么？](#从proto文件生成了什么)
- [标量数值类型](#标量数值类型)
- [默认值](#默认值)
- [枚举](#枚举)
- [使用其他消息类型](#使用其他消息类型)
  - [导入定义](#导入定义)
  - [使用proto2消息类型](#使用proto2消息类型)
- [嵌套类型](#嵌套类型)
- [更新一个消息类型](#更新一个消息类型)
- [Any](#any)
- [Oneof](#oneof)
  - [使用Oneof](#使用oneof)
  - [Oneof 特性](#oneof-特性)
  - [向后兼容性问题](#向后兼容性问题)
- [映射（Maps）](#映射maps)
  - [向后兼容性问题](#向后兼容性问题)
- [包（Packages）](#包packages)
  - [包及名称的解析](#包及名称的解析)
- [定义服务](#定义服务)
- [JSON 映射](#json-映射)
- [选项](#选项)
  - [自定义选项](#自定义选项)
- [生成你的类](#生成你的类)


## 出处
- 英文原文：
  - [Language Guide(proto3)](https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-cn#generating)
- 中文出处：
  - [Protobuf语言指南](http://www.open-open.com/home/space.php?uid=37924&do=blog&id=5873)
  - [Protobuf 语法指南](中文出处是proto2的译文，proto3的英文出现后在原来基础上增改了，水平有限，还请指正)

## 定义一个消息类型

先来看一个非常简单的例子。假设你想定义一个“搜索请求”的消息格式，每一个请求含有一个查询字符串、你感兴趣的查询结果所在的页数，以及每一页多少条查询结果。可以采用如下的方式来定义消息类型的.proto文件了：

```
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

- 文件的第一行指定了你正在使用proto3语法：如果你没有指定这个，编译器会使用proto2。这个指定语法行必须是文件的非空非注释的第一个行。
- SearchRequest消息格式有3个字段，在消息中承载的数据分别对应于每一个字段。其中每个字段都有一个名字和一种类型。

### 指定字段类型
在上面的例子中，所有字段都是标量类型：两个整型（page_number和result_per_page），一个string类型（query）。当然，你也可以为字段指定其他的合成类型，包括枚举（enumerations）或其他消息类型。

### 分配标识号
正如你所见，在消息定义中，每个字段都有唯一的一个数字标识符。这些标识符是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改变。注：[1,15]之内的标识号在编码的时候会占用一个字节。[16,2047]之内的标识号则占用2个字节。所以应该为那些频繁出现的消息元素保留 [1,15]之内的标识号。切记：要为将来有可能添加的、频繁出现的标识号预留一些标识号。

最小的标识号可以从1开始，最大到2^29 - 1, or 536,870,911。不可以使用其中的[19000－19999]（ (从FieldDescriptor::kFirstReservedNumber 到 FieldDescriptor::kLastReservedNumber)）的标识号， Protobuf协议实现中对这些进行了预留。如果非要在.proto文件中使用这些预留标识号，编译时就会报警。同样你也不能使用早期保留的标识号。

### 指定字段规则
所指定的消息字段修饰符必须是如下之一：

- singular：一个格式良好的消息应该有0个或者1个这种字段（但是不能超过1个）。
- repeated：在一个格式良好的消息中，这种字段可以重复任意多次（包括0次）。重复的值的顺序会被保留。

在proto3中，repeated的标量域默认情况虾使用packed。

你可以了解更多的pakced属性在Protocol Buffer 编码。

### 添加更多消息类型
在一个.proto文件中可以定义多个消息类型。在定义多个相关的消息的时候，这一点特别有用——例如，如果想定义与SearchResponse消息类型对应的回复消息格式的话，你可以将它添加到相同的.proto文件中，如：

```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

### 添加注释
向.proto文件添加注释，可以使用C/C++/java风格的双斜杠（//） 语法格式，如：

```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

### 保留标识符（Reserved）
如果你通过删除或者注释所有域，以后的用户可以重用标识号当你重新更新类型的时候。如果你使用旧版本加载相同的.proto文件这会导致严重的问题，包括数据损坏、隐私错误等等。现在有一种确保不会发生这种情况的方法就是指定保留标识符（and/or names, which can also cause issues for JSON serialization不明白什么意思），protocol buffer的编译器会警告未来尝试使用这些域标识符的用户。

```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
注：不要在同一行reserved声明中同时声明域名字和标识号

### 从.proto文件生成了什么？
当用protocol buffer编译器来运行.proto文件时，编译器将生成所选择语言的代码，这些代码可以操作在.proto文件中定义的消息类型，包括获取、设置字段值，将消息序列化到一个输出流中，以及从一个输入流中解析消息。

- 对C++来说，编译器会为每个.proto文件生成一个.h文件和一个.cc文件，.proto文件中的每一个消息有一个对应的类。
- 对Java来说，编译器为每一个消息类型生成了一个.java文件，以及一个特殊的Builder类（该类是用来创建消息类接口的）。
- 对Python来说，有点不太一样——Python编译器为.proto文件中的每个消息类型生成一个含有静态描述符的模块，，该模块与一个元类（metaclass）在运行时（runtime）被用来创建所需的Python数据访问类。
- 对go来说，编译器会位每个消息类型生成了一个.pd.go文件。
- 对于Ruby来说，编译器会为每个消息类型生成了一个.rb文件。
- javaNano来说，编译器输出类似域java但是没有Builder类
- 对于Objective-C来说，编译器会为每个消息类型生成了一个pbobjc.h文件和pbobjcm文件，.proto文件中的每一个消息有一个对应的类。
- 对于C#来说，编译器会为每个消息类型生成了一个.cs文件，.proto文件中的每一个消息有一个对应的类。

你可以从如下的文档链接中获取每种语言更多API(proto3版本的内容很快就公布)。[API Reference](https://developers.google.com/protocol-buffers/docs/reference/overview?hl=zh-cn)

## 标量数值类型
一个标量消息字段可以含有一个如下的类型——该表格展示了定义于.proto文件中的类型，以及与之对应的、在自动生成的访问类中定义的类型：

|.proto Type	|Notes	|C++ Type	|Java Type	|Python Type[2]	|Go Type	|Ruby Type	|C# Type	|PHP Type|
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
|double	|	|double	|double	|float	|float64	|Float	|double	|float|
|float	|	|float	|float	|float	|float32	|Float	|float	|float|
|int32	|使用变长编码，对于负值的效率很低，如果你的域有可能有负值，请使用sint64替代	|int32	|int	|int	|int32	|Fixnum 或者 Bignum（根据需要）	|int	|integer|
|uint32	|使用变长编码	|uint32	|int	|int/long	|uint32	|Fixnum 或者 Bignum（根据需要）	|uint	|integer|
|uint64	|使用变长编码	|uint64	|long	|int/long	|uint64	|Bignum	|ulong	|integer/string|
|sint32	|使用变长编码，这些编码在负值时比int32高效的多	|int32	|int	|int	|int32	|Fixnum 或者 Bignum（根据需要）	|int	|integer|
|sint64	|使用变长编码，有符号的整型值。编码时比通常的int64高效。	|int64	|long	|int/long	|int64	|Bignum	|long	|integer/string|
|fixed32	|总是4个字节，如果数值总是比总是比228大的话，这个类型会比uint32高效。	|uint32	|int	|int	|uint32	|Fixnum 或者 Bignum（根据需要）	|uint	|integer|
|fixed64	|总是8个字节，如果数值总是比总是比256大的话，这个类型会比uint64高效。	|uint64	|long	|int/long	|uint64	|Bignum	|ulong	|integer/string|
|sfixed32	|总是4个字节	|int32	|int	|int	|int32	|Fixnum 或者 Bignum（根据需要）	|int	|integer|
|sfixed64	|总是8个字节	|int64	|long	|int/long	|int64	|Bignum	|long	|integer/string|
|bool	|	|bool	|boolean	|bool	|bool	|TrueClass/FalseClass	|bool	|boolean|
|string	|一个字符串必须是UTF-8编码或者7-bit ASCII编码的文本。	|string	|String	|str/unicode	|string	|String (UTF-8)	|string	|string|
|bytes	|可能包含任意顺序的字节数据。	|string	|ByteString	|str	|[]byte	|String (ASCII-8BIT)	|ByteString	|string|


## 默认值

## 枚举

## 使用其他消息类型

### 导入定义

### 使用proto2消息类型

## 嵌套类型

## 更新一个消息类型

## Any

## Oneof

### 使用Oneof

### Oneof 特性

### 向后兼容性问题

## 映射（Maps）

### 向后兼容性问题

## 包（Packages）

### 包及名称的解析

## 定义服务

## JSON 映射

## 选项

### 自定义选项

## 生成你的类
