---
layout: post
title: "Scala Substitution Model"
comments: True
permalink: "scala-substitution-model"
---

### Introduction

`Scala` model of expression evaluation is based on the principle of substitution
model, in which variable names are replaced by the values they are bound to.The underlying
idea is that all evaluation does is reduce an expression to a value, which is a term that
does not need further evaluation. Values not include only constants, but tuple, constructors, and
functions. The substitution model is formalized in the $\lambda-calculus$, which gives foundation for functional programming.

In `Scala` program expressions are evaluated in the same way we would evaluate a mathematical expression. For example we know the expression $(2*2) + (3 * 4)$ will be evaluated by first evaluating $2 * 2$ and $3*4$ and finally $4 + 12$. Similarly in `Scala` non-primitive expression is evaluated as follows.
