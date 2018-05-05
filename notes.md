# A general outline
_To get my thoughts in order..._

Things I want to cover:

1. `nom` is great for parsers, and what some parsers look like
2. Strategies for dealing with the inevitability to stupid compiler errors
3. A tour of my calculator REPL, and demo of adding a token

Basically, this is:

1. Why you should get nom set up
    - general talk about applications of nom, what even is a parser-combinator
    - some examples
    - the PNG example
2. How to get through the tough part
    - story about how I thought `u8!` was a thing, and the errors that resulted
    - technique of extracting a simpler parser and adding things till it breaks
    - example output from -Z -external-macro-backtrace
3. How happy things are once the going gets better
    - example of the parser in `calc-repl`
    - make up an operator for `calc-repl` and live code it.

Questions I'm still not sure how to answer:

1. Why did you use `nom` and not `LALRPOP` - personally I found it easier
2. Are there grammars that can be represented in `LALRPOP` but not `nom` - probably, but I'm not sure.
3. 