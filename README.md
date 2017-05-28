# GraphQL

此文为GraphQL的一份草稿规范，其中GraphQL是Facebook开发的一套API查询语言。

此规范的目标受众并非客户端开发者，而是有志于亦或正在构建GraphQL实现和工具的人。

为了将GraphQL更广泛的应用于各种后台、框架及语言，需借助跨项目和跨组织间的通力合作，而此规范则提供了协作的基准。

如需帮助，可从[社区](http://graphql.org/community/)中寻找资源.

## Getting Started

GraphQL包含类型系统、查询语言、执行语义、静态验证和类型自省等组件.下文将举例描述GraphQL的这些组件。

这些案例并不复杂，它们仅用于让你在深入了解规范细节或者[GraphQL.js](https://github.com/graphql/graphql-js)参考实现之前，快速入门GraphQL的核心概念。

这些案例~~的前提是我们想要~~使用GraphQL查询《星球大战》三部曲中的人物地点信息。

### Type System/类型系统

所有GraphQL实现的核心都是在GraphQL Schema中用类型系统来描述可以返回的对象。对于我们的《星球大战》案例，
[starWarsSchema.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsSchema.js)文件种定义了如下类型系统。

最基本的类型是`Human`（人类），表示Luke、Leia、Han等角色。每个角色都会有一个名字，因此我们的类型`Human`也会有个字段`name`，属于"String"类型，并且不为空。于是我们定义`name`字段为不为空的String，用我们将在下文使用的一种简记法表示如下：

```
type Human {
  name: String
}
```

这种简记法十分便于描述类型系统中的基础模型，JS的参考实现中还实现了备注等的全功能类型系统。它建立了类型系统和下层数据之间的映射；基于测试目，[这儿](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsData.js)的下层数据是一组JS对象，但是大多数情况下，后台数据都是通过一些服务(service)来获取，此时类型系统层的作用便是将类型和字段映射到服务上去。

通常的API都会给每个对象赋予一个唯一ID，用于重取特定对象，GraphQL中也采用了这种模式，以我们的`Human`类型举例（顺便添加了个String类型的homePlanet字段）。

```
type Human {
  id: String
  name: String
  homePlanet: String
}
```

因为我们在讨论星球大站三部曲，所以给每个人物标注出现在哪一部里面吧。首先定义一个枚举型，列举了三部曲的片名。

```
enum Episode { NEWHOPE, EMPIRE, JEDI }
```

然后我们给`Human`添加一个字段，用于描述这个人物出场的剧集名称,类型为`Episode`的数组。

```
type Human {
  id: String
  name: String
  appearsIn: [Episode]
  homePlanet: String
}
```

接着我们引入另一个类型`Droid`（机器人）:

```
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

```
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

我们可能有疑问：这些字段能否返回`null`（空）呢？默认情况下，GraphQL的所有类型可以为空，因为获取GraphQL所需要的数据通常需要联络多个服务，它们不见得任何时刻都可用,，当然如果类型系统能够保证特定类型不为空，那就可以将指定类型标上Non Null（非空），在我们的简记法中，在类型后面加一个"!"就行。

注意，虽然我们现在的实现能够保证多个字段不为空（因为硬编码的），但我们并没有将其标注未非空，因为我们可能将硬编码内容替换为一个后台服务，而这个服务可能就没那么可靠了，所以对应字段可以是可空型，这样在服务出错的时候可以返回空数据，技能给类型系统一定的灵活性，同时也能向客户端通报后台的错误。

```
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

```
type Query {
  hero(episode: Episode): Character
  human(id: String!): Human
  droid(id: String!): Droid
}
```

案例中我们的schema有三个顶级操作:

 - `hero`返回`Character`类型，它是《星球大战》的主角;它接受一个可选参数用以获取特定剧集的主角。
 - `human`接受一个非空String型参数，人类的ID，返回这个ID对应的人类。
 - `droid`类似，返回机器人。

这些字段展示了另一个类型系统的特性：字段可以接收参数从而返回特定值。

当将整个类型系统打包的时候，在入口上定义`Query`以接受查询，就能够生成GraphQL Schema。

这个案例只是类型系统的冰山一角，本规范将在"Type System"（类型系统）章节更加深入细致地探讨。GraphQL.js的[type](https://github.com/graphql/graphql-js/blob/master/src/type)（类型）目录包含一套兼容GraphQL类型系统规范的实现代码。

### Query Syntax

GraphQL queries declaratively describe what data the issuer wishes
to fetch from whoever is fulfilling the GraphQL query.

For our Star Wars example, the
[starWarsQueryTests.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsQuery-test.js)
file in the GraphQL.js repository contains a number of queries and responses.
That file is a test file that uses the schema discussed above and a set of
sample data, located in
[starWarsData.js](https://github.com/graphql/graphql-js/blob/master/src/__tests__/starWarsData.js).
This test file can be run to exercise the reference implementation.

An example query on the above schema would be:

```
query HeroNameQuery {
  hero {
    name
  }
}
```

The initial line, `query HeroNameQuery`, defines a query with the operation
name `HeroNameQuery` that starts with the schema's root query type; in this
case, `Query`. As defined above, `Query` has a `hero` field that returns a
`Character`, so we'll query for that. `Character` then has a `name` field that
returns a `String`, so we query for that, completing our query. The result of
this query would then be:


```json
{
  "hero": {
    "name": "R2-D2"
  }
}
```

Specifying the `query` keyword and an operation name is only required when a
GraphQL document defines multiple operations.  We therefore could have written
the previous query with the query shorthand:

```
{
  hero {
    name
  }
}
```

Assuming that the backing data for the GraphQL server identified R2-D2 as the
hero. The response continues to vary based on the request; if we asked for
R2-D2's ID and friends with this query:

```
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

then we'll get back a response like this:

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

One of the key aspects of GraphQL is its ability to nest queries. In the
above query, we asked for R2-D2's friends, but we can ask for more information
about each of those objects. So let's construct a query that asks for R2-D2's
friends, gets their name and episode appearances, then asks for each of *their*
friends.

```
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

which will give us the nested response

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

The `Query` type above defined a way to fetch a human given their
ID. We can use it by hardcoding the ID in the query:

```
query FetchLukeQuery {
  human(id: "1000") {
    name
  }
}
```

to get

```json
{
  "human": {
    "name": "Luke Skywalker"
  }
}
```

Alternately, we could have defined the query to have a query parameter:

```
query FetchSomeIDQuery($someId: String!) {
  human(id: $someId) {
    name
  }
}
```

This query is now parameterized by `$someId`; to run it, we must provide
that ID. If we ran it with `$someId` set to "1000", we would get Luke;
set to "1002", we would get Han. If we passed an invalid ID here,
we would get `null` back for the `human`, indicating that no such object
exists.

Notice that the key in the response is the name of the field, by default.
It is sometimes useful to change this key, for clarity or to avoid key
collisions when fetching the same field with different arguments.

We can do that with field aliases, as demonstrated in this query:

```
query FetchLukeAliased {
  luke: human(id: "1000") {
    name
  }
}
```

We aliased the result of the `human` field to the key `luke`. Now the response
is:

```json
{
  "luke": {
    "name": "Luke Skywalker"
  }
}
```

Notice the key is "luke" and not "human", as it was in our previous example
where we did not use the alias.

This is particularly useful if we want to use the same field twice
with different arguments, as in the following query:

```
query FetchLukeAndLeiaAliased {
  luke: human(id: "1000") {
    name
  }
  leia: human(id: "1003") {
    name
  }
}
```

We aliased the result of the first `human` field to the key
`luke`, and the second to `leia`. So the result will be:

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

Now imagine we wanted to ask for Luke and Leia's home planets. We could do so
with this query:

```
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

but we can already see that this could get unwieldy, since we have to add new
fields to both parts of the query. Instead, we can extract out the common fields
into a fragment, and include the fragment in the query, like this:

```
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

Both of those queries give this result:

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

The `UseFragment` and `DuplicateFields` queries will both get the same result, but
`UseFragment` is less verbose; if we wanted to add more fields, we could add
it to the common fragment rather than copying it into multiple places.

We defined the type system above, so we know the type of each object
in the output; the query can ask for that type using the special
field `__typename`, defined on every object.

```
query CheckTypeOfR2 {
  hero {
    __typename
    name
  }
}
```

Since R2-D2 is a droid, this will return

```json
{
  "hero": {
    "__typename": "Droid",
    "name": "R2-D2"
  }
}
```

This was particularly useful because `hero` was defined to return a `Character`,
which is an interface; we might want to know what concrete type was actually
returned. If we instead asked for the hero of Episode V:

```
query CheckTypeOfLuke {
  hero(episode: EMPIRE) {
    __typename
    name
  }
}
```

We would find that it was Luke, who is a Human:

```json
{
  "hero": {
    "__typename": "Human",
    "name": "Luke Skywalker"
  }
}
```

As with the type system, this example just scratched the surface of the query
language. The specification goes into more detail about this topic in the
"Language" section, and the
[language](https://github.com/graphql/graphql-js/blob/master/src/language)
directory in GraphQL.js contains code implementing a
specification-compliant GraphQL query language parser and lexer.

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

```
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

```
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

```
# INVALID: hero is not a scalar, so fields are needed
query HeroNoFieldsQuery {
  hero
}
```

Similarly, if a field is a scalar, it doesn't make sense to query for
additional fields on it, and doing so will make the query invalid:

```
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

```
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

```
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

```
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

```
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

```
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


```
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

```
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


```
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

```
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

```
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

```
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
