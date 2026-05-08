---
icon: books
---

# Best Resources to Learn LLVM

### 🟢 Beginner-Friendly Introductions

* **LLVM Official Tutorials**\
  🔗 [https://llvm.org/docs/tutorial/](https://llvm.org/docs/tutorial/)\
  The _Kaleidoscope_ tutorial (building a toy language) is the canonical starting point.\
  It walks you through creating a simple language with LLVM IR generation and JIT.
* **“Getting Started with LLVM Core Libraries” (book)** by Bruno Cardoso Lopes & Rafael Auler\
  Great if you like structured learning with practical examples.
* **YouTube Talks (LLVM Developers’ Meeting)**\
  The official [LLVM YouTube channel](https://www.youtube.com/@LLVMFoundation) has intro and advanced talks directly from compiler engineers.

***

### 🟡 Intermediate / Hands-On

* **LLVM Language Reference Manual**\
  🔗 [https://llvm.org/docs/LangRef.html](https://llvm.org/docs/LangRef.html)\
  Defines LLVM IR (its “assembly language”) — essential if you want to really understand what you’re generating.
* **Building a JIT Compiler with LLVM (Eli Bendersky’s blog + docs)**\
  Eli has several blog posts demystifying LLVM’s JIT, passes, and internals.
* **Compiler Explorer (godbolt.org)**\
  Lets you type C/C++/Rust and see the generated LLVM IR instantly.

***

### 🔴 Advanced / Internals

* **LLVM Source Code**\
  Nothing beats reading the code — it’s modular and well-documented inline. Look at `lib/IR`, `lib/Transforms`, `lib/CodeGen`.
* **LLVM Developers’ Meetings & Mailing List**\
  🔗 [https://discourse.llvm.org/](https://discourse.llvm.org/)\
  Where the core devs discuss new passes, optimizations, and design.\
  Great for keeping up with bleeding edge.
* **Dragon Book vs. Modern LLVM**\
  While the “Dragon Book” (Compilers: Principles, Techniques, and Tools) gives solid background,\
  you’ll want to marry those concepts with LLVM’s modern IR and pass infrastructure.

***

### ⚡ Suggested Learning Path

1. Do the **Kaleidoscope tutorial**.
2. Play with **LLVM IR on Compiler Explorer**.
3. Read the **LangRef** to understand IR deeply.
4. Implement a toy compiler or JIT using LLVM libraries.
5. Dive into **passes and optimizations**.
6. Watch LLVM Dev Meeting talks and contribute small patches.
