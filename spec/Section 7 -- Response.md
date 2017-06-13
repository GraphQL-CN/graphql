# Response/响应

如果一个GraphQL服务器收到一个请求，它必须返回一个良好格式化的响应。如果请求操作执行成功，服务器的响应则描述其执行结果，否则描述执行期间遇到的错误。

一个响应可能会包含部分是响应，部分是另一个字段上发生错误的时候，响应中这个字段的值被替换成了null。


## Serialization Format/序列化格式

GraphQL并不要求特定的序列化格式，然而客户端应该使用一种支持GraphQL响应中的主要原始类型的序列化格式。特别需要支持一下原始类型：

 * Map
 * List
 * String
 * Null

在执行一节中的{CollectFields()}的定义中，序列化格式如果支持有序映射集，那么它应该保持请求字段的顺序。只支持无序映射集的序列化格式（譬如JSON）应该保持其语法上的顺序。

产生跟请求一致的字段顺序响应有助于提过调试过程中面向人类的可读性和属性相关的响应解析效率。

系列化格式应支持下列原始类型，然而String可能用于替换下列原始类型。

 * Boolean
 * Int
 * Float
 * Enum Value


### JSON Serialization/JSON序列化

虽然如上所述，GraphQL并不要求一种特定的序列化格式，但是还是比较偏好于JSON。为了风格统一和易于标注，本规范中响应的案例都以JSOn的格式表示，特别需要注意的是，我们的JSON案例中，原始类型都以一下JSON概念来表示：

| GraphQL值     | JSON值            |
| ------------- | ----------------- |
| Map           | Object            |
| List          | Array             |
| Null          | {null}            |
| String        | String            |
| Boolean       | {true} or {false} |
| Int           | Number            |
| Float         | Number            |
| Enum Value    | String            |

**Object Property Ordering/对象属性排序**

JSON被描述为[无序键值对集合](https://tools.ietf.org/html/rfc7159#section-4)，而键值对在表示上确实有序的方式。换言之，即便JSON字符串`{ "name": "Mark", "age": 30 }`和`{ "age": 30, "name": "Mark" }`编码了相同的值，但是他们却有不同的属性表示顺序。

选择集的求值结果是有序的，这个顺序来源于请求，并由查询执行器定义，因此JSON序列化可以保持这个顺序，并以这个顺序写入到对象属性中。

譬如，如果查询是`{ name, age }`，则GraphQL服务器响应的JSON会是`{ "name": "Mark", "age": 30 }`而不是`{ "age": 30, "name": "Mark" }`。

NOTE: 这并没有破坏JSON规范，因为客户端还是会将响应转换成无序映射集的有效值。


## Response Format/响应格式

GraphQL的响应应该是映射集类型。

如果操作包含了执行，那其响应映射集的第一个条目的键必须是`data`，其值将在"Data/数据"一章节中描述。如果操作在执行之前就失败，譬如语法错误、缺失信息或者验证错误，则此条目不应显示。

如果操作遇到了错误，则响应映射集的下一个条目的键必须是`errors`，其值将在"Errors/错误"一章节中描述。如果操作并没遇到错误，则此条目不应显示。

响应可以包含键为`extensions`的条目，此条目必须包含值。此条目是作为协议扩展而为实现者保留的，因此并没有对其内容的附加限制要求。

为了保证本协议今后的变化不会破坏已有的服务器和客户端，顶层的响应映射集不能包含上述三个之外的条目。


### Data/数据

响应中的`data`条目是请求的操作执行的结果。如果操作是query/查询，输出则是schema查询的根级类型对象，如果操作是mutation/更改，输出则是schema更改的根级类型对象。

如果在执行前遇到错误，结果中将不应有`data`条目。

如果在执行中遇到错误，并导致不能返回有效响应，则`data`条目应该为`null`。


### Errors/错误

响应中的`errors`是一个非空错误列表，每个错误是一个映射集。

如果在执行请求的操作中未遇到错误，结果中将不应有`errors`条目。

如果响应结果中没有`data`条目，则响应中的`errors`条目不可为空，其必须包含至少一条错误，这个错误应指出为什么没有数据返回。

如果响应中包含`data`条目（包含值为{null}的情况），那么响应中的`errors`条目可以包含执行期间的任何错误。如果执行期发生了错误，那这个错误就应该被包含。

**Error result format/错误结果格式**

每个错误都必须包含键为`message`的条目，其包含了针对开发这错误描述，以便修正错误。

如果一个错误能和请求的GraphQL文档特定点所匹配，它应该包含键为`locations`的条目，其内容为一个定位列表，每个定位都是键为`line`和`column`的映射集，两者都是从`1`开始的正数，用以描述相关的语法元素的起始位置。

如果一个错误能和GraphQL结果中的特定字段关联，它必须包含键为`path`的条目，其描述了响应中哪一个字段面临了错误。这能让客户端鉴别一个`null`是正常逻辑还是运行时错误。

这个字段应该是一个路径段的列表，从响应根级开始直到关联的字段结束。用于表示字段的路径段应该是字符串类型，而表示列表索引的路径段则应该是从0开始的整数。如果错误发生在别名字段，那对应的路径应该使用别名，因为其表示的是响应中的路径而非查询中的。

例如，如果下列查询中，获取一个朋友的名字失败了：

```GraphQL
{
  hero(episode: $episode) {
    name
    heroFriends: friends {
      id
      name
    }
  }
}
```

响应会像这样：

```json
{
  "errors": [
    {
      "message": "Name for character with ID 1002 could not be fetched.",
      "locations": [ { "line": 6, "column": 7 } ],
      "path": [ "hero", "heroFriends", 1, "name" ]
    }
  ],
  "data": {
    "hero": {
      "name": "R2-D2",
      "heroFriends": [
        {
          "id": "1000",
          "name": "Luke Skywalker"
        },
        {
          "id": "1002",
          "name": null
        },
        {
          "id": "1003",
          "name": "Leia Organa"
        }
      ]
    }
  }
}
```

如果发生错误的字段声明为`Non-Null`非空，则`null`错误会冒泡到下一个可空字段，这种情况下，`path`也应该包含到发生错误的字段的完整路径，即便这个字段未出现在响应中。

譬如，如果上述的`name`字段在schema中声明为`Non-Null`非空，那么查询结果可能不同，但是错误还是一样。

```json
{
  "errors": [
    {
      "message": "Name for character with ID 1002 could not be fetched.",
      "locations": [ { "line": 6, "column": 7 } ],
      "path": [ "hero", "heroFriends", 1, "name" ]
    }
  ],
  "data": {
    "hero": {
      "name": "R2-D2",
      "heroFriends": [
        {
          "id": "1000",
          "name": "Luke Skywalker"
        },
        null,
        {
          "id": "1003",
          "name": "Leia Organa"
        }
      ]
    }
  }
}
```

GraphQL服务器可以给error提供附加的条目，以用来展示更为用的或者机器可读的错误，当然，今后本规范也可能为error引入附加条目。
