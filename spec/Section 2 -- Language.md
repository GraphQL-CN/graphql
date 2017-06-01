# Language/语言

客户端使用GraphQL查询语言来请求GraphQL服务，我们称这些请求为文档，文档包含操作（queries/查询，mutations/修改，和subscriptions订阅）和片段（用于组合重用的共有单元）。

GraphQL文档的语法中，将终端符号视为记号，即独立词法单元。这些记号以词法方式定义，满足源字符模式（用`::`定义）。<small>译者案：翻译中使用->表示</small>

## Source Text/源文本

SourceCharacter :: /[\u0009\u000A\u000D\u0020-\uFFFF]/

源字符 -> /\[\u0009\u000A\u000D\u0020-\uFFFF\]/

GraphQL文档可表示为一序列的[Unicode](http://unicode.org/standard/standard.html)（统一码）字符，然而，除了少许例外，大部分GraphQL文档都是用ASCII非控制字符来表示，以便于尽量兼容已有工具、语言和序列化格式，并尽可能避免在编辑器和源代码管理的显示问题。


### Unicode/统一码

UnicodeBOM :: "Byte Order Mark (U+FEFF)"

UnicodeBOM -> "字节顺序标记(U+FEFF)"

GraphQL的{StringValue}（字符串值）和{Comment}（备注）中可以使用非ASCII的Unicode字符。

BOM，又称字节顺序标记，是一个特殊的Unicode字符，它出现在文件的头部，以便程序用以确认当前文本流是Unicode编码，使用了大端还是小端，该用哪一种Unicode编码来转义。


### White Space/空白符

WhiteSpace ::
  - "Horizontal Tab (U+0009)"
  - "Space (U+0020)"

空白符 ->
  - "水平制表符(U+0009)"
  - "空格(U+0020)"

空白符出现在记号的前后，作为记号分隔使用，用于提升源文本的易读性。GraphQL查询文档的空白符可能出现在{String}或{Comment}记号中，但并不会显著影响其语义。

Note: GraphQL不采用Unicode的Zs类别字符作为空白符，以避免编辑器和源代码管理工具的误读。


### Line Terminators/行终止符

LineTerminator ::
  - "New Line (U+000A)"
  - "Carriage Return (U+000D)" [ lookahead ! "New Line (U+000A)" ]
  - "Carriage Return (U+000D)" "New Line (U+000A)"

行终止符 ->
  - "新行 (U+000A)"
  - "回车 (U+000D)"
  - "回车 (U+000D)" "新行 (U+000A)"

跟空白符类似，行终止符也是用于提升源文本的易读性，可出现在记号的前后，对GraphQL查询文档的语义无显著影响。行终止符不应该出现在其他记号之间。

Note: 语法报错时的行号应该由之前{LineTerminator}（行终止符）的总数来生成。


### Comments/注释

Comment :: `#` CommentChar*

CommentChar :: SourceCharacter but not LineTerminator

注释 -> `#` 注释字符

注释字符 -> 源字符，非行终止符

GraphQL查询文档可以包含以{`#`}开头的单行注释。

注释可使用除了{LineTerminator}（行终止符）以外的任意Unicode字符（码点）。所以可以说，注释包含了以{`#`}开头的除行终止符以外的所有字符。

注释与空白符类似，出现在任意记号后面、行终止符前面，对GraphQL查询文档的语义无显著影响。


### Insignificant Commas/无语义逗号

Comma :: ,

逗号 -> ,

与空白符和行终止符类似，逗号({`,`})也是提升源文本的易读性、分隔词法记号，对GraphQL查询文档的语法语义上也无显著影响.

无语义逗号保证了无论有误逗号，都不影响文档的解读，在其他语言中这就可能是一个用户错误。为了源代码易读性和可维护性，列表会使用行终止符和末尾逗号作为分隔符，无语义逗号也有这个样式上的用途。


### Lexical Tokens/词法记号

Token ::
  - Punctuator
  - Name
  - IntValue
  - FloatValue
  - StringValue

记号 ->
  - 标点
  - 命名
  - 整数值
  - 浮点值
  - 字符串型值

一个GraphQL文档由多种独立记号组成，本规范中使用源文本Unicode字符模式来定义这些记号。

在后文GraphQL查询文档句法中，记号将作为终结符使用。


### Ignored Tokens/无语义记号

Ignored ::
  - UnicodeBOM
  - WhiteSpace
  - LineTerminator
  - Comment
  - Comma

无语义记号 ->
  - Unicode字节顺序标记
  - 空白符
  - 行终止符
  - 注释
  - 逗号

在词法记号的前后可能会出现不定量的无语义记号，包括{WhiteSpace}（空白符）和{Comment}备注。源文档的无语义区域都是无语义影响的，但是无语义字符可能以一种有影响的方式出现在源字符词法记号之间，譬如{String}（字符串）可能包含空白字符。

在解析给定记号时，所有字符都不能被忽略，譬如{FloatValue}（浮点值）的字符中不允许出现空白符。
<small>(译者案：本段无力，求大神指点)</small>

### Punctuators/标点

Punctuator :: one of ! $ ( ) ... : = @ [ ] { | }

标点 -> ! $ ( ) ... : = @ \[ \] \{ | \} 之一

GraphQL文档使用了标点符号以描述结构，GraphQL是一种数据描述语言，而非编程语言，因此GraphQL缺乏用于描述数学表达式的标点符号。


### Names/命名

Name :: /[_A-Za-z][_0-9A-Za-z]*/

命名 ->  /\[_A-Za-z\]\[_0-9A-Za-z\]/\*

GraphQL查询文档全是命名的产物：operations（操作），fields（字段），arguments（参数），directives（指令）， fragments（片段）和variables（变量）。所有命名必须遵循以下格式：

GraphQL的命名是大小写敏感的，也就是说`name`，`Name`，和`NAME`是不同的名字，下划线也具有影响，`other_name`和`othername`也是两个不同的名字。

GraphQL的命名限制在上述<acronym>ASCII</acronym>子集内，以便支持尽可能多的其他系统。


## Query Document/查询文档

Document : Definition+

Definition :
  - OperationDefinition
  - FragmentDefinition

文档 -> 定义

定义 ->
  - 操作定义
  - 片段定义

GraphQL查询文档描述了GraphQL服务收到的完整文件或者请求字符串。一个文档可以包含多个操作和片段的定义。一个查询文档只有包含操作时，服务器才能执行。但是无操作的文档也能被解析和验证，以让客户端提供单个跨文档请求。

如果一个文档只有一个操作，那这个操作可以不带命名或者以简写，省略掉query关键字和操作名。否则当一个查询文档包含多个操作时，每个操作都必须命名，并且在提交给服务器的时候，也要指明需要执行的目标操作。


## Operations/操作

OperationDefinition :
  - OperationType Name? VariableDefinitions? Directives? SelectionSet
  - SelectionSet

OperationType : one of `query` `mutation` `subscription`

操作定义 ->
  - 操作类型命名？变量定义？指令？选择集合
  - 选择集合

操作类型 ->
  - `query` `mutation` `subscription`之一
  
GraphQL做了三类操作模型：
  * query/查询 - 只读获取
  * mutation/改变 - 先写入再获取
  * subscription/订阅 - 一个长期请求，根据源事件获取数据

每一个操作都以一个可选操作名和选择集合表示，例如这个mutation（改变）操作，对一个story点赞（like），然后获取了被点赞次数：

```GraphQL
mutation {
  likeStory(storyID: 12345) {
    story {
      likeCount
    }
  }
}
```

**Query shorthand/查询简写**

如果一个文档只包含一个查询操作，也不包含变量和指令，那么这个操作可以省略query关键字和操作名。例如，下面这个无名查询操作就写成了查询简写形式：

```GraphQL
{
  field
}
```

Note: 注意，后文中很多案例都会使用查询简写格式。


## Selection Sets/选择集合

SelectionSet : { Selection+ }

Selection :
  - Field
  - FragmentSpread
  - InlineFragment

选择集合 -> \{选择\}

选择 ->
  - 字段
  - 片段解构
  - 内联片段

一个操作选择了他所需要的信息的集合，然后就会精确地得到他所要的信息，没有一点多余，避免了数据的多取或少取。

```GraphQL
{
  id
  firstName
  lastName
}
```

这个query/查询中，`id`，`firstName`和`lastName`字段构成了选择集合，选择集合也能包含fragment/片段的引用。


## Fields/字段

Field : Alias? Name Arguments? Directives? SelectionSet?

字段 -> 别名？具名参数？指令？选择集合？

一个选择集合主要由字段组成，一个字段描述了选择集合中对请求可用的一个离散信息片段。

有些字段描述了复杂的数据或者与其他数据的关联，为了进一步解明这种数据，一个字段可能包含一个选择集合，从而使能将请求嵌套起来。所有的GraphQL操作都必须依次深入嵌套，指明所有标量值字段，以保证响应数据的形态上没有歧义。

例如，这个操作选择了复杂数据和关联数据，并深入到嵌套内部，直到标量值字段。

```GraphQL
{
  me {
    id
    firstName
    lastName
    birthday {
      month
      day
    }
    friends {
      name
    }
  }
}
```

一个操作中，顶层选择集合的字段通常表示对应用和观察者而言全局可见的信息。典型的案例有顶层字段指向当前登录的观察者，或者引用唯一id来取特定类型数据：

```GraphQL
# `me` could represent the currently logged in viewer.`me`指代当前登录的观察者
{
  me {
    name
  }
}

# `user` represents one of many users in a graph of data, referred to by a
# unique identifier.
# `user`表示一个通过id来从图数据中取出来的用户
{
  user(id: 4) {
    name
  }
}
```


## Arguments/参数

Arguments : ( Argument+ )

Argument : Name : Value

参数 -> 命名: 值

字段在概念上是会返回值的函数，偶尔接受参数以改变其行为。通常这些参数和GraphQL服务器实现的函数参数直接映射。

这个案例中，我们向查询特定用户（通过`id`参数请求）的特定尺寸档案照片（通过`size`参数）：

```GraphQL
{
  user(id: 4) {
    id
    name
    profilePic(size: 100)
  }
}
```

许多参数也能存在于给定字段：

```GraphQL
{
  user(id: 4) {
    id
    name
    profilePic(width: 100, height: 50)
  }
}
```

**Arguments are unordered/参数无需顺序**

参数可以以任意句法顺序排列，都表示同一种语义。

下列两个查询语义上都是一样的：

```GraphQL
{
  picture(width: 200, height: 100)
}
```

```GraphQL
{
  picture(height: 100, width: 200)
}
```


## Field Alias/字段别名

Alias : Name :

别名 -> 命名

默认情况下，返回对象的键名会采用查询的字段名，然后你可以定义不同的键名，亦即别名。

案例中，我们获取了两个不同尺寸的档案照片，并保证了返回对象没有重复键名：

```GraphQL
{
  user(id: 4) {
    id
    name
    smallPic: profilePic(size: 64)
    bigPic: profilePic(size: 1024)
  }
}
```

然后得到结果：

```json
{
  "user": {
    "id": 4,
    "name": "Mark Zuckerberg",
    "smallPic": "https://cdn.site.io/pic-4-64.jpg",
    "bigPic": "https://cdn.site.io/pic-4-1024.jpg"
  }
}
```

顶级query/查询也是一个字段，所以它也可以使用别名：

```GraphQL
{
  zuck: user(id: 4) {
    id
    name
  }
}
```

得到这个结果：

```json
{
  "zuck": {
    "id": 4,
    "name": "Mark Zuckerberg"
  }
}
```

如果使用了别名，那么返回对象的中字段的键名就是别名，否则就是字段名。


## Fragments/片段

FragmentSpread : ... FragmentName Directives?

FragmentDefinition : fragment FragmentName TypeCondition Directives? SelectionSet

FragmentName : Name but not `on`

片段解构/展开 -> ... 片段名 指令？

片段定义 -> fragment 片段名 类型条件 指令？ 选择集

片段名 -> 除了`on`以外的命名

片段是GraphQL组合拼装的基本单元，它通用选择集字段的重用得以实现，减少了文档中的重复文本。内联片段可以直接在选择集合内使用，通常用于interface（接口）或者union（联合）这种存在类型条件的场合。

例如，我们想要获取某个用户的朋友以及和他互为朋友的人的共通信息：

```GraphQL
query noFragments {
  user(id: 4) {
    friends(first: 10) {
      id
      name
      profilePic(size: 50)
    }
    mutualFriends(first: 10) {
      id
      name
      profilePic(size: 50)
    }
  }
}
```

这些重复的字段可以提取进一个fragment（片段）中，然后被父级fragment（片段）或者query（查询）组合：

```GraphQL
query withFragments {
  user(id: 4) {
    friends(first: 10) {
      ...friendFields
    }
    mutualFriends(first: 10) {
      ...friendFields
    }
  }
}

fragment friendFields on User {
  id
  name
  profilePic(size: 50)
}
```

片段可以通过解构操作符(`...`)被消费掉，片段内的字段将会被添加到片段被调用的同层级选择集合，这一过程也会在多级别片段中解构发生。

例如:

```GraphQL
query withNestedFragments {
  user(id: 4) {
    friends(first: 10) {
      ...friendFields
    }
    mutualFriends(first: 10) {
      ...friendFields
    }
  }
}

fragment friendFields on User {
  id
  name
  ...standardProfilePic
}

fragment standardProfilePic on User {
  profilePic(size: 50)
}
```

`noFragments`，`withFragments`和`withNestedFragments`三个查询都会产生相同的返回对象。


### Type Conditions/类型条件

TypeCondition : on NamedType

类型条件 -> on 具名类型

片段需要指定应用于的目标类型，在上述案例中，`friendFields`在查询`User`的上下文中使用。

片段不能应用于任何输入值（标量值，枚举型或者输入型对象）。

片段可应用与对象型，接口和联合。

只有在对象的具体类型和片段的应用目标类型匹配的时候，片段内的选择集合才会返回值。

譬如，下列Facebook数据模型查询：

```GraphQL
query FragmentTyping {
  profiles(handles: ["zuck", "cocacola"]) {
    handle
    ...userFragment
    ...pageFragment
  }
}

fragment userFragment on User {
  friends {
    count
  }
}

fragment pageFragment on Page {
  likers {
    count
  }
}
```

`profiles`根字段将会返回一个列表，其中的元素可能是`Page`或者`User`类型。当`profiles`内的对象是`User`类型时，`friends`会出现，而`likers`不会。反之当结果内的对象是`Page`时，`likers`会出现，`friends`则不会。

```json
{
  "profiles": [
    {
      "handle": "zuck",
      "friends": { "count" : 1234 }
    },
    {
      "handle": "cocacola",
      "likers": { "count" : 90234512 }
    }
  ]
}
```


### Inline Fragments/内联片段

InlineFragment : ... TypeCondition? Directives? SelectionSet

内联/行内片段 -> ... 类型条件？ 指令？ 选择集合？

片段可以在选择集合内以内联格式定义，这用于根据运行时类型条件式地引入字段。这个特性的标准片段引入版本在`query FragmentTyping`中已经演示，我们也可以使用内联片段的方式来实现：

```GraphQL
query inlineFragmentTyping {
  profiles(handles: ["zuck", "cocacola"]) {
    handle
    ... on User {
      friends {
        count
      }
    }
    ... on Page {
      likers {
        count
      }
    }
  }
}
```

内联片段也用于将指令应用于一群字段的场景。如果省略了类型条件，片段则被视为等同于封装所在的上下文。

```GraphQL
query inlineFragmentNoType($expandedInfo: Boolean) {
  user(handle: "zuck") {
    id
    name
    ... @include(if: $expandedInfo) {
      firstName
      lastName
      birthday
    }
  }
}
```


## Input Values/输入值

Value[Const] :
  - [~Const] Variable
  - IntValue
  - FloatValue
  - StringValue
  - BooleanValue
  - NullValue
  - EnumValue
  - ListValue[?Const]
  - ObjectValue[?Const]

值 ->
 - 变量
 - 整数值
 - 浮点值
 - 字符串值
 - 布尔值
 - 空值
 - 枚举值
 - 列表值
 - 对象值
 
 字段和指令的参数接受各种原始类型的输入值，输入值可以是标量值、枚举值、列表值或者输入型对象。
 
如果没有定义为常量（例如，{DefaultValue}（默认值）），输入值就可以被指定为变量，列表和输入对象也可以包含变量（除非被定义为常量）。


### Int Value/整数值

IntValue :: IntegerPart

IntegerPart ::
  - NegativeSign? 0
  - NegativeSign? NonZeroDigit Digit*

NegativeSign :: -

Digit :: one of 0 1 2 3 4 5 6 7 8 9

NonZeroDigit :: Digit but not `0`

指定整数不应该使用小数点或指数符号。(譬如：`1`)


### Float Value/浮点值

FloatValue ::
  - IntegerPart FractionalPart
  - IntegerPart ExponentPart
  - IntegerPart FractionalPart ExponentPart

FractionalPart :: . Digit+

ExponentPart :: ExponentIndicator Sign? Digit+

ExponentIndicator :: one of `e` `E`

Sign :: one of + -

指定浮点数需要包含小数点(例如：`1.0`)或者指数符号(例如：`1e50`)或者两者(例如：`6.0221413e23`)。


### Boolean Value/布尔值

BooleanValue : one of `true` `false`

`true`和`false`两个关键字表示布尔型的两个值。


### String Value/字符串值

StringValue ::
  - `""`
  - `"` StringCharacter+ `"`

StringCharacter ::
  - SourceCharacter but not `"` or \ or LineTerminator
  - \u EscapedUnicode
  - \ EscapedCharacter

EscapedUnicode :: /[0-9A-Fa-f]{4}/

EscapedCharacter :: one of `"` \ `/` b f n r t

字符串是一系列由双引号(`"`)包起来的字符，譬如`"Hello World"`。字符串内的空白符和其他无语义字符都对字符串有影响。

Note: 字符串值字面量内是允许Unicode字符的，但是GraphQL源文本不允许ASCII控制符，因此如要使用这些字符，则需对其进行转义。

**Semantics/语义**

StringValue :: `""`

  * 返回空Unicode字符序列。

StringValue :: `"` StringCharacter+ `"`

  * 返回{StringCharacter}的Unicode字符序列。

StringCharacter :: SourceCharacter but not `"` or \ or LineTerminator

  * 返回{SourceCharacter}的字符值。

StringCharacter :: \u EscapedUnicode

  * 返回{EscapedUnicode}在Unicode基本多文种平面内的16进制对应代码单元。<small>译者案：详情查询Unicode字符平面映射</small>

StringCharacter :: \ EscapedCharacter

  * 返回{EscapedCharacter}在此对照表中的值。

| 转义后字符 | 字符单元值  | 字符名称  |
| -------- | ---------- | -------- |
| `"`      | U+0022     | 双引号    |
| `\`      | U+005C     | 反斜线    |
| `/`      | U+002F     | 正斜线    |
| `b`      | U+0008     | 退格      |
| `f`      | U+000C     | 换页符    |
| `n`      | U+000A     | 换行符    |
| `r`      | U+000D     | 回车符    |
| `t`      | U+0009     | 水平制表符|


### Null Value/空值

NullValue : `null`

{null}代表空值。

GraphQL在语义上有两种方式来表示一个缺值：

  * 显式，使用字面量值：{null}。
  * 隐式，不使用任何值。

例如：下列两个相似的字段查询并不是一样的：

```GraphQL
{
  field(arg: null)
  field
}
```

前者显式使用了{null}给arg参数，后者隐式，没有给arg参数以值。两个形式会被不同解读，譬如一个mutation操作中，会表示成删除一个字段或者不改变一个字段。两者都不能用于非空输入类型参数。

Note: 在使用变量表示一个缺值的时候，也可以使用这两种方法，亦即显式提供{null}，或者隐式，不提供任何值。


### Enum Value/枚举值

EnumValue : Name but not `true`, `false` or `null`

枚举值表现为没有引号包裹的名称，规范建议使用全大写字母表示枚举值，枚举值仅用于准确枚举类型可用的上下文中，因此枚举类型命名上不必使用字面量。


### List Value/列表值

ListValue[Const] :
  - [ ]
  - [ Value[?Const]+ ]

列表是包在方括号`[ ]`中的有序值序列，列表值可以是任意字面量值或者变量，譬如`[1, 2, 3]`。

因为GraphQL中逗号是可选的，因此末尾逗号和重复逗号都是允许的，而不会代表空缺值。

**Semantics/语义**

ListValue : [ ]

  * 返回空列表值。

ListValue : [ Value+ ]

  * 使{inputList}为空.
  * 对于每一个{Value+}
    * 让{value}变成{Value}求值后的值。
    * 将{value}添加到{inputList}尾部。
  * 返回{inputList}。


### Input Object Values/输入型对象值

ObjectValue[Const] :
  - { }
  - { ObjectField[?Const]+ }

ObjectField[Const] : Name : Value[?Const]

输入型对象是无需键值列表，使用花括号`{ }`包起来。对象的值可以是输入字面量值或者变量值(例如：`{ name: "Hello world", score: 1.0 }`)。我们将输入型对象的字面量表示法称作“对象字面量”。

**Input object fields are unordered/输入型对象的字段是无序的**

输入型字段能以各种句法顺位排列，而表示相同的语义。

下面两个查询在语义上是一样的：

```GraphQL
{
  nearestThing(location: { lon: 12.43, lat: -53.211 })
}
```

```GraphQL
{
  nearestThing(location: { lat: -53.211, lon: 12.43 })
}
```

**Semantics/语义**

ObjectValue : { }

  * 返回一个无字段的输入型对象。

ObjectValue : { ObjectField+ }

  * 使{inputObject}为一个无字段的输入型对象。
  * 对于{inputObject}中的每一个{field}。
    * 使{field}的{Name}为{name}。
    * 使{field}的{Value}为{value}求值后的值。
    * 给{inputObject}添加一个字段，键为{name}，值为{value}。
  * 返回{inputObject}


## Variables/变量

Variable : $ Name

VariableDefinitions : ( VariableDefinition+ )

VariableDefinition : Variable : Type DefaultValue?

DefaultValue : = Value[Const]

变量 -> $ 名称

变量定义 -> 变量：类型 默认值？

默认值 -> 值<small>常量</small>

GraphQL查询可以使用变量作为参数，已最大化查询重用，避免客户端运行时耗费巨大的字符串重建。

如果没有被定义为常量（例如{DefaultValue}），{Variable}就能被赋予一个输入类型。

变量必须在查询的顶部定义，并在整个操作的执行周期范围内可用。

下面例子中，我们想要根据特定设备大小获取一个对应大小的档案图片：

```GraphQL
query getZuckProfile($devicePicSize: Int) {
  user(id: 4) {
    id
    name
    profilePic(size: $devicePicSize)
  }
}
```

在向GraphQL请求的时候，这些参数的值也需要一并发送，以便执行期间用以替换。假设以JSON发送参数，我们请求大小为`60`宽度的档案照片：

```json
{
  "devicePicSize": 60
}
```

**Variable use within Fragments/片段内的参数**

片段内也可以使用查询变量，变量在整个操作中拥有全局作用域，所以片段内变量也需要在顶部操作上定义，以便传递到片段上消费。如果一个参数被片段所引用，包含这个片段的操作并没有定义这个变量，那么这个操作将不会被执行。


## Input Types/输入类型

Type :
  - NamedType
  - ListType
  - NonNullType

NamedType : Name

ListType : [ Type ]

NonNullType :
  - NamedType !
  - ListType !

类型 ->
  - 具名类型
  - 列表类型
  - 非空类型
  
具名类型 -> 命名

列表类型 -> \[ 类型 \]

非空类型 ->
  - 具名类型!  
  - 列表类型!

GraphQL描述查询参数需要的类型为输入类型，可以是某种其他输入类型的列表或者其他输入类型的非空变体。

**Semantics/语义**

Type : Name

  * 使{name}为{Name}的值。
  * 使{type}为Schema中定义的类型的{name}
  * {type}不可为{null}
  * 返回{type}

Type : [ Type ]

  * 使{itemType}为{Type}求值后的值。
  * 使{type}为一个列表类型，其中内部类型为{itemType}。
  * 返回{type}。

Type : Type !

  * 使{nullableType}为{Type}计算后的值。
  * 使{Type}为非空类型，其中内部类型为{nullableType}。
  * 返回{type}。


## Directives/指令

Directives : Directive+

Directive : @ Name Arguments?

指令 -> @名称 参数?
 
指令为GraphQL文档提供了另一种运行时执行行为和类型验证行为。

有时候，你需要改变GraphQL的执行行为，而参数并不满足要求，譬如条件性的包含或者跳过一个字段。指令通过向执行器描述附加信息来完成这种需求。

指令需要一个名字和一组参数，可以接受任意输入类型。

指令可以用于描述类型、字段、片段、操作的附加信息。

将来版本的GraphQL会加入可配置的执行能力，他们则可能表现为指令。
