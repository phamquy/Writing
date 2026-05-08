---
icon: database
---

# LSMTree is not a Tree, but kind of a tree

### Why Is LSM Tree Called a "Tree"?

The **Log-Structured Merge Tree (LSM Tree)** is called a "tree" due to its **hierarchical structure and conceptual similarity** to traditional tree-based data structures. However, it is **not** a tree in the strict sense (like a binary tree or B-tree with nodes and pointers).

***

#### 🧱 1. Hierarchical Organization

* Data in an LSM Tree is organized across multiple **levels** (e.g., Level 0, Level 1, ..., Level N).
* Data flows **from memory to disk** in a cascading manner.
* This structure mimics a **tree-like hierarchy**, where each level depends on and merges with the next.

***

#### 🔄 2. Merge Behavior Like Tree Traversals

* Data is periodically **compacted** or **merged** from one level to the next.
* This resembles a **traversal** through layers, similar to traversing nodes in a tree.

***

#### 🌳 3. Conceptual Inheritance from Tree Structures

* LSM Trees were designed as an alternative to **B-trees** for better write performance.
* Although they don’t use pointers like B-trees, the multi-level sorted structure is **inspired** by traditional trees.

***

#### ✅ Summary

* ✅ **"Tree"** refers to the **hierarchical, level-based design**.
* 🚫 It does **not** have explicit tree nodes or branches.
* 🛠️ Think of it as a **log-structured system with tree-like behavior** in compaction and searching.
