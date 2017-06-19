# Execution/执行

GraphQL通过执行来从请求生成响应。

一个用于执行的请求包含以信息的一部分：

* 要用的schema，典型情况下由GraphQL服务单独提供。
* 包含操作和片段的文档。
* 可选：要执行的操作名。
* 可选：操作定义的变量所需的值。
* 一个执行期对应于根级类型的初始值。
概念上而言，初始值代表GraphQL服务下可用的数据的“宇宙”。对于一个GraphQL服务而言，通常执行每个请求都会使用相同初始值。

有了这些信息，{ExecuteRequest()}的结果就能生成响应，然后按照下述响应一节来格式化。

## Executing Requests/执行请求

要执行请求，执行器必须有一个解析过的`Document`/文档（本规范的“Query Language”部分有定义过），如果文档中定义了多个操作还需要有选择的操作名，不然文档将被视为只包含一个操作。请求的结果取决于其操作根据下文“Executing Operations”一节执行的结果。

ExecuteRequest(schema, document, operationName, variableValues, initialValue)：

  * 使{operation}为{GetOperation(document, operationName)}的结果。
  * 使{coercedVariableValues}为{CoerceVariableValues(schema, operation, variableValues)}的结果。
  * 如果{operation}是一个查询操作：
    * 返回{ExecuteQuery(operation, schema, coercedVariableValues, initialValue)}。
  * 或者如果{operation}是一个更改操作：
    * 返回{ExecuteMutation(operation, schema, coercedVariableValues, initialValue)}。
  * 或者如果{operation}是一个订阅操作：
    * 返回{Subscribe(operation, schema, coercedVariableValues, initialValue)}。

GetOperation(document, operationName)：

  * 如果{operationName}是{null}：
    * 如果{document}只包含一个操作。
      * 返回{document}中包含的操作。
    * 否则产生一个要求{operationName}的错误。
  * 否则：
    * 使{operation}为{document}中名为{operationName}的操作。
    * 如果{operation}未找到，产生一个查询错误。
    * 返回{operation}。


### Validating Requests/验证请求

如同验证章节中解释的一样，只有通过了所有验证的请求才应该被执行。如果得到了验证错误，它们将被添加到响应中的"errors"列表中，而这个请求也就不执行直接失败。

典型的情况下，验证将在请求执行之前的上下文内瞬间完成，但是在一个相同请求之前已经验证过的情况下，GraphQL服务可能直接执行而不验证。GraphQL服务只应该执行那些*某个时间点*上没有验证错误也没更改过的请求。

例如：一个请求在开发期已经通过验证，并假设他后面不会改变，或者服务器层验证了一个请求，记住了它的验证结果以避免后续再次验证同样的请求。


### Coercing Variable Values/转换变量值

如果操作定义了任何变量，然后这些变量的值需要根据变量声明的类型的输入转换规则而转换。如果变量值的输入转换中发生了查询错误，操作会不执行直接失败。

CoerceVariableValues(schema, operation, variableValues)：

  * 使{coercedValues}为一个空的无序Map/映射集。
  * 使{variableDefinitions}为{operation}定义的变量。
  * 对于{variableDefinitions}中的每一个{variableDefinition}：
    * 使{variableName}为{variableDefinition}的名字。
    * 使{variableType}为{variableDefinition}的期望类型。
    * 使{defaultValue}为{variableDefinition}的默认值。
    * 使{value}为{variableValues}中给名为{variableName}的变量提供的值。
    * 如果{value}并不存在({variableValues}中并未提供)：
      * 如果{defaultValue}存在(包括{null})：
        * 给{coercedValues}添加一个名为{variableName}值为{defaultValue}的条目。
      * 或者如果{variableType}是一个Non-Nullable/非空类型，抛出一个查询错误。
      * 否则，继续处理下一个变量定义。
    * 或者, 如果{value}无法根据{variableType}的输入转换规则转换，抛出一个查询错误。
    * 使{coercedValue}为根据{variableType}的输入转换规则转换的结果。
    * 给{coercedValues}添加一个名为{variableName}值为{coercedValue}的条目。
  * 返回{coercedValues}。

Note: 这个算法和{CoerceArgumentValues()}的很相似。


## Executing Operations/执行操作

如果本规范的“Type System”/类型系统一章节所描述，类型系统必须提供一个查询的根级对象类型。如果支持更改或者订阅，它也必须提供更改或者订阅对应的根级对象类型。

### Query/查询

如果操作是一个查询，那操作的结果就是用查询根级对象类型执行查询的顶层选择集的结果。

执行一个查询的时候可以提供一个初始值。

ExecuteQuery(query, schema, variableValues, initialValue)：

  * 使{queryType}为{schema}中的根级查询类型。
  * 断言：{queryType}是一个对象类型。
  * 使{selectionSet}为{query}中的顶层选择集。
  * 使{data}为*正常*执行{ExecuteSelectionSet(selectionSet, queryType, initialValue, variableValues)}的结果(允许并行)。
  * 使{errors}为执行选择集期间产生的任何*字段错误*。
  * 返回一个包含{data}和{errors}的无序映射集。

### Mutation/更改

如果操作是一个更改，那操作的结果就是用更改根级对象类型执行更改的顶层选择集的结果。这个选择集应该依次执行。

更改的顶层字段被期望用于在下层数据系统上执行副作用操作。依次执行这些更改，以保证副作用操作期间没有竞态条件。

ExecuteMutation(mutation, schema, variableValues, initialValue)：

  * 使{mutationType}为{schema}的根级更改类型。
  * 断言：{mutationType}是一个对象类型。
  * 使{selectionSet}为{mutation}中的顶层选择集。
  * 使{data}为*依次*执行{ExecuteSelectionSet(selectionSet, mutationType, initialValue, variableValues)}的结果
  * 使{errors}为执行选择集期间产生的任何*字段错误*。
  * 返回一个包含{data}和{errors}的无序映射集。

### Subscription/订阅

如果操作是一个订阅，那结果是一个事件流，称作"Response Stream"响应流，事件流中的每一个事件即是针对下层"Source Stream"源流上的新事件执行操作的结果。

执行订阅会在服务端创建一个永久的函数，用于将下层源流映射成返回的响应流。

Subscribe(subscription, schema, variableValues, initialValue)：

  * 使{sourceStream}为{CreateSourceEventStream(subscription, schema, variableValues, initialValue)}执行的结果。
  * 使{responseStream}为{MapSourceToResponseEvent(sourceStream, subscription, schema, variableValues)}执行的结果。
  * 返回{responseStream}。

Note: 在大型订阅系统中，{Subscribe()}和{ExecuteSubscriptionEvent()}算法可能运行在分离的服务器上，以保持可预测的规模属性。可在下文章节中看到关于支持大规模订阅的内容。

考虑一个聊天应用案例。客户端发送一个如下请求来订阅投递到聊天室的新消息：

```GraphQL
subscription NewMessages {
  newMessage(roomId: 123) {
    sender
    text
  }
}
```

当客户端订阅后，任何时候有ID为"123"的新消息投递到聊天室后，对于"sender"和"text"的选择将会被执行然后发布到客户端。例如：

```json
{
  "data": {
    "newMessage": {
      "sender": "Hagrid",
      "text": "You're a wizard!"
    }
  }
}
```

"新消息投递到聊天室"可以使用了"Pub-Sub"发布订阅系统，其中聊天室ID是"topic"，并且每一个"publish"都包含了"sender"发送者和"text"文本。

**Event Streams**

事件流表示一序列可观测的离散事件。例如，一个"Pub-Sub"系统当"subscribing to a topic"订阅了一个话题可能产生一个事件流，每一次"publish"发布到话题都会在事件流上产生一个事件。事件流可能产生一序列无尽的事件，也可能在任何时候终止。响应中的事件流可能因为发生错误而终止，也可能仅仅因为没有后续事件发生。观察者可以在任何时候取消订阅而停止观察事件流，这之后就不会从事件流中收到后续事件了。

**Supporting Subscriptions at Scale/支持大规模订阅**

支持订阅对GraphQL服务器而言是一个显著的变化。查询和更改操作是无状态的，可以通过克隆GraphQL服务器实例而扩展规模。订阅却相反，它是有状态的，要求在这个订阅的生命周期内维持文档、变量和其他上下文。

考虑下你服务中某个机器宕机状态丢失的时候，你的系统的行为。使用分离的专用服务器来处理订阅状态和客户端连接将能提升系统的持久性和可用性。

#### Source Stream/源流

源流表示会触发GraphQL对应事件执行的一序列事件。就像字段值的解析一样，创建源流的逻辑也是应用特定的。

CreateSourceEventStream(subscription, schema, variableValues, initialValue)：

  * 使{subscriptionType}为{schema}的根级订阅类型。
  * 断言：{subscriptionType}是一个对象类型。
  * 使{selectionSet}。
  * 使{rootField}为{selectionSet}中的第一个顶层字段。
  * 使{argumentValues}为{CoerceArgumentValues(subscriptionType, rootField, variableValues)}的结果。
  * 使{fieldStream}为运行{ResolveFieldEventStream(subscriptionType, initialValue, rootField, argumentValues)}的结果。
  * 返回{fieldStream}。

ResolveFieldEventStream(subscriptionType, rootValue, fieldName, argumentValues)：
  * 使{resolver}为{subscriptionType}提供的内部函数，用于解析名为{fieldName}的订阅字段的事件流。
  * 返回使用{rootValue}和{argumentValues}调用{resolver}的结果。

Note: 这个{ResolveFieldEventStream()}算法有意与{ResolveFieldValue()}相似，以保证给任何操作类型定义解析函数时的一致性。

#### Response Stream/响应流

下层源流中的每一个事件都会触发订阅使用这个事件作为根值执行选择集。

MapSourceToResponseEvent(sourceStream, subscription, schema, variableValues)：

  * 返回产生如下事件的事件流{responseStream}：
  * 对于{sourceStream}上的每一个{event}：
    * 使{response}为运行{ExecuteSubscriptionEvent(subscription, schema, variableValues, event)}的结果。
    * 产生包含{response}的事件。
  * 当{responseStream}完成的时候：终止这个事件流。

ExecuteSubscriptionEvent(subscription, schema, variableValues, initialValue)：

  * 使{subscriptionType}为为{schema}的根级订阅类型。
  * 断言：{subscriptionType}是一个对象类型。
  * 使{selectionSet}为{subscription}中的顶层选择集。
  * 使{data}为*正常*执行的结果{ExecuteSelectionSet(selectionSet, subscriptionType, initialValue, variableValues)}（允许并行）。
  * 使{errors}为执行选择集期间产生的任何*字段错误*。
  * 返回一个包含{data}和{errors}的无序映射集。

Note: 这个{ExecuteSubscriptionEvent()}算法有意与{ExecuteQuery()}相似，因为这便是每个事件结果如何产生的。

#### Unsubscribe/退订

当客户端不再想要收到订阅的载荷时可以通过退订来取消响应流。这可能也同时取消掉了源流。这个是一个清理被这个订阅占用的其他资源的好机会。

Unsubscribe(responseStream)

  * 取消{responseStream}

## Executing Selection Sets/执行选择集

要执行选择集，对象值必须得到，对象类型必须已知，同样还需要知道其需要依次执行还是并列执行。

首先，选择集被转换成分组的字段集合，然后分组字段集合内的每一个字段都会在响应映射集中产生一个条目。

ExecuteSelectionSet(selectionSet, objectType, objectValue, variableValues)：

  * 使{groupedFieldSet}为{CollectFields(objectType, selectionSet, variableValues)}的结果。
  * 初始化{resultMap}进一个空有序映射集。
  * 对于作为{responseKey}和{fields}的每一个{groupedFieldSet}：
    * 使{fieldName}为{fields}中第一个条目的名字。
      Note: 值并不会因引入别名而受影响。
    * 使{fieldType}为{objectType}的{fieldName}定义的的返回类型。
    * 如果{fieldType}是{null}：
      * 继续下一轮{groupedFieldSet}的迭代。
    * 使{responseValue}为{ExecuteField(objectType, objectValue, fields, fieldType, variableValues)}。
    * 将{responseValue}设为{responseKey}中{resultMap}的值。
  * 返回{resultMap}。

Note: {resultMap}依照出现在query中的顺序排序。这在下列字段集合一节中有更详细的介绍。


### Normal and Serial Execution/正常序列执行

正常情况下，不论分组字段集合中的条目顺序为何，执行器都能执行（通常是并行执行）。因为除了顶层更改必然有副作用且具有幂等性，字段解析的执行顺序必然不会影响其结果，因此服务器能够以他认为优化的方式自由地执行字段条目。

例如，有下正常执行的分组字段：

```GraphQL
{
  birthday {
    month
  }
  address {
    street
  }
}
```

一个有效的GraphQL执行器可以以任何他选择的顺序来解析这四个字段（然而，`birthday`必然在`month`之前，同理`address`在`street`之前）。

当执行更改时，最顶层的选择集会依序执行。

当依序执行一个分组字段集的时候，执行器必须以每个条目出现在分组字段集中的顺序来决定在结果映射集对应的条目，使得分组映射集中每一个条目完成之后才继续下一个条目。

例如，有以依序执行的选择集：

```GraphQL
{
  changeBirthday(birthday: $newBirthday) {
    month
  }
  changeAddress(address: $newAddress) {
    street
  }
}
```

执行器必须依序执行：

 - 执行`changeBirthday`的{ExecuteField()}，其中{CompleteValue()}期间，会正常执行`{ month }`次级选择集。
 - 执行`changeAddress`的{ExecuteField()}，其中{CompleteValue()}期间，会正常执行`{ street }`次级选择集。

举一个说明性的例子，假设我们有一个mutation字段`changeTheNumber`，其返回包含一个`theNumber`字段的对象。如果我们依序执行下列选择集：

```GraphQL
{
  first: changeTheNumber(newNumber: 1) {
    theNumber
  }
  second: changeTheNumber(newNumber: 3) {
    theNumber
  }
  third: changeTheNumber(newNumber: 2) {
    theNumber
  }
}
```

执行器会如下依序执行：

 - 解析`changeTheNumber(newNumber: 1)`字段
 - 正常执行`first`次级选择集`{ theNumber }`
 - 解析`changeTheNumber(newNumber: 3)`字段
 - 正常执行`second`次级选择集`{ theNumber }`
 - 解析`changeTheNumber(newNumber: 2)`字段
 - 正常执行`third`次级选择集`{ theNumber }`

正确的执行器，对于上述选择集必然会生成如下结果：

```json
{
  "first": {
    "theNumber": 1
  },
  "second": {
    "theNumber": 3
  },
  "third": {
    "theNumber": 2
  }
}
```


### Field Collection/字段集合

执行之前，通过调用{CollectFields()}，选择集会被转换成分组字段集。分组字段集中的每一个条目都是共享同一个响应键的列表。这保证了同一个响应键（别名或者字段名）内所有的字段，包含通过片段引入的，都能同时执行。

如果，手机如下选择集的字段会收集到两个`a`字段的实体和一个`b`字段的实体：

```GraphQL
{
  a {
    subfield1
  }
  ...ExampleFragment
}

fragment ExampleFragment on Query {
  a {
    subfield2
  }
  b
}
```

{CollectFields()}生成的字段分组的深度优先搜索通过执行来保持，保证字段在执行后响应中以稳定可预测的顺序出现。

CollectFields(objectType, selectionSet, variableValues, visitedFragments)：

  * 如果未提供{visitedFragments}，将其初始化为空集。
  * 初始化{groupedFields}为列表的空的有序集。
  * 对于{selectionSet}中的每一个{selection}：
    * 如果{selection}提供了`@skip`指令，使{skipDirective}为此指令。
      * 如果{skipDirective}的{if}参数是{true}或者是{variableValues}中的一个为{true}的变量，则继续{selectionSet}中的下一个{selection}。
    * 如果{selection}提供了`@include`指令，使{includeDirective}为此指令。
      * 如果{includeDirective}参数不是{true}或者不是{variableValues}中的一个为{true}的变量，则继续{selectionSet}中的下一个{selection}。
    * 如果{selection}是一个{Field}：
      * 使{responseKey}为{selection}中的响应键。
      * 使{groupForResponseKey}为{groupedFields}中的一个{responseKey}列表；如果不存在这么一个列表，则创建一个空列表。
      * 附加{selection}到{groupForResponseKey}上。
    * 如果{selection}是一个{FragmentSpread}：
      * 使{fragmentSpreadName}为{selection}的名字。
      * 如果{fragmentSpreadName}在{visitedFragments}中，则继续{selectionSet}中的下一个{selection}。
      * 添加{fragmentSpreadName}到{visitedFragments}上。
      * 使{fragment}为当前文档中名为{fragmentSpreadName}的片段。
      * 如果不存在那样的{fragment}，则继续{selectionSet}中的下一个{selection}。
      * 使{fragmentType}为{fragment}上的类型条件。
      * 如果{DoesFragmentTypeApply(objectType, fragmentType)}为false，则继续{selectionSet}中的下一个{selection}。
      * 使{fragmentSelectionSet}为{fragment}的顶层选择集。
      * 使{fragmentGroupedFieldSet}为调用{CollectFields(objectType, fragmentSelectionSet, visitedFragments)}的结果。
      * 对于{fragmentGroupedFieldSet}中的每一个{fragmentGroup}：
        * 使{responseKey}为{fragmentGroup}中所有字段共享的响应键。
        * 使{groupForResponseKey}为{groupedFields}中的{responseKey}列表；如果不存在这么一个列表，则创建一个空列表。
        * 附加{fragmentGroup}中所有的元素到{groupForResponseKey}上。
    * 如果{selection}是一个{InlineFragment}：
      * 使{fragmentType}为{selection}上的类型条件。
      * 如果{fragmentType}不为{null}且{DoesFragmentTypeApply(objectType, fragmentType)}是false，则继续{selectionSet}中的下一个{selection}。
      * 使{fragmentSelectionSet}为{selection}的顶层选择集。
      * 使{fragmentGroupedFieldSet}为调用{CollectFields(objectType, fragmentSelectionSet, variableValues, visitedFragments)}的结果。
      * 对于{fragmentGroupedFieldSet}中的每一个{fragmentGroup}：
        * 使{responseKey}为为{fragmentGroup}中所有字段共享的响应键
        * 使{groupForResponseKey}为{groupedFields}中的{responseKey}列表；如果不存在这么一个列表，则创建一个空列表。
                  * 附加{fragmentGroup}中所有的元素到{groupForResponseKey}上。
  * 返回{groupedFields}。

DoesFragmentTypeApply(objectType, fragmentType)：

  * 如果{fragmentType}是一个对象类型：
    * 如果{objectType}和{fragmentType}是同类型，返回{true}，否则返回{false}。
  * 如果{fragmentType}是一个接口类型：
    * 如果{objectType}是{fragmentType}的一个实现，返回{true}，否则返回{false}。
  * 如果{fragmentType}是一个联合：
    * 如果{objectType}是{fragmentType}的一个可能类型，返回{true}，否则返回{false}。


## Executing Fields/执行字段

分组字段集上的每一个请求字段（定义在被选择的对象类型上）都会得到一个响应映射集中的一个条目。字段执行首先会转换任何提供的技术值，然后解析这个字段的值，最后通过递归执行另一个选择集或者转换一个标量来完成这个字段的值。

ExecuteField(objectType, objectValue, fieldType, fields, variableValues)：
  * 使{field}为{fields}中的第一个条目。
  * 使{argumentValues}为{CoerceArgumentValues(objectType, field, variableValues)}的结果
  * 使{resolvedValue}为{ResolveFieldValue(objectType, objectValue, fieldName, argumentValues)}。
  * 返回{CompleteValue(fieldType, fields, resolvedValue, variableValues)}的结果。


### Coercing Field Arguments/转换字段参数

字段可能包含下层运行时产生正确结果的参数，这些参数定义在类型系统中的字段上，都有一个特定的输入类型：Scalars标量、Enum庙举行、Input Object输入对象，或者这三个的List列表和Non-Null非空封装。

对于查询中每个参数的位置，可能是一个字面量值，也可能是运行时提供的变量。

CoerceArgumentValues(objectType, field, variableValues)：
  * 使{coercedValues}为空的无序映射集。
  * 使{argumentValues}为{field}提供的参数值。
  * 使{fieldName}为{field}的名字。
  * 使{argumentDefinitions}为{objectType}上字段名{fieldName}定义的参数。
  * 对于{argumentDefinitions}中的每一个{argumentDefinition}：
    * 使{argumentName}为{argumentDefinition}的名字。
    * 使{argumentType}为{argumentDefinition}的期待类型。
    * 使{defaultValue}为{argumentDefinition}的默认值。
    * 使{value}为{argumentValues}中针对{argumentName}提供的值。
    * 如果{value}是一个变量：
      * 使{variableName}为{value}变量的名字。
      * 使{variableValue}为{variableValues}中针对{variableName}提供的值。
      * 如果{variableValue}存在(包含{null})：
        * 添加名为{argName}值为{variableValue}的条目到{coercedValues}。
      * 否则，如果{defaultValue}存在(包含{null})：
        * 添加名为{argName}值为{defaultValue}的条目到{coercedValues}。
      * 否则，如果{argumentType}是一个Non-Nullable非空类型，抛出字段错误。
      * 否则，继续下一个参数定义。
    * 否则，如果{value}不存在({argumentValues}中未提供)：
      * 如果{defaultValue}存在(包含{null})：
        * 添加名为{argName}值为{defaultValue}的条目到{coercedValues}。
      * 否则，如果{argumentType}是一个Non-Nullable非空类型，抛出字段错误。
      * 否则，继续下一个参数定义。
    * 否则, 如果{value}不能根据{argType}的输入转换规则转换，抛出字段错误。
    * 使{coercedValue}为根据{argType}的输入转换规则转换{value}的结果。
    * 添加名为{argName}值为{variableValue}的条目到{coercedValues}。
  * 返回{coercedValues}。

Note: 变量的值并没有被转换，因为它们应该在执行{CoerceVariableValues()}中的操作前前就被转换，一个有效的查询必须仅允许合适类型的变量。


### Value Resolution/值解析

虽然几乎所有的GraphQL执行都能通用化描述，但最终内部系统暴露给GraphQL接口的时候必须提供值。这通过{ResolveFieldValue}暴露，其生成真实值的类型上给定字段的值，

在案例中，这个可能接收{objectType}/对象类型`Person`，其{field}/字段为{"soulMate"}，这个{objectValue}/类型值表示John Lennon。它可能被期待得到值表示Yoko Ono。

ResolveFieldValue(objectType, objectValue, fieldName, argumentValues)：
  * 使{resolver}为{objectType}提供的内部函数，用以决定名为{fieldName}的字段的解析值。
  * 返回使用{objectValue}和{argumentValues}调用{resolver}的结果。

Note: {resolver}通常可能是异步的，因为其依靠读取下层数据库或者网络服务来产生值。这要求其他的GraphQL执行器能够处理异步执行流。


### Value Completion/值完成

在解析一个字段的值后，再确认其符合期望返回类型即完成。如果返回值是另一个对象类型，然后字段执行将继续递归。

CompleteValue(fieldType, fields, result, variableValues)：
  * 如果{fieldType}是一个Non-Null非空类型：
    * 使{innerType}为{fieldType}的内部类型。
    * 使{completedResult}为调用{CompleteValue(innerType, fields, result, variableValues)}的结果。
    * 如果{completedResult}是{null}，抛出一个字段错误。
    * 返回{completedResult}。
  * 如果{result}是{null} (或者另一种类似于的{null}内部值，譬如{undefined}或{NaN}), 返回{null}。
  * 如果{fieldType}是一个List列表类型：
    * 如果{result}并不是一个值的集合，抛出一个字段错误。
    * 使{innerType}为{fieldType}的内部类型。
    * 返回一个列表，其中每个列表元素都是调用{CompleteValue(innerType, fields, resultItem, variableValues)}的结果，其中{resultItem}为{result}中的每个元素。。
  * 如果{fieldType}是Scalar标量或者Enum枚举类型：
    * 返回“转换”{result}的结果，保证其为{fieldType}的合法值，否则为{null}。
  * 如果{fieldType}是Object对象，Interface接口，或者Union类型：
    * 如果{fieldType}是一个Object对象类型。
      * 使{objectType}为{fieldType}。
    * Otherwise 如果{fieldType}是一个Interface接口，或者Union类型。
      * 使{objectType}为ResolveAbstractType({fieldType}, {result})。
    * 使{subSelectionSet}为调用{MergeSelectionSets(fields)}的结果。
    * 返回ExecuteSelectionSet(subSelectionSet, objectType, result, variableValues)*正常*求值的结果(允许并行)。

**Resolving Abstract Types/解析抽象类型**

当完成一个具有抽象返回类型的字段时，譬如Interface接口或者Union联合返回类型，首先，抽象类型必须接地到一个相关的对象类型上去，这个由内部系统决定哪一个是合适的。

Note: 在面向对象的环境中，譬如Java或者C#，用以决定一个{objectValue}的对象类型的通用方法是使用这个{objectValue}的类名。

ResolveAbstractType(abstractType, objectValue)：
  * 返回调用系统提供的内部方法的结果，此方法用于决定给定值{objectValue}的{abstractType}的对象类型。

**Merging Selection Sets/合并选择集**

当多余一个同名字段并行执行后，他们的选择集在完成这个值的时候被合并，以继续次级选择集的执行。

案例查询中，描述了带次级选择集的同名的并行字段。

```GraphQL
{
  me {
    firstName
  }
  me {
    lastName
  }
}
```

当解析了 `me`的值后，选择集将合并，所以`firstName`和`lastName`可以被解析到一个值上。

MergeSelectionSets(fields)：
  * 使{selectionSet}为一个空列表。
  * 对于 {fields}中的每一个{field}：
    * 使{fieldSelectionSet}为{field}的选择集。
    * 如果{fieldSelectionSet}是null或者empty，继续下一个字段。
    * 将{fieldSelectionSet}中所有的选择集添加到{selectionSet}。
  * 返回{selectionSet}。


### Errors and Non-Nullability/错误与非空

当解析一个字段时抛出了错误，它应该被当作这个字段返回了{null}，且错误必须添加到响应的{"errors"}列表中。

如果一个字段解析的结果就是{null}（不论是字段的解析函数返回了{null}还是发生了错误），且那个字段是`Non-Null`，那么抛出一个字段错误。这个错误必须添加到响应的{"errors"}列表中。

如果字段因为错误而返回了{null}，这个错误也被添加到了响应的{"errors"}列表中，这个{"errors"}列表后续就不应被影响，即是，错误列表中一个字段上只允许添加一个错误。

因为`Non-Null`类型不能为{null}，字段错误将会冒泡到父级字段并被父级字段处理。如果父级字段可能为{null}，那么它就解析为{null}，否则如果父级字段也是`Non-Null`类型，那么这个字段错误将会继续冒泡到上一级父级字段。

如果从请求的根到错误的源的所有字段都返回`Non-Null`条目，那么响应中的{"data"}条目应该为{null}。
