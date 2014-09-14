---
layout: "page"
title: "Notes"
author: "Dan Wysocki"
---

# Introduction

```
compiler(source) -> destination
```
Source can be any programming language. Destination can be machine code,
byte code, or even more source code, even in the same language
(code obfuscators are one such example).

- lexical scanning tokenization
- context-free parsing
    - AST (abstract tree)
        - functions as a parse tree, but is a more convenient representation
    - two types of parsing
        - top down (LL)
		    - recursive descent
        - bottom up (LR)
            - generates a table
            - humans do not write LR parsers by hand
    - semantic analysis
	    - type checking / type inference
        - static semantics
        - dynamic analysis

          ```
          int a = b + c + d;
          // can be represented by //
          mov   b tmp
          add   c tmp
          add   d tmp
          mov tmp   a
          ```

        - dataflow (graph)
            - optimization
                - forward
                - backward
    - code generation
        - multiple approaches
            - tree-based
                - munch
            - graph-based
                - peep-hole
            - bytecode
                - jit

# Tokenization

```java
class A { int x; if (x == 0) ... }
```

- regular expressions
    - `A in Alphabet`
    - `x a`
    - `l == { a, b, ..., z, A, B, ..., Z, _ }`
	- `d == { 0, 1, 2, ..., 9 }`
    - `ID == l(l|d)*`
    - `class` and `if` both match `ID`, so you either make explicit expressions
      for them *before* `ID`, or you just tokenize them as identifiers and
      parse them separately later
    - reason not to like Fortran, number 7813
	    - language cannot be defined with regular expressions

          ```fortran
          C      this is legal
                 do i = 3,6
          C      so is this
                 doi=3
          ```
    - literals
        - int lit (integer literal)
            - do we treat negatives and explicit positives as a unary operator
			  (`+` or `-`) followed by a number, or as a single entity
			  (e.g. `-1`, `+3`)
                - `d+`
                - `(+|-)?d+`
            - what about hexadecimal, binary, and octal literals?
        - float literal
            - Some languages make you do `0.1`, while others allow `.1`
            - What about `1.`?
            - Assuming we allow all of these, we could write the regular
              expression `(d*\.d+)|(d+\.d*)`
            - what about scientific notation? (e.g. `1.3e5`) you could do
              `(d*'.'d+)|(d+'.'d*)(('e'|'E')('-'|'+')?d+)?`
            - some languages let you add an `f` or `d` afterwards to specify
              single vs double precision, so
              `(d*'.'d+)|(d+'.'d*)(('e'|'E')('-'|'+')?d+)?(f|d)?`
        - char literal
            - looks like `'a'`
            - we were using `'` to denote literal characters in regexes already,
              so I guess we'll have to escape it to mean the literal `'` char
            - `\'[^\']|\\.\'`
        - string literal
            - assuming multi-line strings are not allowed,
              `"([^\\"]|\\.)*"`
    - non-deterministic finite state automaton (NDFA)
        - for any NDFA there is a DFA which accepts the same language
        - for any DFA there is a table
        - for any table you can write an interpreter
        - some machines for regexes
            - `x`   ![](/images/nfa_x.svg)
            - `xy`  ![](/images/nfa_xy.svg)
            - `x|y` ![](/images/nfa_x_OR_y.svg)
            - `x+`  ![](/images/nfa_x+.svg)

              |--------+-----------------+---------+---------+--------|
              | *n*    |                 | l       | d       | .      |
              |--------|-----------------|---------|---------|--------|
              | 0      | {b, c, e, g}    | cl(b)   | &perp;  | &perp; |
              |--------+-----------------+---------+---------+--------|
              | 1      | {d, g, b, c, e} | cl(d)/2 | cl(f)/3 | &perp; |
              |--------+-----------------+---------+---------+--------|
              | 2      | {f, g, b, c, e} | 2       | 3       | &perp; |
              |--------+-----------------+---------+---------+--------|

- lexical analysis
    - yacc
	- lex, flex, parser generator

      ```grep
      [0-9]+         { return INTLIT; }
      [a-z][a-z0-9]* { return screen(yynxt); }
      //             { eatto('\n'); }
      ```

- parsing
    - assume we have a language parsable by a CFG
	  (though this is usually a lie)
    - `S -> A B | C d`
    - `A -> A x | ε`
    - `A -> ( A )`
    - `A -> x*`
    - `E -> E * E | E + E | v`
        - ambiguous statement (order of operations)
        - moral: every CFG can have ambiguities
        - every language should not have ambiguities
        - Fortress was a language that let people program math in TeX
            - project failed because it was too hard with all the ambiguities
    - dangling else problem
        - `if ... if ... else ...`
    - top down parsing
        - `S -> A B C | D`
        - `S() { A(); B(); C(); }`
        - `A -> a`
        - `A() { match('a'); }`
            - `match` is a call to your lexical analyzer
        - operations
            - `check(token)`
            - `advance`
            - `match`
                - `check` then `advance`

        ```java
        if (check('a')) { // I think DL might have meant `check('s')`
            A();
            B();
            C();
        } else {
            D();
        }
        ```

        - ll-parsing
        - recursive descent
            - for every left-hand-side of the grammer, write a potentially
              recursive method
        - `S -> A | xA | ε`

          ```antlr
          # ambiguous precedence
          E -> E + E | E * E | ( E ) | v
          # break it apart for unambiguity
          E -> E + T | T
          T -> T * F | F
          F -> ( E ) | v
          # C does 9 levels of this trick
          ```

        - head grammar with `S -> E $`, where `$` denotes `EOF`
            - this makes sure everything ends with the end of file
        - defining something like `E -> E + E ...` would mean writing a method
          which does `E() { E() }`, which is baad.
        - Left Recursion Transformations remove the recursion problem

          ```antlr
          # originally
          A  -> A α1 | A α2 | ... | A αN | β
          # transform to
          A  -> β A'
          A' -> α1 A' | α2 A' | ... | αN A' | ε
          ```

        - using this transformation on the previous example

          ```antlr
          E  -> T E'
          E' -> + T E' | ε
          T  -> F T'
          T' -> * F T' | ε
          # unchanged
          F -> ( E ) | v
          ```

        - Left Factoring

          ```antlr
          A  -> α β1 | α β2 | γ
	      # transforms into
          A  -> α A' | γ
          A' -> β1 | β2
          ```

          ```antlr
          A -> B x | B y
          # transforms into
          A  -> B A'
          A' -> x | y
          ```

- yet another grammar example

```antlr
E : E E
  | E '*'
  | E '|' E
  | '(' E ')'
  | ID
  ;
```

```antlr
E : E '|' T
  | T
  ;
T : T F
  | F
  ;
F : F '*'
  | A
  ;
A : '(' E ')'
  | ID
  ;
```

```antlr
E  : T E_
   ;
E_ : '|' T E_
   | ε
   ;
T  : F T_
   ;
T_ : F T_
   | ε
   ;
F  : A F_
   ;
F_ : '*' F_
   | ε
   ;
A  : '(' E ')'
   | ID
   ;
```

- first and follows
    - `A : a | b;`
    - pick `a` if next token is in `first(a)` if `a = t` and next token is in
      `Follow(A)`
    - algorithm
        1. add `S' -> S $`
        2. `A -> α B β`
		   add `first(β)` to `follow(B)`
		3. `A -> α B` or `A -> α B β` nullable `β`
           add `Follow(A)` to `Follow(B)`

      |---------------------+--------------------|
      |                     | `S -> E $`         |
	  |---------------------+--------------------|
	  | `A  = { (, ID }`    | `f(F')`            |
	  |---------------------+--------------------|
	  | `F' = { ε, A }`     | `f(T')`            |
	  |---------------------+--------------------|
	  | `F  = { (, ID }`    | `f(T')`            |
	  |---------------------+--------------------|
	  | `T' = { ε, (, ID }` | `F(T)`             |
	  |---------------------+--------------------|
	  | `T  = { (, ID }`    | `f(E')`            |
	  |---------------------+--------------------|
	  | `E' = { ε, | }`     | `F(T)`             |
	  |---------------------+--------------------|
	  | `E  = { (, ID }`    | `F(T)`  `)` `$`    |
      |---------------------+--------------------|

- LL parsers
    - transform to be LL-parsable
	- find first/follows
	- for each lhs: `A -> B | C`
		- for each symbol `x` in `first(B)`
			- if `B` is nullable, for each `x` in `Follows(B)`, check and call
              `B`
			- otherwise, throw errors
				- error recovery
					- add
					- delete
					- replace
- LR parsers

  ```
  S' -> S $
  S  -> 'int' L
  L  -> L ',' ID
  L  -> ID
  ```

  ```C
  int a, b
  ```

    - first thing we see is `'int'`, which is a prefix which is part of a viable
      right hand side (`S`), so we push it onto a stack

	  |-------|
	  | stack |
      |-------|
	  | `int` |
	  |-------|

    - then we see `a` which is an `ID`, (though we don't know which RHS it goes
      in), so we push it onto a stack

	  |-------|
	  | stack |
      |-------|
	  | `int` |
	  | `ID`  |
	  |-------|

	- the stack now contains `'int' ID`, which matches `'int' L`, so we reduce

	  |-------|
	  | stack |
      |-------|
	  | `int` |
	  | `L`   |
	  |-------|

    - now we read in `,`, which is part of `L`, and push it onto the stack

	  |-------|
	  | stack |
      |-------|
	  | `int` |
	  | `L`   |
	  | `,`   |
	  |-------|

    - now we read `b`, which is another `ID`

	  |-------|
	  | stack |
      |-------|
	  | `int` |
	  | `L`   |
	  | `,`   |
	  | `ID`  |
	  |-------|

    - we now have `L ',' ID`, which is another valid `L`, so we reduce

      |-------|
	  | stack |
      |-------|
	  | `int` |
	  | `L`   |
	  |-------|

    - we have `'int' L`, which is a valid `S`, so we reduce

	  |-------|
	  | stack |
      |-------|
	  | `S`   |
	  |-------|

    - now we read in `$`, so we push it onto the stack

	  |-------|
	  | stack |
      |-------|
	  | `S`   |
	  | `$`   |
	  |-------|

    - `S $` is a valid `S'`, so we reduce to `S'`, and we're done!
    - Item Sets:
	    - Set 0 (core) `S' -> S $`
		- Closure `S -> 'int' L`
		- the two above are together one item set

      |-----+----------------------+-----------------------|
	  | Set |         Core         |        Closure        |
	  |-----+----------------------+-----------------------|
	  |  0  | `S' -> S $` (1)      | `S -> 'int' L` (2)    |
	  |-----+----------------------+-----------------------|
	  |  1  | `S' -> S . $`        | (acc)                 |
	  |-----+----------------------+-----------------------|
	  |  2  | `S -> int . L` (3)   | `L -> . L ',' ID` (3) |
	  |-----+----------------------+-----------------------|
	  |     |                      | `L -> . ID` (4)       |
	  |-----+----------------------+-----------------------|
	  |  3  | `S -> 'int' L .` (-) |                       |
	  |-----+----------------------+-----------------------|
	  |     | `S -> L . , ID`  (3) |                       |
	  |-----+----------------------+-----------------------|
      |  4  | `L -> ID` (-)        |                       |
	  |-----+----------------------+-----------------------|
	  |  5  | `S -> L ',' . ID`    |                       |
	  |-----+----------------------+-----------------------|
	  |  6  | `S -> L ',' ID .`    |                       |
	  |-----+----------------------+-----------------------|

      |-----+-------+-----------+-------+---------------+-------+-------|
	  | set | `int` | `','`     | `ID`  | `$`           | `S`   | `L`   |
      |-----+-------+-----------+-------+---------------+-------+-------|
	  |  0  |  +2   |           |       |               | goto1 |       |
      |-----+-------+-----------+-------+---------------+-------+-------|
	  |  1  |       |           |       | acc           |       |       |
      |-----+-------+-----------+-------+---------------+-------+-------|
      |  2  |       |           |  +4   |               |       | goto3 |
      |-----+-------+-----------+-------+---------------+-------+-------|
      |  3  |       |  +5       |       | `S -> int L`  |       |       |
      |-----+-------+-----------+-------+---------------+-------+-------|
      |  4  |       | `L -> ID` |       | `L -> ID`     |       |       |
      |-----+-------+-----------+-------+---------------+-------+-------|
      |  5  |       |           |  +6   |               |       |       |
      |-----+-------+-----------+-------+---------------+-------+-------|
      |  6  |       |           |       | `S -> L, int` |       |       |
      |-----+-------+-----------+-------+---------------+-------+-------|

    - now our parse looks something like

      |-----------+-----------+---------|
	  | 0         | 1         | 2       |
	  |-----------+-----------+---------|
	  | `' ' 0`   | `' ' 0`   | `' ' 0` |
	  |-----------+-----------+---------|
	  | `'int' 2` | `'int L'` | `'S'`   |
	  |-----------+-----------+---------|
	  | `'ID' 4`  |           |         |
	  |-----------+-----------+---------|