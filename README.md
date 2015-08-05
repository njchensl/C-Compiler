C Compiler in C#
===

This intends to be a full ANSI C compiler. It generates x86 (32-bit) assembly code in linux. The goal is to produce `.s` files that `gcc`'s assembler and linker could directly use.

### 1. Handwritten Scanner (lexical analysis) - done
* Not automatically generated by flex.
* Written via standard state-machine approach.

The first phase of a compiler is lexical analysis. In this phase, the source code is loaded and scanned through, to produce a list of minimial grammar units - tokens. Tokens are to programming languages just as words are to English. Each token in the C language falls into one of these categories: char, float, identifier, int, keyword, operator, and string.

You might see different classification methods, such as one that has the punctuator, which I group into the operator type.

Let's look at a concrete example. Suppose we have a line of code that looks like this.

```C
int a = ceil(3.2);
```

After lexical analysis, the code will be tranformed into these tokens:

| Source | Token kind | Data |
|:------:|:----------:|:----:|
| `int`  | Keyword    | int  |
| `a`    | Identifier | `"a"` |
| `=`    | Operator   | assignment |
| `ceil` | Identifier | `"ceil"` |
| `(`    | Operator   | open parenthesis |
| `3.2`  | Float      | 3.2, no suffix, indicating that is a double |
| `)`    | Operator   | close parenthesis |
| `;`    | Operator   | semicolon |

Note that each token has a type, and some type-specific data: for a keyword, you need to know exactly which keyword it is; for a floating number, you need to know what number it is, and whether it's a float or a double; etc.

Each kind of token has its unique pattern, making it recognizable by the program. For example, an identifier always starts with an underscore '_' or a letter, so we won't confuse it with an integer.

### 2. Handwritten Parser (grammar analysis) - done

* Not automatically generated by yacc / bison.
* Standard recursive descent parser, with a little hack.

### 3. Semantic Analysis - has a running version, but needs huge modification.
* A type system to perform the C language implicit typecasts and other tasks.
* An environment system to record the user defined symbols.

### Issues in semantic analysis

* Expressions **can** change the environment.

    First, consider the code below:
    
    ```C
    some_int = foo(0);
    some_struct = foo(1);
    ```
    
    `foo` isn't declared before being used, so the compiler doesn't know its prototype, making the first usage of `foo` un-checkable. The compiler will immediately make up a prototype `int foo(int)` based on the first usage. The second usage then causes a type error.
    
    In this case, simply using an (undeclared) function introduces a new prototype, thus changes the environment. 
    
    Then, consider this:

    ```C
    int size = sizeof(struct A { int a; });
    // a new type introduced insize a 'sizeof' expression.
    
    void *ptr = (struct B { int b; } *)0x0;
    // a new type introduced insize a type-cast expression.
    
    struct A a = { 3 };
    struct B b = { 4 };
    // these new types can also be referred to afterwards, but only in the same scope.
    ```
    
    So, we can see that an expression can change the environment. I must modify about 79 methods, which is crazy. So I'll put this issue away first.
    
* Certain statements like `return` or `case` require special environments.

    The current implementation doesn't check that in semantic analysis.
    
* Conversions between pointers, functions, arrays are not made clear.

    I need to read some documents and implement it properly.

### Environment
To determine the semantics of a piece of code, we must examine it in the context.

Take the following two pieces of code as an example.

Code snippet 1:
```C
int a = 3;
int b = 4;
a * b;
```

Code snippet 2:
```C
typedef int a;
a *b;
```

Both have a line `a * b;`, but they mean very different things. The first is a multiplication expression (though performing the multplication doesn't seem to do any good...), while the second is a declaration of a variable b.

It turns out that the environment only has an impact on **identifiers** - we can't conclude what a given identifier refers to, until we're given an environment. So we need to maintain a data structure to record all user-defined identifiers.

So, where could a new identifier come in?

1. Declaration statements.

    Obviously, the purpose of a declaration is just to introduce a new identifier. 

### The conversion between arrays and pointers
Consider the following declaration:

```C
int arr[3][2];
```

This would create an array of arrays, and the memory layout would be:

    arr[0][0] arr[0][1] arr[1][0] arr[1][1] arr[2][0] arr[2][1]
    
If the user writes the following expression

```C
arr[2][1]
```

then the corresponding element would be visited.

The expression above would be implicitly translated to

```C
*(*(arr + 2) + 1)
```

It is worth our time to figure out what each step means and what each intermediate result's type is.

1. `arr + 2`
    
    `arr` is of type `int [3][2]`. Its value is the base address of the array.
    
    The standard says you **can** add an integer to a pointer, but it doesn't say you can add an integer to an array. This means that the array needs to be implicitly cast to a pointer `int (*)[2]`.
    
    Note: `int (*)[2]` is not equal to `int *[2]`. `int (*)[2]` means "a pointer to int[2]", while `int *[2]` means "an array that has 2 int*'s". This is because the operator `[]` has a higher priority over the operator `*`.
    
    A pointer and an array are very different. If you use the `sizeof` operator on a pointer, you would just get 4, since a pointer has the same size as an unsigned integer; if you use the `sizeof` operator on an array, you would get the total size of it.
    
    * `sizeof(int (*)[2]` is 4;
    
    * `sizeof(int [3][2]` is 3 * 2 * 4 = 24.
    
    Now that we've done the implicit type cast, we will perform the actual addition. This is a "pointer + integer" addition, which would be scaled. Since our pointer points to a `int [2]`, the addition would be scaled to `sizeof(int [2])` which is 8.
    
    The result is `(base + 2 * 8)`, and has the same pointer type: `int (*)[2]`.
    
    ```
    arr[0][0] arr[0][1] arr[1][0] arr[1][1] arr[2][0] arr[2][1]
                                            |
                                            result is this address
    ```
    
2. `*(arr + 2)`
    
    The next step is dereference. The type of the result would be `int [2]`, but the value is rather special - still `(base + 2 * 8)`, since the value of an array is just the base address.
    
3. `*(arr + 2) + 1`

    This step is pretty much the same. The array is cast into a pointer. This time the pointer has type `int *` so the scale would be 4. The result of this expression is `(base + 2 * 8 + 1 * 4) and has type `int *`.
    
    ```
    arr[0][0] arr[0][1] arr[1][0] arr[1][1] arr[2][0] arr[2][1]
                                                      |
                                                      result is this address
    ```
    
4. `*(*(arr + 2) + 1)`

    The final step is dereference. Just pick up the integer according to the address. Farely easy step.
    
### 4. Code Generator - round 60%
* Generates x86 assembly code.
