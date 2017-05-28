# GraphQL

此文为GraphQL的一份草稿规范，其中GraphQL是Facebook开发的一套API查询语言。

此规范的目标受众并非客户端开发者，而是有志于亦或正在构建GraphQL实现和工具的人。

为了将GraphQL更广泛的应用于各种后台、框架及语言，需借助跨项目和跨组织间的通力合作，而此规范则提供了协作的基准。

如需帮助，可从[社区](http://graphql.org/community/)中寻找资源。

## Getting Started

GraphQL包含类型系统、查询语言、执行语义、静态验证和类型自省等组件，下文将举例描述GraphQL的这些组件。

这些案例并不复杂，它们仅用于让你在深入了解规范细节或者[GraphQL.js](https://github.com/graphql/graphql-js)参考实现之前，快速入门GraphQL的核心概念。

这些案例~~的前提是我们想要~~使用GraphQL查询《星球大战》三部曲中的人物地点信息。

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

在我们的《星球大战》案例中, GraphQL.js库的[starWarsQueryTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsQuery-test.js)文件包含一系列查询及返回。
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

### Validation

By using the type system, it can be predetermined whether a GraphQL query
is valid or not. This allows servers and clients to effectively inform
developers when an invalid query has been created, without having to rely
on runtime checks.

For our Star Wars example, the file
[starWarsValidationTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsValidation-test.js)
contains a number of queries demonstrating various invalidities, and is a test
file that can be run to exercise the reference implementation's validator.

To start, let's take a complex valid query. This is the `NestedQuery` example
from the above section, but with the duplicated fields factored out into
a fragment:

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

And this query is valid. Let's take a look at some invalid queries!

When we query for fields, we have to query for a field that exists on the
given type. So as `hero` returns a `Character`, we have to query for a field
on `Character`. That type does not have a `favoriteSpaceship` field, so this
query:

```GraphQL
# INVALID: favoriteSpaceship does not exist on Character
query HeroSpaceshipQuery {
  hero {
    favoriteSpaceship
  }
}
```

is invalid.

Whenever we query for a field and it returns something other than a scalar
or an enum, we need to specify what data we want to get back from the field.
Hero returns a `Character`, and we've been requesting fields like `name` and
`appearsIn` on it; if we omit that, the query will not be valid:

```GraphQL
# INVALID: hero is not a scalar, so fields are needed
query HeroNoFieldsQuery {
  hero
}
```

Similarly, if a field is a scalar, it doesn't make sense to query for
additional fields on it, and doing so will make the query invalid:

```GraphQL
# INVALID: name is a scalar, so fields are not permitted
query HeroFieldsOnScalarQuery {
  hero {
    name {
      firstCharacterOfName
    }
  }
}
```

Earlier, it was noted that a query can only query for fields on the type
in question; when we query for `hero` which returns a `Character`, we
can only query for fields that exist on `Character`. What happens if we
want to query for R2-D2s primary function, though?

```GraphQL
# INVALID: primaryFunction does not exist on Character
query DroidFieldOnCharacter {
  hero {
    name
    primaryFunction
  }
}
```

That query is invalid, because `primaryFunction` is not a field on `Character`.
We want some way of indicating that we wish to fetch `primaryFunction` if the
`Character` is a `Droid`, and to ignore that field otherwise. We can use
the fragments we introduced earlier to do this. By setting up a fragment defined
on `Droid` and including it, we ensure that we only query for `primaryFunction`
where it is defined.

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

This query is valid, but it's a bit verbose; named fragments were valuable
above when we used them multiple times, but we're only using this one once.
Instead of using a named fragment, we can use an inline fragment; this
still allows us to indicate the type we are querying on, but without naming
a separate fragment:

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

This has just scratched the surface of the validation system; there
are a number of validation rules in place to ensure that a GraphQL query
is semantically meaningful. The specification goes into more detail about this
topic in the "Validation" section, and the
[validation](https://github.com/graphql/graphql-js/blob/master/src/validation)
directory in GraphQL.js contains code implementing a
specification-compliant GraphQL validator.

### Introspection

It's often useful to ask a GraphQL schema for information about what
queries it supports. GraphQL allows us to do so using the introspection
system!

For our Star Wars example, the file
[starWarsIntrospectionTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsIntrospection-test.js)
contains a number of queries demonstrating the introspection system, and is a
test file that can be run to exercise the reference implementation's
introspection system.

We designed the type system, so we know what types are available, but if
we didn't, we can ask GraphQL, by querying the `__schema` field, always
available on the root type of a Query. Let's do so now, and ask what types
are available.

```GraphQL
query IntrospectionTypeQuery {
  __schema {
    types {
      name
    }
  }
}
```

and we get back:

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

Wow, that's a lot of types! What are they? Let's group them:

 - **Query, Character, Human, Episode, Droid** - These are the ones that we
defined in our type system.
 - **String, Boolean** - These are built-in scalars that the type system
provided.
 - **__Schema, __Type, __TypeKind, __Field, __InputValue, __EnumValue,
__Directive** - These all are preceded with a double underscore, indicating
that they are part of the introspection system.

Now, let's try and figure out a good place to start exploring what queries are
available. When we designed our type system, we specified what type all queries
would start at; let's ask the introspection system about that!

```GraphQL
query IntrospectionQueryTypeQuery {
  __schema {
    queryType {
      name
    }
  }
}
```

and we get back:

```json
{
  "__schema": {
    "queryType": {
      "name": "Query"
    }
  }
}
```

And that matches what we said in the type system section, that
the `Query` type is where we will start! Note that the naming here
was just by convention; we could have named our `Query` type anything
else, and it still would have been returned here if we had specified it
as the starting type for queries. Naming it `Query`, though, is a useful
convention.

It is often useful to examine one specific type. Let's take a look at
the `Droid` type:


```GraphQL
query IntrospectionDroidTypeQuery {
  __type(name: "Droid") {
    name
  }
}
```

and we get back:

```json
{
  "__type": {
    "name": "Droid"
  }
}
```

What if we want to know more about Droid, though? For example, is it
an interface or an object?

```GraphQL
query IntrospectionDroidKindQuery {
  __type(name: "Droid") {
    name
    kind
  }
}
```

and we get back:

```json
{
  "__type": {
    "name": "Droid",
    "kind": "OBJECT"
  }
}
```

`kind` returns a `__TypeKind` enum, one of whose values is `OBJECT`. If
we asked about `Character` instead:


```GraphQL
query IntrospectionCharacterKindQuery {
  __type(name: "Character") {
    name
    kind
  }
}
```

and we get back:

```json
{
  "__type": {
    "name": "Character",
    "kind": "INTERFACE"
  }
}
```

We'd find that it is an interface.

It's useful for an object to know what fields are available, so let's
ask the introspection system about `Droid`:

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

and we get back:

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

Those are our fields that we defined on `Droid`!

`id` looks a bit weird there, it has no name for the type. That's
because it's a "wrapper" type of kind `NON_NULL`. If we queried for
`ofType` on that field's type, we would find the `String` type there,
telling us that this is a non-null String.

Similarly, both `friends` and `appearsIn` have no name, since they are the
`LIST` wrapper type. We can query for `ofType` on those types, which will
tell us what these are lists of.

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

and we get back:

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

Let's end with a feature of the introspection system particularly useful
for tooling; let's ask the system for documentation!

```GraphQL
query IntrospectionDroidDescriptionQuery {
  __type(name: "Droid") {
    name
    description
  }
}
```

yields

```json
{
  "__type": {
    "name": "Droid",
    "description": "A mechanical creature in the Star Wars universe."
  }
}
```

So we can access the documentation about the type system using introspection,
and create documentation browsers, or rich IDE experiences.

This has just scratched the surface of the introspection system; we can
query for enum values, what interfaces a type implements, and more. We
can even introspect on the introspection system itself. The specification goes
into more detail about this topic in the "Introspection" section, and the [introspection](https://github.com/graphql/graphql-js/blob/master/src/type/introspection.js)
file in GraphQL.js
contains code implementing a specification-compliant GraphQL query
introspection system.

### Additional Content

This README walked through the GraphQL.js reference implementation's type
system, query execution, validation, and introspection systems. There's more
in both [GraphQL.js](https://github.com/graphql/graphql-js/) and specification,
including a description and implementation for executing queries, how to format
a response, explaining how a type system maps to an underlying implementation,
and how to format a GraphQL response, as well as a grammar for GraphQL.
