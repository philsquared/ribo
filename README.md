# Ribo

<div align="center">
    <img src="ribo.png" style="width:120px">
</div>

Ribo is a declarative programming language. 
Think something like JSON or YAML - but with types. And nicer.

But Ribo doesn't exist in a vacuum. 
Just as JSON is a declarative subset of Javascript, 
Ribo is a declarative subset of a family of languages, 
with the head of the family being [Pheno](https://pheno-lang.org).

## A Tour of Ribo

### Comments
Ribo uses the `#` character for comments. Any text following a `#` is ignored until the end of the line (unless it is with a quoted string).

```
# This is a Ribo comment
let i = 42 # <-- the code to the left is evaluated, but this comment is ignored
```

### Built in types

Ribo has a set of built-in types, similar to JSON, with the addition of datetime types:

```
booleans = true
integers = 42
floats = 3.141
strings = "hello"
dates = 2018/05/15
times = 19:16:00Z
datetimes = 2018/05/15T19:16:00Z
```

Unlike, JSON, however, these are static types, and can be explicitly annotated:

```
booleans: bool = true
integers: int = 42
floats: float = 3.141
strings: str = "hello"
dates: date = 2018/05/15
times: time = 19:16:00Z
datetimes: datetime = 2018/05/15T19:16:00Z
```

With type inference you'll rarely have to specify types where it wouldn't otherwise be needed.


Ribo also supports four container types:

```
arrays: [int] = [1, 2, 3]
dicts: [str: int] = ["one": 1, "two": 2, "three": 3]
sets: set<int> = set([1, 2, 3])
optionals: str? = "another string"
```

Again, the type annotations are usually optional. They can all be inferred except the last one.
Types suffixed with ? are optional types. 
Again this is a common shortcut and could also have been written `optional<str>` in this case.
Optionals may be considered containers (they can contain zero or one values),
but more accurately they are examples of sum types, which we'll introduce in a moment.

Before we do that we should introduce the multiline format for containers.
Ribo is an indentation sensitive language (like Python or OCaml).
So those first three container examples could also be written (and this time without the annotations):

```
arrays =
    1
    2
    3

dicts = 
    "one": 1
    "two": 2
    "three": 3

sets = set
    1
    2
    3
```
Notice that the comma separators are not necessary in this format. Neither are the square brackets.
The round brackets for the set constructor are also optional (in both singleline and multiline formats).
This has implications for arrays of objects, as we'll see in a moment, but where each element is unambiguously single line the only sepatator necessary is the newline itself.

### Custom types

Ribo supports Algebraic Data Types (ADTs).
These are becoming common in modern programming languages and allow you to be very expressive with minimal syntax.
New types are introduced with the `type` keyword. More specifically this introduces a type *binding*, so we use the `=` operator:

```
type Widget =
    name: str
    size: int    
```

This is an example of a "Product Type". In other languages these may be called structs, classes, records or even tuples (tuples usually have unnamed fields, which we can do, too, but we'll look at that later).
A product type is simply a sequence of fields of some other types (which could be built-ins or other ADTs).
Each field typically has a name and type and, optionally, a default value.
(technically the name and type are also optional, and we may have associated metadata - but we'll get to all that).

Values of product types can be created using constructor syntax, 
which simply involves using the name of the type, followed by a sequence of values to initialise each field with.
This sequence is usually enclosed in round brackets. The brackets are optional if the constructor is unambiguous in context.
The type name is also optional if it can be inferred.
E.g.:

```
let widget1 = Widget("My widget", 42)
let widget2 = Widget "My other widget", 7
let widget3 : Widget = ("Inferred widget", 9)
```

Values can be bound to names using "let bindings" as above, 
although in Ribo this is not usually necessary.
In a Ribo module (source file) the last expression evaluated becomes the "module binding",
which is what the processor usually receives.

We said the names and types of the fields are optional. 
Types can be optional if the types can be inferred (which implies a type that is immediately used, rather than as part of a named binding).
Names are also optional, and can be useful for small, limited use, types (we would usually think of these as tuples).
But if both are optional how do we know if a name on its own is a field name or a type?
If field names are omitted then the colon for the type annotation must be present.

So a type with no field names might look like this:

```
type MyTuple = (:int, :str)

let my_tuple = MyTuple(42, "unnamed")
```

As well as product types Ribo supports "Sum Types". 
These are often also called Discriminated or Tagged Unions, Variants or event "enums".
The latter is becoming common in languages like Rust or Swift, but may be confusing if you come from most C-family languages.
A Sum Type is a sequence of fields of some other type - just like a product type - 
except rather than having *all* the field values it only has one at a time.
Sum type fields start with the `|` operator (except the first field in the single line form) and may omit a type annotation (in which case `()` is implied).
So in the simplest case they really are like C-style enums:

```
type CardSuits = Clubs | Spades | Hearts | Diamonds
```

But because each field may have any type (which may be defined inline) we can get quite sophisticated:

```
type Contact:
    | None
    | Phone: (area: int, number: int)
    | Email: str
    
let contact1 = Contact.None
let contact2 = Contact.Phone(123, 4567890)
let contact3 : Contact.Email = "abc@example.com"
let contact4 : Contact = .Email("abc@example.com")
```

Notice the type inference at play at multiple levels. With Phone we infer the whole type from the constructor.
With Email, for contact3, we annotate the qualified subtype, so supply just the arguments (round brackets optional).
For contact4 we annotate just the Sum type, so need to specify the subtype for the constructor. 
The subtype is Contact.Email but we can use partial inference (with the leading `.`) to elide the Contact.

Notice, also, that we can define product types inline (the type of Phone), without needing to create a type binding up front.

So Sum Types are just compositions of types using the `|` operator to mean "this type or that type".
We can also compose types using the `&` operator to mean "this type *and* that type".
This is known as an "Intersection Type":

```
type A = (name: str)
type B = (size: int)
type C = A & B
```

Intersection types may only compose composite types (ie those with one or more fields).
So you can't compose, say, `str` and `int` this way.

If fields with the same name exist in more than one arm of an intersection type then the later one overrides the earlier one.
Because types may be composed without needing named bindings we can use intersection types to extend existing types with new fields
(like OO inheritence).
E.g.

```
type A = (name: str)
type B = A & (size: int)

let b = B("Extended", 42)
```

Notice, also, that intersection types still get constructors do memberwise initialisation.

Talking of those constructors, as well as positional arguments, as we have been using, 
they also take named arguments:

```
let b = B(name: "Extended", size: 42)
```

Notice that `:` is used to bind an argument name to a value.

Named and positional arguments may be mixed but, as is usual in languages with this support,
all positional arguments must come first, followed by any named arguments.
Any arguments not specified either positionally or by name take their default values, if any - 
otherwise it is an error.

### Multiline considerations
Most types of expressions or statements have a single line and a multiline form
and we have seen most of them already.

Arrays of object literals need some extra explanation.

Given:

```
type Widget = (name: str, size: int)
```

An array of widgets may be written

```
let widgets =
    Widget("one", 1)
    Widget("two", 2)
    Widget("three", 3)
```

But repeating `Widget` a lot may get tedious, so we may want to use type inference:

```
let widgets : [Widget]=
    ("one", 1)
    ("two", 2)
    ("three", 3)
```

that also works. The round brackets make it unambiguous that each line is a separate element.
But if we want to write the object constructors over multiple lines, too, we might try to write:

```
let widgets: [Widget] =
  "one"
  1
  "two"
  2
  "three"
  3
```

This could be made to work for this case but (a) gets hard to read and (b) breaks down when you have optional fields.
Instead we introduce a special "object separator" syntax. This involves write two or more `-` characters on a line as a separator:

```
let widgets: [Widget] =
  "one"
  1
  --
  "two"
  2
  --
  "three"
  3
```

This is somewhat similar to YAML's `-` prefix for each object but 
(a) is used as a separator with no extra indentation and
(b) is only necessary for arrays of objects in this multiline of multiline formats.

### Attributes
Any value literal can be passed to the processor as metadata by prefixing it with the `@` character.
This is most commonly applied to constructors, e.g.

```
type Name = (:str)

type Widget =
    @Name("Widget's name)
    name: str

    @Name("Widget's size")
    size: int
```

Because round brackets are optional when unambigous, 
unit types (anything that evaluates to `()`) are often used as marker attributes:

```
type Deprecated = ()
type Required = ()

type Widget =
    @Required
    name: str
    
    @Deprecated
    @Required
    size: int
```

For types, attributes must always precede the type binding.
For fields attributes may either precede the whole field (on the same or preceding lines),
or *trail* the field *on the same line*:

```
@Deprecated
type Widget =    
    @Name("Widget's name)
    name: str @Required
        
    size: int @Deprecated @Required
```

### Importing other modules

A Ribo module is scoped to the file it is defined within.
Other modules can be imported into the current module using the `import` statement.

```
# other_module.ribo:
type Widget = (name: str, size: int)
```

```
import "some_path/other_module.ribo" as m2 # Import whole module into a namespace

m2.Widget("my widget", 42)
```

```
import from "some_path/other_module.ribo" Widget # Import specific entities into current namespace

Widget("my widget", 42)
```
