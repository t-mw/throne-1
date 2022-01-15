![Throne](Throne.png)


A scripting language for game prototyping and story logic:

```
// Declare the initial state as 'phrases', with one phrase per line.
Mary is sister of David
Sarah is child of Mary
Tom is child of David

// Define rules with the format: INPUT = OUTPUT.
CHILD is child of PARENT . AUNT is sister of PARENT .
    COUSIN is child of AUNT = COUSIN is cousin of CHILD

// The final state will be:
//    Sarah is cousin of Tom
```

## Motivation

The original inspiration for Throne comes from languages used to author interactive fiction, such as [Inform](http://inform7.com/).
As described in [this](https://brunodias.dev/2017/05/05/inform-prototyping.html) article, Inform can be used to prototype games from genres besides interactive fiction. By defining gameplay through rules, some of the verbosity of general purpose programming languages can be avoided. However, Inform and other interactive fiction authoring systems are too slow to execute their rules in every frame of a real-time gameplay loop and are difficult to embed in an existing engine.

Throne allows gameplay logic to be defined through rules and so provides some of the benefits of a rule-based language like Inform, but is also fast to execute and easy to embed in an existing engine. Its rule syntax and mechanics began as simplified versions of those found in the [Ceptre](https://www.cs.cmu.edu/~cmartens/ceptre.pdf) programming language, which was the main influence for the design of Throne.

## Examples
- [throne-playground](https://github.com/xkbc/throne-playground-suite) - a web-based editor for Throne, made possible by compiling Throne to WebAssembly.
- [blocks](examples/blocks.throne) - a simple tile matching game run with `cargo run --example blocks`.

## Reference

Rules are of the format `INPUT = OUTPUT`, where `INPUT` and `OUTPUT` are lists that use period (`.`) as a separator between items:
- `INPUT` is a list of one or more conditions that must pass for the rule to be executed. The conditions can either be state [phrases](#phrases) that must exist or [predicates](#predicates) that must evaluate to true. Any [matching](#matching) state phrases are consumed by the rule on execution.
- `OUTPUT` is a list of state phrases that will be generated by the rule if it is executed.
- Identifiers in rules that use only uppercase letters (`CHILD`, `PARENT`, `AUNT` and `COUSIN` in the snippet above) are variables that will be assigned when the rule is executed.

Evaluating a Throne script involves executing any rule that matches the current state until the set of matching rules is exhausted. Rules are executed in a random order and may be executed more than once.

### Phrases

The building blocks of a Throne script are phrases, which are collections of arbitrary character sequences (referred to in this section as *atoms*) separated by spaces. In other words, a phrase is a list of atoms separated by spaces.

A list of phrases completely defines the state of a Throne script. Combined with [predicates](#predicates), lists of phrases also completely define the inputs and outputs of the rules in a Throne script.

The following are examples of valid phrases:

| Example | Notes |
| --- | --- |
| `1 2-abC Def_3` | ASCII alphanumeric characters, optionally mixed with dashes and underscores, are the simplest way to define a phrase. This phrase contains three atoms. |
| `"!complex$" "1 2"` | Atoms containing non-ASCII alphanumeric characters, or that should include whitespace, must be double-quoted. This phrase contains two atoms. |
| `foo MATCH_1ST AND-2ND` | Atoms that contain only uppercase ASCII alphanumeric characters, optionally mixed with dashes and underscores, define a variable when included in a rule's list of inputs. This phrase contains three atoms, the last two of which are variables. |
| <code>this&nbsp;(is&nbsp;(a&nbsp;nested)&nbsp;phrase)</code> | Surrounding a set of atoms with parentheses causes them to be nested within the phrase. This affects variable [matching](#matching). |

### Predicates

The following predicates can be used as one of the items in a rule's list of inputs:

| Syntax | Effect | Example |
| --- | --- | --- |
| `+ X Y Z` | Matches when the sum of `X` and `Y` equals `Z` | `+ HEALTH 10 HEALED` |
| `- X Y Z` | Matches when the sum of `Y` and `Z` equals `X` | `- HEALTH 10 DAMAGED` |
| `< X Y` | Matches when `X` is less than `Y` | `< MONEY 0` |
| `> X Y` | Matches when `X` is greater than `Y` | `> MONEY 0` |
| `<= X Y` | Matches when `X` is less than or equal to `Y` | `<= MONEY 0` |
| `>= X Y` | Matches when `X` is greater than or equal to `Y` | `>= MONEY 0` |
| `% X Y Z` | Matches when the modulo of `X` with `Y` equals `Z` | `% DEGREES 360 REM` |
| `= X Y` | Matches when `X` equals `Y` | `= HEALTH 100` |
| `!X` | Matches when `X` does not exist in the state | `!this does not exist` |
| `^X` | Calls the host application and matches depending on the response | `^os-clock-hour 12` |

When a predicate accepts two input variables, both variables must be assigned a value for the predicate to produce a match. A value is assigned either by writing a constant inline or by sharing a variable with another of the rule's inputs.
When a predicate accepts three input variables and one of the variables remains unassigned, it will be assigned the expected value according to the effect of the predicate e.g. `A` will be assigned the value `8` in `+ 2 A 10`.

### Matching

Before a rule is executed, each of its inputs must be successfully matched to a unique state phrase.
An input matches a state phrase if they are equal after all variables have been assigned.

The following are examples of potential matches:

| Rule Inputs | State Phrases | Outcome |
| --- | --- | --- |
| `health H = ...` | `health 100` | A successful match if `H` is unassigned or was already set to `100` by another input in the rule. `H` is then assigned the value `100`. |
| `health H . health H = ...` | `health 100` | A failed match because only one compatible state phrase exists. |
| `health H . health H = ...` | <pre>health 100<br/>health 200</pre> | A failed match because `H` cannot be set to both `100` and `200`. |
| `health H1 . health H2 = ...` | <pre>health 100<br/>health 200</pre> | A successful match if `H1` and `H2` are unassigned or already assigned compatible values by other inputs in the rule. `H1` is then assigned the value of `100` or `200`, and `H2` is assigned the remaining value. |
| `health H1 . health H2 . + H1 10 H2 = ...` | <pre>health 100<br/>health 200</pre> | A failed match because the predicate cannot be satisfied. |
| `health H = ...` | `health ((min 20) (max 100))` | A successful match if `H` is unassigned or was already set to `(min 20) (max 100)` by another input in the rule. `H` is then assigned the phrase `(min 20) (max 100)`. |
| `health (_ (max H)) = ...` | `health ((min 20) (max 100))` | A successful match if `H` is unassigned or was already set to `100` by another input in the rule. `H` is then assigned the value `100`. `_` is a wildcard that will match anything, in this case the phrase `min 20`. |

### Constructs

Special syntax exists to make it easier to write complex rules, but in the end these constructs compile down to the simple form described in the introduction. The following table lists the available constructs:

| Syntax | Effect | Example | Compiled Form |
| --- | --- | --- | --- |
| Input phrase prefixed with `$` | Copies the input phrase to the rule output. | `$foo = bar` | `foo = bar . foo` |
| A set of rules surrounded by curly braces prefixed with `INPUT:` where `INPUT` is a list of phrases | Copies `INPUT` to each rule's inputs. | <pre>foo . bar: {<br/>  hello = world<br/>  123 = 456<br/>}</pre> | <pre>foo . bar . hello = world<br/>foo . bar . 123 = 456</pre> |
| `<<PHRASE . PHRASES` where `PHRASE` is a single phrase and `PHRASES` is a list of phrases | Replaces `<<PHRASE` with `PHRASES` wherever it exists in a rule's list of inputs. | <pre><<math A B . + A 1 B<br/><<math A C . - A 1 B . % B 2 C<br/>foo X . <<math X Y = Y</pre> | <pre>foo X . + X 1 Y = Y<br/>foo X . - X 1 B . % B 2 Y = Y</pre> |

### The `()` Phrase

The `()` phrase represents the absence of state. When present in a rule's list of outputs it has no effect, besides making it possible to write rules that produce no output (e.g. `foo = ()`).
When present in a rule's list of inputs it has the effect of producing a match when no other rules can be matched. For example, in the following script the first rule will only ever be matched last:

```
foo           // initial state
() . bar = () // matched last
foo = bar     // matched first
```

In this way `()` can be used as a form of control flow, overriding the usual random order of rule execution.

### Stage Phrases

Prefixing a phrase with `#` marks it as a 'stage' phrase.
Stage phrases behave in largely the same way as normal phrases, but their presence should be used to indicate how far a script has executed within a sequence of 'stages'. A stage phrase can be included in a rule's inputs to only execute the rule within that stage of the script's execution, and included in a rule's outputs to define the transition to a new stage of the script's execution.

Stage phrases only differ in their behavior to normal phrases when used as a prefix to a set of curly braces. In this case the stage phrase will be copied to not only the inputs of the rules within the braces, but also the outputs, except when a rule includes `()` as an input phrase. This makes it easy to scope execution of the prefixed set of rules to a stage and finally transition to a second stage once execution of the first stage is complete.

| Example | Compiled Form |
| --- | --- |
| <pre>#first-stage: {<br/>  foo = bar<br/>  () = #second-stage<br/>}</pre> | <pre>#first-stage . foo = #first-stage . bar<br/>#first-stage . () = #second-stage</pre> |

## Build for WebAssembly

1. Run `cargo install wasm-pack` to install [wasm-pack](https://github.com/rustwasm/wasm-pack).
1. Run `npm install ; npm start` in this directory.
