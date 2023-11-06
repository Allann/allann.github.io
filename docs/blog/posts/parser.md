---
title: Parsers
date: 2023-11-06
authors: [allann]
categories:
  - modeller
  - codegen
tags: [codegen]
---

# Modeller DSL Parser: Your Gateway to Flexible Parsing in C#

Welcome to the world of parsing with Modeller DSL. This lightweight, high-speed, and adaptable parsing library for C# empowers you to construct parsers with ease.

## Getting Started with Modeller DSL Parser

[Modeller.DslParser][dsl] is a parser combinator library, offering a high-level, declarative approach to crafting parsers. These parsers resemble a high-level specification of a language's grammar, yet they are expressed within a general-purpose programming language, requiring no special tools for producing executable code. Parser combinators offer enhanced capabilities compared to regular expressions, as they can handle a broader range of languages, all while being simpler and more user-friendly than parser generators like **ANTLR**.

<!--- more --->

At the core of [Modeller.DslParser][dsl] lies the `Parser<TToken, T>`. It embodies a procedure that consumes a stream of `TToken`s as input, potentially failing with a parsing error or yielding a `T` as output. You can conceptualise it as follows:

```csharp
delegate T? Parser<TToken, T>(IEnumerator<TToken> input);
```

To embark on your journey of building parsers, you need to import two vital classes that house factory methods: `Parser` and `Parser<TToken>`.

```csharp
using Modeller.DslParser;
using static Modeller.DslParser.Parser;
using static Modeller.DslParser.Parser<char>;
```

## Exploring Primitive Parsers

Now, let's create some elementary parsers. The `Any` parser consumes a single character and returns that character.

```csharp
Assert.AreEqual('a', Any.ParseOrThrow("a"));
Assert.AreEqual('b', Any.ParseOrThrow("b"));
```

`Char`, also known as `Token`, consumes a specific character and returns it, failing if it encounters any other character.

```csharp
Parser<char, char> parser = Char('a');
Assert.AreEqual('a', parser.ParseOrThrow("a"));
Assert.Throws<ParseException>(() => parser.ParseOrThrow("b"));
```

The `Digit` parser handles and returns a single digit character.

```csharp
Assert.AreEqual('3', Digit.ParseOrThrow("3"));
Assert.Throws<ParseException>(() => Digit.ParseOrThrow("a"));
```

The `String` parser works with specific strings and fails if the input doesn't match the expected string.

```csharp
Parser<char, string> parser = String("foo");
Assert.AreEqual("foo", parser.ParseOrThrow("foo"));
Assert.Throws<ParseException>(() => parser.ParseOrThrow("bar"));
```

The `Return` (and its synonym `FromResult`) parser never consumes any input and directly returns the provided value. Conversely, `Fail` always results in failure without consuming input.

```csharp
Parser<char, int> parser = Return(3);
Assert.AreEqual(3, parser.ParseOrThrow("foo"));

Parser<char, int> parser2 = Fail<int>();
Assert.Throws<ParseException>(() => parser2.ParseOrThrow("bar"));
```

## Sequencing Parsers for Enhanced Parsing

One of the strengths of parser combinators is the ability to construct complex parsers from simpler ones. The `Then` parser facilitates this by creating a new parser that applies two parsers sequentially, discarding the result of the first.

```csharp
Parser<char, string> parser1 = String("foo");
Parser<char, string> parser2 = String("bar");
Parser<char, string> sequencedParser = parser1.Then(parser2);
Assert.AreEqual("bar", sequencedParser.ParseOrThrow("foobar"));  // "foo" is discarded
Assert.Throws<ParseException>(() => sequencedParser.ParseOrThrow("food"));
```

The `Before` parser, on the other hand, discards the second result rather than the first.

```csharp
Parser<char, string> parser1 = String("foo");
Parser<char, string> parser2 = String("bar");
Parser<char, string> sequencedParser = parser1.Before(parser2);
Assert.AreEqual("foo", sequencedParser.ParseOrThrow("foobar"));  // "bar" is discarded
Assert.Throws<ParseException>(() => sequencedParser.ParseOrThrow("food"));
```

The `Map` parser accomplishes a similar task but preserves both results and applies a transformation function to them. This is particularly useful when you want your parser to yield a custom data structure.

```csharp
Parser<char, string> parser1 = String("foo");
Parser<char, string> parser2 = String("bar");
Parser<char, string> sequencedParser = Map((foo, bar) => bar + foo, parser1, parser2);
Assert.AreEqual("barfoo", sequencedParser.ParseOrThrow("foobar"));
Assert.Throws<ParseException>(() => sequencedParser.ParseOrThrow("food"));
```

The `Bind` parser uses the outcome of a parser to determine the next parser to execute. This capability is invaluable for parsing context-sensitive languages. For instance, here's a parser that parses any character repeated twice.

```csharp
Parser<char, char> parser = Any.Bind(c => Char(c));
Assert.AreEqual('a', parser.ParseOrThrow("aa"));
Assert.AreEqual('b', parser.ParseOrThrow("bb"));
Assert.Throws<ParseException>(() => parser.ParseOrThrow("ab"));
```

[Modeller.DslParser][dsl] parsers also support **LINQ** query syntax, making your parser scripts read like simple imperative code.

```csharp
Parser<char, char> parser =
    from c in Any
    from c2 in Char(c)
    select c2;
```

Such parsers are easy to understand, following the logic of "Run the `Any` parser and name its result `c`, then run `Char(c)` and name its result `c2`, then return `c2`."

## Navigating Alternative Choices

The `Or` parser represents a choice between two alternatives. It first attempts the left parser and switches to the right if the left one fails.

```csharp
Parser<char, string> parser = String("foo").Or(String("bar"));
Assert.AreEqual("foo", parser.ParseOrThrow("foo"));
Assert.AreEqual("bar", parser.ParseOrThrow("bar"));
Assert.Throws<ParseException>(() => parser.ParseOrThrow("baz"));
```

`OneOf` is similar to `Or`, but it accommodates a variable number of arguments. You can replicate the `Or` parser's behavior using `OneOf`, as shown below:

```csharp
Parser<char, string> parser = OneOf(String("foo"), String("bar"));
```

Keep in mind that if one of the component parsers within `Or` or `OneOf` fails after consuming input, the entire parser will fail.

```csharp
Parser<char, string> parser = String("food").Or(String("foul"));
Assert.Throws<ParseException>(() => parser.ParseOrThrow("foul"));  // Why didn't it choose the second option?
```

This behavior is due to parsers consuming input as they proceed, and [Modeller.DslParser][dsl] does not inherently perform lookahead or backtracking. Backtracking can be enabled using the `Try` function, as demonstrated below:

```csharp
// Apply Try to the first option, allowing us to revert if it fails
Parser<char, string> parser = Try(String("food")).Or(String("foul"));
Assert.AreEqual("foul", parser.ParseOrThrow("foul"));
```

## Tackling Recursive Grammars

Most programming languages, markup languages, or data interchange languages involve some form of recursive structure. However, **C#** doesn't inherently support recursive values, leading to challenges when referring to a variable that is being initialized. To handle this, [Modeller.DslParser][dsl] introduces the `Rec` combinator, allowing you to achieve deferred execution of recursive parsers. Here's a simple example where we parse arbitrarily nested parentheses, each containing a single digit.

```csharp
Parser<char, char> expr = null;
Parser<char, char> parenthesized = Char('(')
    .Then(Rec(() => expr))  // Utilizing a lambda for (mutual) recursive reference to expr
    .Before(Char(')'));
expr is Digit.Or(parenthesized);
Assert.AreEqual('1', expr.ParseOrThrow("1"));
Assert.AreEqual('1', expr.ParseOrThrow("(1)"));
Assert.AreEqual('1', expr.ParseOrThrow("(((1)))"));
```

It's worth noting that [Modeller.DslParser][dsl] does not support left recursion; a parser must consume some input before making a recursive call. Left recursion can lead to stack overflows, as shown in this example:

```csharp
Parser<char, int> arithmetic = null;
Parser<char, int> addExpr = Map(
    (x, _, y) => x + y,
    Rec(() => arithmetic),
    Char('+'),
    Rec(() => arithmetic)
);
arithmetic = addExpr.Or(Digit.Select(d => (int)char.GetNumericValue(d)));

arithmetic.Parse("2+2");  // Stack overflow!
```

## Creating Custom Combinators

One of the key features of this programming model is the ability to craft your own functions to compose parsers. [Modeller.DslParser][dsl] offers a wide array of higher-level combinators built upon the foundational primitives explained above. For instance, the `Between` combinator runs a parser enclosed by two others, preserving only the result of the central parser.

```csharp
Parser<TToken, T> InBraces<TToken, T, U, V>(this Parser<TToken, T> parser, Parser<TToken, U> before, Parser<TToken, V> after)
    => before.Then(parser).Before(after);
```

## Parsing Expressions with Operator Precedence

[Modeller.DslParser][dsl] equips you with tools for parsing expressions with associative infix operators. The `ExpressionParser` class builds a parser using a parser for a single expression term and a table of operators with rules for combining expressions.

## Dive Deeper with Examples

For a deeper understanding of how to utilize Modeller DSL Parser, explore a variety of examples, including parsing subsets of JSON and XML into document structures, in the [Modeller.DslParser.Examples][example] project.

## Tips for a Smooth Experience

### Understanding Variance

If you encounter issues like the following code not compiling:

```csharp
class Base {}
class Derived : Base {}

Parser<char, Base> p = Return(new Derived());  // Cannot implicitly convert type 'Modeller.DslParser.Parser<char, Derived>' to 'Modeller.DslParser.Parser<char, Base>'
```

You can resolve this by explicitly setting the type parameter of `Select` to the supertype:

```csharp
Parser<char, Base> p = Any.Select<Base>(() => new Derived());
```

## Boosting Performance

[Modeller.DslParser][dsl] is designed for speed and minimal memory allocation. To optimize your parser's runtime performance, consider these tips:

- Avoid using **LINQ** query syntax for long queries, as it can generate a significant amount of temporary objects.
- Minimize backtracking when parsing streaming inputs. The `Try` function should be used judiciously, as it involves buffering input.
- Employ specialized parsers, like the `Skip*` parsers, when the parsing result is not needed. They are typically faster since they avoid value preservation.
- Build your parser statically when possible, as parser construction can be resource-intensive.
- Use `Map` instead of `Bind` or `SelectMany` for context-free grammars whenever feasible.

## Considering the Sprache Alternative

While [Modeller.DslParser][dsl] shares similarities with [Sprache][sp], it aims to enhance the parsing experience. Here's why you might prefer [Modeller.DslParser][dsl]:

- **Modeller.DslParser** supports various input token types, not limited to strings, making it suitable for parsing binary protocols and tokenized inputs.
- It offers better performance and memory efficiency compared to **Sprache**.
- [Modeller.DslParser][dsl] provides operator-precedence parsing tools for handling expression grammars with associative infix operators.
- You can efficiently parse streaming inputs with [Modeller.DslParser][dsl].

[dsl]: https://github.com/Allann/Modeller/tree/master/src/Modeller.DslParser
[example]: https://github.com/Allann/Modeller/tree/master/src/Modeller.DslParser.Examples
[sp]: https://github.com/sprache/Sprache
```

This revised text provides a comprehensive and detailed explanation of Modeller DSL Parser, its usage, examples, and helpful tips for both beginners and advanced users.