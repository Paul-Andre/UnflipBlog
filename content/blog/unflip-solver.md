---
title:
  "Search, Nullspaces, MiniZinc"
date: 2025-10-03T22:57:04+05:30
draft: false
category: Blogging
slug: unflip-solver
summary:
  First Blog post
description:
  Creating a solver for the Unflip puzzle game.
math: true

  

---

Hi, I'm Paul-Andre Henegar, and welcome to my tech blog.

Today I'll be sharing with you all the different approaches I took to write a solver for my puzzle game [Unflip](https://unflipgame.com).
As you'll see there's quite a lot of diversity in terms of the algorithms and math that went into it.

## 0. Preliminaries

Unflip (aka Unsquare, aka SquareFlip) is an original puzzle game with a very simple
mechanic.

Each level you're given a grid of black and white tiles.
Your goal is to make all tiles white.
Each move consists of selecting a square of size 2x2 or larger and inverting the
colors of the tiles within. [Try it here](https://unflipgame.com).

*INSERT PREVIEW GIF if not widger*

My goal was to create a *solver*, a program that would find the solution with
the minimum number of moves for any any given level.


### Input

I needed to have a way to pass each level as input to my solver.
I was already using JSON inside my game.
However, for the purpose of the solver, I found it simpler to represent each level as a simple text files and pass them into my solver through stdin.
C++ doesn't have JSON in its standard library, and this way I can easily parse my levels using `cin >>`.
Another reason I did this is it's the tradition in competitive programming, and this felt like a very "competitive programming"-type problem.

Here's an example:
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

By the way, the way you would read this, in C++, is like this:
```c++ { title="asdfa.cpp"}
int w, h;
cin >> w >> h;
vector<vector<bool>> grid;
for (int i=0; i<h; i++) {
  for (int j=0; j<w; j++) {
    char c;
    cin>>c;
    grid

```

Note: Although I have distinct width and height parameters, they were always equal in the levels I tested.


## 1. Search

The first reflex when when solving such a problem is *search*. You can think of the possible game states as a tree, with the beginning state as its root, and find the shortest path to the solved state.

The most basic technique that can be used for solving such puzzles is
*search*. Think of the possible game states as a *tree*.
Each node in the tree is a possible grid configuration, and the edges between them are moves.
Finding a solution to the puzzle corresponds to finding a path from the root to a node repr

!!! Picture.

Since we're trying to find a solution with the smallest number of moves, we want to find the shorted path through the tree.
In order to do this we will use *breadth-first search* (BFS).

## 2. Optimizing the Representation

Let's optimize how we represent our data (and also make it simpler to code).

One optimization is to "flatten" the grid.
Instead of representing a 8x8 grid as 8 arrays of 8 elements, we can represent it as a single array of 64 elements.
While we're at it, since all of our tiles are either black or white, we can encode each tile as a bit. (0 = white, 1 = black).
Assuming we don't ever go bigger than 8x8, we can thus represent each grid configuration as a 64 bit integer (a `uint64_t` in C++).

Another optimization is to "pre-compute" all the possible moves.
For each valid "square", the tiles inside the selected square will flip, while the ones outside will stay the same.
We can use a bit of 1 for the tiles that change and a bit of 0 for the tiles that don't.

In this representation, to apply a given move to grid, we simply use the bitwise XOR operator (represented in C++ as `^`).

``` C++
uint64_t grid;
uint64_t move;
...
// Apply move to grid.
grid ^= move;
```

And here's how you would precompute the possible.
``` C++
!!!

```

The complete code for the seach + optimization can be found here:  !!!

!!! Link to github


## 3. Matrix Multiplication and Brute-Force

Let's further analyze this problem.

Notice that there's no point in ever doing the same move more than once. (Because doing the same move a second time will simply undo the first time.)

Notice also that the order in which you do the moves doesn't matter.

We can think of each move as either being `toggled` or `untoggled`.


We can therefore represent this as an equation modulo 2.

!!!



When you combine two moves together, that basically adding together two integer vectors modulo 2.

You can think of this problem as a problem with vectors of integers modulo 2.


Also, notice you never need to do the same move twice. 
(Doing fact doing a move a second time just undoes the first move.)

In fact you could think of the solution as being a vector of 1s and 0s,
representing which of the moves are "toggled on" or "toggled off".

4\*4 + 3\*3 + 2\*2 + 1\*1 = 30

So the combinations of all possible is 2^30
That's a lot of 

However

## 4. MiniZinc
[MiniZinc](https://www.minizinc.org/) is a very cool tool. It is a constraint
programming language that allows you to describe exactly what you're trying to
optimize in a high level language, and then allows you to swap out different
industry-grade constraint solvers/optimizers in the backend.

!!! picture of MiniZinc IDE

Once it comes time to
"productionize" your solver, you can put your actual constraints into a `.mzn` file, and provide the variable data in `.dzn` files.
You can then run the constraint solver from the CLI as:
```sh
minizinc minXor.mzn input6.dzn -a --solver highs
```



Starting from the flattened/optimized version of the problem, here's how I would model this 

The MiniZinc file to find the solution to this problem would be (supposing the
grid has been flattened, and I'm going to be using the representations of the
moves as calculated in C++) like this:

When I started using MiniZinc I had already 
I used minizinc directly on the "flattened/optimized" version of the problem --
and it's much simpler that way anyway.

In this specific instance, `W` is the size of the grid once flattened, and `H`

`A` is a matrix containing all the possible moves.

I c


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


The `--solver highs` indicates which constraint solver to use on the backend,
and the `-a` flag tells MiniZinc to print out intermediate solutions as it runs, not just
the final one (it's fun to see that the solver is actually doing something, in
case you're feeling impatient.)

In my experience


## 5. Gaussian Elimination

Ok, so we have a system of linear equations modulo 2.
Why don't we just solve it?
Let's do that.



(If
you're feeling fancy you can say "over the finite field of order 2", or over
\(
\mathbb Z/2\mathbb Z
\). The nice thing is that \(Z/2Z\) is what we call in mathematics a "field",
which 

And we know how to solve systems of linear equations.

Gaussian elimination is the algorithm
As 

Here's a little trick when you're programming something like this. Use `assert` a lot!. 
Also, once you found the solution to your equation, actually do the multiplication to see if it actually is the solution.

However, by running this 

!!!!! 




## 6. Null Spaces and the "Dual" Problem

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




## 7. Branch and Bound (Dual)

## 8. MiniZinc (Dual)


In the same way we can use search in the "primal space" to find what moves to
apply to solve our level, we can use search in the "dual space" to find what
kernel vectors we should apply to our existing solution in order to find the
minimal one.

The difference however is that we're not trying to minimize the number of steps
in our search, but rather, so we can no longer use DFS.


## 9. Putting it all together. Glue code

In my not-very-scientific experimentation, I have found the best results came
when using MiniZinc with the HiGHS constraint solver on the "dual" problem,
with the kernel basis as originally generated by the gaussian elimination.

The only thing that remained was to write the glue code that would take my JSON
file containing all levels (120 at the time of writing), convert the JSON to
the textual format, run the C++ code to do the gaussian elimination, pipe that
to MiniZinc and run the constraint solver, parse the solution, and output a new
json.

I used Cursor and kinda vibe coded it. It's a very different form of activity.

I had to do a lot of directing to get it
to work. For example, I had to tell it that the output from `minXorDual.mzn`
was already the correct solution, and that it didn't need to modify it or do
anything weird with it.

I also asked it to do a sanity check and see whether the solution it got
actually matched the puzzle. 

It got really confused about the indexing and asked 




## Bonus 1: CUDA

I have never coded in CUDA before this point, so I decided to switch to the
Linux partition of my desktop computer.

This document gives a quickstart to CUDA.

## Bonus 2: The "other" dual?
When representing, however there's another way to represent the puzzle. Instead
of thinking in terms of the solid square tiles, we can think of the contours of
the squares and the borders between tiles. Similarly to how black tile = 1 and white tile = 0,
we can make, for each border between tiles: tiles have the same color = 0,
tiles have different color = 1.

From there, we can do all the same things I have done (linear algebra, etc...)
but on the "border space" instead of the "tile space". Will this be more or
less effective? It's something to be tried.








