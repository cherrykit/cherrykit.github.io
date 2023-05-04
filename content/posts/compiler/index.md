---
title: "A Joos 1W compiler"
date: 2023-05-03
description: The design and implmentation of the Joos 1W compiler
menu:
  sidebar:
    name: Joos 1W compiler
    identifier: compiler
    weight: 10
# tags: []
# categories: []
---

The Joos 1W compiler was written in C++ in a group of three people for the "Compiler Construction" course at the University of Waterloo (CS444). The compiler is for the Joos 1W language, a subset of Java 1.3 including objects, interfaces, arrays and strings. This post gives an overview of the different parts of the compiler and their implementation and is based off of reports written for the course.

## Overview

The program consists of a series of compiler passes over all Java files passed as input. The course supplied a minimal version of the Java standard library, which is also passed to the compiler as input. 

First, the program will call the lexer and parser. The parser will construct the AST on reduction of grammar rules. Additionally, the part of the environment corresponding to the current file’s AST is constructed. After parsing and environment construction has successfully completed for all files, the program iterates through the AST’s to perform various checks (eg. type checking). After all the checks have passed, it is converted to IR, then lowered into canonical IR, optimized and compiled to assembly following the steps described below.

## Compiler Stages

### Lexing

For the lexer generation, we used [Flex](https://github.com/westes/flex), a lexical analyzer generator. The tool takes in a set of lexing rules defining all valid input tokens and generates code that reads in an input file and outputs a sequence of those tokens.

### Parsing

For the parser generation, we used [Bison](https://www.gnu.org/software/bison/), which is already installed on the cs servers. The tool takes in a context-free grammar into an LALR(1) parser which checks that the input files follow the correct syntax. When the correct syntax is followed, the parser outputs an Abstract Syntax Tree as described in the next section. Our grammar is based off of the grammar defined in the [Java Language Specification (JLS)](https://web.archive.org/web/20111225035254/http://java.sun.com:80/docs/books/jls/second_edition/html/jTOC.doc.html), with some simplifications where Joos 1W does not support features.

For more complex requirements, like only allowing specific modifier combinations, we used weeding instead of the grammar. Weeding manually checks part of the syntax and is done in the constructors of the Abstract Syntax Tree. An error is thrown if the syntax is incorrect.

### Abstract Syntax Tree

Our Abstract Syntax Tree (AST) nodes are a hierarchy of C++ classes which we designed based on the [Eclipse AST for Java](https://help.eclipse.org/2019-12/index.jsp?topic=%2Forg.eclipse.jdt.doc.isv%2Freference%2Fapi%2Forg%2Feclipse%2Fjdt%2Fcore%2Fdom%2Fpackage-summary.html), simplifying where certain language features are not supported by Joos 1W. When some parts of a program are implicitly defined (for example for the implicit java.lang import and for the implicit inheritance from Object), we generate appropriate AST nodes for the implicit parts and link these with the rest of the AST. We implemented the Visitor pattern, adding an “accept” method to each of the AST nodes to let us easily add AST traversals to our compiler in the future.

### Environment Building

The environment is a datastructure separate from the AST that contains information on all types declared in the input files. To build the environment, we define another hierarchy of C++ “Context” classes, each subclass representing the definition of some type and/or a scope. We have specific contexts for packages, classes, interfaces, methods, fields, blocks and local variables. Additionally, we have a special RootContext which holds pointers to all declared packages. Each context contains a pointer back to the AST node where it was declared as well as pointers to other contexts, where applicable.

We create instances of context classes when the declaring AST nodes are created. Then, we use a subclass of our AST Visitor to build the environment by linking together the already existing context classes. Once the environment is built, we traverse it and check that no fields, variables or parameters in the same scope have the same name. Furthermore, we check here to see if the prefix of a declared package resolves to a type not in the default package, in which case we throw an error.

### Type Linking

Type linking links all uses of simple type and local variable names to the corresponding context in the environment. This compiler stage uses a subclass of Visitor, called TypeLinker. While type linking the AST corresponding to each file, the type linker saves the currently imported types, which lets it easily resolve all types in the current environment. After each file, this information is reset.

Our logic differentiates between simple names and qualified names. Each type of name has a different resolution order which is defined in 6.5.2 of the JLS2 specification. After types have been linked, we check that only one method with each signature is declared in each class.

### Hierarchy Checking

Hierarchy checking uses a subclass of Visitor, called HierarchyChecker. For each defined type (a class or an interface), we traverse through all implemented interfaces and superclasses and check that the hierarchy does not contain any cycles. To check for any cycles in the hierarchy tree, we must recursively traverse the entire hierarchy tree. Thus, as an optimization, we perform all the inheritance during this traversal. Once we recursively reach the top of the hierarchy, we iterate down through all classes/interfaces in the hierarchy and save all inherited methods and fields in the contexts of interfaces and classes. This includes code that checks that replacement of inherited methods is permissible. During hierarchy checking, we also construct a flattened map of all the class subtypes which we will need in later stages.

After traversing the hierarchy, for each class we check that a non-abstract class does not declare or inherit any abstract methods. Furthermore, we check that final classes are not extended at this stage.

### Disambiguation

The disambiguation phase links all remaining names to contexts in the environment. Some of the disambiguation is implemented in the TypeLinker, to take advantage of the existing information about imported packages and types. Remaining disambiguation is done by a separate Disambiguator subclass of Visitor. 

We try to disambiguate qualified names, field access and simple names that correspond to static fields or local variables. Disambiguator also links ThisLiterals to the type and checks that this is not used in static methods or fields. Disambiguator also checks for field use before declaration in the field declarations of a class.

### Type Checking

Type checking is the final stage in which program errors may be found. In this stage, all expressions are assigned a type and operations are checked for type compatibility. Type checking is done in the TypeChecker subclass of Visitor. 

In TypeChecker, we resolve all method invocations (as these require type checking of arguments) and all non-static fields. Special checks are implemented to ensure rules are followed for protected fields/methods and constructors. Furthermore, checks in TypeChecker ensure that classes have constructors with the correct name, have zero-argument constructors when required and no objects of abstract classes are created. The remaining type checking is performed according to assignability rules and type checking rules from the JLS.

### Converting to Intermediate Representation

The compiler construction course supplied an Intermediate Representation (IR) implemented in Java. We converted the Java implmentation to C++. Furthermore, we made additions to accommodate object-oriented features. The IR includes a `_start` function which our compiler ensures is called initially.

Converting to the IR is implemented as an AST visitor which traverses the AST and generates an IR node called `IrCompUnit` for each file. To ensure that static fields are initialized before code starts running, these are initialized by the `_start` function before the `main` function is called. Since many of the IR conversions are very straight forward (eg. Literals, Block, etc), we will only be explaining the complex conversions in detail.

#### Static Fields

To allocate and initialize static fields, each `IrCompUnit` node has an `init` function. All static field initializers for that class are put into this `init` function which gets called by the `_start` function.
	
When we translate accesses of static fields to assembly, we will access them through a memory label, so the names of these Global IR nodes must be unique. This is done by concatenating the package name, the class name, and the field name together.
	
Because each class is responsible for allocating their own static fields, we keep track of which static fields are used by which `IrCompUnit` so that when we convert to assembly, any static fields used that are not owned by itself are declared to be `extern`.

#### Object-Oriented Features

To allocate a new object, we need to know the size of each object. Since the size of a class is determined by the size of the parent class, we must traverse the class hierarchy to calculate the size of each class object. We utilized our existing class hierarchy traversal in HierarchyChecker to do this. During this step, we also calculate the data offsets of every field and method of every class.

Using this information we can convert class instance creation to IR. Before calling the constructor, we first allocate the necessary heap memory for the object, set the Class Dispatch Vector (CDV) as the first element in the memory, and initialize the rest of the memory to zero. Then the call to the constructor is called. Every constructor first calls its parent’s default constructor, then executes any instance field initializations, before executing any user-provided code in the constructor. When a constructor is called, the memory of the object is passed in as the first argument so that any instance field access or instance method calls can be made. Once the constructor is finished, that same object is returned to the caller.

The Class Dispatch Vector (CDV) of each object contains 3 things. The first element is the IDT of the object, the second element is the unique type id of the object, and the rest is populated by the labels to all the instance methods available to that class object. The CDV is allocated as static data in the respective class file and labeled using a unique name. The unique type id is a unique numerical id assigned to each type that is used to identify which type an object is.

The Interface Dispatch Table (IDT) is a hash table required to call methods on interface objects. The labels to instance methods are stored in indices determined by the hashcode of the method. Collision resolution uses a unique id for each method determined from the memory address of the method. For each index in the IDT where a collision happens, we generate a small procedure to do collision resolution. This is custom code generated for each index where a collision happens and it checks the given unique method id with the expected ids. Once a match is found, it jumps to the correct method label for that id.

Dealing with String objects is a special case. String literals are handled by creating a char array, populating it with the string literal data, and then calling the String object constructor to create a String object with the data of the string literal. String concatenations are handled in similar ways. We heavily utilize the stdlib String class functions to help us with string concatenations.
	
For arrays, we create a new internal Array class which implements the Cloneable and Serializable interfaces. While the JLS requires one Array class for each base type, we decided to use only one class since Joos does not support method calls on arrays. Instead, we create only a unique CDV for each base type, where the CDVs differ only in their last entry. We use the last entry to hold the type id of the base type. For primitive types which are not assigned ids, we use negative identifiers to distinguish the base type. When assigning to array elements at runtime, we do an instanceOf check against the base type ID to ensure that the right-hand side is compatible with the array type.

Our instanceof checks are implemented using a global MxM table, where M is the number of reference types. Entry [i][j] is equal to 1 only if the type with type id i is a subtype of the type with type id j. At compile time, this information is set up using the existing flattened subtype data created during hierarchy checking. If the right-hand side of the instanceof is an object, we retrieve the type id of the object from the CDV and check the appropriate entry of the table. If the right-hand side of the instanceof is an array, we first check that the object is an array by checking the type id from the CDV. If this is the case, we retrieve the basetype id from the last entry of the Array CDV. If the right-hand side basetype is a primitive type, we do a simple comparison to finish the instanceof check. Otherwise, we check the global subtype table. Casts to reference types or arrays use the same code as the instanceof checks, with the exception that assigning null is allowed in casts while a null value results in false in instanceof checks.

### Lowering to Canonical Form

Lowering the IR into canonical form involves flattening the IR node trees into one top-level sequence of statements for each function. Furthermore, expressions may no longer have any side effects. These transformations make the IR more assembly-like, implifying instruction selection in the next compiler pass.

We implement lowering uing the `lowerToCanonical` function in `IrCompUnit`. This function calls an `intoCanonical` method on each function in the compilation unit and concatenates the resulting sequences of statements for each function into one top-level sequence. The `intoCanonical` methods traverse the IR tree. For statements, they return a flat vector of IR statements. For expressions, they return a flat vector of statements along with a pure IR expression (not containing any side effects).

### Conversion to Assembly

Instruction selection is done using tiling, where a subtree of the IR tree is matched with a tile and that tile is translated to a sequence of abstract assembly instructions. Abstract assembly uses abstract registers, of which there is an infinite supply.

We represent a Tile as a tree of TileNodes, with each TileNode representing a particular IR node. Since some TileNodes might need to specify particular constraints beyond the type of IR node (e.g. we might only want constants with the value 0 to match), a TileNode may specify an optional constraint function on the IR node. For each Tile, we supply a translate function, which is given the root of the tile and generates the abstract assembly for that tile. The translation function is passed an instance of a FunctionContext class, which is used to map the unmatched subtrees to abstract registers.

To avoid having to specify too many different tiles, parts of the IR that can be directly translated to assembly operands are covered by dummy tiles that themselves do not generate any assembly code. Here, the FunctionContext class will directly create the appropriate assembly operand instead of mapping the unmatched part of the IR to an abstract register. This is mainly used for constants, variables and global data.

Functions are tiled using a `tile_function` method. This function generates the function prologue and epilogue. Then, for each statement in the function, a `find_tiles` function implementing optimal tiling using dynamic programming and memoization is called. Costs of tiles are computed based on the number of memory accesses and number of instructions used by the translation. The function returns the matched tiles along with the IR nodes that are the roots of each tile in post-traversal order. Given this optimal tiling, `tile_function` calls `translate_tile` on each tile to obtain a sequence of assembly instructions.

### Simple Register Allocation

Translation from abstract assembly to concrete assembly is handled by an instance of the RegisterAllocator class. SimpleAllocator spills all abstract registers onto the stack, using specific registers to shuttle the registers on and off the stack.

## Warnings

Our compiler generates warnings for dead assignments to variables and unreachable code. We find such parts of the code using dataflow analysis.

### Dataflow Analysis

We implemented a Control Flow Graph (CFG) node class containing predecessor and successor links. The CFG nodes are owned by AST statement nodes and created by the AST constructors. Some AST nodes, such as for loops, might own multiple CFG nodes. For Live Variable Analysis (LVA), we also require use and define sets for each CFG node. The links between nodes as well as the use and define sets are populated by an AST visitor called CFGBuilder. The links are set up by having blocks link together all consecutive statements inside of them, while accounting for special cases like loops and if statements where we may need to link from multiple CFG nodes or from one specific CFG node. The use and define sets are computed recursively over AST expressions and are added to CFG nodes while linking the CFG nodes in CFG builder. 

The CFG builder sets up a full CFG for each method separately and then invokes Reachable Statement Analysis (RSA) and Live Variable Analysis once construction has completed. Both dataflow analyses use a general dataflow analysis framework implemented in CFG builder, which has arguments for a top element, meet and transfer functions as well as the direction of the analysis. The results of RSA are used to throw warnings when statements are unreachable or when a return statement is missing. The results of LVA are used to generate warnings for dead assignments to local variables.

## Optimizations

### Constant Propagation

Constant propagations is implemented as a dataflow analysis over canonical form IR. Each IR statement creates its CFG node at initialization time and the nodes are linked together by an IrAnalyzer class. For each method, IrAnalyzer links the CFG nodes together using two passes of the sequence of statements. In the first pass, IrAnalyzer finds all labels and creates a map from the label names to their CFG nodes. In the second pass, IrAnalyzer links each statement to the next statement, unless the statement is a jump. In the case of jumps or conditional jumps, links are created to the nodes stored in the label map.

For the dataflow analysis, we reuse our existing `dataflow_analysis` template method. After doing the dataflow analysis, IrAnalyzer iterates over the dataflow values and sets the right hand side of Move operations to constants if they evaluate to an integer. Furthermore, IrAnalyzer removes unreachable nodes by checking if the dataflow value contains an unreachable flag. Here, conditional jumps are replaced with unconditional jumps. These additional jumps are removed in a final pass over all statements if they directly precede the corresponding label.

### Register Allocation

Optimized register allocation is implemented in a separate subclass of RegisterAllocator called ChaitinAllocator. ChaitinAllocator implements Chaitin’s algorithm for register allocation. The algorithm uses graph colouring over an interference graph to assign each abstract register to a concrete register. When graph colouring fails, the abstract assembly code is rewritten to spill some abstract registers onto the stack before retrying register allocation. Our implementation of Chaitin’s algorithm closely follows the [textbook](https://ocul-wtl.primo.exlibrisgroup.com/permalink/01OCUL_WTL/pa2qcq/alma9933268623505162).

The final conversion from abstract to concrete instructions after coloring is performed simply replaces all abstract registers with the corresponding machine register. This is handled using an enum. Any move instructions where the concrete source and destination registers are identical are dropped.

### Benchmarks

We implemented benchmarks for each of our optimizations. 

Here are the results for constant propagation:
- The first test case focused on removing unreachable branches from if statements. Turning on constant propagation reduced the runtime from 2669ms to 1231ms, a speedup of 2.17 times.
- The second test case focused on propagation constant values through a computation. Turning on constant propagation reduced the runtime from 2145ms to 508ms, a speedup of 4.2 times.

Here are the results for register allocation:
- The first test case involved the computation of the fibonacci numbers. Switching to Chaitin's algorithm reduced the runtime from 5878ms to 781ms, a speedup of 7.53 times.
- The second test case involved the computation of squares of numbers. Switching to Chaitin's algorithm reduced the runtime from 6661ms to 983ms, a speedup of 6.78 times.

As expected, register allocation is more impactful than constant propagation, with register allocation having a speedup of 7.5 times and constant propagation having a speedup of up to 4.7 times. For constant propagation, the tests show that constant propagation is more impactful than the removal of unreachable code. Register allocation has a very significant speedup since we designed the benchmarks to have few variables, such that spilling to the stack can be avoided completely.


## Testing

We have a few categories of tests, including code that should pass compilation, code that should fail compilation, code that should generate warnings and tests that are fully compile to binary and executed to ensure the return value is correct. We wrote a script to automatically run all test cases and print out information about failed tests.

	