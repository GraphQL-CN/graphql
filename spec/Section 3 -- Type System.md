# Type System/类型系统

GraphQL的类型系统用于描述服务器的能力以及判断一个查询是否有效。类系统也描述了查询参数的输入类型，用于运行时检查有效性。

GraphQL服务器的能力是同schema来描述，schema使用其支持的类型和指令来定义。

一个给定GraphQL schema其自身首先必须要通过内部有效性验证，本节将会讲述这个验证过程的相关规则。

一个GraphQL schema使用每种操作的根级类型表示：query/查询、mutation/更改和subscription/订阅，这表示schema是这三种操作开始的地方。

所有GraphQL schema内的类型都必须要有唯一的名字，任意两个类型都不应该有相同的名字，任意类型也不应该有和内建类型冲突的名字（包含Scalar/标量和Introspection/内省类型）。

所有GraphQL schema内的指令也必须拥有唯一的名字，指令和类型可以拥有相同的名字，因为两者之间并没有歧义。

所有schema内定义的类型和指令都不能以{"__"}（双下划线）开头命名，因为这是GraphQL内省系统专用。


## Types

任何GraphQL Schema的最基本单元都是类型，GraphQL中有8种类型。

最基本的类型是`Scalar`/标量，一个标量代表一个原始值，例如字符串或者整数。有时候，一个标量字段的返回值可能是可枚举的，对应这种场景，GraphQL提速了`Enum`/枚举类型，其指定了响应结果的有效范围。

标量和枚举型组成了响应结果树的叶子节点，而中间的分支节点则是`Object`/对象类型，其定义了一套字段，每个字段是系统中的另一个类型，从而能够定义任意层次的类型层级。

GraphQL支持两种抽象类型：interface/接口和union/联合。

`Interface`定义了一系列字段，`Object`类型通过实现了其中的字段来实现它。当类型系统表明要返回一个接口时，其返回的都是一个实现这个接口的类型。

`Union`定义了一个可能类型的列表，与接口相似，当系统表明要返回一个联合时，其返回的是联合中的一个类型。

这些类型都是可为空且为单数，譬如，一个标量字符串会返回一个null或者一个字符串。通常有需要表示某个类型的列表，于是GraphQL中提供了`List`类型，将其他类型封装在其中。类似的`Non-Null`类型也是封装其他类型，用以标注返回结果不可为空。这两种类型称为“封装类型”，非封装类型称为基础类型，每个封装类型里面都有一个基础类型，通过不断解封装来找到基础类型。

向GraphQL提供复杂结构作为输入参数是十分有用的，GraphQL为此提供了`Input Object`类型，让客户端从schema中获知服务端具体需要什么样的数据。


### Scalars/标量

如名字所示，GraphQL中一个标量代表这一个原始值，GraphQL的响应采用的是树形层级结构，其叶子节点即是标量。

所有GraphQL标量都能以字符串形式表示，虽然取决于所用的返回格式，可能有准确的原始类型来表示指定标量，服务器在适当的时候也应采取这种类型。

GraphQL提供了一些内建标量，类型系统也允许根据语义添加其他标量。假设GraphQL中要定义一个标量`Time`/时间，可将字符串转换成ISO-8601的格式，当查询一个`Time`字段时，客户端可以使用ISO-8601解析器，将这个字段类型转换成客户端特有的原始类型。另一个有潜在用途的标量是`Url`，通常会序列化成字符串，但是会由服务器保证是有效的URL。

服务器可能会在schema中省略内建标量，譬如，服务器并未使用浮点数，那么它可能并不会包含`Float`类型。 但是一旦schema包含了本规范所述的类型，那么一定会继承本规范描述的对应行为，譬如，服务器一定不会使用名为`Int`的类型去表示128-bit的数字，或者国际化信息。


**Result Coercion**

A GraphQL server, when preparing a field of a given scalar type, must uphold the
contract the scalar type describes, either by coercing the value or
producing an error.

For example, a GraphQL server could be preparing a field with the scalar type
`Int` and encounter a floating-point number. Since the server must not break the
contract by yielding a non-integer, the server should truncate the fractional
value and only yield the integer value. If the server encountered a boolean
`true` value, it should return `1`. If the server encountered a string, it may
attempt to parse the string for a base-10 integer value. If the server
encounters some value that cannot be reasonably coerced to an `Int`, then it
must raise a field error.

Since this coercion behavior is not observable to clients of the GraphQL server,
the precise rules of coercion are left to the implementation. The only
requirement is that the server must yield values which adhere to the expected
Scalar type.

**Input Coercion**

If a GraphQL server expects a scalar type as input to an argument, coercion
is observable and the rules must be well defined. If an input value does not
match a coercion rule, a query error must be raised.

GraphQL has different constant literals to represent integer and floating-point
input values, and coercion rules may apply differently depending on which type
of input value is encountered. GraphQL may be parameterized by query variables,
the values of which are often serialized when sent over a transport like HTTP. Since
some common serializations (ex. JSON) do not discriminate between integer
and floating-point values, they are interpreted as an integer input value if
they have an empty fractional part (ex. `1.0`) and otherwise as floating-point
input value.

For all types below, with the exception of Non-Null, if the explicit value
{null} is provided, then the result of input coercion is {null}.

**Built-in Scalars**

GraphQL provides a basic set of well-defined Scalar types. A GraphQL server
should support all of these types, and a GraphQL server which provide a type by
these names must adhere to the behavior described below.


#### Int

The Int scalar type represents a signed 32-bit numeric non-fractional value.
Response formats that support a 32-bit integer or a number type should use
that type to represent this scalar.

**Result Coercion**

GraphQL servers should coerce non-int raw values to Int when possible
otherwise they must raise a field error. Examples of this may include returning
`1` for the floating-point number `1.0`, or `2` for the string `"2"`.

**Input Coercion**

When expected as an input type, only integer input values are accepted. All
other input values, including strings with numeric content, must raise a query
error indicating an incorrect type. If the integer input value represents a
value less than -2<sup>31</sup> or greater than or equal to 2<sup>31</sup>, a
query error should be raised.

Note: Numeric integer values larger than 32-bit should either use String or a
custom-defined Scalar type, as not all platforms and transports support
encoding integer numbers larger than 32-bit.


#### Float

The Float scalar type represents signed double-precision fractional values
as specified by [IEEE 754](http://en.wikipedia.org/wiki/IEEE_floating_point).
Response formats that support an appropriate double-precision number type
should use that type to represent this scalar.

**Result Coercion**

GraphQL servers should coerce non-floating-point raw values to Float when
possible otherwise they must raise a field error. Examples of this may include
returning `1.0` for the integer number `1`, or `2.0` for the string `"2"`.

**Input Coercion**

When expected as an input type, both integer and float input values are
accepted. Integer input values are coerced to Float by adding an empty
fractional part, for example `1.0` for the integer input value `1`. All
other input values, including strings with numeric content, must raise a query
error indicating an incorrect type. If the integer input value represents a
value not representable by IEEE 754, a query error should be raised.


#### String

The String scalar type represents textual data, represented as UTF-8 character
sequences. The String type is most often used by GraphQL to represent free-form
human-readable text. All response formats must support string representations,
and that representation must be used here.

**Result Coercion**

GraphQL servers should coerce non-string raw values to String when possible
otherwise they must raise a field error. Examples of this may include returning
the string `"true"` for a boolean true value, or the string `"1"` for the
integer `1`.

**Input Coercion**

When expected as an input type, only valid UTF-8 string input values are
accepted. All other input values must raise a query error indicating an
incorrect type.


#### Boolean

The Boolean scalar type represents `true` or `false`. Response formats should
use a built-in boolean type if supported; otherwise, they should use their
representation of the integers `1` and `0`.

**Result Coercion**

GraphQL servers should coerce non-boolean raw values to Boolean when possible
otherwise they must raise a field error. Examples of this may include returning
`true` for any non-zero number.

**Input Coercion**

When expected as an input type, only boolean input values are accepted. All
other input values must raise a query error indicating an incorrect type.


#### ID

The ID scalar type represents a unique identifier, often used to refetch an
object or as the key for a cache. The ID type is serialized in the same way as
a `String`; however, it is not intended to be human-readable. While it is
often numeric, it should always serialize as a `String`.

**Result Coercion**

GraphQL is agnostic to ID format, and serializes to string to ensure consistency
across many formats ID could represent, from small auto-increment numbers, to
large 128-bit random numbers, to base64 encoded values, or string values of a
format like [GUID](http://en.wikipedia.org/wiki/Globally_unique_identifier).

GraphQL servers should coerce as appropriate given the ID formats they expect.
When coercion is not possible they must raise a field error.

**Input Coercion**

When expected as an input type, any string (such as `"4"`) or integer (such
as `4`) input value should be coerced to ID as appropriate for the ID formats
a given GraphQL server expects. Any other input value, including float input
values (such as `4.0`), must raise a query error indicating an incorrect type.


### Objects

GraphQL queries are hierarchical and composed, describing a tree of information.
While Scalar types describe the leaf values of these hierarchical queries, Objects
describe the intermediate levels.

GraphQL Objects represent a list of named fields, each of which yield a value of
a specific type. Object values should be serialized as ordered maps, where the
queried field names (or aliases) are the keys and the result of evaluating
the field is the value, ordered by the order in which they appear in the query.

All fields defined within an Object type must not have a name which begins with
{"__"} (two underscores), as this is used exclusively by GraphQL's
introspection system.

For example, a type `Person` could be described as:

```
type Person {
  name: String
  age: Int
  picture: Url
}
```

Where `name` is a field that will yield a `String` value, and `age` is a field
that will yield an `Int` value, and `picture` is a field that will yield a
`Url` value.

A query of an object value must select at least one field. This selection of
fields will yield an ordered map containing exactly the subset of the object
queried, which should be represented in the order in which they were queried.
Only fields that are declared on the object type may validly be queried on
that object.

For example, selecting all the fields of `Person`:

```GraphQL
{
  name
  age
  picture
}
```

Would yield the object:

```json
{
  "name": "Mark Zuckerberg",
  "age": 30,
  "picture": "http://some.cdn/picture.jpg"
}
```

While selecting a subset of fields:

```GraphQL
{
  age
  name
}
```

Must only yield exactly that subset:

```json
{
  "age": 30,
  "name": "Mark Zuckerberg"
}
```

A field of an Object type may be a Scalar, Enum, another Object type,
an Interface, or a Union. Additionally, it may be any wrapping type whose
underlying base type is one of those five.

For example, the `Person` type might include a `relationship`:

```
type Person {
  name: String
  age: Int
  picture: Url
  relationship: Person
}
```

Valid queries must supply a nested field set for a field that returns
an object, so this query is not valid:

```!graphql
{
  name
  relationship
}
```

However, this example is valid:

```GraphQL
{
  name
  relationship {
    name
  }
}
```

And will yield the subset of each object type queried:

```json
{
  "name": "Mark Zuckerberg",
  "relationship": {
    "name": "Priscilla Chan"
  }
}
```

**Field Ordering**

When querying an Object, the resulting mapping of fields are conceptually
ordered in the same order in which they were encountered during query execution,
excluding fragments for which the type does not apply and fields or
fragments that are skipped via `@skip` or `@include` directives. This ordering
is correctly produced when using the {CollectFields()} algorithm.

Response serialization formats capable of representing ordered maps should
maintain this ordering. Serialization formats which can only represent unordered
maps should retain this order grammatically (such as JSON).

Producing a response where fields are represented in the same order in which
they appear in the request improves human readability during debugging and
enables more efficient parsing of responses if the order of properties can
be anticipated.

If a fragment is spread before other fields, the fields that fragment specifies
occur in the response before the following fields.

```GraphQL
{
  foo
  ...Frag
  qux
}

fragment Frag on Query {
  bar
  baz
}
```

Produces the ordered result:

```json
{
  "foo": 1,
  "bar": 2,
  "baz": 3,
  "qux": 4
}
```

If a field is queried multiple times in a selection, it is ordered by the first
time it is encountered. However fragments for which the type does not apply does
not affect ordering.

```GraphQL
{
  foo
  ...Ignored
  ...Matching
  bar
}

fragment Ignored on UnknownType {
  qux
  baz
}

fragment Matching on Query {
  bar
  qux
  foo
}
```

Produces the ordered result:

```json
{
  "foo": 1,
  "bar": 2,
  "qux": 3
}
```

Also, if directives result in fields being excluded, they are not considered in
the ordering of fields.

```GraphQL
{
  foo @skip(if: true)
  bar
  foo
}
```

Produces the ordered result:

```json
{
  "bar": 1,
  "foo": 2
}
```

**Result Coercion**

Determining the result of coercing an object is the heart of the GraphQL
executor, so this is covered in that section of the spec.

**Input Coercion**

Objects are never valid inputs.


#### Object Field Arguments

Object fields are conceptually functions which yield values. Occasionally object
fields can accept arguments to further specify the return value. Object field
arguments are defined as a list of all possible argument names and their
expected input types.

All arguments defined within a field must not have a name which begins with
{"__"} (two underscores), as this is used exclusively by GraphQL's
introspection system.

For example, a `Person` type with a `picture` field could accept an argument to
determine what size of an image to return.

```
type Person {
  name: String
  picture(size: Int): Url
}
```

GraphQL queries can optionally specify arguments to their fields to provide
these arguments.

This example query:

```GraphQL
{
  name
  picture(size: 600)
}
```

May yield the result:

```json
{
  "name": "Mark Zuckerberg",
  "picture": "http://some.cdn/picture_600.jpg"
}
```

The type of an object field argument can be any Input type.


#### Object Field deprecation

Fields in an object may be marked as deprecated as deemed necessary by the
application. It is still legal to query for these fields (to ensure existing
clients are not broken by the change), but the fields should be appropriately
treated in documentation and tooling.


#### Object type validation

Object types have the potential to be invalid if incorrectly defined. This set
of rules must be adhered to by every Object type in a GraphQL schema.

1. An Object type must define one or more fields.
2. The fields of an Object type must have unique names within that Object type;
   no two fields may share the same name.
3. Each field of an Object type must not have a name which begins with the
   characters {"__"} (two underscores).
4. An object type may declare that it implements one or more unique interfaces.
5. An object type must be a super-set of all interfaces it implements:
   1. The object type must include a field of the same name for every field
      defined in an interface.
      1. The object field must be of a type which is equal to or a sub-type of
         the interface field (covariant).
         1. An object field type is a valid sub-type if it is equal to (the same
            type as) the interface field type.
         2. An object field type is a valid sub-type if it is an Object type and
            the interface field type is either an Interface type or a Union type
            and the object field type is a possible type of the interface field
            type.
         3. An object field type is a valid sub-type if it is a List type and
            the interface field type is also a List type and the list-item type
            of the object field type is a valid sub-type of the list-item type
            of the interface field type.
         4. An object field type is a valid sub-type if it is a Non-Null variant
            of a valid sub-type of the interface field type.
      2. The object field must include an argument of the same name for every
         argument defined in the interface field.
         1. The object field argument must accept the same type (invariant) as
            the interface field argument.
      3. The object field may include additional arguments not defined in the
         interface field, but any additional argument must not be required.


### Interfaces

GraphQL Interfaces represent a list of named fields and their arguments. GraphQL
objects can then implement an interface, which guarantees that they will
contain the specified fields.

Fields on a GraphQL interface have the same rules as fields on a GraphQL object;
their type can be Scalar, Object, Enum, Interface, or Union, or any wrapping
type whose base type is one of those five.

For example, an interface may describe a required field and types such as
`Person` or `Business` may then implement this interface.

```
interface NamedEntity {
  name: String
}

type Person implements NamedEntity {
  name: String
  age: Int
}

type Business implements NamedEntity {
  name: String
  employeeCount: Int
}
```

Fields which yield an interface are useful when one of many Object types are
expected, but some fields should be guaranteed.

To continue the example, a `Contact` might refer to `NamedEntity`.

```
type Contact {
  entity: NamedEntity
  phoneNumber: String
  address: String
}
```

This allows us to write a query for a `Contact` that can select the
common fields.

```GraphQL
{
  entity {
    name
  }
  phoneNumber
}
```

When querying for fields on an interface type, only those fields declared on
the interface may be queried. In the above example, `entity` returns a
`NamedEntity`, and `name` is defined on `NamedEntity`, so it is valid. However,
the following would not be a valid query:

```!graphql
{
  entity {
    name
    age
  }
  phoneNumber
}
```

because `entity` refers to a `NamedEntity`, and `age` is not defined on that
interface. Querying for `age` is only valid when the result of `entity` is a
`Person`; the query can express this using a fragment or an inline fragment:

```GraphQL
{
  entity {
    name
    ... on Person {
      age
    }
  },
  phoneNumber
}
```

**Result Coercion**

The interface type should have some way of determining which object a given
result corresponds to. Once it has done so, the result coercion of the interface
is the same as the result coercion of the object.

**Input Coercion**

Interfaces are never valid inputs.


#### Interface type validation

Interface types have the potential to be invalid if incorrectly defined.

1. An Interface type must define one or more fields.
2. The fields of an Interface type must have unique names within that Interface
   type; no two fields may share the same name.
3. Each field of an Interface type must not have a name which begins with the
   characters {"__"} (two underscores).


### Unions

GraphQL Unions represent an object that could be one of a list of GraphQL
Object types, but provides for no guaranteed fields between those types.
They also differ from interfaces in that Object types declare what interfaces
they implement, but are not aware of what unions contain them.

With interfaces and objects, only those fields defined on the type can be
queried directly; to query other fields on an interface, typed fragments
must be used. This is the same as for unions, but unions do not define any
fields, so **no** fields may be queried on this type without the use of
typed fragments.

For example, we might have the following type system:

```
union SearchResult = Photo | Person

type Person {
  name: String
  age: Int
}

type Photo {
  height: Int
  width: Int
}

type SearchQuery {
  firstSearchResult: SearchResult
}
```

When querying the `firstSearchResult` field of type `SearchQuery`, the
query would ask for all fields inside of a fragment indicating the appropriate
type. If the query wanted the name if the result was a Person, and the height if
it was a photo, the following query is invalid, because the union itself
defines no fields:

```!graphql
{
  firstSearchResult {
    name
    height
  }
}
```

Instead, the query would be:

```GraphQL
{
  firstSearchResult {
    ... on Person {
      name
    }
    ... on Photo {
      height
    }
  }
}
```

**Result Coercion**

The union type should have some way of determining which object a given result
corresponds to. Once it has done so, the result coercion of the union is the
same as the result coercion of the object.

**Input Coercion**

Unions are never valid inputs.


#### Union type validation

Union types have the potential to be invalid if incorrectly defined.

1. The member types of a Union type must all be Object base types;
   Scalar, Interface and Union types may not be member types of a Union.
   Similarly, wrapping types may not be member types of a Union.
2. A Union type must define one or more unique member types.

### Enums

GraphQL Enums are a variant on the Scalar type, which represents one of a
finite set of possible values.

GraphQL Enums are not references for a numeric value, but are unique values in
their own right. They serialize as a string: the name of the represented value.

**Result Coercion**

GraphQL servers must return one of the defined set of possible values. If a
reasonable coercion is not possible they must raise a field error.

**Input Coercion**

GraphQL has a constant literal to represent enum input values. GraphQL string
literals must not be accepted as an enum input and instead raise a query error.

Query variable transport serializations which have a different representation
for non-string symbolic values (for example, [EDN](https://github.com/edn-format/edn))
should only allow such values as enum input values. Otherwise, for most
transport serializations that do not, strings may be interpreted as the enum
input value with the same name.


### Input Objects

Fields can define arguments that the client passes up with the query,
to configure their behavior. These inputs can be Strings or Enums, but
they sometimes need to be more complex than this.

The `Object` type defined above is inappropriate for re-use here, because
`Object`s can contain fields that express circular references or references
to interfaces and unions, neither of which is appropriate for use as an
input argument.  For this reason, input objects have a separate type in the
system.

An `Input Object` defines a set of input fields; the input fields are either
scalars, enums, or other input objects. This allows arguments to accept
arbitrarily complex structs.

**Result Coercion**

An input object is never a valid result.

**Input Coercion**

The value for an input object should be an input object literal or an unordered
map, otherwise an error should be thrown. This unordered map should not contain
any entries with names not defined by a field of this input object type,
otherwise an error should be thrown.

If any non-nullable fields defined by the input object do not have corresponding
entries in the original value, were provided a variable for which a value was
not provided, or for which the value {null} was provided, an error should
be thrown.

The result of coercion is an environment-specific unordered map defining slots
for each field both defined by the input object type and provided by the
original value.

For each field of the input object type, if the original value has an entry with
the same name, and the value at that entry is a literal value or a variable
which was provided a runtime value, an entry is added to the result with the
name of the field.

The value of that entry in the result is the outcome of input coercing the
original entry value according to the input coercion rules of the
type declared by the input field.

Following are examples of Input Object coercion for the type:

```GraphQL
input ExampleInputObject {
  a: String
  b: Int!
}
```

Original Value          | Variables       | Coerced Value
----------------------- | --------------- | -----------------------------------
`{ a: "abc", b: 123 }`  | {null}          | `{ a: "abc", b: 123 }`
`{ a: 123, b: "123" }`  | {null}          | `{ a: "123", b: 123 }`
`{ a: "abc" }`          | {null}          | Error: Missing required field {b}
`{ a: "abc", b: null }` | {null}          | Error: {b} must be non-null.
`{ a: null, b: 1 }`     | {null}          | `{ a: null, b: 1 }`
`{ b: $var }`           | `{ var: 123 }`  | `{ b: 123 }`
`{ b: $var }`           | `{}`            | Error: Missing required field {b}.
`{ b: $var }`           | `{ var: null }` | Error: {b} must be non-null.
`{ a: $var, b: 1 }`     | `{ var: null }` | `{ a: null, b: 1 }`
`{ a: $var, b: 1 }`     | `{}`            | `{ b: 1 }`

Note: there is a semantic difference between the input value
explicitly declaring an input field's value as the value {null} vs having not
declared the input field at all.

#### Input Object type validation

1. An Input Object type must define one or more fields.
2. The fields of an Input Object type must have unique names within that
   Input Object type; no two fields may share the same name.
3. The return types of each defined field must be an Input type.


### Lists

A GraphQL list is a special collection type which declares the type of each
item in the List (referred to as the *item type* of the list). List values are
serialized as ordered lists, where each item in the list is serialized as per
the item type. To denote that a field uses a List type the item type is wrapped
in square brackets like this: `pets: [Pet]`.

**Result Coercion**

GraphQL servers must return an ordered list as the result of a list type. Each
item in the list must be the result of a result coercion of the item type. If a
reasonable coercion is not possible they must raise a field error. In
particular, if a non-list is returned, the coercion should fail, as this
indicates a mismatch in expectations between the type system and the
implementation.

**Input Coercion**

When expected as an input, list values are accepted only when each item in the
list can be accepted by the list's item type.

If the value passed as an input to a list type is *not* a list and not the
{null} value, it should be coerced as though the input was a list of size one,
where the value passed is the only item in the list. This is to allow inputs
that accept a "var args" to declare their input type as a list; if only one
argument is passed (a common case), the client can just pass that value rather
than constructing the list.

Note that when a {null} value is provided via a runtime variable value for a
list type, the value is interpreted as no list being provided, and not a list of
size one with the value {null}.


### Non-Null

By default, all types in GraphQL are nullable; the {null} value is a valid
response for all of the above types. To declare a type that disallows null,
the GraphQL Non-Null type can be used. This type wraps an underlying type,
and this type acts identically to that wrapped type, with the exception
that {null} is not a valid response for the wrapping type. A trailing
exclamation mark is used to denote a field that uses a Non-Null type like this:
`name: String!`.

**Nullable vs. Optional**

Fields are *always* optional within the context of a query, a field may be
omitted and the query is still valid. However fields that return Non-Null types
will never return the value {null} if queried.

Inputs (such as field arguments), are always optional by default. However a
non-null input type is required. In addition to not accepting the value {null},
it also does not accept omission. For the sake of simplicity nullable types
are always optional and non-null types are always required.

**Result Coercion**

In all of the above result coercions, {null} was considered a valid value.
To coerce the result of a Non-Null type, the coercion of the wrapped type
should be performed. If that result was not {null}, then the result of coercing
the Non-Null type is that result. If that result was {null}, then a field error
must be raised.

**Input Coercion**

If an argument or input-object field of a Non-Null type is not provided, is
provided with the literal value {null}, or is provided with a variable that was
either not provided a value at runtime, or was provided the value {null}, then
a query error must be raised.

If the value provided to the Non-Null type is provided with a literal value
other than {null}, or a Non-Null variable value, it is coerced using the input coercion for the wrapped type.

Example: A non-null argument cannot be omitted.

```!graphql
{
  fieldWithNonNullArg
}
```

Example: The value {null} cannot be provided to a non-null argument.

```!graphql
{
  fieldWithNonNullArg(nonNullArg: null)
}
```

Example: A variable of a nullable type cannot be provided to a non-null argument.

```GraphQL
query withNullableVariable($var: String) {
  fieldWithNonNullArg(nonNullArg: $var)
}
```

Note: The Validation section defines providing a nullable variable type to
a non-null input type as invalid.

**Non-Null type validation**

1. A Non-Null type must not wrap another Non-Null type.


## Directives

A GraphQL schema includes a list of the directives the execution
engine supports.

GraphQL implementations should provide the `@skip` and `@include` directives.


### @skip

The `@skip` directive may be provided for fields, fragment spreads, and
inline fragments, and allows for conditional exclusion during execution as
described by the if argument.

In this example `experimentalField` will be queried only if the `$someTest` is
provided a `false` value.

```GraphQL
query myQuery($someTest: Boolean) {
  experimentalField @skip(if: $someTest)
}
```


### @include

The `@include` directive may be provided for fields, fragment spreads, and
inline fragments, and allows for conditional inclusion during execution as
described by the if argument.

In this example `experimentalField` will be queried only if the `$someTest` is
provided a `true` value.

```GraphQL
query myQuery($someTest: Boolean) {
  experimentalField @include(if: $someTest)
}
```

Note: Neither `@skip` nor `@include` has precedence over the other. In the case
that both the `@skip` and `@include` directives are provided in on the same the
field or fragment, it *must* be queried only if the `@skip` condition is false
*and* the `@include` condition is true. Stated conversely, the field or fragment
must *not* be queried if either the `@skip` condition is true *or* the
`@include` condition is false.


## Initial types

A GraphQL schema includes types, indicating where query, mutation, and
subscription operations start. This provides the initial entry points into the
type system. The query type must always be provided, and is an Object
base type. The mutation type is optional; if it is null, that means
the system does not support mutations. If it is provided, it must
be an object base type. Similarly, the subscription type is optional; if it is
null, the system does not support subscriptions. If it is provided, it must be
an object base type.

The fields on the query type indicate what fields are available at
the top level of a GraphQL query. For example, a basic GraphQL query
like this one:

```GraphQL
query getMe {
  me
}
```

Is valid when the type provided for the query starting type has a field
named "me". Similarly

```GraphQL
mutation setName {
  setName(name: "Zuck") {
    newName
  }
}
```

Is valid when the type provided for the mutation starting type is not null,
and has a field named "setName" with a string argument named "name".

```GraphQL
subscription {
  newMessage {
    text
  }
}
```

Is valid when the type provided for the subscription starting type is not null,
and has a field named "newMessage" and only contains a single root field. 
