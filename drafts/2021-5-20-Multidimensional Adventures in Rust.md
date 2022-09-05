---
layout: post
title: Multidimensional Adventures in Rust
---

## But first: why?
Multidimensional array libraries have been done to death in the scientific computing industry. NumPy, TensorFlow, PyTorch, Eigen C++, xtensor, and not to mention Rust's existing ndarray and ArrayFire crates all handle operations on multidimensional sets of data. Languages like Julia already have first-class multidimensional array support. So why bother creating your own?

In systems level languages, I've found that building numerical tools is an amazing strategy for learning the idioms and obscure details you don't traditionally learn. When I taught myself C++ by attempting to build a convnet, I ended up discovering friend classes and functions. When working on my current Rust project, Arrs, I learned about const generics, deserialization, FFI, and procedural macros.

These small, niche details help me develop a strong understanding for how the language works, and how it approaches difficult problems in language design. They give me a glimpes past the norms that remain largely unchanged across programming languages.

## Arrs and Rust
* Started with a lot of focus on deserialization
* Wanted to be efficient with memory and sharing, so overengineered a shared base system
* Wanted to put everything on the stack using static compiler checks
* Learned proc macros for fancy parsing method
* Now using RustaCUDA to do graph computations, inspired by TensorFlow.


## Why
Why create something that already exists and has been refined by experts?
I found it's a really good way to learn a new language

I recently took a serious interest in numerical computing and multidimensional array processing last summer, where I created [Quantum Python](https://github.com/QnnOkabayashi/Quantum), a C matrix library wrapped for Python usage. 

