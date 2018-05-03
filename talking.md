First I want to give you a tour of two example parsers that do different things. One
of them is for parsing an old binary file format for Zork games, and the second
one is for parsing strings into the tokens of a simple, scheme-like language.

Here's the first parser that I want to talk about:

``` rust
named!( take_z_word<&[u8],ZWord>,
    bits!(
        do_parse!(
            word_end_bit: take_bits!( u8, 1 ) >>            
            first: take_bits!( u8, 5 ) >>
            second: take_bits!( u8, 5 ) >>
            third: take_bits!( u8, 5 ) >>
            (
                ZWord {
                    first: first,
                    second: second,
                    third: third,
                    word_end_bit: word_end_bit
                }
            )
        )
    )   
);

```

This is the nom part of a parser that unpacks the way Zork games store strings. Basically,
Zork has three 