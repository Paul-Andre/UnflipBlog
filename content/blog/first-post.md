---
title:
  "Search, Nullspaces, MiniZinc: How I created a solver for the Unflip puzzle game"
date: 2023-03-15T22:57:04+05:30
draft: false
category: Blogging
slug: first-post
summary:
  First Blog post
description:
  adfads
math: true

  

---


Hi, my name is Paul-Andre and today I'll walk through all the different approaches I attempted while creating a
solver for my puzzle game Unflip, and show you what worked best in the end.

Figuring out
Today I will show you what it took me to write a solver for my
puzzle game [unflip](https://unflipgame.com). I will examine search and
brute-force approaches, linear algebra and gaussian elimination, branch-and-bound,
MiniZinc, and even an attempt of using CUDA.


## 0. Preliminaries

[Unflip](https://unflipgame.com) (aka Unsquare, aka SquareFlip) is a puzzle game with a very simple
mechanic. Each level is a grid of black and white tiles. Your goal is to make
all tiles white. Each move consists of selecting a square of size 2x2 or larger and inverting the
colors of the tiles within. [Try it here](https://unflipgame.com).

My goal was to make a *solver*, a program that for any given level will find a solution with the minimum
number of moves.

While I keep,

Since my goal is to 

Since I'll be using a lot of Having a background in competitive programming, I find it's nice for the input
to be a simple text file (that I can easily read using `cin <<` in vanilla C++).


The first line will contain two numbers, *w* and *h*. (Frankly, as \(x=y\)
``` { title="input6.txt" }
6 6
000000
001100
010010
010010
001100
000000
```

```
6 6
......
..##..
.#..#.
.#..#.
..##..
......
```

The way you would read it in, in C++, is like this:
```c++ { title="asdfa.cpp"}
int w, h;
cin << w << h;
vector<vector<int>> grid;

```


## 1. Search

The most basic technique that can be used for solving any such puzzle is
*search*. You can think of the possible game states as a tree, and you want to
find the shortest

For completeness, I have written 

The implementation

## 2. Optimizations

Ok, but I don't need.

## 2. Brute-Force

Now, notice that the order of moves doesn't actually matter. 

Also, notice you never need to do the same move twice. 
(Doing fact doing a move a second time just undoes the first move.)

In fact you could think of the solution as being a vector of 1s and 0s,
representing which of the moves are "toggled on" or "toggled off".

4\*4 + 3\*3 + 2\*2 + 1\*1 = 30

So the combinations of all possible is 2^30
That's a lot of 

However

## 3. MiniZinc
[MiniZinc](https://www.minizinc.org/) is a very cool tool. It is a constraint
programming language that allows you to describe exactly what you're trying to
optimize in a high level language, and then allows you to swap out different
industry-grade constraint solvers/optimizers in the backend.

It provides a nice IDE to edit your code in. And when it comes time to
"productionize" your solver, you can split your code into a `.mzn` file for
the constraints, and provide the changing data in `.dzn` files, and execute it
from the CLI.

Starting from the flattened/optimized version of the problem, here's how I would model this 

The MiniZinc file to find the solution to this problem would be (supposing the
grid has been flattened, and I'm going to be using the representations of the
moves as calculated in C++) like this:

When I started using MiniZinc I had already 
I used minizinc directly on the "flattened/optimized" version of the problem --
and it's much simpler that way anyway.

In this specific instance, `W` is the size of the grid once flattened, and `H`

`A` is a matrix containing all the possible moves.


```MiniZinc {title = "minXor.mzn"}
set of int: W;
set of int: H;
array[W,H] of 0..1: a;

array[H] of var 0..1: v;
array[W] of 0..1: target;

constraint forall(i in W) (
    sum (j in H)(a[i,j]*v[j]) mod 2 == target[i]
    );
    
var int: s = sum(j in H) (v[j]);

solve
minimize s;

output [show(v) ++ "; " ++ show(s)]
```

In the above code, `a` refers to the matrix of possible moves, `target` refers
to the pattern I'm trying to solve for, and `v` is the variable I'm solving for
(thus the `var` keyword)

I constraint that `A\*x = target`, and finally I ask it to minimize the sum



The data is provided like this:


```sh
minizinc minXor.mzn input6.dzn -a --solver highs
```

The `--solver highs` indicates which constraint solver to use on the backend,
and the `-a` flag tells MiniZinc to print out intermediate solutions as it runs, not just
the final one (it's fun to see that the solver is actually doing something, in
case you're feeling impatient.)

In my experience


## 4. Gaussian Elimination

Ok, so we have basically we have a system of linear equations, modulo 2. (If
you're feeling fancy you can say "over the finite field of order 2", or over
Z/2Z. The nice thing is that "Z/2Z" is what we call in mathematics a "field",
which 

And we know how to solve systems of linear equations.

Gaussian elimination is the algorithm
As 

## 5. Null Spaces and the "Dual" Problem

Ok, so gaussian elimination provides *a* solution, but not necessarily the best
solution. Could we perhaps use that solution as a starting point and improve
it?

Here's a very intersting concept from linear algebra:
Given a matrix or linear transformation M, its *nullspace*, aka *kernel* is the set of all vectors that
get mapped by it into the zero vector. In other words, all vectors x such that:

x \in Ker(M) if and only if Mx = 0

Because of the linearity of the transformation M, it's easy to show that Ker(M)
forms a vector space. Supposing both x and y are in the nullspace of M you will.

If vectors x,y \in Ker(M), and a is a scalar then:

M(ax) = aMx = a\*0 = 0
M(x + y) = Mx + My = 0 + 0 = 0

Now suppose the optimal solution to our level is x, but the gaussian elimination algorithm gave us y. The difference (x-y) will be in the 

Mx = t
My = t
M(x-y) = t-t = 0

Note that (x-y) is in the nullspace of M!

Since the nullspace is a vector space, has finite dimensions, we can
represent it using basis of vectors.

Actually, it's possible to modify the 





## 6. MiniZinc (Dual)

## 7. Branch and Bound (Dual)

In the same way we can use search in the "primal space" to find what moves to
apply to solve our level, we can use search in the "dual space" to find what
kernel vectors we should apply to our existing solution in order to find the
minimal one.

The difference however is that we're not trying to minimize the number of steps
in our search, but rather, so we can no longer use DFS.


## 6. Putting it all together. Glue code

So far, the best solution was using Gaussian elimination, followed.
In my not-very-scientific experimentation, I have found the best results can
when using 

The only thing that remained was to write the glue code that would take my json
file containing all levels, and run my C++ code and MiniZinc on each one, and
output a new json.

I di

$$ x + p $$

## Bonus 1: CUDA

## Bonus 2: The "other" dual?








