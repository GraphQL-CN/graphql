# Introspection/内省

一个GraphQL服务器支持基于它的Schema来支持内省，这个Schema可以通过GraphQL自身来查询，创建了一个强大的工具构建平台。

拿一个琐碎的app的样例查询举例。样例中有一个User类型，其中有字段：id，name和birthday。

例如，假设服务器有下列类型定义：

```GraphQL
type User {
  id: String
  name: String
  birthday: Date
}
```

查询为

```GraphQL
{
  __type(name: "User") {
    name
    fields {
      name
      type {
        name
      }
    }
  }
}
```

会返回

```json
{
  "__type": {
    "name": "User",
    "fields": [
      {
        "name": "id",
        "type": { "name": "String" }
      },
      {
        "name": "name",
        "type": { "name": "String" }
      },
      {
        "name": "birthday",
        "type": { "name": "Date" }
      }
    ]
  }
}
```


## General Principles/基本原则


### Naming conventions/命名约定

GraphQL内省系统要求的类型和字段与用户定义的类型和系统共享相同的上下文，其字段名以{"__"}双下划线开头，以避免和用户定义的GraphQL类型命名冲突。相反地，GraphQL类型系统作者不能定义任何以双下划线开头的类型、字段、参数和其他类型系统工件。<small>artifact的翻译不确定</small>


### Documentation/文档

所有内省系统中的类型必须提供`String`类型的`description`字段，以便类型设计者发布文档以增强能力。GraphQL服务可以返回使用Markdown语法（[CommonMark](http://commonmark.org/)中指定）的`description`字段。因此建议所有展示`description`字段的工具使用CommonMark兼容的Markdown渲染器。


### Deprecation/弃用

为了支持向后兼容管理，GraphQL字段和枚举值可以指出其是否弃用(`isDeprecated: Boolean`)和一个为何弃用的描述(`deprecationReason: String`)。

基于GraphQL内省系统建造的工具应该通过隐含信息或者面向开发者的警告信息来减少弃用字段的使用。


### Type Name Introspection/类型命名内省

GraphQL支持类型命名自行，查询任意对象/接口/联合时可以在一个查询语句的任意位置通过元字段`__typename: String!`，得到当前查询的对象类型。

这在查询接口或者联合类型的真实类型的时候用得最为频繁。

这是个隐式字段，并不会出现在定义的类型的字段列表中。


## Schema Introspection/Schema内省

Schema的内省系统可通过查询操作的根级类型上的元字段`__schema`和`__type`来接入。

```
__schema: __Schema!
__type(name: String!): __Type
```

这两个字段也是隐式的，不会出现在查询操作根节点类型的字段列表中。

GraphQL Schema内省系统的Schema：

```GraphQL
type __Schema {
  types: [__Type!]!
  queryType: __Type!
  mutationType: __Type
  subscriptionType: __Type
  directives: [__Directive!]!
}

type __Type {
  kind: __TypeKind!
  name: String
  description: String

  # OBJECT and INTERFACE only
  fields(includeDeprecated: Boolean = false): [__Field!]

  # OBJECT only
  interfaces: [__Type!]

  # INTERFACE and UNION only
  possibleTypes: [__Type!]

  # ENUM only
  enumValues(includeDeprecated: Boolean = false): [__EnumValue!]

  # INPUT_OBJECT only
  inputFields: [__InputValue!]

  # NON_NULL and LIST only
  ofType: __Type
}

type __Field {
  name: String!
  description: String
  args: [__InputValue!]!
  type: __Type!
  isDeprecated: Boolean!
  deprecationReason: String
}

type __InputValue {
  name: String!
  description: String
  type: __Type!
  defaultValue: String
}

type __EnumValue {
  name: String!
  description: String
  isDeprecated: Boolean!
  deprecationReason: String
}

enum __TypeKind {
  SCALAR
  OBJECT
  INTERFACE
  UNION
  ENUM
  INPUT_OBJECT
  LIST
  NON_NULL
}

type __Directive {
  name: String!
  description: String
  locations: [__DirectiveLocation!]!
  args: [__InputValue!]!
}

enum __DirectiveLocation {
  QUERY
  MUTATION
  SUBSCRIPTION
  FIELD
  FRAGMENT_DEFINITION
  FRAGMENT_SPREAD
  INLINE_FRAGMENT
}
```


### The __Type Type/__Type类型

`__Type`是这个类型内省系统的核心，它代表了这个系统中的标量、接口、对象类型、联合、枚举型。

`__Type`也表示类型修改器，其通常用于表示修改一个类型(`ofType: __Type`)，这正是我们如何表示列表类型和非空类型，以及他们的组合类型。


### Type Kinds/类型种类

类型系统中存在多个种类的类型，每种类型都有不同的有效字段，这些类型都被列举在`__TypeKind`枚举值中。


#### Scalar/标量

标量中譬如整数型、字符串型、布尔型都不能拥有字段。

GraphQL类型设计者需要描述标量的`description`字段描述数据格式以及标量的类型转换规则。

字段

* `kind`必须返回`__TypeKind.SCALAR`。
* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* 其他字段必须返回{null}空值。


#### Object/对象

对象类型表示一系列字段的具体实例，内省类型（譬如`__Type`,`__Field`等）亦是典型的对象类型。

字段

* `kind`必须返回`__TypeKind.OBJECT`。
* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `fields`：本类型上可被查询的字段的集合。
  * 接受参数`includeDeprecated`，默认为{false}。如过为{true}则弃用的字段也将返回。
* `interfaces`：对象实现的接口的集合。
* 其他字段必须返回{null}空值。


#### Union/联合

联合是一个抽象类型，其不声明任何字段。联合的可能类型要显式的在`possibleTypes`中列出。类型可不加修改的直接作为联合的一部分。

字段

* `kind`必须返回`__TypeKind.UNION`。
* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `possibleTypes`必须返回联合内可能的类型的列表，它们都必须是对象类型。
* 其他字段必须返回{null}空值。


#### Interface/接口

接口是一个抽象类型，其声明了通用字段。所有实现接口的对象必须定义完全匹配的名字和类型的字段。实现了此接口的类型都有在`possibleTypes`中列出。

字段

* `__TypeKind.INTERFACE`。
* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `fields`: 接口要求的字段的集合。
* `possibleTypes`返回实现了此接口的类型，它们都必须是对象类型。
* 其他字段必须返回{null}空值。


#### Enum/枚举型

枚举型是特殊的标量，他只能拥有有限集合的标量。

字段

* `kind`必须返回`__TypeKind.ENUM`。
* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `enumValues`：`EnumValue`的列表。必须至少有一个值，并且必须有不同的名字。
  * 接受参数`includeDeprecated`，默认为{false}。如过为{true}则弃用的字段也将返回。


#### Input Object/输入对象

输入对象是复合类型，通常以具名输入值列表的形式作为查询的输入。

例如输入对象`Point`可以定义成：

```GraphQL
input Point {
  x: Int
  y: Int
}
```

字段

* `kind`必须返回`__TypeKind.INPUT_OBJECT`。
* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `inputFields`：`InputValue`的列表。


#### List/列表

列表表示一系列GraphQL值。列表类型是类型修改器：它在`ofType`中封装了另一个类型的实例，其定义了列表中的每个元素。

字段

* `kind`必须返回`__TypeKind.LIST`。
* `ofType`：任意类型。
* 其他字段必须返回{null}空值。


#### Non-Null/非空

GraphQL类型都是可空的，{null}值是有效的字段类型返回值。

非空类型是类型修改器：它它在`ofType`中封装了另一个类型的实例。非空类型不允许{null}作为返回，也用于表示必要输入参数和必要输入对象字段。

* `kind`必须返回`__TypeKind.NON_NULL`。
* `ofType`：除了Non-null外的所有类型。
* 其他字段必须返回{null}空值。


#### Combining List and Non-Null/列表和非空的组合

列表和非空可以通过组合以表示更为复杂的类型。

如果列表修改的类型是非空，那么列表不能包含任何{null}空值元素。

如果非空修改的类型是列表，那么不能接受{null}空值，但是空数组可接受。

如果列表修改的类型是列表，那么第一个列表的元素是第二个列表类型的列表。

非空类型不能修改另一个非空类型。


### The __Field Type/__Field类型

`__Field`类型表示对象或者接口类型的每一个字段。

字段

* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `args`返回`__InputValue`的列表，表示这个字段接受的参数。
* `type`必须返回`__Type`，表示这个字段值的类型。
* `isDeprecated`返回{true}如果这个字段不应再被使用，否则{false}。
* `deprecationReason`可选，提供字段被弃用的原因。


### The __InputValue Type/__InputValue类型

`__InputValue`类型表示字段和指令的参数，如同输入对象的`inputFields`字段。

字段

* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `type`必须返回表示这个输入值期待的类型的`__Type`。
* `defaultValue`返回一个（使用GraphQL语言）字符串，表示这个输入值在运行时未提供值的情况下的默认值，如果这个输入值没有默认值，返回{null}空值。

### The __EnumValue Type/__EnumValue类型

`__EnumValue`表示枚举型的可能值之一。

字段

* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `isDeprecated`返回{true}如果这个字段不应再被使用，否则{false}。
* `deprecationReason`可选，提供字段被弃用的原因。

### The __Directive Type/__Directive类型

`__Directive`类型表示服务器支持的一个指令。

字段

* `name`必须返回字符串。
* `description`必须返回字符串或者{null}空值。
* `locations`返回`__DirectiveLocation`列表，表示指令可以放置的有效位置。
* `args`返回`__InputValue`的列表，表示这个字段接受的参数。
