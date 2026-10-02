<!---
{
  "id": "e9a90c71-845e-4e2c-a6af-1128a7921d93",
  "teaches": "GNU Make: From Shell Script to Makefile",
  "depends_on": [
    "05e99c97-2969-45b9-b955-5b5a62a0786e"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-01",
  "keywords": [
    "make",
    "Makefile",
    "build system",
    "target",
    "dependency",
    "recipe",
    "incremental build"
  ]
}
--->

# GNU Make: From Shell Script to Makefile

> In this exercise you will take a working build script for a small C project with a static library, turn it into a `Makefile` with explicit targets and dependencies, and watch `make` skip work that is already done. No `sudo` is required: the library is linked from the project directory.

## Introduction

As soon as a C project consists of more than one file, building it means running several commands in the right order: compile the library, pack it into an archive, compile the main program, link everything. You could put those commands into a shell script. That works, and many people stop there.

`make` goes one step further. Instead of listing *commands*, a `Makefile` lists **targets** (files that should exist), their **dependencies** (files they are built from) and the **recipe** (the commands to get from the dependencies to the target):

```make
target: dependency1 dependency2
	command
```

The line with the command must start with a real **tab character**, not spaces. This is the single most common mistake with `make`, and `vim` will happily insert spaces if you have `expandtab` set. Check with `:set noexpandtab` before you start, or type the tab with `Ctrl-V Tab`.

When you run `make target`, it compares timestamps. If the target is newer than all of its dependencies, nothing happens. If a dependency is newer, the recipe runs. Because dependencies can themselves be targets, `make` walks the whole chain and rebuilds exactly what is out of date, and nothing more. That is what the lecture calls an **incremental build**.

### 1.1) Further Readings and Other Sources

* [GNU Make Manual: An Introduction to Makefiles](https://www.gnu.org/software/make/manual/html_node/Introduction.html)
* [Makefile Tutorial By Example](https://makefiletutorial.com/)
* [The lecture's `compile.sh` and `Makefile` side by side (Gist)](https://gist.github.com/maxclerkwell/a7f31732a3426d27cb532c78d84217f4)

## Tasks

### Task 1: Set Up the Project

Create a project directory with a tiny static library and a main program. If you still have `vec2.h` and `vec2.c` from the static library exercise, reuse them. Otherwise:

```bash
mkdir -p ~/make-intro && cd ~/make-intro
vim vec2.h
```

```c
// file: vec2.h
#ifndef VEC2_H
#define VEC2_H
typedef struct { double x, y; } vec2;
vec2   VEC2_add(vec2 a, vec2 b);
double VEC2_len(vec2 a);
#endif
```

```c
// file: vec2.c
#include <math.h>
#include "vec2.h"
vec2   VEC2_add(vec2 a, vec2 b) { vec2 r = { a.x + b.x, a.y + b.y }; return r; }
double VEC2_len(vec2 a)         { return sqrt(a.x * a.x + a.y * a.y); }
```

```c
// file: main.c
#include <stdio.h>
#include "vec2.h"
int main(void) {
    vec2 a = { 3, 4 }, b = { 1, 2 };
    vec2 c = VEC2_add(a, b);
    printf("c = (%.1f, %.1f), |a| = %.1f\n", c.x, c.y, VEC2_len(a));
    return 0;
}
```

### Task 2: Write the Build as a Shell Script

```bash
vim build.sh
```

```bash
#!/bin/bash
gcc -Wall -Wextra -O2 -c vec2.c -o vec2.o
ar rcs libvec2.a vec2.o
gcc -Wall -Wextra -O2 main.c -o main_program -L. -lvec2 -lm
echo "done"
```

```bash
chmod +x build.sh
./build.sh
./main_program
./build.sh
```

Run the script twice and watch the second run. Everything is compiled again, although nothing changed.

### Task 3: Translate the Script Into a Makefile

Each command in the script produces one file. That file becomes a target, the files it reads become its dependencies:

```bash
vim Makefile
```

```make
main_program: main.c libvec2.a
	gcc -Wall -Wextra -O2 main.c -o main_program -L. -lvec2 -lm

libvec2.a: vec2.o
	ar rcs libvec2.a vec2.o

vec2.o: vec2.c
	gcc -Wall -Wextra -O2 -c vec2.c -o vec2.o
```

Remove the old build products and run `make`:

```bash
rm -f vec2.o libvec2.a main_program
make
./main_program
```

Which recipe ran first, which one last? Compare with the order in which the rules are written. `make` starts with the **first** target in the file and resolves dependencies depth-first.

### Task 4: Run It Again

```bash
make
```

Read the message. Then modify one file and rebuild:

```bash
touch vec2.c
make
```

Which recipes ran this time? Why did `main_program` have to be relinked although `main.c` did not change?

Now touch `main.c` instead and run `make` again. Which recipes ran?

### Task 5: Build a Single Target

```bash
rm -f vec2.o libvec2.a main_program
make libvec2.a
ls
```

Only the archive and its dependency were built. Then:

```bash
make vec2.o
make
```

### Task 6: Break the Tab

Replace the tab in front of one recipe with four spaces (in `vim`: put the cursor on the line, `0`, then `xi    <Esc>` and check with `:set list`). Run `make` and read the error message. Fix it again.

### Task 7: Dry Run

```bash
touch vec2.c
make -n
```

`-n` prints what `make` *would* do without doing it. Compare the output with what Task 4 actually did. This is the fastest way to understand an unfamiliar `Makefile`.

## Questions

- In which order did the three recipes run in Task 3, and why does that order not match the order in the file?
- Why did `touch vec2.c` cause two recipes to run, but `touch main.c` only one?
- What exactly does `make` compare to decide whether a target is up to date? What happens if the system clock is wrong?
- What would happen if you listed `vec2.h` nowhere in the `Makefile` and then changed the struct? Try it.
- Why does `make` insist on a tab? Look up the history of this decision; the author has apologised for it.
- The shell script and the `Makefile` contain the same three commands. Name three things the `Makefile` version can do that the script cannot.

## Advice

The most important habit to build from this exercise: **every file that is read by a recipe belongs in the dependency list**. If a file is missing there, `make` will not notice when it changes, and you will spend an afternoon debugging a stale object file. The next exercise is entirely about this failure mode.

Keep the `Makefile` from this exercise. The following three exercises extend it step by step until it is a reusable template for small C projects.
