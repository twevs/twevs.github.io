# Leveraging clangd to implement structured navigation and editing of C/C++ code

>source > text => source editor > text editor. i'm ready to try an editor that only allows ast transformations.

[Kent Beck](https://twitter.com/KentBeck/status/311983951218630656) ([bio](https://en.wikipedia.org/wiki/Kent_Beck))

## Table of contents
- [Background: the relationship between source code, AST, and the editing experience](#background)
- [The experiment: implementing structured navigation and editing in a Visual Studio Code extension](#experiment)
- [Difficulties encountered along the way](#difficulties)
- [Conclusion: what to take away from this?](#conclusion)

## Background: the relationship between source code, AST, and the editing experience <a name="background"></a>

Structured editing, that is, editing source code in such a way as to manipulate the abstract syntax tree directly, is not a novel idea, at least outside of the world of C/C++; indeed, practical implementations of it have long been available as [Emacs minor modes](https://www.gnu.org/software/emacs/manual/html_node/emacs/Minor-Modes.html) for one of the oldest extant (family of) programming languages, namely, Lisp. This language lends itself all the more naturally and easily to structured editing on account of its being written as nested lists of [S-expressions](https://en.wikipedia.org/wiki/S-expression), in consequence of which in Lisp, syntax and AST are _arguably_ one and the same. An example of the ease and fluidity with which the Lisp programmer is afforded the ability directly to work on the AST is offered by the following video on the subject of [paredit](https://www.emacswiki.org/emacs/ParEdit), one such Emacs minor mode:

[![Emacs Rocks! Episode 14: Paredit](https://img.youtube.com/vi/D6h5dFyyUX0/0.jpg)](https://www.youtube.com/watch?v=D6h5dFyyUX0)

More recently, the idea that the AST rather than text might be the manner in which programs' internal representation is stored was the object of an impressive proof of concept in the form of the Dion Systems language, an overview of which was given by its creators Allen Webster and Ryan Fleury in the very interesting presentation [Dion Systems - The How And Why Of Reinventing The Wheel](https://guide.handmade-seattle.com/c/2020/dion-systems-reinventing-wheel/) for Handmade Seattle 2020; they show how a programming language thus specified and implemented could allow the programmer trivially to work directly with the AST as well as prevent classes of problems typically associated with text-first languages. The overall look and feel of the language as presented is C-like, but crucially, the syntax in this case does not precede and _produce_ the AST but is merely a way to _represent_ it according to the preferences of the programmer, such that while Bob and Alice work on the very same code, Bob can see the Allman brace style and four-space indentations and Alice can enjoy her K&R brace style and two-space indentations. (Still it remains that Bob has the correct view of things, of course. :-)

One slide from that presentation which is particularly illustrative for our purposes is the following:

![image](https://user-images.githubusercontent.com/77587819/233859684-c7c01c99-d547-4e17-8a23-8033cde43a52.png)

C/C++ programs have their source code stored as text, which is then lexed and parsed into an AST; the consequence of this trivial and obvious fact is that we take it just as much for granted that the primary way to edit source code is to edit text by operating on the tokens that compose it: we add and delete characters, either individually or in groups (as words or lines, or in the case of Vim users, as visual blocks). But this is not how we actually _think_ of our source code: when we look at it, what we see is, say, a function definition containing an `if` statement, a `while` statement and a function call with several arguments. Thus, there exists a fundamental cognitive mismatch between the level at which programmers _conceptualize_ their code and the level at which they _edit_ it.

How might this gap be bridged? What would it be like if the editor met the programmer at the level at which they think of their code?

## The experiment: implementing structured navigation and editing in a Visual Studio Code extension <a name="experiment"></a>

As one of its proprietary extensions to the Language Server Protocol, clangd offers the possibility of inspecting the AST using a [`textDocument/ast` request](https://clangd.llvm.org/extensions#ast). Given a file and a region of source code to inspect (specified as a range whose start and end points are defined using line number and intra-line character index), clangd returns an `ASTNode` object whose most interesting members are the `kind` string (eg, "BinaryOperator") and the `children` array of `ASTNode`s.

This suggested itself as a means of delivering a proof of concept of what this might look like. I decided to create a fork of the [Vim extension for Visual Studio Code](https://github.com/VSCodeVim/Vim) with the very modest aim of implementing for C/C++ some of the functionality offered by paredit for Lisp, the thought being that after having implemented highlighting of the current AST node, I would simply add the ability to make a given statement an elder sibling of its parent node, substitute it for its parent node, or delete it outright; the project expanded in scope as I came to a greater understanding of what it would truly mean to make the AST node the _basic_ unit of editing, resulting in a wholesale replacement of Vim navigational keybindings which truly puts the AST node at the centre of the editing experience.

Instead of navigating to the previous _character_, the _h_ key now navigates to the previous _sibling_ of the current AST node (wrapping around if one requests the previous sibling of the first child; note that `internal` is `#define`d as `static`):

![previous_sibling_2](https://user-images.githubusercontent.com/77587819/233864268-ef0af8c9-847b-425c-b63f-08b966470f44.gif)

Likewise, the _l_ key now navigates not to the next character, but to the following sibling of the current AST node (with the same wraparound behaviour):

![next_sibling_2](https://user-images.githubusercontent.com/77587819/233864283-3ff3b8f7-01af-41b2-80fd-4986b7b8f635.gif)

_j_ is repurposed and no longer goes to the character below, but to the first child of the current node. In the following example, we progressively narrow down our scope from the function to one of its innermost statements, changing siblings along the way as necessary:

![child](https://user-images.githubusercontent.com/77587819/233864493-433c7a15-e814-42df-aa67-f15563d0f07a.gif)

_k_ meanwhile no longer goes to the character above, but to the parent node. Here, we take the contrary journey, from the statement to the function:

![parent](https://user-images.githubusercontent.com/77587819/233864529-4d75d744-7ed0-4933-b0a7-193f7ab1de56.gif)

_e_ (for "extract") makes the current node an elder sibling of its parent:

![extract](https://user-images.githubusercontent.com/77587819/234072392-651e32af-0002-4598-93c7-adb5108a0e4d.gif)

_s_ (for "substitute") replaces the parent node with the current node:

![substitute](https://user-images.githubusercontent.com/77587819/234072410-438cf4dd-c9c0-4184-b03e-de05ae9bc5b1.gif)

And _x_ now deletes the current node:

![delete](https://user-images.githubusercontent.com/77587819/234072423-5dc54d91-6d47-4f8b-b177-4ee5ca4d9955.gif)

## Difficulties encountered along the way <a name="difficulties"></a>

### Accessing the parent node

One major problem was that the information returned by clangd in answer to a request for details about the current AST node does not provide an easy way to access the parent node; this is an area where the disadvantage of C/C++ syntax in going from text to syntax tree made itself particularly felt. The naïve solution is simply to work backwards through the text sending AST node requests to clangd until one inevitably encounters the parent, identified by finding the current node among its children; but in fact, there are situations in which the only text for which clangd would return the parent lies _after_ that for the child, as in the case of the following function call:

```cpp
void foo(Bar *bar);
// ...
Bar *bar;
foo(bar);
```

Whose tree in clangd is as follows:

```
Call
├─ ImplicitCast (FunctionToPointerDecay)
|  └─ DeclRef (foo)
└─ ImplicitCast (LValueToRValue)
   └─ DeclRef (bar)
```

If the cursor is in the function name `foo`, making AST node requests for any text prior to it will fail to return the parent Call node, which clangd instead provides only in response to a request made on either of the parentheses enclosing the arguments _following_ the function name. A further difficulty is suggested by the fact that clangd returns the DeclRef node for `foo`, while its immediate parent is not the Call node representing the whole function, but rather an ImplicitCast node whose range is coextensive with its own. Thus, it was necessary to emend the backwards search by ensuring that the current node was sought not only among the queried node's immediate children, but among all its descendants. Further care had to be taken not to pursue a query further in the first place if a node returned during this backwards search was among the children of the current node.

Another example of how one might encounter trouble in finding the parent is a function definition:
```cpp
void foo(Bar *bar)
{
    // ...
}
```
One has to consider the fact that the only text for which clangd returns the whole Function is `foo` — awkward if the cursor is in `void`, with the end result being that the best way to handle the situation is to make this a special case in which one _does_ search forwards. Prefixing qualifiers complicate things further: `static` immediately returns the Function node, while given `extern "C" __declspec(dllexport)`, the Function node itself becomes a _child_ of a LinkageSpec node.

### Working with preprocessor definitions and macros

Another difficulty in trying to facilitate working directly with the AST of C/C++ code is the preprocessor, which is, of course, but a plain and simple text-replacement step that runs before any other code processing does. We generally only think of preprocessor definitions and macros as something that matters when we intentionally compile our code rather than at the editing stage; but as anyone who has had to work with Unreal code in anything less than the most recent versions of Visual Studio knows from drowning in IntelliSense's flood of red squiggles (IntelliSense sometimes getting so confused as to find error with the `class` keyword itself), preprocessor definitions can be a real challenge at the editor level. So it turned out to be here, since clangd logically produces its AST after the text has been preprocessed, necessarily creating a disconnect between the text that one is working with and the actual, substituted text from which clangd forms its AST. Some macros (such as Windows's `GET_(X|Y)_LPARAM()`) were particularly problematic to deal with; the solution was to skip character ranges when nodes had `null` ranges.

### Problems with clangd's AST information

In many instances, the AST information returned by clangd was inconsistent and frustrating to work with. One such problem was having to deal with nodes of the UnresolvedLookup type, which clangd is only supposed to yield when awaiting argument-dependent lookup or dealing with either an overloaded function or a function template; it was quite puzzling, therefore, to have UnresolvedLookup nodes be returned for non-overloaded, non-templated functions dealing with `glm::vec3` ... but not `glm::vec2`, even though both are almost identically defined typedefs for a templated GLM vector type. Another such instance was clangd returning Recovery nodes, meant for "semantic errors that prevent creating other well-formed expressions", for simply accessing the `glm::vec3` member of a pointed-to struct. In the end, working through the developments needed to produce generalized AST-based navigation overcame these particular hurdles.

### Impossibility of acting as a second client alongside the VSCode clangd extension

Quite apart from these issues, which were all quite central to the challenge of implementing AST-based editing of C/C++ code, there was also the practical problem of having to deal with the fact that [the Language Server Protocol states](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#languageServerProtocol):

>The protocol currently assumes that one server serves one tool. There is currently no support in the protocol to share one server between different tools. Such a sharing would require additional protocol e.g. to lock a document to support concurrent editing.

The consequence of this is that one cannot have two Visual Studio Code extensions both acting as clients to the clangd server and, therefore, that being unable to have this Vim extension fork run _alongside_ vscode-clangd, I made the choice of including the latter.

## Conclusion: what to take away from this? <a name="conclusion"></a>

Overall, this project (published [here](https://github.com/twevs/VSCodeStructuredEditing)) has proved to be an interesting one, yielding a proof of concept which will hopefully spur both further discussion about both the desirability of changing some of the ways we work with source code and further experimentation with what those ways might be.

As it is, the extension works fairly well for navigational purposes with the code that I have tried it with, but the editing experience per se I do not find to be a particularly satisfactory one in its current state. Beyond stability and fluidity issues that one might expect from a rudimentary proof of concept, it still bears too much the mark of being a haphazardly nurtured Vim extension fork in which the ideal wished for by Kent Beck, ie only allowing AST transformations, is sketched only in the embryonic form of a few select actions, in relation to which more traditional text insertion operations now function awkwardly.

To reach a practical implementation of that ideal would require work going far beyond the modest ambition set out for this project. While contributions are welcome despite these provisos, I should be content to think this might serve as humble food for thought for others interested in the area of interaction between programmers and code.
