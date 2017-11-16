---
# Mandatory fields. See more on aka.ms/skyeye/meta.
title: Quantum computer trace simulator | Microsoft Docs 
description: Overview of quantum computer trace simulator 
#keywords: Don’t add or edit keywords without consulting your SEO champ. 
author: Vadym 
ms.author: vadym@microsoft.com 
ms.date: 11/12/2017 
ms.topic: article-type-from-white-list 
# Use only one of the following. Use ms.service for services, ms.prod for on-prem. Remove the # before the relevant field. 
# ms.service: service-name-from-white-list
# product-name-from-white-list

# Optional fields. Don't forget to remove # if you need a field.
# ms.custom: can-be-multiple-comma-separated
# ms.devlang:devlang-from-white-list
# ms.suite: 
# ms.tgt_pltfrm:
# ms.reviewer:
# manager: MSFT-alias-manager-or-PM-counterpart
---

# Overview 

The Microsoft quantum computer trace simulator executes a quantum program without actually simulating the state of a quantum computer.  For this reason, the trace simulator can execute quantum programs that use thousands of qubits.  It is useful for two main purposes: 

* Debugging classical code that is part of a quantum program. 
* Estimating the resources required to run a given instance of a quantum program
  on a quantum computer.

The trace simulator relies on additional information provided by the user when
measurements must be performed. See Section [Providing the probability of
measurement outcomes](#providing-the-probability-of-measurement-outcomes) for more
details on this. 

# Providing the probability of measurement outcomes

There are two kinds of measurements that appear in quantum algorithms. The first
kind plays an auxiliary role where the user usually knows the
probability of the outcomes. In this case the user can write
`AssertProb` from the `Microsoft.Quantum.Primitive` module to express this knowledge. The following example illustrates this: 

```qsharp
operation Teleportation (source : Qubit, target : Qubit) : () {
    body {
        using (ancillaRegister = Qubit[1]) {
            let ancilla = ancillaRegister[0];

            H(ancilla);
            CNOT(ancilla, target);

            CNOT(source, ancilla);
            H(source);

            AssertProb([PauliZ], [source], Zero, 0.5, "Outcomes must be equally likely", 1e-5);
            AssertProb([PauliZ], [ancilla], Zero, 0.5, "Outcomes must be equally likely", 1e-5);

            if (M(source) == One)  { Z(target); X(source); }
            if (M(ancilla) == One) { X(target); X(ancilla); }
        }
    }
}
```

When the trace simulator executes `AssertProb` it will record that measuring
`PauliZ` on `source` and `ancilla` should given an outcome of `Zero` with probability
0.5. When the simulator executes `M` later, it will find the recorded values of
the outcomes probabilities and `M` will return `Zero` or `One` with probability
0.5. When the same code is executed on a simulator that keeps track of the
quantum state, such a simulator will check that the provided probabilities in
`AssertProb` are correct. 

The second kind of measurement is used to read out the answer of the quantum
algorithm and the user usually does not know the probability of such measurement
outcomes. The quantum computer trace simulator provides a function `ForceMeasure` in
namespace `Microsoft.Quantum.Simulation.Simulators.QCTraceSimulators` to force
the simulator to take the measurement outcome preferred by the user. 

`[TODO: add reference to
Microsoft.Quantum.Simulation.Simulators.QCTraceSimulators.ForceMeasure qsharp
API doc.]`

# Running your program with the quantum computer trace simulator 

The quantum computer trace simulator may be viewed as just another simulator. The C# driver program for using it is very similar to the one for any other Quantum Simulator: 

```csharp
using Microsoft.Quantum.Simulation.Core;
using Microsoft.Quantum.Simulation.Simulators;
using Microsoft.Quantum.Simulation.Simulators.QCTraceSimulators;

namespace Quantum.MyProgram
{
    class Driver
    {
        static void Main(string[] args)
        {
            QCTraceSimulator sim = new QCTraceSimulator();
            var res = MyQuantumProgram.Run().Result;
            System.Console.WriteLine("Press any key to continue...");
            System.Console.ReadKey();
        }
    }
}
```

Note that if there is at least one measurement not annotated using `AssertProb`
or `ForceMeasure` the simulator will throw `UnconstraintMeasurementException`
from `Microsoft.Quantum.Simulation.QCTraceSimulatorRuntime` namespace. 

`[TODO: add reference to
Microsoft.Quantum.Simulation.QCTraceSimulatorRuntime.UnconstraintMeasurementException
csharp API doc.]`

In addition to running quantum programs the trace simulator comes with five
components for detecting bugs in the programs and performing quantum program
resource estimates: 

* Distinct Inputs Checker `[TODO: Add reference to the documentation file here]`
* Invalidated Qubits Use Checker `[TODO: Add reference to the documentation file
  here]`
* Gate Counter `[TODO: Add reference to the documentation file here]`
* Circuit Depth Counter `[TODO: Add reference to the documentation file here]`
* Circuit Width Counter `[TODO: Add reference to the documentation file here]`

Each of these components can be enabled by setting appropriate flags in
`QCTraceSimulatorConfiguration`. More details about using each of these
components are provided in the corresponding reference files.

`[TODO: add reference to
Microsoft.Quantum.Simulation.Simulators.QCTraceSimulators.QCTraceSimulatorConfiguration
csharp API doc. ]`
