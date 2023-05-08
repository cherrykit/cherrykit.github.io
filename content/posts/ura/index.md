---
title: "Inequality Comparisons over Homomorphically Encrypted Data"
date: 2023-05-04
description: Contrasts a new inequality comparison protocol with existing inequality comparison schemes
menu:
  sidebar:
    name: Homomorphic Encryption
    identifier: ura
    weight: 10
# tags: []
# categories: []
---

Homomorphic encryption allows the evaluation of functions over encrypted data without requiring decryption first. However, only a limited number of computations are possible before decryption becomes infeasible. In particular, the number of possible consecutive multiplications, called multiplicative depth, is limited. Hence, efficient protocols are necessary for basic operations such as comparisons.

Under supervision of Professor Florian Kerschbaum, I spent four months creating and implementing a new algorithm that reduces inequality comparisons over homomorphically encrypted data to a linear number of equality comparisons. The new protocol allows the user to choose the desired multiplicative depth while varying the number of overall multiplications.

In a [course paper](ura.pdf) written for the "Algorithm Design and Analysis" course at the University of Waterloo (CS466), I proved the correctness and runtime properties of the new protocol. Furthermore, I contrast the protocol with a prevalent inequality algorithm.