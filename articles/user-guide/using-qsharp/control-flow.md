---
title: Control flow in Q#
description: Loops, conditionals, etc.
author: gillenhaalb
ms.author: a-gibec@microsoft.com
ms.date: 03/05/2020
ms.topic: article
uid: microsoft.quantum.guide.using.controlflow
---

# Control flow in Q#

## from ops and fns in qdevtechs
Split some of this to ops/fns page? 
## Control Flow

Within an operation or function, each statement executes in order, similar to most common imperative classical languages.
This flow of control can be modified, however, in three distinct ways:

- `if` Statements
- `for` Loops
- `repeat`-`until` Loops

We defer discussion of the latter until we discuss [Repeat Until Success (RUS)](xref:microsoft.quantum.techniques.qubits#measurements) circuits.
The `if` and `for` control flow constructs, however, proceed in a familiar sense to most classical programming languages.
In particular, an `if` statement can take a condition, may be followed by one or more `elif` statements, and may end with an `else`:

```qsharp
if (pauli == PauliX) {
    X(qubit);
} elif (pauli == PauliY) {
    Y(qubit);
} elif (pauli == PauliZ) {
    Z(qubit);
} else {
    fail "Cannot use PauliI here.";
}
```

Similarly, `for` loops indicate iteration over a range of integers or over the elements of an array:

```qsharp
for (idxQubit in 0..nQubits - 1) {
    // Do something to idxQubit...
}
```

Importantly, `for` loops and `if` statements can even be used in operations for which specializations are auto-generated. In that case the adjoint of a `for` loop reverses the direction and takes the adjoint of each iteration.
This follows the "shoes-and-socks" principle: if you wish to undo putting on socks and then shoes, you must undo putting on shoes and then undo putting on socks.
It works decidedly less well to try and take your socks off while you're still wearing your shoes!

## Operations and Functions as First-Class Values

One critical technique for reasoning about control flow and classical logic using functions rather than operations is to utilize that operations and functions in Q# are *first-class*.
That is, they are each values in the language in their own right.
For instance, the following is perfectly valid Q# code, if a little indirect:

```qsharp
operation FirstClassExample(target : Qubit) : Unit {
    let ourH = H;
    ourH(target);
}
```

The value of the variable `ourH` in the snippet above is then the operation <xref:microsoft.quantum.intrinsic.h>, such that we can call that value like any other operation.
This allows us to write operations that take operations as a part of their input, forming higher-order control flow concepts.
For instance, we could imagine wanting to "square" an operation by applying it twice to the same target qubit.

```qsharp
operation ApplyTwice(op : (Qubit => Unit), target : Qubit) : Unit {
    op(target);
    op(target);
}
```

In this example, the `=>` arrow that appears in the type `(Qubit => Unit)` denotes that the input field `op` is an operation which takes as its input the type `Qubit` and produces an empty tuple as its output.
Additionally we specify the characteristics of that operation type, which contain the information about which functors are supported.
An operation of type `(Qubit => Unit)` supports neither the `Adjoint` nor the `Controlled` functor. 
If we want to indicate that an operation of that type has to support e.g. the `Adjoint` functor, we have to declare it as being adjointable. This is done by using the annotation `is Adj` to the type. 
Similarly, `(Qubit => Unit is Ctl)` denotes that an operation of that type supports the `Controlled` functor. 
We will explore this further when we discuss [types in Q#](xref:microsoft.quantum.language.type-model) more generally.

For now, we emphasize that we can also return operations as a part of outputs, such that we can isolate some kinds of classical conditional logic as a classical function which returns a description of a quantum program in the form of an operation.
As a simple example, consider the teleportation example, in which the party receiving a two-bit classical message needs to use the message to decode their qubit into the proper teleported state.
We could write this in terms of a function that takes those two classical bits and returns the proper decoding operation.

```qsharp
function TeleporationDecoderForMessage(hereBit : Result, thereBit : Result)
: (Qubit => Unit is Adj + Ctl) {

    if (hereBit == Zero && thereBit == Zero) {
        return I;
    } elif (hereBit == One && thereBit == Zero) {
        return X;
    } elif (hereBit == Zero && thereBit == One) {
        return Z;
    } else {
        return Y;
    }
}
```

This new function is indeed a function, in that if we call it with the same values of `hereBit` and `thereBit`, we will always get back the same operation.
Thus, the decoder can be safely run inside operations without having to reason about how the decoding logic interacts with the definitions of the different operation specializations.
That is, we have isolated the classical logic inside a function, guaranteeing to the compiler that the function call can be reordered with impunity so long as the input is preserved.

We can also treat functions as first-class values, as we will see in more detail when we discuss [operations and function types](#operations-and-functions-as-first-class-values).

## Partially Applying Operations and Functions

We can do significantly more with functions that return operations by using *partial application*, in which we can provide one or more parts of the input to a function or operation without actually calling it. For example, recalling the `ApplyTwice` example above, we can indicate that we don't want to specify, right away, to which qubit the input operation should apply:

```qsharp
operation PartialApplicationExample(op : (Qubit => Unit), target : Qubit) : Unit {
    let twiceOp = ApplyTwice(op, _);
    twiceOp(target);
}
```

In this case, the local variable `twiceOp` holds the partially applied operation `ApplyTwice(op, _)`, where parts of the input that have not yet been specified are indicated by `_`.
When we actually call `twiceOp` in the next line, we pass as input to the partially applied operation all of the remaining parts of the input to the original operation.
Thus, the above snippet is effectively identical to having called `ApplyTwice(op, target)` directly, save for that we have introduced a new local variable that allows us to delay the call while providing some parts of the input.

Since an operation that has been partially applied is not actually called until its entire input has been provided, we can safely partially apply operations even from within functions.

```qsharp
function SquareOperation(op : (Qubit => Unit)) : (Qubit => Unit) {
    return ApplyTwice(op, _);
}
```

In principle, the classical logic within `SquareOperation` could have been much more involved, but it is still isolated from the rest of an operation by the guarantees that the compiler can offer about functions.
This approach will be used throughout the Q# standard library for expressing classical control flow in a way that can be readily used within quantum programs.





# from 'Statements'
## Control Flow

### For Loop

The `for` statement supports iteration over an integer range or over an array.
The statement consists of the keyword `for`, an open parenthesis `(`,
followed by a symbol or symbol tuple, the keyword `in`, an expression of type `Range` or array, a close parenthesis `)`, and a statement block.

The statement block (the body of the loop) is executed repeatedly,
with the defined symbol(s) (the loop variable(s)) bound to each value in the range or array.
Note that if the range expression evaluates to an empty range or array,
the body will not be executed at all.
The expression is fully evaluated before entering the loop,
and will not change while the loop is executing.

The binding of the declared symbol(s) is immutable and follows the same rules as other variable bindings. 
In particular, it is possible to destruct e.g. array items for an iteration over an array upon assignment to the loop variable(s).

For example,

```qsharp
// ...
for (qubit in qubits) { // qubits contains a Qubit[]
    H(qubit);
}

mutable results = new (Int, Results)[Length(qubits)];
for (index in 0 .. Length(qubits) - 1) {
    set results w/= index <- (index, M(qubits[index]));
}

mutable accumulated = 0;
for ((index, measured) in results) {
    if (measured == One) {
        set accumulated += 1 <<< index;
    }
}
```

The loop variable is bound at each entrance to the loop body, and unbound at the end of the body.
In particular, the loop variable is not bound after the for loop is completed.

### Repeat-Until-Success Loop

The `repeat` statement supports the quantum �repeat until success� pattern.
It consists of the keyword `repeat`, followed by a statement block
(the _loop_ body), the keyword `until`, a Boolean expression, 
and either a terminating semicolon or 
the keyword `fixup` followed by another statement block (the _fixup_).
The loop body, condition, and fixup are all considered to be a single scope,
so symbols bound in the body are available in the condition and fixup.

```qsharp
mutable iter = 1;
repeat {
    ProbabilisticCircuit(qubits);
    let success = ComputeSuccessIndicator(qubits);
}
until (success || iter > maxIter)
fixup {
    iter += 1;
    ComputeCorrection(qubits);
}
```

The loop body is executed, and then the condition is evaluated.
If the condition is true, then the statement is completed;
otherwise, the fixup is executed, and the statement is re-executed starting
with the loop body.
Note that completing the execution of the fixup ends the scope for the
statement, so that symbol bindings made during the body or fixup are not
available in subsequent repetitions.

For example, the following code is a probabilistic circuit that implements
an important rotation gate $V_3 = (\boldone + 2 i Z) / \sqrt{5}$ using the
Hadamard and T gates.
The loop terminates in $\frac{8}{5}$ repetitions on average.
See [*Repeat-Until-Success: Non-deterministic decomposition of single-qubit unitaries*](https://arxiv.org/abs/1311.1074)
(Paetznick and Svore, 2014) for details.

```qsharp
using (qubit = Qubit()) {
    repeat {
        H(qubit);
        T(qubit);
        CNOT(target, qubit);
        H(qubit);
        Adjoint T(qubit);
        H(qubit);
        T(qubit);
        H(qubit);
        CNOT(target, qubit);
        T(qubit);
        Z(target);
        H(qubit);
        let result = M(qubit);
    } until (result == Zero);
}
```

> [!TIP]   
> Avoid using repeat-until-success loops inside functions. 
> The corresponding functionality is provided by while loops in functions. 

### While Loop

Repeat-until-success patterns have a very quantum-specific connotation. They are widely used in particular classes of quantum algorithms -- hence the dedicated language construct in Q#. 
However, loops that break based on a condition and whose execution length is thus unknown at compile time need to be handled with particular care in a quantum runtime. 
Their use within functions on the other hand is unproblematic, since these only contain code that will be executed on conventional (non-quantum) hardware. 

Q# therefore supports to use of while loops within functions only. 
A `while` statement consists of the keyword `while`, an open parenthesis `(`,
a condition (i.e. a Boolean expression), a close parenthesis `)`, and a statement block.
The statement block (the body of the loop) is executed as long as the condition evaluates to `true`.

```qsharp
// ...
mutable (item, index) = (-1, 0);
while (index < Length(arr) && item < 0) { 
    set item = arr[index];
    set index += 1;
}
```

### Conditional Statement

The `if` statement supports conditional execution.
It consists of the keyword `if`, an open parenthesis `(`, a Boolean
expression, a close parenthesis `)`, and a statement block (the _then_ block).
This may be followed by any number of else-if clauses, each of which consists
of the keyword `elif`, an open parenthesis `(`, a Boolean expression,
a close parenthesis `)`, and a statement block (the _else-if_ block).
Finally, the statement may optionally finish with an else clause, which
consists of the keyword `else` followed by another statement block
(the _else_ block).
The condition is evaluated, and if it is true, the then block is executed.
If the condition is false, then the first else-if condition is evaluated;
if it is true, that else-if block is executed.
Otherwise, the second else-if block is tested, and then the third, and so on
until either a clause with a true condition is encountered or there are no
more else-if clauses.
If the original if condition and all else-if clauses evaluate to false,
the else block is executed if one was provided.

Note that whichever block is executed is executed in its own scope.
Bindings made inside of a then, else-if, or else block are not visible
after the end of the if statement.

For example,

```qsharp
if (result == One) {
    X(target);
} 
```

or

```qsharp
if (i == 1) {
    X(target);
} elif (i == 2) {
    Y(target);
} else {
    Z(target);
}
```

### Return

The return statement ends execution of an operation or function
and returns a value to the caller.
It consists of the keyword `return`, followed by an expression of the
appropriate type, and a terminating semicolon.

A callable that returns an empty tuple, `()`, does not require a
return statement.
If an early exit is desired, `return ()` may be used in this case.
Callables that return any other type require a final return statement.

There is no maximum number of return statements within an operation.
The compiler may emit a warning if statements follow a return statement
within a block.

For example,

```qsharp
return 1;
```

or

```qsharp
return ();
```

or

```qsharp
return (results, qubits);
```

### Fail

The fail statement ends execution of an operation and returns an error value
to the caller.
It consists of the keyword `fail`, followed by a string
and a terminating semicolon.
The string is returned to the classical driver as the error message.

There is no restriction on the number of fail statements within an operation.
The compiler may emit a warning if statements follow a fail statement within
a block.

For example,

```qsharp
fail $"Impossible state reached";
```

or

```qsharp
fail $"Syndrome {syn} is incorrect";
```
