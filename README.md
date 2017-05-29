# GraphQL

此文为GraphQL的一份草稿规范，其中GraphQL是Facebook开发的一套API查询语言。

此规范的目标受众并非客户端开发者，而是有志于亦或正在构建GraphQL实现和工具的人们。

为了将GraphQL更广泛的应用于各种后台、框架及语言，需借助跨项目和跨组织间的通力合作，而此规范则提供了协作的基准。

如需帮助，可从[社区](http://graphql.org/community/)中寻找资源。

## Getting Started

GraphQL包含类型系统、查询语言、执行语义、静态验证和类型自省等组件，下文将举例描述GraphQL的这些组件。

这些案例并不复杂，它们仅用于让你在深入了解规范细节或者[GraphQL.js](https://github.com/graphql/graphql-js)参考实现之前，快速入门GraphQL的核心概念。

这些案例使用GraphQL查询《星球大战》三部曲中的人物地点信息。

### Type System/类型系统

所有GraphQL实现的核心都是在GraphQL Schema中用类型系统来描述可以返回的对象。对于我们的《星球大战》案例，
[starWarsSchema.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsSchema.js)文件种定义了如下类型系统。

最基本的类型是`Human`（人类），表示Luke、Leia、Han等角色。每个角色都会有一个名字，因此我们的类型`Human`也会有个字段`name`，属于"String"类型，并且不为空。于是我们定义`name`字段为不为空的String，用我们将在下文使用的一种简记法表示如下：

```GraphQL
type Human {
  name: String
}
```

这种简记法十分便于描述类型系统中的基础模型，JS的参考实现中还实现了备注等的全功能类型系统。它建立了类型系统和下层数据之间的映射；基于测试目，[这儿](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsData.js)的下层数据是一组JS对象，但是大多数情况下，后台数据都是通过一些服务(service)来获取，此时类型系统层的作用便是将类型和字段映射到服务上去。

通常的API都会给每个对象赋予一个唯一ID，用于重取特定对象，GraphQL中也采用了这种模式，以我们的`Human`类型举例（顺便添加了个String类型的homePlanet字段）。

```GraphQL
type Human {
  id: String
  name: String
  homePlanet: String
}
```

因为我们在讨论星球大站三部曲，所以给每个人物标注出现在哪一部里面吧。首先定义一个枚举型，列举了三部曲的片名。

```GraphQL
enum Episode { NEWHOPE, EMPIRE, JEDI }
```

然后我们给`Human`添加一个字段，用于描述这个人物出场的剧集名称,类型为`Episode`的数组。

```GraphQL
type Human {
  id: String
  name: String
  appearsIn: [Episode]
  homePlanet: String
}
```

接着我们引入另一个类型`Droid`（机器人）:

```GraphQL
type Droid {
  id: String
  name: String
  appearsIn: [Episode]
  primaryFunction: String
}
```

现在我们有两种类型了，需要一个方法来关联他们：人类和机器人都会有朋友，而朋友也可能是人类或者机器人，怎么将`friends`（朋友）字段关联到人类或者机器人上呢？

仔细看看，我们发现人类和机器人具有一些共性，都用ID、名字、出场剧集的名称。因此我们添加一个interface（接口）`Character`，并且让`Human`和`Droid`都实现它，从而添加`friends`（朋友）字段，返回`Character`数组。

这样下来，我们的类型系统就变成了这样

```GraphQL
enum Episode { NEWHOPE, EMPIRE, JEDI }

interface Character {
  id: String
  name: String
  friends: [Character]
  appearsIn: [Episode]
}

type Human implements Character {
  id: String
  name: String
  friends: [Character]
  appearsIn: [Episode]
  homePlanet: String
}

type Droid implements Character {
  id: String
  name: String
  friends: [Character]
  appearsIn: [Episode]
  primaryFunction: String
}
```

我们可能有疑问：这些字段能否返回`null`（空）呢？默认情况下，GraphQL的所有类型可以为空，因为获取GraphQL所需要的数据通常需要联络多个服务，它们不见得任何时刻都可用，当然如果类型系统能够保证特定类型不为空，那就可以将指定类型标上Non Null（非空），在我们的简记法中，在类型后面加一个"!"就行。

注意，虽然我们现在的实现能够保证多个字段不为空（因为硬编码的），但我们并没有将其标注未非空，因为我们可能将硬编码内容替换为一个后台服务，而这个服务可能就没那么可靠了，所以对应字段可以是可空型，这样在服务出错的时候可以返回空数据，技能给类型系统一定的灵活性，同时也能向客户端通报后台的错误。

```GraphQL
enum Episode { NEWHOPE, EMPIRE, JEDI }

interface Character {
  id: String!
  name: String
  friends: [Character]
  appearsIn: [Episode]
}

type Human implements Character {
  id: String!
  name: String
  friends: [Character]
  appearsIn: [Episode]
  homePlanet: String
}

type Droid implements Character {
  id: String!
  name: String
  friends: [Character]
  appearsIn: [Episode]
  primaryFunction: String
}
```

我们还缺少拼图的最后一块儿：类型系统的入口。

我们来schema(模式)，首先定义个对象类型作为所有查询的基础，按照惯例这个类型的名称是`Query`，它描述类型系统的顶级公开API，我们的案例的`Query`如下：

```GraphQL
type Query {
  hero(episode: Episode): Character
  human(id: String!): Human
  droid(id: String!): Droid
}
```

案例中我们的schema有三个顶级操作:

 - `hero`返回`Character`类型，它是《星球大战》的主角;它接受一个可选参数用以获取特定剧集的主角。
 - `human`接受一个非空String型参数（人类的ID），返回这个ID对应的人类。
 - `droid`类似，返回机器人。

这些字段展示了另一个类型系统的特性：字段可以接收参数从而返回特定值。

当将整个类型系统打包的时候，在入口上定义`Query`以接受查询，就能够生成GraphQL Schema。

这个案例只是类型系统的冰山一角，本规范将在"Type System"（类型系统）章节更加深入细致地探讨。GraphQL.js的[type](https://github.com/graphql/graphql-js/blob/master/src/type)（类型）目录包含一套兼容GraphQL类型系统规范的实现代码。

### Query Syntax/查询语法

GraphQL查询语句声明式地描述了"取回什么样的数据"，而不管数据来源，只要数据提供者能满足GraphQL查询语句的要求就行。

在我们的《星球大战》案例中, GraphQL.js库的[starWarsQueryTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsQuery-test.js)文件包含若干查询及返回。
这是个测试文件，使用了上述的schema和一组样本数据，数据在[starWarsData.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsData.js)。这个测试文件是用于检测参考实现的。

查询上述schema的样例语句如下：

```GraphQL
query HeroNameQuery {
  hero {
    name
  }
}
```

首行的`query HeroNameQuery`以schema根级类型`Query`起头，定义了一个名为`HeroNameQuery`的查询操作。如上文所述，`Query`拥有一个`hero`字段，返回`Character`类型，这是我们要查的，`Character`具有一个`name`字段，返回`String`类型，这也是我们要查的，这样就完成了一个查询语句。其返回结果可能如下：

```json
{
  "hero": {
    "name": "R2-D2"
  }
}
```

只有在一个GraphQL文档定义了多个操作的时候，才需要指定`query`关键字和操作名。所以我们上面的查询可以简写为：

```GraphQL
{
  hero {
    name
  }
}
```

假设R2-D2被后台数据当成hero，如果我们请求R2-D2的ID和朋友，那么返回值会根据我们的查询变化而变化：

```GraphQL
query HeroNameAndFriendsQuery {
  hero {
    id
    name
    friends {
      id
      name
    }
  }
}
```

返回值会是：

```json
{
  "hero": {
    "id": "2001",
    "name": "R2-D2",
    "friends": [
      {
        "id": "1000",
        "name": "Luke Skywalker"
      },
      {
        "id": "1002",
        "name": "Han Solo"
      },
      {
        "id": "1003",
        "name": "Leia Organa"
      }
    ]
  }
}
```

GraphQL的一个关键特性即是嵌套查询(nested query)。上述案例中，我们查询了R2-D2的friends（朋友），并可以查询了这些对象的进一步信息。我们来构造一个查询语句，用来查询R2-D2和它朋友们的name（名字）和出场episode（剧集）：

```GraphQL
query NestedQuery {
  hero {
    name
    friends {
      name
      appearsIn
      friends {
        name
      }
    }
  }
}
```

然后会得到这个结果：

```json
{
  "hero": {
    "name": "R2-D2",
    "friends": [
      {
        "name": "Luke Skywalker",
        "appearsIn": [ "NEWHOPE", "EMPIRE", "JEDI" ],
        "friends": [
          { "name": "Han Solo" },
          { "name": "Leia Organa" },
          { "name": "C-3PO" },
          { "name": "R2-D2" }
        ]
      },
      {
        "name": "Han Solo",
        "appearsIn": [ "NEWHOPE", "EMPIRE", "JEDI" ],
        "friends": [
          { "name": "Luke Skywalker" },
          { "name": "Leia Organa" },
          { "name": "R2-D2" }
        ]
      },
      {
        "name": "Leia Organa",
        "appearsIn": [ "NEWHOPE", "EMPIRE", "JEDI" ],
        "friends": [
          { "name": "Luke Skywalker" },
          { "name": "Han Solo" },
          { "name": "C-3PO" },
          { "name": "R2-D2" }
        ]
      }
    ]
  }
}
```

上面的`Query`类型定义了一种通过ID获取human(人类)的信息，我们将ID硬编码再查询语句中：

```GraphQL
query FetchLukeQuery {
  human(id: "1000") {
    name
  }
}
```

得到

```json
{
  "human": {
    "name": "Luke Skywalker"
  }
}
```

其外我们也可以在查询语句中定义查询参数:

```GraphQL
query FetchSomeIDQuery($someId: String!) {
  human(id: $someId) {
    name
  }
}
```

现在查询语句里面有了参数`$someId`，如果想要运行，我们需要提供ID，譬如1000对应Luke，1002对应Han，如果传递的是无效ID，那么就会得到`null`，表示没有哪个对象。

注意，默认情况下，返回内容的名字和字段名一致，有时候有必要修改键名，以避免键名冲突（譬如以不同参数获取相同字段）。

我们通过字段别名来实现：

```GraphQL
query FetchLukeAliased {
  luke: human(id: "1000") {
    name
  }
}
```

我们将`human`字段别名为键名`luke`，于是返回内容：
is:

```json
{
  "luke": {
    "name": "Luke Skywalker"
  }
}
```

注意键名为"luke"而不是"human"，因为它存在前一案例中，所以我们不使用这个别名。

特别是我们想要在一次查询中使用不同参数查询两个相同字段，如下所示：

```GraphQL
query FetchLukeAndLeiaAliased {
  luke: human(id: "1000") {
    name
  }
  leia: human(id: "1003") {
    name
  }
}
```

我们将第一个`human`字段别名为`luke`，第二个别名为`leia`。得到如下结果：

```json
{
  "luke": {
    "name": "Luke Skywalker"
  },
  "leia": {
    "name": "Leia Organa"
  }
}
```

如果我们想要得到Luke和Leia的home planets（母星），我们可以如下构件查询语句：

```GraphQL
query DuplicateFields {
  luke: human(id: "1000") {
    name
    homePlanet
  }
  leia: human(id: "1003") {
    name
    homePlanet
  }
}
```

但是这样写依然不够明智，因为我们在两部分添加了同样的内容。我们提取共同字段，放进一个fragment（片段）里面，然后在查询语句中包含这个片段，就像这样：

```GraphQL
query UseFragment {
  luke: human(id: "1000") {
    ...HumanFragment
  }
  leia: human(id: "1003") {
    ...HumanFragment
  }
}

fragment HumanFragment on Human {
  name
  homePlanet
}
```

上述两个查询都会返回一样的结果：

```json
{
  "luke": {
    "name": "Luke Skywalker",
    "homePlanet": "Tatooine"
  },
  "leia": {
    "name": "Leia Organa",
    "homePlanet": "Alderaan"
  }
}
```

`UseFragment`和`DuplicateFields`查询都会获得一样的结果，但是`UseFragment`更简洁，如果我们需要获取更多的字段，直接在共有的fragment（片段）中加而不是复制到多个地方。

我们之前定义了类型系统，所以我们知道返回值的每一个对象的类型；而查询语句也能通过特殊字段`__typename`来查询每个对象的类型：

```GraphQL
query CheckTypeOfR2 {
  hero {
    __typename
    name
  }
}
```

因为R2-D2是机器人,所以得到：

```json
{
  "hero": {
    "__typename": "Droid",
    "name": "R2-D2"
  }
}
```

因为`hero`返回的类型`Character`是一个接口，所以在这儿这种查询十分有用，如果我们想要知道实际返回的具体类型的话。如果我们想要查询Episode V（第五集）的hero（主角）：

```GraphQL
query CheckTypeOfLuke {
  hero(episode: EMPIRE) {
    __typename
    name
  }
}
```

于是得到主角是Luke，他是个Human(人类)：

```json
{
  "hero": {
    "__typename": "Human",
    "name": "Luke Skywalker"
  }
}
```

跟类型系统一样，这个案例也只是查询语言的冰山一角。本规范的"Language"（语言）章节会有更加深入细致的讨论。GraphQL.js库的[language](https://github.com/graphql/graphql-js/blob/master/src/language)（语言）目录包含了一套兼容GraphQL查询规范的语言分析器和词法分析器。

### Validation/验证

通过使用类型系统，你可以预先判定一个GraphQL查询是否有效。这能让服务端和客户端有效地给开发者预先通告当前查询语句是否有效，而不必只能依赖运行时检查。

我们的《星球大战》案例中，[starWarsValidationTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsValidation-test.js)文件包含了若干使用了验证的查询，这是个测试文件，用于检测参考实现的验证器。

首先，我们构造一个有效的复杂查询，这是来自上文的`NestedQuery`案例，其中的重复字段已经被提取到了一个fragment片段中：

```GraphQL
query NestedQueryWithFragment {
  hero {
    ...NameAndAppearances
    friends {
      ...NameAndAppearances
      friends {
        ...NameAndAppearances
      }
    }
  }
}

fragment NameAndAppearances on Character {
  name
  appearsIn
}
```

当然这个查询是有效的。那我们再来看看无效的查询！

当我查询某些字段的时候，我们查询存在于某个Type（类型）中的字段，譬如`hero`返回的是`Character`，我们就查询的字段就必须在`Character`内存在。如果查询不存在字段，譬如`favoriteSpaceship`：

```GraphQL
# INVALID: favoriteSpaceship does not exist on Character（无效：Character中不存在favoriteSpaceship）
query HeroSpaceshipQuery {
  hero {
    favoriteSpaceship
  }
}
```

这个查询就是无效的了。

如果我们查询的字段返回的不是标量或者枚举型，那么还需要指定我们所需要的内部的字段。譬如`hero`返回的是`Character`，我们在之前的案例中请求过了`name`或者`appearsIn`之类的字段。如果我们省略这些，查询就变成无效的了：

```GraphQL
# INVALID: hero is not a scalar, so fields are needed（无效：hero不是标量，需要提供内部字段）
query HeroNoFieldsQuery {
  hero
}
```

类似的，如果一个字段是标量，取内部字段也没有意义，那样做会导致查询无效：

```GraphQL
# INVALID: name is a scalar, so fields are not permitted（无效：name是标量，不允许查询内部字段）
query HeroFieldsOnScalarQuery {
  hero {
    name {
      firstCharacterOfName
    }
  }
}
```

之前的案例中，你可能注意到查询语句中的字段只能是被请求类型上的，譬如我们请求的是`hero`，它会返回`Character`，我们就只能查询`Character`上的字段。如果我们要查询R2-D2的primaryFunction（基本功能），那么该怎么构建查询呢？

```GraphQL
# INVALID: primaryFunction does not exist on Character（无效，Character中不存在primaryFunction）
query DroidFieldOnCharacter {
  hero {
    name
    primaryFunction
  }
}
```

这个查询是无效的，因为Character中不存在primaryFunction字段，我们需要一种方法在`Character`是`Droid`的时候返回`primaryFunction`字段，否则就忽略。我们可以通过上文引入的fragment（片段）来解决这个问题：建立一个只包含`primaryFunction`的`Droid`片段，然后在查询中引入：

```GraphQL
query DroidFieldInFragment {
  hero {
    name
    ...DroidFields
  }
}

fragment DroidFields on Droid {
  primaryFunction
}
```

这个查询是有效的，但是有些啰嗦， named fragment（具名片段/命名片段）仅仅在多次使用的场景才能发挥作用，但是这儿只使用了一次。换言之，相较于具名片段，我们可以使用inline fragment（内联片段/行内片段），这样我们依然能查询我们需要的类型，但不必单独命名一个片段。

```GraphQL
query DroidFieldInInlineFragment {
  hero {
    name
    ... on Droid {
      primaryFunction
    }
  }
}
```

这也只是验证系统的冰山一角，这之外还有很多验证规则来保证一个查询语句的语义性，本规范将在 "Validation"（验证）章节更加深入细致地讨论。GraphQL.js的[validation](https://github.com/graphql/graphql-js/blob/master/src/validation)（验证）目录包含一套兼容GraphQL规范的验证器代码。

### Introspection/内省

我们经常需要知道一个GraphQL schema（模式）支持的所有查询类型，而GraphQL的introspection（内省）系统就是用来完成这个的。

我们的《星球大战》案例中，[starWarsIntrospectionTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsIntrospection-test.js)文件包含了若干使用了内省系统的查询。这是个测试文件，用于检测参考实现的内省系统。

我们定义了类型系统，所以我们知道那些类型是可用的，如果不知道，还可以通过向GraphQL的查询`__schema`字段得到这些，这个字段是一定存在于Query根级类型上的。不妨一试：

```GraphQL
query IntrospectionTypeQuery {
  __schema {
    types {
      name
    }
  }
}
```

然后得到:

```json
{
  "__schema": {
    "types": [
      {
        "name": "Query"
      },
      {
        "name": "Character"
      },
      {
        "name": "Human"
      },
      {
        "name": "String"
      },
      {
        "name": "Episode"
      },
      {
        "name": "Droid"
      },
      {
        "name": "__Schema"
      },
      {
        "name": "__Type"
      },
      {
        "name": "__TypeKind"
      },
      {
        "name": "Boolean"
      },
      {
        "name": "__Field"
      },
      {
        "name": "__InputValue"
      },
      {
        "name": "__EnumValue"
      },
      {
        "name": "__Directive"
      }
    ]
  }
}
```

有一大堆类型啊，他们都是些啥呢？我们将他们分个组：

 - **Query, Character, Human, Episode, Droid** - 这是我们在类型系统中定义的类型。
 - **String, Boolean** - 这是类型系统内置的标量。
 - **__Schema, __Type, __TypeKind, __Field, __InputValue, __EnumValue,
__Directive** - 这些都有个双下划线前缀，表明他们都属于内省系统。

现在让我们好好开始探讨一下那些查询是可用的吧！我们定义类型系统的时候，指定了所有类型从哪儿开始，看看怎么向内省系统查询：

```GraphQL
query IntrospectionQueryTypeQuery {
  __schema {
    queryType {
      name
    }
  }
}
```

然后得到了:

```json
{
  "__schema": {
    "queryType": {
      "name": "Query"
    }
  }
}
```

这符合我们在类型系统中说的`Query`是所有查询的起点，当然这个命名只是惯例，我们也可以将`Query`类型改成其他名字，它依然会返回，只是说`Query`作为约定俗称的惯例，最便于理解。

有时候也需要验证特定的类型，不妨看看`Droid`类型：


```GraphQL
query IntrospectionDroidTypeQuery {
  __type(name: "Droid") {
    name
  }
}
```

然后得到:

```json
{
  "__type": {
    "name": "Droid"
  }
}
```

如果我们想得到`Droid`的更多信息呢？譬如，他是个interface（接口）还是object（对象）呢？

```GraphQL
query IntrospectionDroidKindQuery {
  __type(name: "Droid") {
    name
    kind
  }
}
```

然后得到:

```json
{
  "__type": {
    "name": "Droid",
    "kind": "OBJECT"
  }
}
```

`kind`得到了`__TypeKind`枚举类型，其中之一便是`OBJECT`。如果我们查询`Character`：


```GraphQL
query IntrospectionCharacterKindQuery {
  __type(name: "Character") {
    name
    kind
  }
}
```

然后会得到:

```json
{
  "__type": {
    "name": "Character",
    "kind": "INTERFACE"
  }
}
```

我们发现他是个interface（接口）。

通常我们需要知道一个类型内有什么字段。继续以`Droid`为例，向内省系统查询：

```GraphQL
query IntrospectionDroidFieldsQuery {
  __type(name: "Droid") {
    name
    fields {
      name
      type {
        name
        kind
      }
    }
  }
}
```

然后得到:

```json
{
  "__type": {
    "name": "Droid",
    "fields": [
      {
        "name": "id",
        "type": {
          "name": null,
          "kind": "NON_NULL"
        }
      },
      {
        "name": "name",
        "type": {
          "name": "String",
          "kind": "SCALAR"
        }
      },
      {
        "name": "friends",
        "type": {
          "name": null,
          "kind": "LIST"
        }
      },
      {
        "name": "appearsIn",
        "type": {
          "name": null,
          "kind": "LIST"
        }
      },
      {
        "name": "primaryFunction",
        "type": {
          "name": "String",
          "kind": "SCALAR"
        }
      }
    ]
  }
}
```

这就是我们在`Droid`上定义的字段！

`id`看上去有些奇怪，它并没有类型名。那是因为他被`NON_NULL`类型封装。如果我们在字段的类型上查询`ofType`就能得到`String`，亦即它是一个non-null（非空）String。

类似的，`friends`和`appearsIn`也没有名字，因为他们是`LIST`封装类型。我们也能在它们上面查询`ofType`，然后得到他们是什么list。

```GraphQL
query IntrospectionDroidWrappedFieldsQuery {
  __type(name: "Droid") {
    name
    fields {
      name
      type {
        name
        kind
        ofType {
          name
          kind
        }
      }
    }
  }
}
```

然后得到：

```json
{
  "__type": {
    "name": "Droid",
    "fields": [
      {
        "name": "id",
        "type": {
          "name": null,
          "kind": "NON_NULL",
          "ofType": {
            "name": "String",
            "kind": "SCALAR"
          }
        }
      },
      {
        "name": "name",
        "type": {
          "name": "String",
          "kind": "SCALAR",
          "ofType": null
        }
      },
      {
        "name": "friends",
        "type": {
          "name": null,
          "kind": "LIST",
          "ofType": {
            "name": "Character",
            "kind": "INTERFACE"
          }
        }
      },
      {
        "name": "appearsIn",
        "type": {
          "name": null,
          "kind": "LIST",
          "ofType": {
            "name": "Episode",
            "kind": "ENUM"
          }
        }
      },
      {
        "name": "primaryFunction",
        "type": {
          "name": "String",
          "kind": "SCALAR",
          "ofType": null
        }
      }
    ]
  }
}
```

我们用一个内省系统在工具开发中特别有用的特性来收尾吧：向内省系统查询文档！

```GraphQL
query IntrospectionDroidDescriptionQuery {
  __type(name: "Droid") {
    name
    description
  }
}
```

得到

```json
{
  "__type": {
    "name": "Droid",
    "description": "A mechanical creature in the Star Wars universe."
  }
}
```

这样我们就能通过内省系统得到文档了，进一步制作文档阅读器，或者丰富IDE体验。

这也只是内省系统的冰山一角，我们可以查询枚举型的值，也可查询一个类型实现了什么interface（接口）等等，我们甚至可以内省这个内省系统本身，本规范将在"Introspection"（内省）章节更加深入细致地讨论。GraphQL.js的[introspection](https://github.com/graphql/graphql-js/blob/master/src/type/introspection.js)文件包含一套兼容GraphQL规范的查询内省系统代码。

### Additional Content/附加内容

这个README概述了GraphQL.js参考实现的类型系统、查询执行、验证器和内省系统。在[GraphQL.js](https://github.com/graphql/graphql-js/)和规范里面，能找到查询执行的描述和实现、如何格式化响应的描述和实现，并阐述了类型系统和下层实现之间的映射、如何格式化响应以及GraphQL的语法。
