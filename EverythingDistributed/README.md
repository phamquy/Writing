# Classic distributed algorithm & TLA+

Below is a **curated learning path + high-quality resources** for **TLA+**, **Paxos**, and **RAFT** (assuming you meant **RAFT**, not “DRAFT”).\
This is tailored for an **experienced software engineer** who wants **deep, formal understanding**, not just surface-level intuition.

***

### 1. TLA+ (Formal Specification & Verification)

#### Why TLA+ matters

* Used to **formally specify and verify** distributed systems
* Widely used at **Amazon, Microsoft, AWS, etc.**
* Excellent for reasoning about **Paxos, Raft, consistency, safety, liveness**

#### Core Resources (Authoritative)

**📘 Book (Primary)**

* _**Specifying Systems**_**&#x20;– Leslie Lamport**
  * Free PDF from Lamport himself
  * Covers:
    * TLA+ language
    * Temporal logic
    * Model checking
  * Link:\
    https://lamport.azurewebsites.net/tla/book.html

**📘 Book (Modern, Practical)**

* _**Practical TLA+**_**&#x20;– Hillel Wayne**
  * Best hands-on intro for engineers
  * Includes:
    * Real-world examples
    * How to _think_ in TLA+
  * https://www.apress.com/9781484238288

***

#### Online Tutorials & Courses

**🎓 Learn TLA+ (Official)**

* https://lamport.azurewebsites.net/tla/learning.html

**🧪 TLA+ Examples Repo**

* https://github.com/tlaplus/Examples

**📝 Learn by Doing**

* https://learntla.com/
  * Step-by-step
  * Extremely beginner-friendly

***

#### Tools

* **TLA+ Toolbox**
  * https://lamport.azurewebsites.net/tla/toolbox.html
* Model checker: **TLC**

***

### 2. Paxos (Consensus Algorithm)

#### Goal

Understand **why Paxos works**, not just the message flow.

***

#### Must-Read Papers (In Order)

**📄 Paxos Made Simple**

* Leslie Lamport
* Short, canonical explanation
* https://lamport.azurewebsites.net/pubs/paxos-simple.pdf

**📄 The Part-Time Parliament**

* Original Paxos paper
* Dense but foundational
* https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf

***

#### Practical & Explanatory Resources

**📘 Book Chapter**

* **Designing Data-Intensive Applications – Martin Kleppmann**
  * Chapter on consensus
  * Excellent intuition
  * https://dataintensive.net/

**📄 Paxos for Engineers**

* https://understandingpaxos.wordpress.com/
  * Clear, step-by-step explanations

***

#### Paxos + TLA+

* Official Paxos TLA+ specs:
  * https://github.com/tlaplus/Examples/tree/master/specifications/Paxos
* This is where Paxos _really_ clicks

***

### 3. Raft (Consensus Algorithm)

#### Why Raft

* Designed to be **understandable**
* Used in:
  * etcd
  * Consul
  * CockroachDB
  * TiKV

***

#### Core Resources

**📄 Original Raft Paper**

* _**In Search of an Understandable Consensus Algorithm**_
* https://raft.github.io/raft.pdf

**🌐 Raft Website (Excellent)**

* https://raft.github.io/
  * Visualizations
  * Extended explanations

***

#### Interactive Learning

**🧠 Raft Visualization**

* https://thesecretlivesofdata.com/raft/
  * Best interactive Raft explainer ever built

***

#### Code-Level Understanding

**🧩 Raft Implementations**

* MIT 6.824 Lab (Go)
  * https://pdos.csail.mit.edu/6.824/
* etcd Raft
  * https://github.com/etcd-io/raft

***

#### Raft + TLA+

* Raft TLA+ spec (official):
  * https://github.com/ongardie/raft.tla

***

### 4. Suggested Learning Order (Strongly Recommended)

```
1. Raft (intuition first)
2. Paxos (theory & rigor)
3. TLA+ basics
4. Paxos TLA+ specification
5. Raft TLA+ specification
6. Write your own TLA+ spec for a toy consensus algorithm
```
