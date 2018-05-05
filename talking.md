Tonight I'm going to talk about `nom`. `nom` is a parser-combinator framework
that makes nice, clean macros for parsing through data. These macros make a
DSL for describing how to pull apart a file (byte by byte).

Using macros for this framework is a trade-off with, in my opinion, a big win
and a big loss. The big win is that the resulting code is very readable. I
hope you agree that the examples of nom parsers I'll show you are fairly easy
to read. The big loss from this tradeoff is that it can make some *bizarre* compiler
errors. This loss is a particularly big disadvantage in Rust, since so much
of the time we get such nice errors.

To start with, let's write a simple parser that recognizes the string `"cargo"`.

``` rust
named!(parse_cargo,
    alt!(tag!(&b"cargo"[..]) | tag!(&b"Cargo"[..]))
);
```

`tag!` is a macro that expands to a function for parsing a literal. `alt!` is a macro
that will be happy as long as any of the pipe-separated sub-parsers are happy. `named!` is
a macro that makes a function named what you want. 

Let's take a look at calling `parse_cargo()`, the function that `named!` gave us in the
previous example.

``` rust
fn print_parse_results(input: &str) {
    match parse_cargo(input.as_bytes()) {
        nom::IResult::Done(input, output) => println!("Rest of input: {:?} \n output: {:?}", input, output),
        nom::IResult::Error(err) => println!("Error: {:?}", err),
        nom::IResult::Incomplete(needed) => println!("Incompled, needed {:?}", needed),
    }
}
```

`named!`, by default, makes a function that accepts a byte slice and returns a `nom::IResult`.

