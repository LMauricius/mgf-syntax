# MGF

Function Grammar Form

It's ***not*** Mauricius' Grammar Form 😉.

## Motivation - Why MGF?

Originally I started working on MGF due to minor frustrations with the usual EBNF -
big differences from the real parsed code, significant usage of commas,
separation of the lexer and the parser
and the usually required combination with Turing-complete languages.

While I'm not against using different tools best suited for different purposes,
I *am* bothered by combining different tools for the *same* purpose
because one of them is insufficient.
To me, that meant the first tool wasn't good enough to be used for the task in the first place,
and if almost every usage requires occasional usage of the more powerful tool,
the first one is bad in itself.

Another pet-peeve of mine were the toolsets built around existing grammar specifications - 
every one of them imposes some kind of a limitation on the grammar itself due to the limitations of the parser generator.
Examples being lacking left-recursion and ambiguities introduced by the algorithm.
Some of these are trivial to fix, meaning the grammar was never bad, just our communication with the program.

Since I believe humans shouldn't have to adapt to the machine but the other way around,
I decided to make something more obvious and flexible.

## Ideals

When defining the MGF language I went through many small iterations, trying to make a language that is:
- Clearly readable
- Intuitive, even for someone who doesn't 100% understand MGF
- Usable when teaching the target language's grammar to a newbie
- Unambiguous and rigorously defined
- Extensible
- Not tied to string parsing
- Complete, to minimize the work required in another language for parser development
- Powerful enough to define languages higher in the Chomsky hierarchy
- Simple for regular languages

MGF isn't always easy to *write* in. Instead, its clarity should help the writer to reduce the cognitive load needed approach the grammar design and focus on what matters.

## Features
As a modern language, MGF provides the following features:
- Modules
- Scopes
- Function functions
- Function modifiers for specifying properties
- Structural approach for storing data
- Class system for enum-like fields
- Simple escape characters
- WYSiWYG for terminal symbols
- Unicode support OotB
- Potential for tweaking error messages

## Syntax

### Overview
- MGF is structured as a sequence of ***items***, separated by spaces
- `#` starts a ***comment***, till the end of the line
    - Example: `# This is a comment`
- ***Identifiers*** are named using letters, numbers and underscores (unicode categories `Letter`, `Mark`, `Number` and `Connector Punctuation`)
    - Identifiers are the most basic MGF items
    - Example: `Digit`
- Parentheses `( )` enclose a ***group*** of items, separated by spaces
    - Groups are bigger items themselves
    - Example: `(Digit Letter Digit)`
- Pipe operator `|` separates ***alternative choices***
    - Example: `Digit | Letter`
- ***Symbols*** are used for giving arguments to functions. Any symbol except for `# ( ) | ' \` can be used ( unicode categories `Punctuation` excluding `Connector Punctuation` and `Symbol`)
    - Examples: `+` `-` `=`
- ***Single quotes*** `' '` are used to treat *any* character as part of an identifier.
    - Example: `'+'`
    - Quotes can be part of identifiers with multiple characters. `Identifier' 'with' 'a' ''<'space'>'` is a single identifier, for example
- Backslash `\` is used to write ***textual symbols***. The `\` immediately followed by an identifier is treated as a symbol, not identifier
    - This is commonly used when we want more explicit function naming, if pure symbols wouldn't be recognizable enough
    - Example: `\optional` is a symbol
- A sequence of at least one symbol and one item, with no spaces inbetween, is a ***function call***
    - Items are used as arguments to the call. There must be at least one symbol between them.
    - Function calls are bigger items themselves
    - The meaning of functions and identifiers depends on the context in which they are used
    - Example: `\repeat{1-5}Letter`
- A standalone `=` symbol (not part of a function call), with an identifier or a function call on the left, and one or more items on the right, is a ***production***
    - Productions are bigger items themselves
    - Productions are used to reuse items, name groups for readability, define new functions or write expressions that couldn't be defined without them, like recursive productions
    - The identifier or function call on the left side of `=` is the production's name
    - Example: `Digit = 0|1|2|3|4|5|6|7|8|9`
    - Recursion example: `Expression = Expression ('+'|'-') Number`
    - Function example: `\thrice:Pattern = Pattern Pattern Pattern`

### Scoping
MGF doesn't use proper namespaces. Instead, it uses a robust prefix system.

Scopes have names. The scope names are used as prefixes to access the productions existing inside them.
When productions are written in a sub-scope, they will also exist in the outside scope with slightly modified names:
- If its name is an identifier, the identifier gets the prefix
- If the name is a function call, we have two cases:
    - If it starts with a textual symbol, the identifier *inside* the first symbol gets the prefix
    - If it starts with an item or a non-textual symbol, it gets a *new* starting textual symbol, having only the prefix

### Contexts

### Built-in functions
Several functions are provided by default:
- `\import[ModuleName]`
    - Loads the module `ModuleName` and expands into a sequence of productions from that module, prefixed with `ModuleName`
- `\using[Prefix]Selection`
    - Expands into a sequence of productions, mapping productions with prefixes removed to prefixed ones
    - `Selection` can either be a single item or a group of all items we want to remove prefixes from.
    - The items can either be identifiers, symbols or function calls
        - If it's an identifier, it selects productions named that identifier, prefixed with `Prefix`
        - If it's a textual symbol, it selects all function productions starting with that symbol, prefixed with `Prefix`
        - If it's a function call, it selects the productions with that exact function call as their names, prefixed with a textual symbol `\Prefix`
    - The removal is done backwards from the scope prefixing rule:
        - If the production is named with an identifier, the prefix gets removed from the identifier
        - If the production is a function, the prefix gets removed from the textual symbol it starts with. If the symbol was only the prefix, it gets removed completely
- `\include[Prefix]`
    - Expands into a sequence of productions, mapping **all** productions with prefixes `Prefix` removed to prefixed ones
    - Like `\using`, but it automagically selects all the productions prefixed with `Prefix`
- `\include(ScopeGroup)`
    - Expands into a sequence of all the productions existing inside the `ScopeGroup`
    - Note the lack of square brackets
- `\scope[Prefix]:ScopeGroup`
    - Introduces a new scope, with the specified prefix
    - Expands into a sequence of productions from the `ScopeGroup`, except prefixed with `Prefix` according to scoping rules

### Standard module
The module is named `standard_`. After importing, all standard productions will use the `standard_` prefix.