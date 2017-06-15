# Validation/验证

GraphQL不仅会检测一个请求是否句法上正确，还会其在给定GraphQL schema上下文内无歧义无错误。

一个无效请求依然是技术上可执行的，也能通过执行章节的步骤产生稳定的结果，但是相对于这个有验证错误的请求，其结果可能是有歧义的、意外不可预知的，所以对于无效请求，不应该予以执行。

典型的验证会在请求执行前的上下文中执行，但是如果给定请求之前已经通过验证，那么GraphQL服务在执行这个请求前可能不会显式的验证它，譬如一个请求在开发期已经通过验证，并假设他后面不会改变，或者服务器层验证了一个请求，记住了它的验证结果以避免后续再次验证同样的请求。因此任何客户端或者开发期工具，都应该汇报验证错误，并阻止构建或者执行当时已知错误的请求。

**Type system evolution/类型系统演变**

GraphQL类型系统可能会随着时间添加一些类型和字段，从而发生了演变，之前有效的请求之后可能就变得无效。任何让之前有效的请求变得无效的变化都称之为*破坏性变化*。GraphQL服务和schema维护者被鼓励要避免破坏性变化，但是为了保证面对破坏性变化的弹性，复杂的GraphQL系统可能依然允许执行*在某个点上*没有错误也未曾改变的请求。

**Examples/案例**

至于本章节的schema，我们假定有如下类型系统用于描述案例：

```GraphQL
enum DogCommand { SIT, DOWN, HEEL }

type Dog implements Pet {
  name: String!
  nickname: String
  barkVolume: Int
  doesKnowCommand(dogCommand: DogCommand!): Boolean!
  isHousetrained(atOtherHomes: Boolean): Boolean!
  owner: Human
}

interface Sentient {
  name: String!
}

interface Pet {
  name: String!
}

type Alien implements Sentient {
  name: String!
  homePlanet: String
}

type Human implements Sentient {
  name: String!
}

enum CatCommand { JUMP }

type Cat implements Pet {
  name: String!
  nickname: String
  doesKnowCommand(catCommand: CatCommand!): Boolean!
  meowVolume: Int
}

union CatOrDog = Cat | Dog
union DogOrHuman = Dog | Human
union HumanOrAlien = Human | Alien

type QueryRoot {
  dog: Dog
}
```


## Operations/操作

### Named Operation Definitions/具名操作定义

#### Operation Name Uniqueness/操作名唯一性

**Formal Specification/形式规范**

  * 对于文档中的每一个操作定义{operation}
  * 使{operationName}为{operation}的名字。
  * 如果存在{operationName}
    * 使{operations}为文档中名为{operationName}的所有的操作定义。
    * {operations}必然是只有一个值的集合。

**Explanatory Text/解释文本**

每一个具名操作定义必须是在其文档中唯一，以便于使用其名字指代。

例如下列文档就是有效的：

```GraphQL
query getDogName {
  dog {
    name
  }
}

query getOwnerName {
  dog {
    owner {
      name
    }
  }
}
```

然而这个文档是无效的：

```!graphql
query getName {
  dog {
    name
  }
}

query getName {
  dog {
    owner {
      name
    }
  }
}
```

即便两个操作类型是不同的，它也是无效的：

```!graphql
query dogOperation {
  dog {
    name
  }
}

mutation dogOperation {
  mutateDog {
    id
  }
}
```

### Anonymous Operation Definitions/匿名操作定义

#### Lone Anonymous Operation/单独匿名操作

**Formal Specification/形式规范**

  * 使{operations}为文档中所有的操作定义。
  * 使{anonymous}为文档中所有的匿名操作定义。
  * 如果{operations}集合多余1个值:
    * {anonymous}必须为空.

**Explanatory Text/解释文本**

GraphQL允许在文档只有一个操作存在时用简写形式定义查询操作。

例如下列文档就是有效的：

```GraphQL
{
  dog {
    name
  }
}
```

然而这个文档是无效的：

```!graphql
{
  dog {
    name
  }
}

query getName {
  dog {
    owner {
      name
    }
  }
}
```

### Subscription Operation Definitions/订阅操作定义

#### Single root field/单个根级字段

**Formal Specification/形式规范**

  * 对于文档中的每一个订阅定义{subscription}。
  * 使{rootFields}为{subscription}上的顶级选择集。
    * {rootFields}必然是只有一个值的集合。

**Explanatory Text/解释文本**

订阅操作必须只有一个根字段。

有效案例：

```GraphQL
subscription sub {
  newMessage {
    body
    sender
  }
}
```

```GraphQL
fragment newMessageFields on Message {
  body
  sender
}

subscription sub {
  newMessage {
    ... newMessageFields  
  }
}
```

无效案例：

```!graphql
subscription sub {
  newMessage {
    body
    sender
  }
  disallowedSecondRootField
}
```

内省字段也是计算在内的，以下案例是无效的：

```!graphql
subscription sub {
  newMessage {
    body
    sender
  }
  __typename
}
```

## Fields/字段

### Field Selections on Objects, Interfaces, and Unions Types/对象、接口和联合上的字段选择

**Formal Specification/形式规范**

  * 对于文档中的每一个{selection}。
  * 使{fieldName}为{selection}的目标字段。
  * {fieldName}必须定义在范围内的类型上。

**Explanatory Text/解释文本**

字段选择的目标字段必须定义在选择集的范围类型上。对别名没有限制。

譬如下列片段无法通过验证：

```!graphql
fragment fieldNotDefined on Dog {
  meowVolume
}

fragment aliasedLyingFieldTargetNotDefined on Dog {
  barkVolume: kawVolume
}
```

对于接口，直接的字段选择只能在字段上操作，（接口）具体实现（对象）的字段与给定接口类型选择集的有效性没有相关性。

例如，以下是有效的：

```GraphQL
fragment interfaceFieldSelection on Pet {
  name
}
```

而以下是无效的：

```!graphql
fragment definedOnImplementorsButNotInterface on Pet {
  nickname
}
```

因为联合上并没有定义字段，字段没法直接从联合类型选择集上选出，除了元字段{__typename}这个特例。联合类型选择集上的字段必须仅通过片段查询。

例如，以下是有效的：

```GraphQL
fragment inDirectFieldSelectionOnUnion on CatOrDog {
  __typename
  ... on Pet {
    name
  }
  ... on Dog {
    barkVolume
  }
}
```

但是以下是无效的：

```!graphql
fragment directFieldSelectionOnUnion on CatOrDog {
  name
  barkVolume
}
```


### Field Selection Merging/字段选择合并

**Formal Specification/形式规范**

  * 使{set}为GraphQL文档上定义的任意选择集。
  * {FieldsInSetCanMerge(set)}必然为true。

FieldsInSetCanMerge(set)：
  * 使{fieldsForName}为包含访问片段和内联片段的{set}中给定响应名的选择集。
  * 假设{fieldsForName}有一对成员{fieldA}和{fieldB}：
    * {SameResponseShape(fieldA, fieldB)}必须为true。
    * 如果{fieldA}和{fieldB}的父类型一样或者都不为对象类型：
      * {fieldA}和{fieldB}必然有相同的字段名。
      * {fieldA}和{fieldB}必然有相同的参数集。
      * 使{mergedSet}为选择集{fieldA}和{fieldB}相加的结果。
      * {FieldsInSetCanMerge(mergedSet)}必然为true。

SameResponseShape(fieldA, fieldB)：
  * 使{typeA}为{fieldA}的返回类型。
  * 使{typeB}为{fieldB}的返回类型。
  * 如果{typeA}或者{typeB}是Non-Null非空类型。
    * {typeA}和{typeB}必然两个都是Non-Null类型。
    * 使{typeA}为{typeA}的可空类型
    * 使{typeB}为{typeB}的可空类型
  * 如果{typeA}或者{typeB}是List列表类型。
    * {typeA}和{typeB}必然两个都是List类型
    * 使{typeA}为{typeA}的元素类型
    * 使{typeB}为{typeB}的元素类型
    * 从第3步重复。
  * 如果{typeA}或者{typeB}是Scalar标量或者Enum枚举型。
    * {typeA}和{typeB}必然是相同类型。
  * 断言：{typeA}和{typeB}都是组合类型。
  * 使{mergedSet}为选择集{fieldA}和{fieldB}相加的结果。
  * 使{fieldsForName}为包含访问片段和内联片段的{set}中给定响应名的选择集。
  * 假设{fieldsForName}有一对成员{subfieldA}和{subfieldB}：
    * {SameResponseShape(subfieldA, subfieldB)}必然为true。

**Explanatory Text/解释文本**

如果执行期间遇到了相同响应名的多个字段选择，执行的字段和参数以及结果值都应该避免歧义。然而，任意两个字段选择能在同一个对象里面遇到还是有效的，那只能是它们等价的情况。

对于简单的手写GraphQL，这个规则明显是一个开发者错误，然而对于嵌套片段，就很难手动检查到这个问题。

以下选择正确地合并：

```GraphQL
fragment mergeIdenticalFields on Dog {
  name
  name
}

fragment mergeIdenticalAliasesAndFields on Dog {
  otherName: name
  otherName: name
}
```

以下则无法合并：

```!graphql
fragment conflictingBecauseAlias on Dog {
  name: nickname
  name
}
```

如果它们有相同的参数，那么这个相同的参数也会被合并。值和参数都能正确地合并。

例如以下正确地合并：

```GraphQL
fragment mergeIdenticalFieldsWithIdenticalArgs on Dog {
  doesKnowCommand(dogCommand: SIT)
  doesKnowCommand(dogCommand: SIT)
}

fragment mergeIdenticalFieldsWithIdenticalValues on Dog {
  doesKnowCommand(dogCommand: $dogCommand)
  doesKnowCommand(dogCommand: $dogCommand)
}
```

以下并不正确得合并：

```!graphql
fragment conflictingArgsOnValues on Dog {
  doesKnowCommand(dogCommand: SIT)
  doesKnowCommand(dogCommand: HEEL)
}

fragment conflictingArgsValueAndVar on Dog {
  doesKnowCommand(dogCommand: SIT)
  doesKnowCommand(dogCommand: $dogCommand)
}

fragment conflictingArgsWithVars on Dog {
  doesKnowCommand(dogCommand: $varOne)
  doesKnowCommand(dogCommand: $varTwo)
}

fragment differingArgs on Dog {
  doesKnowCommand(dogCommand: SIT)
  doesKnowCommand
}
```

下面的字段不会合并在一起，然而它们也不会在同一个对象上相遇，所以它们是安全的：

```GraphQL
fragment safeDifferingFields on Pet {
  ... on Dog {
    volume: barkVolume
  }
  ... on Cat {
    volume: meowVolume
  }
}

fragment safeDifferingArgs on Pet {
  ... on Dog {
    doesKnowCommand(dogCommand: SIT)
  }
  ... on Cat {
    doesKnowCommand(catCommand: JUMP)
  }
}
```

然而，字段响应必须形状上能合并，譬如，标量不可变。下列例子中，`someValue`可能是`String`或者`Int`：

```!graphql
fragment conflictingDifferingResponses on Pet {
  ... on Dog {
    someValue: nickname
  }
  ... on Cat {
    someValue: meowVolume
  }
}
```


### Leaf Field Selections/叶子节点选择

**Formal Specification/形式规范**

  * 对于文档中的每一个{selection}
  * 使{selectionType}为{selection}的结果类型
  * 如果{selectionType}是一个标量：
    * 这个选择的下级选择必须为空
  * 如果{selectionType}是一个接口、联合或者对象
    * 这个选择的下级选择必**不**为空

**Explanatory Text/解释文本**

标量上的字段选择是不允许的，标量在任何GraphQL查询的都只是叶子节点。

下列是有效的。

```GraphQL
fragment scalarSelection on Dog {
  barkVolume
}
```

下列是无效的。

```!graphql
fragment scalarSelectionsNotAllowedOnBoolean on Dog {
  barkVolume {
    sinceWhen
  }
}
```

相反的，GraphQL查询的叶子字段选择必须为标量。在接口、联合或者对象上的选择也不允许没有下级选择。

假设schema有下列查询根类型的补充内容：

```GraphQL
extend type QueryRoot {
  human: Human
  pet: Pet
  catOrDog: CatOrDog
}
```

以下案例是无效的

```!graphql
query directQueryOnObjectWithoutSubFields {
  human
}

query directQueryOnInterfaceWithoutSubFields {
  pet
}

query directQueryOnUnionWithoutSubFields {
  catOrDog
}
```


## Arguments/参数

参数在字段和指令上都有使用，下列验证规则可应用于这两种情况。


### Argument Names/参数名

**Formal Specification/形式规范**

  * 对于文档中的每一个{argument}。
  * 使{argumentName}为{argument}的名字。
  * 使{argumentDefinition}为父字段提供的参数定义或者名为{argumentName}的的定义。
  * {argumentDefinition}必然存在。

**Explanatory Text/解释文本**

提供给字段或者指令的每一个参数，必须在字段或者指令的可能参数集合中定义。

譬如下列是有效的：

```GraphQL
fragment argOnRequiredArg on Dog {
  doesKnowCommand(dogCommand: SIT)
}

fragment argOnOptional on Dog {
  isHousetrained(atOtherHomes: true) @include(if: true)
}
```

下列是无效，因为`command`并没有定义在`DogCommand`上。

```!graphql
fragment invalidArgName on Dog {
  doesKnowCommand(command: CLEAN_UP_HOUSE)
}
```

而这个也是无效，因为`unless`并没有定义在`@include`上。

```!graphql
fragment invalidArgName on Dog {
  isHousetrained(atOtherHomes: true) @include(unless: false)
}
```


为了展示更复杂的参数案例，我们添加下列（补充内容）到我们的类型系统：

```GraphQL
type Arguments {
  multipleReqs(x: Int!, y: Int!): Int!
  booleanArgField(booleanArg: Boolean): Boolean
  floatArgField(floatArg: Float): Float
  intArgField(intArg: Int): Int
  nonNullBooleanArgField(nonNullBooleanArg: Boolean!): Boolean!
  booleanListArgField(booleanListArg: [Boolean]!): [Boolean]
}

extend type QueryRoot {
  arguments: Arguments
}
```

参数的顺序并不重要，因此下列两个案例对有效的：

```GraphQL
fragment multipleArgs on Arguments {
  multipleReqs(x: 1, y: 2)
}

fragment multipleArgsReverseOrder on Arguments {
  multipleReqs(y: 1, x: 2)
}
```


### Argument Uniqueness/参数唯一性

字段和指令将参数视作参数名到从参数值的映射，一个参数集合内有多于一个参数拥有一个样的参数名时将会产生歧义，也是无效的。

**Formal Specification/形式规范**

  * 对于文档中的每一个{argument}。
  * 使{argumentName}为{argument}的名字。
  * 使{arguments}为参数集合中所有具有{argumentName}的名字且包含{argument}的所有参数。
  * {arguments}必然是只包含{argument}的集合。


### Argument Values Type Correctness/参数值类型正确性

#### Compatible Values/兼容值

**Formal Specification/形式规范**

  * 对于文档中的每一个{argument}。
  * 使{value}为{argument}的值。
  * 如果{value}不是一个变量
    * 使{argumentName}为{argument}的名字。
    * 使{argumentDefinition}为父字段提供的参数定义或者名为{argumentName}的的定义。
    * 使{type}为{argumentDefinition}所期望的类型。
    * {literalArgument}的类型必须被转换为{type}。

**Explanatory Text/解释文本**

字面量值必须和使用它的参数的类型相兼容，兼容规则为类型系统章节所定义的转换规则。

譬如，一个Int型可以被转换为Float型。

```GraphQL
fragment goodBooleanArg on Arguments {
  booleanArgField(booleanArg: true)
}

fragment coercedIntIntoFloatArg on Arguments {
  floatArgField(floatArg: 1)
}
```

而String到Int则是不可转换的，因此下面的案例是无效的：

```!graphql
fragment stringIntoInt on Arguments {
  intArgField(intArg: "3")
}
```


#### Required Non-Null Arguments/必要非空参数

  * 对于文档中的每一个字段或指令。
  * 使{arguments}为字段或指令提供的参数。
  * 使{argumentDefinitions}为字段或指令的参数定义集合。
  * 对于{argumentDefinitions}上的每一个{definition}：
    * 使{type}为{definition}所需要的类型。
    * 如果{type}是Non-Null非空：
      * 使{argumentName}为{definition}的名字。
      * 使{argument}为{arguments}中名为{argumentName}的参数。
      * {argument}必然存在。
      * 使{value}为{argument}的值。
      * {value}不可为{null}字面量。

**Explanatory Text/解释文本**

参数也能是必要的，只要参数的类型是非空，那么这个参数就是必须的，且显式值{null}也不能用于此。否则参数是可选的。

例如以下是有效的：

```GraphQL
fragment goodBooleanArg on Arguments {
  booleanArgField(booleanArg: true)
}

fragment goodNonNullArg on Arguments {
  nonNullBooleanArgField(nonNullBooleanArg: true)
}
```

可空参数的字段，参数可以省略。

因此以下是有效的：

```GraphQL
fragment goodBooleanArgDefault on Arguments {
  booleanArgField
}
```

但是这对于一个非空参数就不是有效的了。

```!graphql
fragment missingRequiredArg on Arguments {
  nonNullBooleanArgField
}
```

使用显式值{null}也是无效的。

```!graphql
fragment missingRequiredArg on Arguments {
  notNullBooleanArgField(nonNullBooleanArg: null)
}
```

## Fragments/片段

### Fragment Declarations/片段声明

#### Fragment Name Uniqueness/片段名唯一性

**Formal Specification/形式规范**

  * 对于文档中的每一个{fragment}。
  * 使{fragmentName}为{fragment}的名字。
  * 使{fragments}为文档中名为{fragmentName}的所有片段定义。
  * {fragments}必然是只有一个值的集合。

**Explanatory Text/解释文本**

片段定义在片段解构中使用名字引用。为避免歧义，每一个片段都必须是片段内唯一。

行内片段并不被当作片段定义，也不被这些验证规则影响。

例如以下文档是有效的：

```GraphQL
{
  dog {
    ...fragmentOne
    ...fragmentTwo
  }
}

fragment fragmentOne on Dog {
  name
}

fragment fragmentTwo on Dog {
  owner {
    name
  }
}
```

然而这个文档是无效的：

```!graphql
{
  dog {
    ...fragmentOne
  }
}

fragment fragmentOne on Dog {
  name
}

fragment fragmentOne on Dog {
  owner {
    name
  }
}
```


#### Fragment Spread Type Existence/片段解构类型存在性

**Formal Specification/形式规范**

  * 对于文档中的每一个文档解构{namedSpread}
  * 使{fragment}为{namedSpread}的目标
  * {fragment}的目标类型必须在schema中定义过的。

**Explanatory Text/解释文本**

片段必须在schema中存在的类型上指定，这同时适用于具名片段和内联片段。如果它们不存在于schema上，那么这个查询则无法通过验证。

譬如下列片段是有效的：

```GraphQL
fragment correctType on Dog {
  name
}

fragment inlineFragment on Dog {
  ... on Dog {
    name
  }
}

fragment inlineFragment2 on Dog {
  ... @include(if: true) {
    name
  }
}
```

而下列无法通过验证：

```!graphql
fragment notOnExistingType on NotInSchema {
  name
}

fragment inlineNotExistingType on Dog {
  ... on NotInSchema {
    name
  }
}
```


#### Fragments On Composite Types/组合类型上的片段

**Formal Specification/形式规范**

  * 对于定义在文档内的每一个{fragment}。
  * 片段的目标类型必须是{UNION}、{INTERFACE}或者{OBJECT}类型。

**Explanatory Text/解释文本**

片段只能在联合、接口和对象上声明，在标量上是无效的，只能应用与非叶子节点，这个规则同时适用于具名片段和行内片段。

下列片段声明是有效的：

```GraphQL
fragment fragOnObject on Dog {
  name
}

fragment fragOnInterface on Pet {
  name
}

fragment fragOnUnion on CatOrDog {
  ... on Dog {
    name
  }
}
```

而下列是无效的：

```!graphql
fragment fragOnScalar on Int {
  something
}

fragment inlineFragOnScalar on Dog {
  ... on Boolean {
    somethingElse
  }
}
```


#### Fragments Must Be Used/必须使用的片段

**Formal Specification/形式规范**

  * 对于文档中的每一个{fragment}。
  * {fragment}必然是文档中至少一个解构的目标。

**Explanatory Text/解释文本**

已定义的片段必须在查询文档中使用。

例如下列是一个无效的查询文档：

```!graphql
fragment nameFragment on Dog { # unused
  name
}

{
  dog {
    name
  }
}
```


### Fragment Spreads/片段解构

字段选择也被片段解构之间的互相调用决定。譬如目标片段的选择集和同级的引用目标片段相联合。


#### Fragment spread target defined/片段解构目标必须预先定义

**Formal Specification/形式规范**

  * 对于文档中的每一个{namedSpread}。
  * 使{fragment}为{namedSpread}的目标。
  * {fragment}必须在文档中预先定义。

**Explanatory Text/解释文本**

具名片段解构必须指定文档中已经定义的片段。如果结构的目标没有定义，则将会报错：

```!graphql
{
  dog {
    ...undefinedFragment
  }
}
```


#### Fragment spreads must not form cycles/片段解构不可造成循环

**Formal Specification/形式规范**

  * 对于文档中的每一个{fragmentDefinition}
  * 使{visited}为一个空集。
  * {DetectCycles(fragmentDefinition, visited)}

{DetectCycles(fragmentDefinition, visited)}：
  * 使{spreads}为为{fragmentDefinition}的所有片段解构后代。
  * 对于{spreads}的每一个{spread}
    * {visited}必不包含{spread}
    * 使{nextVisited}为包含{spread}和{visited}成员的集合
    * 使{nextFragmentDefinition}为{spread}的目标
    * {DetectCycles(nextFragmentDefinition, nextVisited)}

**Explanatory Text/解释文本**

片段结构的图不可造成任何循环，即便是对自身的解构。否则一个操作会下层数据的环上无尽地结构无尽地执行。

这使可能导致无尽解构的片段失效；

```!graphql
{
  dog {
    ...nameFragment
  }
}

fragment nameFragment on Dog {
  name
  ...barkVolumeFragment
}

fragment barkVolumeFragment on Dog {
  barkVolume
  ...nameFragment
}
```

如果上述片段被引入，则会导致结果无限大：

```GraphQL
{
  dog {
    name
    barkVolume
    name
    barkVolume
    name
    barkVolume
    name
    # forever...
  }
}
```

这也会使导致循环数据上结果无限递归的片段无效：

```!graphql
{
  dog {
    ...dogFragment
  }
}

fragment dogFragment on Dog {
  name
  owner {
    ...ownerFragment
  }
}

fragment ownerFragment on Dog {
  name
  pets {
    ...dogFragment
  }
}
```


#### Fragment spread is possible/片段结构必须可行

**Formal Specification/形式规范**

  * 对于文档中定义的每样个（具名的和内联的）{spread}。
  * 使{fragment}为{spread}的目标。
  * 使{fragmentType}为{fragment}的类型条件
  * 使{parentType}为包含{spread}的类型选择集合
  * 使{applicableTypes}为{GetPossibleTypes(fragmentType)}和{GetPossibleTypes(parentType)}的交集。
  * {applicableTypes}必不为空。

GetPossibleTypes(type)：
  * 如果{type}是一个object/对象类型非，返回包含{type}的集合
  * 如果{type}是一个interface/接口类型非，返回实现{type}的集合
  * 如果{type}是一个union/联合类型非，返回{type}的可能类型的集合

**Explanatory Text/解释文本**

片段在一个类型上声明，并只在这个运行时对象匹配类型条件的时候应用。它们也能在父类型的上下文中解构。片段解构只有在类型条件能够应用与父类型的时候有效。


##### Object Spreads In Object Scope/对象范围内的对象解构

在一个对象类型范围内，唯一有效的对象类型片段解构是能够应用与范围内同一类型的（片段）。

例如：

```GraphQL
fragment dogFragment on Dog {
  ... on Dog {
    barkVolume
  }
}
```

而下列是无效的

```!graphql
fragment catInDogFragmentInvalid on Dog {
  ... on Cat {
    meowVolume
  }
}
```


##### Abstract Spreads in Object Scope/对象范围内的抽象解构

在对象类型的范围，联合或者接口解构仅在对象类型实现了接口或者是联合的成员的时候可用。

例如

```GraphQL
fragment petNameFragment on Pet {
  name
}

fragment interfaceWithinObjectFragment on Dog {
  ...petNameFragment
}
```

是有效的，因为{Dog}实现了{Pet}

同样

```GraphQL
fragment catOrDogNameFragment on CatOrDog {
  ... on Cat {
    meowVolume
  }
}

fragment unionWithObjectFragment on Dog {
  ...catOrDogNameFragment
}
```

是有效的，因为{Dog}是{CatOrDog}联合的成员。如果观察{CatOrDogNameFragment}的内容，结果你发现没有任何有效结果可以返回，那么这个解构是没有意义的。但我们并不说这个是无效的，因为我们仅仅考虑片段声明，而非其主体。


##### Object Spreads In Abstract Scope/抽象范围内的对象解构

在对象类型片段范围内，联合或者接口解构仅在对象类型是接口或者联合的可能类型之一时可用。

例如，下列片段是有效的：

```GraphQL
fragment petFragment on Pet {
  name
  ... on Dog {
    barkVolume
  }
}

fragment catOrDogFragment on CatOrDog {
  ... on Cat {
    meowVolume
  }
}
```

{petFragment}有效是因为{Dog}实现了{Pet}。
{catOrDogFragment}有效是因为{Cat}是{CatOrDog}联合的成员。

相反地，下列片段是无效的：

```!graphql
fragment sentientFragment on Sentient {
  ... on Dog {
    barkVolume
  }
}

fragment humanOrAlienFragment on HumanOrAlien {
  ... on Cat {
    meowVolume
  }
}
```

{Dog}并没有实现{Sentient}接口，因此{sentientFragment}永远不会返回有意义的结果。因此这这个片段是无效的。同样的{Cat}并不是{HumanOrAlien}联合的成员，它也永远不会返回有意义的结果，从而是它无效了。


##### Abstract Spreads in Abstract Scope/抽象范围内的抽象解构

联合或者接口解构能在互相内部使用。只要存在*一个*对象在可能集合的交集中，这个对象的解构就被视为有效的。

因此下例

```GraphQL
fragment unionWithInterface on Pet {
  ...dogOrHumanFragment
}

fragment dogOrHumanFragment on DogOrHuman {
  ... on Dog {
    barkVolume
  }
}
```

被视作有效，以内{Dog}实现了{Pet}并是{DogOrHuman}的成员。

然而

```!graphql
fragment nonIntersectingInterfaces on Pet {
  ...sentientFragment
}

fragment sentientFragment on Sentient {
  name
}
```

并不有效，因为不存在同时实现了{Pet}和{Sentient}的类型。


## Values/值


### Input Object Field Uniqueness/输入对象字段唯一性

**Formal Specification/形式规范**

  * 对于文档中的每一个输入对象值{inputObject}。
  * 对于{inputObject}中的每一个{inputField}
    * 使{name}为{inputField}的名字
    * 使{fields} be all Input Object Fields named {name} in {inputObject}.
    * {fields}必然是仅包含{inputField}的集合。

**Explanatory Text/解释文本**

输入对象不能包含多余一个同名的字段，否则存在让部分句法被忽略的歧义。

例如，下列查询并不会通过验证。

```!graphql
{
  field(arg: { field: true, field: false })
}
```


## Directives/指令


### Directives Are Defined/指令必须预先定义

**Formal Specification/形式规范**

  * 对于文档中的每一个{directive}。
  * 使{directiveName}为{directive}的名字。
  * 使{directiveDefinition}为名为{directiveName}的指令。
  * {directiveDefinition}必然存在。

**Explanatory Text/解释文本**

GraphQL服务器定义他们支持的指令，对于每个指令的使用，必须要指令在服务器上可用。


### Directives Are In Valid Locations/指令必须在有效位置

**Formal Specification/形式规范**

  * 对于文档中的每一个{directive}。
  * 使{directiveName}为{directive}的名字。
  * 使{directiveDefinition}为名为{directiveName}的指令。
  * 使{locations}为{directiveDefinition}的有效位置。
  * 使{adjacent}为收到指令影响的AST（抽象语法树）节点。
  * {adjacent}必须以{locations}内的元素来表示。

**Explanatory Text/解释文本**

GraphQL定义了他们支持的指令，在何处支持。每一指令的使用，都必须在服务器声明支持使用的地方。

譬如，下列查询无法通过验证，因为`@skip`在`QUERY`上，并不是有效位置。

```!graphql
query @skip(if: $foo) {
  field
}
```


### Directives Are Unique Per Location/每个位置的指令都必须唯一

**Formal Specification/形式规范**

  * 对于文档中指令可是应用的{location}：
    * 使{directives}为应用于{location}的指令集合。
    * 对于{directives}中的每一个{directive}：
      * 使{directiveName}为{directive}的名字。
      * 使{namedDirectives}为{directives}中名为{directiveName}的指令集合。
      * {namedDirectives}必然是只有一个值的集合。

**Explanatory Text/解释文本**

指令用于描述其应用的定义的元数据或者表现的改变。当同名的指令多次使用时，期望的源数据或者表现就会变得有歧义，因此一个位置上只允许使用一个指令。

譬如，下列查询不会通过验证，因为`@skip`在同一个字段上用了两次：

```!graphql
query ($foo: Boolean = true, $bar: Boolean = false) {
  field @skip(if: $foo) @skip(if: $bar)
}
```

但是下面案例是有效的，因为`@skip`在每个位置只用了一次，即便在查询中同名的字段上用了两次：

```GraphQL
query ($foo: Boolean = true, $bar: Boolean = false) {
  field @skip(if: $foo) {
    subfieldA
  }
  field @skip(if: $bar) {
    subfieldB
  }
}
```


## Variables/变量

### Variable Uniqueness/变量唯一性

**Formal Specification/形式规范**

  * 对于文档中的每一个{operation}
    * 对于{operation}中定义的每一个{variable}
      * 使{variableName}为{variable}的名字
      * 使{variables}为{operation}上名为{variableName}的变量集合
      * {variables}必然是只有一个值的集合。

**Explanatory Text/解释文本**

任意操作定了多于一个同名变量，将会变得有歧义，并是无效的。即便重复变量的值是一样的，他也是无效的。

```!graphql
query houseTrainedQuery($atOtherHomes: Boolean, $atOtherHomes: Boolean) {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```


多个操作可以具有相同的变量名，特别是两个操作引用了同一个片段，这时候同名变量就是必要的了：

```GraphQL
query A($atOtherHomes: Boolean) {
  ...HouseTrainedFragment
}

query B($atOtherHomes: Boolean) {
  ...HouseTrainedFragment
}

fragment HouseTrainedFragment {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```


### Variable Default Values Are Correctly Typed/变量默认值必须是正确的类型

**Formal Specification/形式规范**

  * 对于文档中的每一个{operation}
  * 对于每一个{operation}上的每一个{variable}
    * 使{variableType}为{variable}的类型
    * 如果{variableType}是non-null非空，则不能有默认值。
    * 如果{variable}有默认值，则其必须是同样的类型或者能转换到{variableType}

**Explanatory Text/解释文本**

操作中定一个变量都是可以定义默认值的，如果对应类型不是non-null。

例如，下列查询能通过验证。

```GraphQL
query houseTrainedQuery($atOtherHomes: Boolean = true) {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```

但是如果变量被定义为非空类型，那么默认值是不可达的，因此下列如同这样的查询就不会通过验证

```!graphql
query houseTrainedQuery($atOtherHomes: Boolean! = true) {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```

默认值必须和变量类型兼容，必须是匹配或者能转换到这种类型。

不匹配的类型会失败，如下例：

```!graphql
query houseTrainedQuery($atOtherHomes: Boolean = "true") {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```

然而如果类型是可转换的，查询就能通过。

例如：

```GraphQL
query intToFloatQuery($floatVar: Float = 1) {
  arguments {
    floatArgField(floatArg: $floatVar)
  }
}
```


### Variables Are Input Types/变量必须是输入类型

**Formal Specification/形式规范**

  * 对{document}中的每一个{operation}
  * 对每一个{operation}上的每一个{variable}
    * 使{variableType}为{variable}的类型
    * 当{variableType}是{LIST}或者{NON_NULL}
      * 使{variableType}为{variableType}引用的类型
    * {variableType}必然是{SCALAR}、{ENUM}或者{INPUT_OBJECT}类型

**Explanatory Text/解释文本**

变量只能是标量、枚举型和输入对象，或者这些类型的列表和非空封装变体，这些是输入类型。而对象、联合、接口不能作为输入。

对于这些的案例，假设有如下类型系统补充内容：

```
input ComplexInput { name: String, owner: String }

extend type QueryRoot {
  findDog(complex: ComplexInput): Dog
  booleanList(booleanListArg: [Boolean!]): Boolean
}
```

下列查询是有效的：

```GraphQL
query takesBoolean($atOtherHomes: Boolean) {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}

query takesComplexInput($complexInput: ComplexInput) {
  findDog(complex: $complexInput) {
    name
  }
}

query TakesListOfBooleanBang($booleans: [Boolean!]) {
  booleanList(booleanListArg: $booleans)
}
```

下列查询是无效的：

```!graphql
query takesCat($cat: Cat) {
  # ...
}

query takesDogBang($dog: Dog!) {
  # ...
}

query takesListOfPet($pets: [Pet]) {
  # ...
}

query takesCatOrDog($catOrDog: CatOrDog) {
  # ...
}
```


### All Variable Uses Defined/所有变量的使用必须预先定义

**Formal Specification/形式规范**

  * 对于文档中的每一个{operation}
    * 对于范围内的每一个{variableUsage}，变量必须在{operation}的变量列表中。
    * 使{fragments}为{operation}传递引用的每一个片段
    * 对于{fragments}中的每一个{fragment}
      * 对于{fragment}范围内的每一个{variableUsage}，变量必须在{operation}的变量列表中。

**Explanatory Text/解释文本**

变量的有效范围是在每一个操作内的，也就是说任何在操作上下文中要使用的变量，必须在那个操作的顶部预先定义。

例如：

```GraphQL
query variableIsDefined($atOtherHomes: Boolean) {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```

是有效的，${atOtherHomes}由这个操作定义。

相反，下面这个操作是无效的：

```!graphql
query variableIsNotDefined {
  dog {
    isHousetrained(atOtherHomes: $atOtherHomes)
  }
}
```

${atOtherHomes}并没有被操作定义。

片段使这个规则复杂，被操作引入的任何片段都能接入那个操作定义的变量。片段可以出现在多个操作内，因此变量的使用必须和所有操作上的变量定义对应。

譬如，下列是有效的：

```GraphQL
query variableIsDefinedUsedInSingleFragment($atOtherHomes: Boolean) {
  dog {
    ...isHousetrainedFragment
  }
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

因为{isHousetrainedFragment}在{variableIsDefinedUsedInSingleFragment}操作的上下文中使用，其参数也被也在这个操作上定义。

另一面，如果操作内引入的片段所使用的参数并没有在操作上定义，那么这个查询是无效的。

```!graphql
query variableIsNotDefinedUsedInSingleFragment {
  dog {
    ...isHousetrainedFragment
  }
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

这也是传递应用的，所以下列会失败：

```!graphql
query variableIsNotDefinedUsedInNestedFragment {
  dog {
    ...outerHousetrainedFragment
  }
}

fragment outerHousetrainedFragment on Dog {
  ...isHousetrainedFragment
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

片段使用的变量必须在所有的操作上定义。

```GraphQL
query housetrainedQueryOne($atOtherHomes: Boolean) {
  dog {
    ...isHousetrainedFragment
  }
}

query housetrainedQueryTwo($atOtherHomes: Boolean) {
  dog {
    ...isHousetrainedFragment
  }
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

然而下面这个并不通过验证：

```!graphql
query housetrainedQueryOne($atOtherHomes: Boolean) {
  dog {
    ...isHousetrainedFragment
  }
}

query housetrainedQueryTwoNotDefined {
  dog {
    ...isHousetrainedFragment
  }
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

这是因为{housetrainedQueryTwoNotDefined}并没有定变量${atOtherHomes}，但是这个变量被{isHousetrainedFragment}操作引入使用。


### All Variables Used/所有变量都必须被使用

**Formal Specification/形式规范**

  * 对于文档中的每一个{operation}。
  * 使{variables}为{operation}上定义的变量。
  * {variables}中的每一个{variable}必须至少被使用一次，无论是操作本身使用，还是操作引入的片段通过传递引用使用。

**Explanatory Text/解释文本**

操作上定义的所有变量必须至少被使用一次，无论是操作本身使用，还是操作引入的片段通过传递引用使用。未使用的变量会导致验证错误。

例如，下列是无效的：

```!graphql
query variableUnused($atOtherHomes: Boolean) {
  dog {
    isHousetrained
  }
}
```

因为${atOtherHomes}没有被引用过。

这个规则也适用于片段解构的传递引用：

```GraphQL
query variableUsedInFragment($atOtherHomes: Boolean) {
  dog {
    ...isHousetrainedFragment
  }
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

上面是有效的，因为${atOtherHomes}在{isHousetrainedFragment}中被使用过了，其由{variableUsedInFragment}引入。

如果那个片段没有引用${atOtherHomes}，则是无效的：

```!graphql
query variableNotUsedWithinFragment($atOtherHomes: Boolean) {
  ...isHousetrainedWithoutVariableFragment
}

fragment isHousetrainedWithoutVariableFragment on Dog {
  isHousetrained
}
```

一个文档中的所有操作都必须使用他们所有的变量。

从而，下列文档是无效的：

```!graphql
query queryWithUsedVar($atOtherHomes: Boolean) {
  dog {
    ...isHousetrainedFragment
  }
}

query queryWithExtraVar($atOtherHomes: Boolean, $extra: Int) {
  dog {
    ...isHousetrainedFragment
  }
}

fragment isHousetrainedFragment on Dog {
  isHousetrained(atOtherHomes: $atOtherHomes)
}
```

这个文档是无效的，因为{queryWithExtraVar}定义了额外的一个变量。


### All Variable Usages are Allowed/所有变量都允许使用

**Formal Specification/形式规范**

  * {document}中的每一个{operation}
  * 使{variableUsages}为{operation}引入的所有传递使用。
  * 对于{variableUsages}中的每一个{variableUsage}
    * 使{variableType}为{operation}上的变量定义的的类型。 
    * 使{argumentType}为传递进去的参数的类型。
    * 使{hasDefault}为true，如果参数定义有默认值。
    * AreTypesCompatible({argumentType}, {variableType}, {hasDefault})必然为true

  * AreTypesCompatible({argumentType},{variableType}, {hasDefault})：
    * 如果{hasDefault}为true，把{variableType}当作non-null非空。
    * 如果{argumentType}和{variableType}的内部类型不相同，返回falae
    * 如果{argumentType}和{variableType}的列表纬度不相同，返回falae
    * 如果{variableType}的任意列表层级不是non-null，而{argument}对应的是non-null，则类型不兼容。

**Explanatory Text/解释文本**

变量使用必须和它们传递进去的参数兼容。

验证失败会发生在变量在类型上下文完全不匹配和传递可空类型参数给非空类型变量的时候。

类型必须匹配：

```!graphql
query intCannotGoIntoBoolean($intArg: Int) {
  arguments {
    booleanArgField(booleanArg: $intArg)
  }
}
```

${intArg}是{Int}类型，不能作为参数传递给{Boolean}类型的{booleanArg}。

列表基数也必须一样。例如，列表不能传递给单数值。

```!graphql
query booleanListCannotGoIntoBoolean($booleanListArg: [Boolean]) {
  arguments {
    booleanArgField(booleanArg: $booleanListArg)
  }
}
```

可空性也同样重要。通常，空值变量不可传递给非空变量。

```!graphql
query booleanArgQuery($booleanArg: Boolean) {
  arguments {
    nonNullBooleanArgField(nonNullBooleanArg: $booleanArg)
  }
}
```

主要注意的例外是有默认参数的情况，这个参数会被视作非空。

```GraphQL
query booleanArgQueryWithDefault($booleanArg: Boolean = true) {
  arguments {
    nonNullBooleanArgField(nonNullBooleanArg: $booleanArg)
  }
}
```

对于列表类型，跟可空性类似的规则同时应用与外部和内部类型，一个可空列表不能传入一个非空列表，一个可空值的列表不能传递给一个非空值的列表。
下列是有效的：

```GraphQL
query nonNullListToList($nonNullBooleanList: [Boolean]!) {
  arguments {
    booleanListArgField(booleanListArg: $nonNullBooleanList)
  }
}
```

然而，一个可空列表不能传入一个非空列表：

```!graphql
query listToNonNullList($booleanList: [Boolean]) {
  arguments {
    nonNullBooleanListField(nonNullBooleanListArg: $booleanList)
  }
}
```

这个的验证会失败，因为`[T]`不能传递给`[T]!`。

类似的，`[T]`也不能传递给`[T!]`。
