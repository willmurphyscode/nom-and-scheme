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

As you can see, this is a lot of code to generate, and it chains together just fine. I think
one real strength of `nom` is it's composability.

Here's a parser that matches the beginning of a PND file and gives us a struct with the
height, width, and other basic metadata:

``` rust
named!(png_header( &[u8] ) -> PngHeader,
    do_parse!(
        _signature: png_signature >>
        _chunk_length: take!(4) >>
        _chunk_type: take!(4) >>
        width: u32!(nom::Endianness::Big) >>
        height: u32!(nom::Endianness::Big) >>
        bit_depth: take!(1) >>
        // snip ...

);
```

This has some useful properties.

1. The combined parser fails if its constituents fail - easy validation
2. You can make parsers that return user defined types
3. Pull structs you define out of byte arrays in safe Rust


First, it makes a function with this signature:

``` rust
for<'r> fn(&'r [u8]) -> nom::IResult<&'r [u8], png_demo::PngHeader>
```

Which is great. We give it a byte slice, and we might get back a PngHeader.

Validations are pretty easy. For example:

``` rust
static PNG_FILE_SIGNATURE : [u8; 8] = [
    137, 80, 78, 71, 13, 10, 26, 10
];

named!(png_signature<&[u8], &[u8]>, tag!(&PNG_FILE_SIGNATURE[..]));
```

Valid PNGs all contain these bytes at the beginning of the file. 
Notice that I'm not actually using this result in my struct, but
the parser will still fail if I don't include it.


# Sources:

- [PNG File Specification](https://www.w3.org/TR/PNG-Chunks.html)
