---
layout: default
title: "Assignment 3"
---

**Due date**: Wednesday, Oct 23rd by 11pm Baltimore time

# Compiler: semantic analysis

In this assignment you will implement semantic analysis and type
checking for a compiler for a subset of the C programming language.

## Getting started

Download [assign03.zip](assign03.zip) and unzip it.

To compile the program, use the commands

```
make depend
make -j
```

You will be (primarily) modifying the source files `include/semantic_analysis.h` and
`sema/semantic_analysis.cpp`. You will also almost certainly need to modify the
`NodeBase` class (defined in `include/node_base.h` and `ast/node_base.cpp`) to add
data members and member functions. Note that you are free to modify other source files if
necessary.

In this assignment, the expected output of the executable
(which is called "`nearly_cc`") is a textual representation of the symbol
table entries created by the semantic analyzer, or (if appropriate)
an error message printed to `stderr`.  The executable should be invoked
using a command of the form

<div class="highlighter-rouge"><pre>
$ASSIGN03_DIR/nearly_cc -a <i>sourcefile</i>
</pre></div>

where `$ASSIGN03_DIR` expands to the directory containing your work, and
*sourcefile* is the C source file to analyze.

The "`-a`" command line option stands for "analyze."

## Your task

Your task is to implement the node visitation member functions in
the `SemanticAnalysis` class so that

* symbol tables are created and populated for the input source program, and
* the input source is checked for semantic errors, which should be reported
  using the `SemanticError::raise` function

## Visualizing the AST

Your semantic analyzer will need to work with the AST representation of the input
code. We strongly recommend that you make use of the `-p` command line option
to have your `nearly_cc` program print a text representation of the AST. This will
give you detailed knowledge of how the AST is represented for whichever part of
the AST you are working on.  For example:

<div class='highlighter-rouge'><pre>
$ <b>cat input/example01.c</b>
int main(void) {
  return 0;
}
$ <b>$ASSIGN03_DIR/nearly_cc -p input/example01.c</b>
AST_UNIT
+--AST_FUNCTION_DEFINITION
   +--AST_BASIC_TYPE
   |  +--TOK_INT[int]
   +--TOK_IDENT[main]
   +--AST_FUNCTION_PARAMETER_LIST
   +--AST_STATEMENT_LIST
      +--AST_RETURN_EXPRESSION_STATEMENT
         +--AST_LITERAL_VALUE
            +--TOK_INT_LIT[0]
</pre></div>


You might also find it helpful to refer to
`parse_buildast.y`, which is the Bison parser, since its semantic actions are
responsible for creating the AST.

## Testing

As with Assignments 1 and 2, **a public test suite is provided to serve as a
detailed specification of the expected functionality of your program**.
(I've emphasized this text because I want to emphasize that the public tests
will probably be at least as useful as a functional specification as the text
in this assignment description.)

To use the test suite, start by setting the `ASSIGN03_DIR` environment
variable to the path of the directory containg your work, e.g.

```
export ASSIGN03_DIR=~/compilers/assign03
```

assuming that `~/compilers/assign03` is the directory containing your code.

You will want to change directory into the directory `assign03` of
your clone of [the public test repository](https://github.com/jhucompilers/fall2024-tests).
(Running `git pull` first is a good idea, since you might not have the Assignment 3
tests yet, or we might have added some since the last time you pulled.)

The test inputs are in the "`input`" directory.  To run a test, use the
`run_test.rb` script as follows, specifying the base name of the test you want
to run:

```
./run_test.rb example01
```

The command above will test your semantic analyzer on the test input
called `input/example01.c`.

You can see the actual output and/or error message produced by your program
in the `actual_output` and `actual_error` directories. You can compare these
to the expected output and error in the `expected_output` and `expected_error`
directories.

You can also use the command

```
./run_all.rb
```

to run all of the tests and report the results.

## Detailed requirements, specifications, and advice

This will be a fairly complex assignment, but if you approach it methodically
you should be able to make steady progress.

### Suggested approach

One way to get started is to work on the `visit_basic_type` member function.
This function should analyze the basic type and type qualifier keywords,
and then create a `BasicType` object (wrapped in a `std::shared_ptr<Type>` smart
pointer) to represent the basic type. If either of the type qualifiers
were specified, then the `BasicType` can be wrapped in a `QualifiedType`.)
Of course, if the combination of keywords doesn't identify a valid basic
type (e.g., `long char`) then use `SemanticError::raise` to report the error.
If the basic type is valid, annotate its AST node with a (shared)
pointer to the type object.

Once basic types are handled, you could tackle variable declarations.
You will need to first process the base type, then (based on the type
representation determined for the base type) you can work your way
through the declarators and add each one as a symbol table entry (since
each one will be defining a variable.)

If you're at the point where declarations like

```c
unsigned int x, y[10], *z;
```

are working well, then you're off to a very good start.

Analyzing expressions should be fairly straightforward, as long as you can
handle the leaf nodes, which will be `AST_VAR_REF` and the various kinds of
literals. Each leaf should be annotated with (at least) a type. Variable reference
nodes should be annotated with a pointer to the `Symbol` representing the
symbol table entry for the referred-to variable or function.

### SemamticAnalysis class, ASTVisitor

The `SemanticAnalysis` class derives from `ASTVisitor`. The `Context::analyze`
function invokes the `visit` member function on the instance of
`SemanticAnalysis`, passing the root node of the AST.  This begins
a traversal of the AST.  Starting with the root node, which has the
tag `AST_UNIT`, the appropriate `visit_` member function will be
called for each visited node in the AST. Because the default behavior
of each visit member function is to recursively visit children,
this effects a traversal of the AST.

Note, however, that certain `visit_` member functions are overridden.
These are the ones which you will likely need to implement to
perform semantic analysis.

The goal of semantic analysis is to build symbol tables recording what
each name used in the input refers to (variables, functions, and struct
types), as well as annotating AST nodes with pointers to symbol table
entries (`Symbol` objects) or types (`std::shared_ptr<Type>` objects
pointing to a reference-counted instance of the `Type` class)
as appropriate.

`AST_VAR_REF` nodes should be annotated with a pointer to the symbol
table entry for the variable or function the name refers to.
(Note that `AST_VAR_REF` nodes *can* refer to functions.)
For all expressions that are not variable references, the node should
be annotated with a type.

You will need to add fields and member functions to the `NodeBase` class
to store these annotations.  Note that if an `AST_VAR_REF` node is annotated with
a pointer to a `Symbol` object, the `Symbol` object contains a shared
pointer to the variable (or function's) type, so it isn't necessary
to represent the type separately.  Here is a possible approach to
adding member functions to the `NodeBase` class to support symbol and
type annotations:

```c++
void NodeBase::set_symbol(Symbol *symbol) {
  assert(!has_symbol());
  assert(m_type == nullptr);
  m_symbol = symbol;
}

void NodeBase::set_type(const std::shared_ptr<Type> &type) {
  assert(!has_symbol());
  assert(!m_type);
  m_type = type;
}

bool NodeBase::has_symbol() const {
  return m_symbol != nullptr;
}

Symbol *NodeBase::get_symbol() const {
  return m_symbol;
}

std::shared_ptr<Type> NodeBase::get_type() const {
  // this shouldn't be called unless there is actually a type
  // associated with this node

  if (has_symbol())
    return m_symbol->get_type(); // Symbol will definitely have a valid Type
  else {
    assert(m_type); // make sure a Type object actually exists
    return m_type;
  }
}
```

This code assumes that the `m_symbol` member variable has the type `Symbol *`
and that the `m_type` member variable has the type `std::shared_ptr<Type>`.
You are free to use or adapt this code, but you are not required to.

### Symbol and SymbolTable classes

The `SymbolTable` class plays more or less the same role in the compiler
as the `Environment` class played in [Assignment 1](assign01.html) and
[Assignment 2](assign02.html). Specifically, it records information
about each named variable, function, and struct type in the program.
Unlike `Environment`, it is not a runtime data structure, because
the compiler will be generating x86-64 assembly language code to carry
out the execution of the program.

A `SymbolTable` is a collection of *symbol table entries*, which are
represented by dynamically-allocated instances of the `Symbol` class
managed by `std::shared_ptr` objects.
There are three kinds of symbol table entries (defined by the
`SymbolTableKind` enumeration type): variables, functions, and types.
Type entries are only used for struct data types.

You can use the `add_entry` member function to add a new entry to
a `SymbolTable`.

Like the interpreter language in Assignments 1 and 2, C is a block
scoped language. Each statement list (represented by `AST_STATEMENT_LIST`
nodes in the AST) is a scope nested within its parent scope.
It is not allowed to define to variables or functions with the same
name in the same scope. However, it is allowed to define a variable
in an inner scope whose name is the same as a variable in an outer
(enclosing) scope.

<div class='admonition info'>
  <div class='title'>Note</div>
  <div class='content' markdown='1'>
Note that in C, a function's parameters are considered to be in the same
scope as the body of the function. So, the statement list defining the
body of a function should use the same symbol table as the one in which
the function's parameters are defined.
  </div>
</div>

Each `SymbolTable` has a link to its "parent" `SymbolTable`, representing
the enclosing scope. The `SymbolTable` representing the global scope
does not have a parent.

The following helper functions for entering and leaving scopes in the source program
are provided:

```c++
SymbolTable *SemanticAnalysis::enter_scope(const std::string &name) {
  SymbolTable *symtab = new SymbolTable(m_cur_symtab, name);
  m_all_symtabs.push_back(symtab);
  m_cur_symtab = symtab;
  return symtab;
}

void SemanticAnalysis::leave_scope() {
  assert(m_cur_symtab->get_parent() != nullptr);
  m_cur_symtab = m_cur_symtab->get_parent();
}
```

Note that the `m_cur_symtab` member variable will always point to the
`SymbolTable` representing the current scope.

### Expectations for symbol tables

We will test your compiler's semantic analyzer by using the `-a` command line
option, which prints the contents of all symbol tables after semantic analysis completes.
This section describes the expectations for creating and populating symbol tables.

A `SymbolTable` should be created (using the `enter_scope` function) for each
statement list in the input source code. Note that in a function definition,
the statement list defining the body of the function should use the same symbol
table as the one containing the function's parameters. Symbol tables should be
created in the same order as the order in which the corresponding statement
lists appear in the input source code.

Symbol table entries should be added to the current scope's `SymbolTable` in the
order in which they appear in the input source code.

A function should have a single "canonical" `SymbolTable` representing its
parameters and body. This symbol table should be created when the first declaration
or definition of the function is encountered. This means that when a function is
encountered a second time, the semantic analyzer must "re-enter" the symbol
table created when the function was encountered for the first time. This
can be as simple as doing something like:

```c++
SymbolTable *fn_symtab = /* pointer to original SymbolTable */;
m_cur_symtab = fn_symtab;
```

When a function is encountered for the first time, the semantic analyzer should
determine its return type, visit its parameters (creating symbol table entries
for them), create a `FunctionType` representing the function's type, and create
a symbol table entry for the function in the `SymbolTable` representing the global scope.

When a function is encountered for a *second* time (or even a third time), the
semantic analyzer should re-enter the function's parameter/body `SymbolTable`
and check the "current" function declaration or definition to determine whether it
matches the previous declaration or definition. Specifically, if the return type,
number of parameters, or types of parameters in the "current" function does
do not exactly match the previous declaration or definition, a
`SemanticError` exception should be raised.

Usually, when a function appears multiple times, the first occurrence will be
a declaration (a.k.a. function prototype), and the second occurrence will be a
definition. However, C allows any number of occurrences of a function, as long as
the return types and parameter types match exactly, and at most one occurrence
is a definition.

### Parameter names

It is legal for a function declaration to use different parameter names than
a function definition. For example:

```c
int sum(int a, int b);

int sum(int x, int y) {
  return x + y;
}
```

To handle this possibility, but still maintain the expectation that a
function's symbol table is created at the point of the first declaration
or definition of the function, your semantic analyzer will need to
rename the symbol table entries for the function's parameters prior to
visiting the function's body, since any variable references in the body
will use the *definition*'s parameter names.

<div class='admonition tip'>
  <div class='title'>Tip</div>
  <div class='content' markdown='1'>
The autograder will not test your compiler on any inputs
where the function declaration and definition use different parameter
names but otherwise are a perfect match. We mention this issue
only because you may see test cases in the public test repository
where a function's declaration and definition use different parameter
names.
  </div>
</div>

### Symbol table names

Each symbol table must have a name:

1. The name of the global symbol table is "`global`"
2. For a symbol table representing the parameters and body of a function,
   the symbol table's name should be "<code class="language-plaintext highlighter-rouge">function <i>name</i></code>",
   where <i>name</i> is the name of the function.
3. For a symbol table representing any statement list other than the one
   defining a function's parameters and body, the symbol table's name
   should be "<code class="language-plaintext highlighter-rouge">block <i>N</i></code>", where <i>N</i> is the line
   number stored in the statement list node's `Location`

You can see the expected symbol table names by looking at the expected output
of the test cases in the test repository.

### Array parameters

In a function declaration or definition, if any parameter has an array type,
it should be converted to an "equivalent" pointer type. For example, the
function

```c
void sum_arr(int p[0], int n);
```

should be handled as though it were actually

```c
void sum_arr(int *p, int n);
```

This should be reasonably easy to do: in `visit_function_parameter`, once the
type of the parameter is known, check whether it's an array type, and if so,
determine the equivalent pointer type.

### Type representations

As discussed in [Lecture 11](../lectures/lecture11-public.pdf), trees are a
good way to represent C data types. The starter code defines a base class
called `Type`, with concrete derived classes `BasicType`, `QualifiedType`,
`PointerType`, `ArrayType`, `StructType`, and `FunctionType`.

You should read through the header file `type.h` to familiarize yourself with
these types and the operations they support.  Note that the base class
`Type` defines a "wide" interface of operations, meaning that any operation
that would be meaningful for any of the derived classes is defined in the
base class. This makes `Type` objects easy to work with, but it also raises
the possibility that your program could invoke an operation that is not
meaningful. For example, if your semantic analyzer calls the `get_num_members()`
function on an instance of `BasicType`, an exception will be thrown.

Because of the way that C variable declarations are split into a base type
and declarators, it is natural to want to allow type representations to
share parts of the representation. For this reason, `Type` objects
(or, more specifically, objects that are derived from `Type`) are meant
to be wrapped in `std::shared_ptr<Type>` smart pointers. This allows
`Type` objects to be reference counted, so that the object is deleted when
the last reference to it disappears.

To make the reference counting work correctly, you should follow these
rules:

1. When a type object is dynamically allocated, it should *immediately*
   be wrapped in a `std::shared_ptr<Type>`
2. All further references to the object should also be managed by
   `std::shared_ptr<Type>` instances created via copying or assignment

The easiest way to ensure both rules are followed is to consistently use
`std::make_shared` when creating objects to be managed by `std::shared_ptr`.
For example:

```c++
std::shared_ptr<Type> type =
  std::make_shared<BasicType>(BasicTypeKind::CHAR, false);
```

This is equivalent to

```c++
std::shared_ptr<Type> type(new BasicType(BasicTypeKind::CHAR, false));
```

and actually slightly more efficient.

`std::shared_ptr` instances can be assigned, copied, returned, etc.,
just like "normal" pointers, with the advantage that the managed object
will be deleted once there are no `std::shared_ptr` instances pointing
to it.

Under no circumstances should you allow a "plain" dynamically allocated
object to be managed by two "unrelated" `std::shared_ptr` instances.
For example:

```c++
// dangerous raw pointer
Type *ushort_type = new BasicType(BasicTypeKind::SHORT, true);

std::shared_ptr<Type> copy3(ushort_type); // problematic
std::shared_ptr<Type> copy4(ushort_type); // problematic
```

In the code above, `copy3` and `copy4` will use different (unrelated) reference counts
for the type object, leading to a high likelihood that the type object will be
destroyed at a point when there are still `shared_ptr` objects referring to it.

This situation is easy to avoid if you use `std::make_shared` consistently.

### Struct types

Handling the definition of a struct type will require a somewhat specific approach,
which is described here. (This is a guide to implementing the
`SemanticAnalysis::visit_struct_type_definition` member function.)

To start with, an instance of `StructType` should be created, and immediately
entered into the current symbol table.  This code could look something like
the following:

```c++
std::string name = /* the name of the struct type */;
Location loc = /* Location of the struct type */;
std::shared_ptr<Type> struct_type(new StructType(name));

m_cur_symtab->add_entry(loc,
                        SymbolTableKind::TYPE,
                        "struct " + name,
                        struct_type);
```
The reason that the struct type will need to be entered into the symbol
table of the current scope immediately is that struct types can refer to
themselves when defining a recursive data type, such as a linked list node:

```c
struct Node {
  int data;
  struct Node *next;
};
```

In the example above, the base type of the `next` member is pointer to
`struct Node`, so for this declaration to be legal, that type must
already exist in the symbol table for the enclosing scope.

Your semantic analyzer will need to treat the body
of a struct type as being a scope containing variable definitions, where the
variable definitions are the members (fields) of the struct type.
This should be very similar to how a statement list is handled.

Once all of the members have been processed, you can use the
`add_member` member function to add a `Member`
to the instance of `StructType` representing each field.
Once that is done, the representation of the struct type is complete.

Note that it is important to make sure that the names of struct types don't
conflict with the names of variables and functions. Prepending
"`struct `" to the name recorded in the symbol table entry (as shown
above) is a simple way to accomplish this, and is also the approach
that is expected by the public tests.

### LiteralValue

The `LiteralValue` class (defined in `literal_value.h` and `literal_value.cpp`)
is intended to help the compiler

1. decode the lexemes of literal values (integers, characters, and strings)
2. represent literal values

One place where you may find `LiteralValue` to be useful is in determining
the size of an array when the semantic analyzer sees an array declarator.
When later on you implement code generation, `LiteralValue` will likely be
useful for representing integer, character, and string constant values.

### Dealing with variable declarations

Here is a suggested strategy for dealing with variable declarations
and function parameters:

Start by visiting the base type. The initial assumption is that
this will be the type of the variable, but that assumption could be
modified if there are pointer and/or array declarators.

Recursively visit the child or children representing the declarator
or declarators.  Each such child will define one variable. When the
visitation of a child declarator is finished, the semantic analyzer
will know both the name and the exact type of the variable, which is
the information it needs in order to create a symbol table entry for
the variable.

It will help to have member variables for the name and type for the
variable that the semantic analyzer is currently working on. The member
variable for the type could be something like

```c++
std::shared_ptr<Type> m_var_type;
```

A variable's type can be updated incrementally as declarators are handled.
You'll need to think about whether this should be done when traversing
"down" the tree (towards the `AST_NAMED_DECLARATOR` node) or when traversing
back "up" the tree. You should find it useful to use the `-p` option
to print the AST so that you can see how the nodes representing declarators
are structured in the tree.

### Operators

C has a fairly large number of operators. To reduce the complexity of
the compiler project sequence, the test cases will only use the following
operators:

* Arithmetic with two operands: `+`, `-`, `*`, `/`, `%`
* Unary arithmetic: `-`
* Pointer and address operations: `*` (dereference) and `&` (address-of)
* Struct member access: `.` and `->`
* Relational operators: `<`, `<=`, `>`, `>=`, `==`, `!=`
* Logic with two operands: `&&`, `||`
* Unary logic: `!` (not)
* Assignment: `=`
* Array subscript (i.e., `a[i]` where `a` is an array or pointer, and `i` is an
  integer element index)

You will explicitly *not* need to support bitwise operators, compound
assignment (`+=` and similar), or increment/decrement (`++` and `--`).

### C semantic rules

Note that the [slides for Lecture 11](../lectures/lecture11-public.pdf) are a
good overview of the semantic rules your semantic analyzer is expected to
check.

**Note also**: to a large degree, the tests in the public test repository are
the best specification of the behavior expected of your semantic analyzer.
For all of the important semantics specified here, you should find at least
one related test.

**Rules for operators**:

If in a use of the `+` and `-` operators one operand is a pointer and the
other is an integer (belonging to any integer data type), then it is performing
pointer arithmetic, and the result type is the same type as the type of the
pointer operand.  The integer operand should be promoted to `int` or `unsigned int`
if its representation is less precise.

Arithmetic on two pointers is never legal.

For arithmetic on two integers (`+`, `-`, `*`, etc.), the following rules apply:

1. If either operand is has a type less precise than `int` or
   `unsigned int`, it is promoted to `int` or `unsigned int`
2. If one operand is less precise than the other, it is promoted
   to the more precise type
3. If the operands differ in signedness, the signed operand is
   implicitly converted to unsigned

For unary (one operand) operations on integer values, the operand value
should be promoted to `int` or `unsigned int` if it belongs to a less-precise
type.

**Assignments**:

An assignment is not legal if the type of the left hand operand is
qualified as `const`.

The left hand side of an assignment must be an *lvalue*.

An *lvalue* is

* a reference to a variable
* an array subscript reference
* a pointer dereference
* a reference to a struct instance
* a reference to a field of a struct instance

If the types of the left and right sides of an assignment are both
integer (`char`, `short`, `int`, etc.) then the assignment is legal.
The right hand type is implicitly converted to the left hand (lvalue)
type.

If the left and right sides are both pointers, then the assignment is
legal if and only if

1. the unqualified base types of each pointer type are identical, and
2. the base type on the left hand side does not lack any qualifiers
   that the base type on the right hand side has

For example, the following code is legal:

```c
char a;
char *right;
right = &a;
const char *left;
left = right; // a legal assignment
```

In the code above, `left`'s base type is `const char` and `right`'s base type
is `char`. The unqualified base types are both `char`, an exact match.
`right`'s base type does not have any qualifiers that `left`'s base type lacks.

The following code is *not* legal:

```c
const char a;
const char *right;
right = &a;
char *left;
left = right; // illegal: discards "const" qualifier from base type
```

An assignment involving both pointer and non-pointer operands is never legal.

If the left and right sides of an assignment both have a struct type,
the assignment is legal as long as the type of both the left and right
sides are the same struct type.

**Function calls**:

A function call is legal if

* the name of the called function refers to a function that was previously
  either declared or defined
* the number of arguments passed is equal to the function type's number of
  parameters
* each argument expression has a type that could be legally assigned to
  the corresponding parameter (according to the rules for assignment)

The result of a function call has the type indicated by the function's
return value.

**Literals**:

The type of an integer literal defaults to `int`. If its suffix
contains `U` or `u`, then it is unsigned.  If its suffix contains
`L` or `l`, then it is `long`. If both are specified, it is
`unsigned long`. The [LiteralValue class](#literalvalue) has member
functions `is_unsigned()` and `is_long()` to help you decode
and work with integer literals.

The type of a character literal is `int`.

The type of a string literal is `const char *` (pointer to `const char`.)

### Representing implicit conversions

Because code generation will need to know which implicit conversions are needed,
it's a good idea to explicitly represent them in the AST.

The `AST_IMPLICIT_CONVERSION` node tag is intended to allow explicit representation
of implicit conversions. For example, let's say that in checking the left operand
of an operator with two integer operands, the left operand's type is less precise
than `int`, and needs to be promoted. That code could look something like this:

```c++
Node *left = n->get_kid(1);

if (left->get_type()->get_basic_type_kind() < BasicTypeKind::INT)
  n->set_kid(1, left = promote_to_int(left));
```

The `promote_to_int` member function could be implemented this way:

```c++
Node *SemanticAnalysis::promote_to_int(Node *n) {
  assert(n->get_type()->is_integral());
  assert(n->get_type()->get_basic_type_kind() < BasicTypeKind::INT);
  std::shared_ptr<Type> type =
    std::make_shared<BasicType>(BasicTypeKind::INT,
                                n->get_type()->is_signed());
  return implicit_conversion(n, type);
}

Node *SemanticAnalysis::implicit_conversion(Node *n,
                                            std::shared_ptr<Type> type) {
  std::unique_ptr<Node> conversion(
    new Node(AST_IMPLICIT_CONVERSION, {n}));
  conversion->set_type(type);
  return conversion.release();
}
```

Although these implicit conversion nodes aren't strictly needed for this assignment,
they will very likely be useful for code generation in assignments 4 and 5.

## `README.txt`

Please submit a `README.txt` with your submission which briefly discusses anything
interesting or significant you want us to know about your implementation.
If there are any features you weren't able to get fully working, or if you implemented
extra functionality, this is a place you could document that.

## Submitting

Create a zipfile using the command

```
make solution.zip
```

**Note**: make sure your `README.txt` is in the top-level directory (i.e., the one
that contains the `Makefile`.)

Upload the `solution.zip` zipfile to Gradescope as **Assignment 3**.
