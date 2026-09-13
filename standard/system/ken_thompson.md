# The Design Lesson
The lasting lesson of Ken Thomson approach is that he found the smallest general mechanism
that made many special cases unnecessary.

One process abstraction instead of many execution models
One file interface instead of device-specific APIs
`fork() + exec()` instead of one complicated process-launch operation
`pipes` instead of programs directly integrating with one another

The goal is not minimalism for its own sake.
The goal is condensation without loss of capability:

Find a small set of orthogonal mechanisms from which the larger system can be composed.

---

# The Ken Thompson Approach

This principle is inspired by the simple, elegant, and deeply considered mechanisms at the heart of Unix.
Unix solved Complex behaviour by from simple mechanisms:

```
process
fork()
exec()
pipe()
read()
write()
```

Each primitive had a narrow purpose. Together, they made an entire operating system possible.

1. The Process Model

Instead of treating programs as batch jobs, Unix introduced the process as an independent execution unit.
Each process has:

PID
address space
registers
file descriptors

This abstraction survives almost unchanged in modern operating systems.

2. fork()

Thompson designed and implemented fork().

```
parent
   |
 fork()
   |
+--------+
| parent |
| child  |
+--------+
```
fork() creates a new process by duplicating the current process. Its elegance comes from doing one thing only:

create another execution context

It does not decide which program the child should run. That responsibility belongs to another primitive.

3. exec()

exec() is the second half of the Unix process model.

```
fork()
   ↓
child process
   ↓
exec("/bin/ls")
```
The process remains, but its program image is replaced.

This creates a clean separation:

fork()  → create a process
exec()  → replace its program

Instead of one complicated “create and run a program” operation, Unix uses two small, composable mechanisms.

4. Everything Is a File

One of Unix’s most influential architectural principles was: Everything is a file.
Different resources could be accessed through the same interface:

* disk
* terminal
* printer
* pipe
* device

using the same small set of operations:

- read()
- write()
- close()

The power came from removing unnecessary distinctions.

Programs did not need a different interface for every type of resource. They could operate on file descriptors and remain largely unaware of what was behind them.



# The core principle

> Do not add exceptions. Find the smallest general mechanism that makes the exceptions disappear.

For the Scribe compiler, that means avoiding case-specific repair rules such as:

```text
if substitution:
    do X

if exclusion:
    do Y

if missing unit coverage:
    do Z
```

Instead, build around a few orthogonal primitives:

```text
Source obligation
Atomic decomposition
Executable carrier
Unsupported residue
Validation
Deterministic materialization
```

Everything should be expressible as composition of those mechanisms.

A Ken Thompson-style test is:

1. Does the mechanism solve more than one class of problem?
2. Does it remove special cases rather than add them?
3. Are semantics separated cleanly from representation?
4. Can independent parts be composed without knowing each other?
5. Can the invariant be stated in one sentence?

For unsupported handling, the root invariant is:

```text
Emit executable AST only for independently executable source obligations.
Preserve everything else as unsupported source residue.
```

From that one rule follow:

* no invented rules for referenced operands;
* no unit-code coverage patches;
* no special substitution workaround;
* no approximation of exclusions or approvals;
* no duplicate executable and unsupported representation;
* no coupling between semantic preservation and fallback presentation.

The minimal architecture is:

```text
classify(source obligation)
    -> executable atoms
    -> unsupported residue
```

Then:

```text
validate(executable atoms)
materialize(unsupported residue)
```

`validate` never creates meaning.

`materialize` never creates executable meaning.

That is the condensation point. Anything that requires another exception probably indicates the core abstraction is still wrong.


---

# Question to ask
!!! Do not delete these until finalized !!!
- look for the clue, specific pattern, characteristics that you can exploit for the case you are handling, that is generci to the patern and no overfit or hack.
- have clear statement for the goal to achieve, what are the core traits of the goal enabler.
- always revisit to check and ask: am I handling the right problem?
- occasionally try a random idea outside the box. It may give a new breakthrough. 