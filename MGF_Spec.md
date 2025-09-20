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
every one of them imposes some kind of a limitation on the grammar itself
due to the limitations of the parser generator.
Examples being lacking left-recursion and ambiguities introduced by the algorithm.
Some of these are trivial to fix, meaning the grammar was never bad,
just our communication with the program.

Since I believe humans shouldn't have to adapt to the machine but the other way around,
I decided to make something more obvious and flexible.

## Ideals

When defining the MGF language I went through many small iterations,
trying to make a language that is:
- Clearly readable
- Intuitive, even for someone who doesn't 100% understand MGF
- Usable when teaching the target language's grammar to a newbie
- Unambiguous and rigorously defined
- Extensible
- Not tied to string parsing
- Complete, to minimize the work required in another language for parser development
- Powerful enough to define languages higher in the Chomsky hierarchy
- Simple for regular languages

MGF isn't always easy to *write* in. Instead, its clarity should help the writer
to reduce the cognitive load needed approach the grammar design and focus on what matters.

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
- ***Identifiers*** are named using letters, numbers and underscores
  (unicode categories `Letter`, `Mark`, `Number` and `Connector Punctuation`)
    - Identifiers are the most basic MGF items
    - Example: `Digit`
- Parentheses `( )` enclose a ***group*** of items, separated by spaces
    - Groups are bigger items themselves
    - Example: `(Digit Letter Digit)`
- Pipe operator `|` separates ***alternative choices***
    - Example: `Digit | Letter`
    - Note: it has to be surrounded by spaces
- ***Symbols*** are used for giving arguments to functions.
  Any symbol except for `# ( ) | ' \ =` can be used
  (unicode categories `Punctuation` (excluding `Connector Punctuation`) and `Symbol`)
    - Examples: `+` `-` `=`
- ***Single quotes*** `' '` are used to treat *any* character as part of an identifier.
    - Example: `'+'`
    - Quotes can be part of identifiers with multiple characters.
      `Identifier' 'with' 'a' ''<'space'>'` is a single identifier, for example
- Brackets `[ ]` are used to write ***textual symbols***.
  The `[` immediately followed by an identifier is treated as a symbol, not identifier
    - This is commonly used when we want more explicit function naming,
      if pure symbols wouldn't be recognizable enough
    - Example: `[optional]` is a symbol
- A sequence of at least two symbols or items, with no spaces inbetween, is a ***function call***
    - Items are used as arguments to the call. There must be at least one symbol between them.
    - Function calls are bigger items themselves
    - The meaning of functions and identifiers depends on the context in which they are used
    - Example: `[repeat]{1-5}Letter`
- A standalone `=` symbol (not part of a function call),
  with only an identifier or a function call on the left and zero or more items on the right,
  is a ***production***
    - The right side goes until the end of the line
    - Productions are bigger items themselves
    - Productions are used to reuse items, name groups for readability,
      define new functions or write expressions that couldn't be defined without them,
      like recursive productions
    - The identifier or function call on the left side of `=` is the production's name
    - An item of the same name, or fitting function calls expand into the items on the right
    - A line *starting* with just `=` makes an alternative choice
      for the right side of the production on the previous line
    - Example: `Digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9`
    - Recursion example: `Expression = Expression ('+' | '-') Number`
    - Function example: `[thrice]Pattern = Pattern Pattern Pattern`

### Scoping
MGF doesn't use proper namespaces. Instead, it uses a robust prefix system.

Scopes have names. The scope names are used as prefixes
to access the productions existing inside them.
When productions are written in a sub-scope,
they will also exist in the outside scope with slightly modified names:
- If its name is an identifier, the identifier gets the prefix
- If the name is a function call, we have two cases:
    - If it starts with a textual symbol, the identifier *inside* the first symbol gets the prefix
    - If it starts with an item or a non-textual symbol,
      it gets a *new* starting textual symbol, having only the prefix

### Contexts
Items interact with the **context** they are evalated in. A single context can have 
The following contexts are defined and used by the standard functions:
- **Choice context**: Allows multiple choices of a result
    - All context properties can be different for every choice
    - When matching, allows multiple possible matches
    - When having a value, allows multiple possible independent values
    - Has an intrinsic order of choices, which is useful when deciding on one of them
    - Purpose: Recognition of different inputs and chosing the most suitable interpretation
- **Matching context**: A context in which items are evaluated to match inputs from a stream
    - It consists of a stream of input values, which can be textual characters,
      but also tokens with classes or other values.
    - Expanding a function into items in this context means processing the inputs
      by the items it expanded into
    - When an item matches a string of inputs, the following items can only match following inputs
      after the matched ones.
    - Alternative choices can match different strings starting at the same place in the stream.
      Each choice branches into its own match, with it's own following matches.
    As the branches are independent, each introduced branch effectively copies the whole context
- **Value context**: A context in which items produce meaningful values
    - This context itself is a value
    - The values of evaluated items are *included* in the context.
      The fields of the item's value are added to the context,
      with list-based fields being appended to the fields of the context.
      The context is a **parent** value to the included values
    - Purpose: constructing semantic data, ASTs, or annotated values during parsing
- **Group context**: A context used only within an item group
    - Usually has a parent context, being the context that surrounds the evaluated group
    - Purpose: properties that won't affect its parent, like temporary fields
- **Scope context**: The scope we're evaluating in
    - Has its prefix and a parent it is a sub-scope of
    - All productions that are defined within it also exist in its parent,
      following the prefixing rules
    - Items evaluated in this context that refer to productions defined in it don't need prefixes


### Built-in functions
Several functions are provided by default:
- `[import]ModuleName`
    - Loads the module `ModuleName` and expands into a sequence of productions from that module,
      prefixed with `ModuleName`
- `[using]Prefix::Selection`
    - Expands into a sequence of productions,
      mapping productions with prefixes removed to prefixed ones
    - `Selection` can either be a single item or a group of all items
      we want to remove prefixes from.
    - The items can either be identifiers or function calls
        - If it's an identifier, it selects productions named that identifier, prefixed with `Prefix`
        - If it's a function call, it selects the productions
          with that exact function call as their names, prefixed with a textual symbol `[Prefix]`.
          If it already starts with a textual symbol, the text itself is prefixed.
    - The removal is done backwards from the scope prefixing rule:
        - If the production is named with an identifier, the prefix gets removed from the identifier
        - If the production is a function,
          the prefix gets removed from the textual symbol it starts with.
          If the symbol was only the prefix, it gets removed completely
- `[include]Prefix`
    - Expands into a sequence of productions,
      mapping **all** productions with prefixes `Prefix` removed to prefixed ones
    - Like `[using]`, but it automagically selects all the productions prefixed with `Prefix`
- `[scope]Prefix::ScopeGroup`
    - In a scope context: Introduces a new sub-scope, with the specified prefix
    - Expands into a sequence of productions from the `ScopeGroup`,
      except prefixed with `Prefix` according to scoping rules

### Standard module
The module is named `standard_`.
After importing, all standard productions will use the `standard_` prefix.

`standard_` functions and identifiers:
- `[optional]Pattern`
    - In a matching context: matches `Pattern` and empty stream
- `[repeat]{Lower-Upper}Pattern`
    - In a matching context: matches at minimum `Lower`,
      up to `Upper` number of repeating `Pattern` matches
- `[repeat]{Lower+}Pattern`
    - In a matching context: matches any number greater than `Lower` repeating `Pattern` matches
- `[first]Options` `[shortest_first]Options` and `[longest_first]Options`
    - In a matching context: When multiple options match the stream, chooses one of them.
      So any option matches the stream only if the options with a higher priority *don't* match it.
        - `first`: Chooses the option that is written first
        - `longest_first`/`shortest_first`: Chooses the option that matches the longest/shortest
          sequence of the stream. If multiple match that length, choose the first one.
- `[class]ClassId`
    - In a value context: adds class `ClassId` to the value
- `[match_classes_from]Pattern`
    - Expands into a sequence of productions,
      one per each class mentioned indirectly in the Pattern.
    - Each production has the corresponding `ClassId` as its name,
      and matches *one* stream item if that item has that `ClassId`
    - Used to match inputs already parsed through MGF, after applying classes to their outputs
- `[field]Name:Value`
    - In a value context: expands into the `Value`,
      but instead of including the value in its surrounding context
      it stores it into the field `Name` of the surronding context
    - The value can be a matching pattern whose value will be stored into the field,
      or any kind of value-returning item
- `[push]Name:Value`
    - In a value context: expands into the `Value`, but instead of including the value
      in its surrounding context it appends it to the list in field `Name` of the surronding context
    - The value can be a matching pattern whose value will be appended into the field,
      or any kind of value-returning item
- `[temp]Name`
    - In a value-group context: Makes a temporary field `Name` in the group.
      The temporary field will exist inside the group, but won't be in the parent value context.
    - All references to the field `Name` in this group context will be of this temporary field
- `[multistep]Step->...`
    - In a matching context: matches each `Step`,
      and uses each `Step`'s value as a list of inputs
      to build a matching context for the next `Step`.
      The first `Step`'s matching context is the same as the surrounding one,
      while each next `Step` has its own one
    - In a value context: expands into the value of the last `Step`, after evaluating all the others
    - Example: `[multistep]TokenStream->PreprocessedTokenStream->ParserTree`
- `[set]First[to]Last`
    - In a matching context with string inputs: matches all characters between `First` and `Last`
- `wildchar`: Wild character
    - Matches any single input

String matching `standard_` scopes:
- `standard_unicode_characters_`: Contains all single-character identifiers
  matching unicode characters in a textual input stream
    - Examples: `'"' a b c '"' ' ' '[' 1 2 3 ']' `
- `standard_unicode_categories_short_`: Contains all two-character shorthands
  for matching characters of unicode categories
    - Each identifier consists of one uppercase and one lowercase letter.
      'Letter, uppercase' is written as `Lu`.
    - Examples: `Name = Lu [repeat]{1+}Ll`