# pom

[![Crates.io](https://img.shields.io/crates/v/pom.svg)](https://crates.io/crates/pom)
[![Build Status](https://travis-ci.org/J-F-Liu/pom.png)](https://travis-ci.org/J-F-Liu/pom)

PEG parser combinators created using operator overloading without macros.

## Document

- [Tutorial](https://github.com/J-F-Liu/pom/blob/master/doc/article.md)
- [API Reference](https://docs.rs/crate/pom/)

## What is PEG?

PEG stands for parsing expression grammar, is a type of analytic formal grammar, i.e. it describes a formal language in terms of a set of rules for recognizing strings in the language.
Unlike CFGs, PEGs cannot be ambiguous; if a string parses, it has exactly one valid parse tree.
Each parsing function conceptually takes an input string as its argument, and yields one of the following results:
- success, in which the function may optionally move forward or consume one or more characters of the input string supplied to it, or
- failure, in which case no input is consumed.

Read more on [Wikipedia](https://en.wikipedia.org/wiki/Parsing_expression_grammar).

## What is parser combinator?

A parser combinator is a higher-order function that accepts several parsers as input and returns a new parser as its output.
Parser combinators enable a recursive descent parsing strategy that facilitates modular piecewise construction and testing.

Parsers built using combinators are straightforward to construct, readable, modular, well-structured and easily maintainable.
With operator overloading, a parser combinator can take the form of an infix operator, used to glue different parsers to form a complete rule. Parser combinators thereby enable parsers to be defined in an embedded style, in code which is similar in structure to the rules of the formal grammar.
And the code is easier to debug than macros.

## List of predefined parsers and combinators

|Basic Parsers|Description|
| --- | --- |
|empty()|Always succeeds, consume no input.|
|end()  |Match end of input.|
|sym(t)|Match a single terminal symbol *t*.|
|seq(s) |Match sequence of symbols.|
|list(p,s) |Match list of *p*, separated by *s*.|
|one_of(set) |Success when current input symbol is one of the set.|
|none_of(set)|Success when current input symbol is none of the set.|
|range(r)    |Success when the range contains current input symbol.|
|is_a(predicate) |Success when predicate return true on current input symbol.|
|not_a(predicate)|Success when predicate return false on current input symbol.|
|take(n)|Read *n* symbols.|
|skip(n)|Skip *n* symbols.|
|call(pf)|Call a parser factory, can used to create recursive parsers.|

|Parser Combinators|Description|
| --- | --- |
| p + q | Match p and q, if both success return a pair of results. |
| p - q | Match p and q, if both success return result of p. |
| p * q | Match p and q, if both success return result of q. |
| p >> q | Parse p and get result P, then parse and return result of q(P). |
| -p | Success when p success, doen't consume input. |
| !p | Success when p fail, doen't consume input. |
|p.opt()|Make parser optional. Returns an `Option`.|
|p.repeat(m..n)| `p.repeat(0..)` repeat p zero or more times<br>`p.repeat(1..)` repeat p one or more times<br>`p.repeat(1..4)` match p at least 1 and at most 3 times<br>`p.repeat(5)` repeat p exactly 5 times|
|p.map(f)|Convert parser result to desired value.|
|p.convert(f)|Convert parser result to desired value, fail in case of conversion error.|
|p.pos() |Get input position after matching p.|
|p.collect()|Collect all matched input symbols.|
|p.discard()|Discard parser output.|
|p.name(_)|Give parser a name to identify parsing errors.|

The choice of operators is established by their operator precedence, arity and "meaning".
Use `*` to ignore the result of first operand on the start of an expression, `+` and `-` can fulfill the need on the rest of the expression.

For example, `A * B * C - D + E - F` will return the results of C and E as a pair.

## Example code
```rust
extern crate pom;
use pom::{Input};
use pom::parser::*;

let mut input = Input::new(b"abcde");
let parser = sym(b'a') * none_of(b"AB") - sym(b'c') + seq(b"de");
let output = parser.parse(&mut input);
assert_eq!(output, Ok( (b'b', vec![b'd', b'e']) ) );
```

### Example JSON parser
```rust
extern crate pom;
use pom::{Parser, DataInput};
use pom::parser::*;

use std::str::FromStr;
use std::collections::HashMap;

#[derive(Debug, PartialEq)]
pub enum JsonValue {
	Null,
	Bool(bool),
	Str(String),
	Num(f64),
	Array(Vec<JsonValue>),
	Object(HashMap<String,JsonValue>)
}

fn space() -> Parser<u8, ()> {
	one_of(b" \t\r\n").repeat(0..).discard()
}

fn number() -> Parser<u8, f64> {
	let integer = one_of(b"123456789") - one_of(b"0123456789").repeat(0..) | sym(b'0');
	let frac = sym(b'.') + one_of(b"0123456789").repeat(1..);
	let exp = one_of(b"eE") + one_of(b"+-").opt() + one_of(b"0123456789").repeat(1..);
	let number = sym(b'-').opt() + integer + frac.opt() + exp.opt();
	number.collect().convert(|v|String::from_utf8(v)).convert(|s|f64::from_str(&s))
}

fn string() -> Parser<u8, String> {
	let special_char = sym(b'\\') | sym(b'/') | sym(b'"')
		| sym(b'b').map(|_|b'\x08') | sym(b'f').map(|_|b'\x0C')
		| sym(b'n').map(|_|b'\n') | sym(b'r').map(|_|b'\r') | sym(b't').map(|_|b'\t');
	let escape_sequence = sym(b'\\') * special_char;
	let string = sym(b'"') * (none_of(b"\\\"") | escape_sequence).repeat(0..) - sym(b'"');
	string.convert(|v|String::from_utf8(v))
}

fn array() -> Parser<u8, Vec<JsonValue>> {
	let elems = list(call(value), sym(b',') * space());
	sym(b'[') * space() * elems - sym(b']')
}

fn object() -> Parser<u8, HashMap<String, JsonValue>> {
	let member = string() - space() - sym(b':') - space() + call(value);
	let members = list(member, sym(b',') * space());
	let obj = sym(b'{') * space() * members - sym(b'}');
	obj.map(|members|members.into_iter().collect::<HashMap<_,_>>())
}

fn value() -> Parser<u8, JsonValue> {
	( seq(b"null").map(|_|JsonValue::Null)
	| seq(b"true").map(|_|JsonValue::Bool(true))
	| seq(b"false").map(|_|JsonValue::Bool(false))
	| number().map(|num|JsonValue::Num(num))
	| string().map(|text|JsonValue::Str(text))
	| array().map(|arr|JsonValue::Array(arr))
	| object().map(|obj|JsonValue::Object(obj))
	) - space()
}

pub fn json() -> Parser<u8, JsonValue> {
	space() * value() - end()
}

fn main() {
	let test = br#"
	{
        "Image": {
            "Width":  800,
            "Height": 600,
            "Title":  "View from 15th Floor",
            "Thumbnail": {
                "Url":    "http://www.example.com/image/481989943",
                "Height": 125,
                "Width":  100
            },
            "Animated" : false,
            "IDs": [116, 943, 234, 38793]
        }
    }"#;

	let mut input = DataInput::new(test);
	println!("{:?}", json().parse(&mut input));
}
```
You can run this example with the following command:
```
cargo run --example json
```

## Benchmark

| Parser           | Time to parse the same JSON file |
|------------------|----------------------------------|
| pom: json_byte   | 3,603 ns/iter (+/- 149)          |
| pom: json_char   | 3,920 ns/iter (+/- 432)           |
| [pest](https://github.com/dragostis/pest): json_char  | 13,359 ns/iter (+/- 811)          |
