# Stubbing behavior

In test double parlance, to "stub" a method or function is to configure it to return a particular response for  a given set of inputs. When used as a noun or referred to as a "stubbing", a person could be referring to the configured test double itself or the act of returning an artificial response. When the code that you're testing depends on other units of code to do its job, the (hopefully, vast) majority of those units are invoked with a set of inputs to return some useful output. As a result, when practicing an outside-in test-driven-development workflow, being able to specify that type of interaction in a readable and specific way is very important.

In testdouble.js, we stub behavior with the `testdouble.when()` function. In this chapter, we'll discuss simple stubbings, matching inexact arguments, and advanced configuration options that testdouble.js supports.

All of the examples in this document presume the reader has aliased `testdouble` to `td`.

## testdouble.when()

The basic structure of a stubbing configuration follows:

``` javascript
var quack = td.function('quack')

td.when(quack()).thenReturn('some return value')

quack() // 'some return value'
```

As JavaScript APIs go, this should strike readers as unusual, but it's an intentional design. Note that inside the invocation operator `()` of `td.when` that the entire call to the test double is, for lack of a better term, *rehearsed*. The goal is to demonstrate an invocation of the test double function just as the test expects the subject to invoke the dependency it represents—to a level of precision desired by the situation.

> If the above is still bothering you, note that the implementation of testdouble.js doesn't care what is "passed" into `td.when()`'s first argument position. In fact, what the library will actually do is simply pop the most recent call to _any test double_ off its global call stack whenever `when()` is invoked and presume that it was a "rehearsal" invocation for the next stubbing.

The `when()` function returns an object containing the function `thenReturn`, which is used to specify the value the test double should return if it's invoked as specified in the `when()` call. For now, only `thenReturn` is supported as a response type. For whatever reason, we haven't found an urgent need to implement two other typical responses, `thenDo` (to invoke a function that has some side effect, [#8](https://github.com/testdouble/testdouble.js/issues/8)) and `thenThrow` (to throw some error, [#7](https://github.com/testdouble/testdouble.js/issues/7)), but we'd gladly accept pull requests for either.

## Simple, precise argument stubbing

When practicing outside-in TDD, unit tests are typically completely isolated. As a result, in most cases it's fair to expect that the test will be able to (and will desire to) specify the exact interactions the subject should have with each of its dependencies. The stubbing API exposed by testdouble.js assumes that the "rehearsed" invocation in a stubbing is made with deeply-equivalents arguments to what the subject is expected to use. (Granted, that is not always desired; later, we'll show how to use so-called "argument matchers" as a means to loosen that specification.)

### No-argument stubbings

The easiest invocation to stub is a no-arg invocation. In fact, you've already seen an example!

``` javascript
var quack = td.function('quack')

td.when(quack()).thenReturn('some return value')

quack() // 'some return value'
quack('anything else at all') // undefined
```

### n-argument stubbings

You can also specify arbitrarily many arguments. Given the example above, you might say:

``` javascript
td.when(quack('soft')).thenReturn('quack')
td.when(quack('soft', 2)).thenReturn('quack quack')
td.when(quack('soft', 2, 'hard', 3)).thenReturn('quack quack QUACK QUACK QUACK')

quack('soft') // 'quack'
quack('soft', 2) // 'quack quack'
quack('soft', 2, 'hard', 3) // 'quack quack QUACK QUACK QUACK'
```

As you can see above, a test double function can be stubbed multiple times for different sets of arguments.

> It's worth noting that stubbing rules are "last-in-wins", meaning that if multiple stubbings are satisfied by an invocation of the test double, the one configured latest is the one that's returned. This is almost certainly the behavior you would want in a test, because you could start with a very generic stubbing at the top of a test, and override it later from a point in the test that represents a more specific context.

### One-liner stubbings

Sometimes, when all you need from a test double is a one-off stubbing, you'll want to express both the creation of the test double and that stubbing as tersely as possible. You can do this, since `thenReturn` will return the test double instance. That means that:

``` javascript
var woof = td.function()
td.when(woof()).thenReturn('bark')
```

Is equivalent to this:

``` javascript
var woof = td.when(td.function()).thenReturn('bark')
```

### Stubbing sequential return values

Sometimes your subject should invoke a dependency multiple times in the same way. For instance perhaps your subject wants to invoke a `randomSound()` function to return `'quack'`, then `'honk'`, then `'moo'`. This can be configured by passing additional arguments to `thenReturn()`, like so:

``` javascript
var randomSound = td.function('randomSound')

td.when(randomSound()).thenReturn('quack', 'honk', 'moo')

randomSound() // 'quack'
randomSound() // 'honk'
randomSound() // 'moo'
randomSound() // 'moo'
```

As you can see, any additional invocations will continue to return the final stubbing in the sequence.

### Loosening stubbings with argument matchers

#### A note on unconditional stubbing

Note that these stubbings will only return their stubbed value if called exactly as they're demonstrated when `td.when()` is invoked. That means, given the examples above, that invoking things like:

``` javascript
quack('hi')
quack([1,2,3],4)
quack('anything','at','all')
```

Will each return `undefined`. This differs from the default stubbings one might create with Jasmine or Sinon.js, but it's by design. When your code calls another function, the arguments it passes _probably matter_. As a result, testdouble.js defaults to assuming that the inputs passed to stubbed test doubles are necessary.

For instance, if the stubbing was unconditional, we might pat ourselves on the back for writing this test:

``` javascript
function sitInTraffic(horn){
  return horn() + '!'
}

var horn = td.function()
td.when(horn()).thenReturn('beep')
assert.equal('beep!', sitInTraffic(horn))
```

But if our stubbings were unconditional, that means the test would still pass even if we changed the implementation as follows:

``` javascript
function sitInTraffic(horn){
  return horn('no really do not honk') + '!'
}
```

Which would probably not be our intention. As a result, testdouble.js only loosens its rules about whether a stub is said to be "satisfied" when you explicitly indicate that it should.

#### What are argument matchers?

An "argument matcher" is a special function which, when passed to a test double function during its `td.when()` "rehearsal" invocation, can modify the default behavior of whether a particular argument will match a given invocation. (That default behavior, recall, is to expect each actual argument to be deeply equivalent to the rehearsed argument.)

Out of the box, testdouble.js ships with a handful of matchers. They are:

#### testdouble.matchers.anything()

When passed `td.matchers.anything()`, any invocation of that test double function will ignore that parameter when determining whether an invocation satisfies the stubbing. For example:

``` javascript
var bark = td.function()

td.when(bark(td.matchers.anything())).thenReturn('woof')

bark(1) // 'woof'
bark('lol') // 'woof'
bark() // undefined
bark(2, 'other stuff') // undefined
```

#### testdouble.matchers.isA()

When passed `td.matchers.isA(someType)`, then invocations of the test double function will satisfy the stubbing when the actual type matches what is passed to `isA`. For example:

``` javascript
var eatBiscuit = td.function()

td.when(eatBiscuit(td.matchers.isA(Number))).thenReturn('yum')

eatBiscuit(5) // 'yum'
eatBiscuit('stuff') // undefined
eatBiscuit() // undefined
```

While `Number` is shown above, it will work for any built-in type (e.g. `String` or `Date`) or any named custom prototypal constructors defined by the user.

#### testdouble.matchers.contains()

When passed `td.matchers.contains()`, then a stubbed argument can be loosened to be satisfied by any invocations that simply contain the portion of the argument. This works for several types, including strings, arrays, and objects.

##### Strings

Using `contains` on a  string argument is pretty straightforward:

``` javascript
var yell = td.function()

td.when(yell(td.matchers.contains('ARGH'))).thenReturn('AYE')

yell('ARGH') // 'AYE'
yell('ARGHHHHHHH') // 'AYE'
yell('ARG') // undefined
```

##### Arrays

Here's how to use `contains` with an array argument:

``` javascript
var jellyBeans = td.function()

td.when(jellyBeans(td.matchers.contains('popcorn', 'apple'))).thenReturn('yum')

jellyBeans(['grape', 'popcorn', 'strawberry', 'apple']) // 'yum'
jellyBeans(['grape', 'popcorn', 'strawberry']) // undefined

```

##### Objects

You can also use `contains` to specify only part of an object. This is especially
useul when a large object is being passed around, but only part of it matters to
the interaction being tested.

``` javascript
var brew = td.function()

td.when(brew(td.matchers.contains({ingredient: 'beans'}))).thenReturn('coffee')

brew({ingredient: 'beans', temperature: 'hot'}) // 'coffee'
brew({ingredient: 'hops', temperature: 'hot'}) // undefined
```

Better yet, the `contains` matcher is capable of doing deep searches of sparse
properties:

``` javascript
var brew = td.function()

td.when(brew(td.matchers.contains({container: {size: 'S'}}))).thenReturn('small coffee')

brew({ingredient: 'beans', container: { type: 'cup', size: 'S'}}) // 'small coffee'
brew({ingredient: 'beans', container: { type: 'cup', size: 'L'}}) // undefined
brew({}) // undefined
```

#### testdouble.matchers.argThat()

If the other built-in matchers don't serve your needs and you don't want to roll
your own custom matcher, you can use `argThat()` to pass a truth test function
to determine whether an invocation will satisfy the stubbing.

``` javascript
var pet = td.function()

td.when(pet(td.matchers.argThat(function(animals){ return animals.length > 2 }))).thenReturn('goood')

pet(['cat', 'dog', 'horse']) // 'goood'
pet(['cat', 'dog']) // undefined
pet({length: 81}) // 'goood'
```

If you find yourself needing a matcher that isn't defined above, you can
[define your own custom matchers](8-custom-matchers.md) as well.

### Configuring stubbings

So far, we've seen `td.when()` only invoked with one argument, but it sports a
second configuration parameter as well, with a handful of options.

Keep in mind that these additional options are made available because they each
have occassional genuine value, but they ought to only be needed sparingly. If
you find yourself reaching for any of these options on a very frequent basis,
we recommend you pause and ask what about the design of your code is encouraging
the perceived need of that option.

#### ignoreExtraArgs

Sometimes, a subject will call a dependency with arbitrarily many arguments, but
for the purpose of a test, only the earlier arguments are significant to the
interaction being considered "correct". Once in a great while, none of the
arguments passed to a depended-on function matter at all. In either of these
cases, you can use the `ignoreExtraArgs` configuration property when stubbing
with `when()`

Here's an example:

``` javascript
logger = td.function()

td.when(logger("Outcomes are:"), {ignoreExtraArgs: true}).thenReturn('loggy')

logger("Outcomes are:") // 'loggy'
logger("Outcomes are:", "stuff") // 'loggy'
logger("Outcomes are:", "stuff", "that", "keeps", "going") // 'loggy'
logger("Outcomes are not:", "stuff") // undefined
```

Or, in the case where _literally none of the arguments matter_:

``` javascript
whatever = td.function()

td.when(whatever(), {ignoreExtraArgs: true}).thenReturn('yesss')

whatever() // 'yesss'
whatever(1,2,3,4,5) // 'yesss'
```

#### times

Sometimes you want to ensure that a function will only return a particular value
at most `n` times.

Note that for simple cases, one could use
[sequential stubbing](#stubbing-sequential-return-values) like
`td.when(...).thenReturn('foo','foo',undefined)` to return `'foo'` at most two
times.

For more complex cases, the `times` property can be configured to effectively
disable a stubbing after it's been satisfied `n` times. This is especially
useful in cases where a generic stubbing is configured first, but a more specific
stubbing is configured later and you want to fall back on the earlier generic
stubbing later. If this sounds incredibly obtuse and convoluted, you'd be right
to think so.

Here's an example of using the `times` property for a complex stubbing
arrangement:

``` javascript
var nextToken = td.function()

td.when(nextToken(td.matchers.isA(Number))).thenReturn("foo")
td.when(nextToken(3), {times: 2}).thenReturn("bar")

nextToken(3) // 'bar'
nextToken(5) // 'foo'
nextToken(3) // 'bar'
nextToken(3) // 'foo'
```

## Congratulations!

And that's about all there is to say about stubbing. Great news, because the
other major feature of any test double library—verifying an invocation took
place—has an API that was carefully-designed to be completely symmetrical! Read
on about [verifying interactions with `verify()`](6-verifying-invocations.md),
and rest easy knowing that you already know exactly how to do it.
