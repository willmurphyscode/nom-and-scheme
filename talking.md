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
The nom result wraps whatever type you want, but the default is a byte slice.
A `nom::IResult::Done` has two fields: the rest of the input, and the part of the input that matched.

Let's take a look at what makes `nom` a parser-combinator. That is, what makes us able to combine parse
functions. We can go back to this simple example:

 ``` rust
named!(parse_cargo,
    alt!(tag!(&b"cargo"[..]) | tag!(&b"Cargo"[..]))
);
```

`alt!` is a macro that combines to parsers to make another parser that's happy if any of the
parsers it combines match. It accepts any number of parsers separated by pipes.

Some other handy combiners of parsers are `many0!` and `many1!` (these work like `*` and `+` in regex).

Let's look at the expansion of our little `parse_cargo`, or at least snippets of it. It makes a lot of code:

``` rust
    #[allow(unused_variables)]
    fn parse_cargo(i: &[u8]) -> ::IResult<&[u8], &[u8], u32> {
        {
            {
                let i_ = i.clone();
                let res =
                    {
                        use ::{Compare, CompareResult, InputLength, Slice};
                        let res: ::IResult<_, _> =
                            match (i_).compare(&b"cargo"[..]) {
                                CompareResult::Ok => {
                                    let blen = (&b"cargo"[..]).input_len();
                                    ::IResult::Done(i_.slice(blen..),
                                                    i_.slice(..blen))
                                }
                                CompareResult::Incomplete => {
                                    ::IResult::Incomplete(::Needed::Size((&b"cargo"[..]).input_len()))
                                }
                                CompareResult::Error => {
                                    ::IResult::Error(::ErrorKind::Tag)
                                }
                            };
                        res
                    };
                match res {
                    ::IResult::Done(_, _) => res,
                    ::IResult::Incomplete(_) => res,
                    ::IResult::Error(e) => {
                        let out =
                            {
                                let i_ = i.clone();
                                let res =
                                    {
                                        use ::{Compare, CompareResult,
                                               InputLength, Slice};
                                        let res: ::IResult<_, _> =
                                            match (i_).compare(&b"Cargo"[..])
                                                {
                                                CompareResult::Ok => {
                                                    let blen =
                                                        (&b"Cargo"[..]).input_len();
                                                    ::IResult::Done(i_.slice(blen..),
                                                                    i_.slice(..blen))
                                                }
                                                CompareResult::Incomplete => {
                                                    ::IResult::Incomplete(::Needed::Size((&b"Cargo"[..]).input_len()))
                                                }
                                                CompareResult::Error => {
                                                    ::IResult::Error(::ErrorKind::Tag)
                                                }
                                            };
                                        res
                                    };
                                match res {
                                    ::IResult::Done(_, _) => res,
                                    ::IResult::Incomplete(_) => res,
                                    ::IResult::Error(e) => {
                                        let out =
                                            ::IResult::Error(::ErrorKind::Alt);
                                        fn unify_types<T>(_: &T, _: &T) { }
                                        if let ::IResult::Error(ref e2) = out
                                               {
                                            unify_types(&e, e2);
                                        }
                                        out
                                    }
                                }
                            };
                        fn unify_types<T>(_: &T, _: &T) { }
                        if let ::IResult::Error(ref e2) = out {
                            unify_types(&e, e2);
                        }
                        out
                    }
                }
            }
        }
    }
```