---
layout: post
title:  "Solving optimization problems using Integer Programming"
subtitle: Let's talk a little about mathematical optimization and some cool programming paradigms we can use to solve some really hard problems.
subtitleImage: https://i.imgur.com/2VUWRXM.jpg
date:   2018-09-15 10:23:00
language: en
image: https://i.imgur.com/2VUWRXM.jpg
description: Integer Programming.
---

Lately I have been working with some discrete optimization problems, learning about some really interesting programming paradigms that can be used to solve optimization and feasibility problems. I think this is a really interesting area, and although I'd say I'm still a beginner on the topic, I wanted to share a few words about it.

In this post I'll do a quick intro about integer programming, then describe and solve one problem where this technique can be helpful.

## Integer programming

So there are different ways we can solve problems with programming, and one way is to use [integer programming](https://en.wikipedia.org/wiki/Integer_programming). An integer programming problem is a mathematical optimization/feasibility problem where we model our problem using integer variables.

The idea is to decide on a model that describes our problem. Then we can describe an "objective" and some "constraints" that are necessary for this problem to be solved. The same problem can be describe in different ways, some better than others, but all those models can be used with integer programming to reach a solution.

This is the basic idea of this programming paradigm. I believe things become a lot clearer when we actually solve one problem using this method, so let's get to it.

## The N Queens Puzzle

<div align="center">
<a href="https://i.imgur.com/2VUWRXM.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/2VUWRXM.jpg" style="width:40%;border-style:solid;border-width:1px"/></a>
</div><br>
One problem that has been around for a long time is the [N Queens Puzzle](https://en.wikipedia.org/wiki/Eight_queens_puzzle). The problem can be described really easily: given a _n_ by _n_ chessboard, how can we place the most amount of queens on the board in such a way that they don't attack each other. Remember, queens move horizontally, vertically and diagonally as much as they want, so you cant have overlapping queens in those areas.

### Tools

There are plenty of libs we can use to solve optimization problems, here I'll program using [SCIP](http://scip.zib.de/), which is free for non-commercial use, and the interface lib [PySCIPOpt](https://github.com/SCIP-Interfaces/PySCIPOpt) to write our problems in python. It's an incredibly easy (and powerful) library to use, it has other interface libraries for other languages, but python really makes the code easier to write.

### The modeling

Ok, so there are multiple ways we can model this problem. One way that I think is probably the most intuitive is an n by n matrix that holds only 0 and 1 values. This means:

$$board_{i,j} \in \{0,1\}, \text{where i and j} \in [0,n)$$

So we have our $$board_{i,j}$$ that represents the chessboard. In this matrix if the value is 1 then it means "there is a queens in this coordinate", and 0 means there is nothing there. The code to create the variables is really simple:

{% highlight python %}
model = Model("Queens") # our problem model

# First create variables
board = [[model.addVar(name="board_%s_%s"%(i, j), vtype="BINARY") for j in range(width)] for i in range(height)]
{% endhighlight %}

This is, of course, one of the ways you can do it. There are plenty of other ways you can try, maybe they even are better than this one. For instance, instead of having a matrix of binary values you could have only one array of integer values: $$board_{i} = j$$ which would mean "at row _i_ on our board, there is a queen at column _j_". You can probably think of other models too.

But we will keep going with our matrix modeling for now.

### The objective

What we want is to place the most amount of queen pieces in our board as possible. We have the variables described previously, so if we want the highest possible value we want to _maximize_ the sum of all variables. Since they are binary values that mean "there is a queen at this coordinate, or not", then if we maximize the sum of these variables it means we placed the maximum number of queens.

$$\text{Objective: Maximize} \sum_{i,j} board_{i,j}$$

Again, really easy to describe in code using a helper function from the SCIP lib called _quicksum_:

{% highlight python %}
model.setObjective(quicksum(board[i][j] for j in range(width) for i in range(height)), "maximize")
{% endhighlight %}

### Constraints

With only the objective we still haven't fully described the problem. What we are missing are the constraints. Since we are using integer programming, we also need to describe all these constraints as inequations, so we need to model them as such.

Like I have said before, we want to place the most amount of non-attacking queens in our board, so we end up with three constraints:

* There can only be at most one queen per row. This means that for each row _i_, the sum of every variable on the _column_ is between 0 and 1 (either there is no queen, or there is at most one).

$$\forall i, \text{then } 0 \leq \sum_{j} board_{i,j} \leq 1$$

* There can only be at most one queen per column. This means that for each column _j_, the sum of every variable on the _row_ is between 0 and 1 (either there is no queen, or there is at most one).

$$\forall j, \text{then } 0 \leq \sum_{i} board_{i,j} \leq 1$$

* There can only be at most one queen per diagonal. This means considering both the diagonals "directions" (say, downwards from left to right and downwards from right to left). For the sake of simplicity let's call the set of these diagonals _d1_ and _d2_.

$$
\forall \text{ pair of } (i,j) \in d1, \text{then } 0 \leq \sum_{i,j} board_{i,j} \leq 1 \\
\forall \text{ pair of } (i,j) \in d2, \text{then } 0 \leq \sum_{i,j} board_{i,j} \leq 1
$$

Our code is also becomes simple to describe:

{% highlight python %}
# each row has at most 1 queen
for i in range(height):
    model.addCons(quicksum(board[i][j] for j in range(width)) <= 1)

# each column has at most 1 queen
for j in range(width):
    model.addCons(quicksum(board[i][j] for i in range(height)) <= 1)

# each diagonal has at most 1 queen
for diagonals in diagonals_top_right_to_bottom_left(board):
    model.addCons(quicksum(board[i][j] for i, j in diagonals) <= 1)

# each diagonal, considering other direction, has at most 1 queen
for diagonals in diagonals_top_left_to_bottom_right(board):
    model.addCons(quicksum(board[i][j] for i, j in diagonals) <= 1)
{% endhighlight %}

## Running our model

The problem now is completely described with all the necessary information. You can check out the whole code [here](https://github.com/fvcaputo/optimization-problems/blob/master/queens-optimality.mip.py). All that was missing is to call `solve` and then create a way to print the results, but this is not really related to the model so you can check out the code at the link to see how I did it.

When running it to, say, a board of size 10 we get:

{% highlight shell %}
SCIP Status        : problem is solved [optimal solution found]
Solving Time (sec) : 0.01
Solving Nodes      : 1
Primal Bound       : +1.00000000000000e+01 (5 solutions)
Dual Bound         : +1.00000000000000e+01
Gap                : 0.00 %
. . . . . Q . . . .
. . . Q . . . . . .
. . . . . . Q . . .
. . . . . . . . . Q
. . . . . . . Q . .
. . Q . . . . . . .
Q . . . . . . . . .
. . . . . . . . Q .
. Q . . . . . . . .
. . . . Q . . . . .
{% endhighlight %}

Awesome! we just go a result showing how the board looks, where `Q` represents the board positions where we can place the queens, and `.` are simply empty positions. You can see see that an optimal solution was found, and that there was actually 5 solutions found overall. This is not wrong, as a matter of fact there are [a lot of possible solutions to this problem](https://en.wikipedia.org/wiki/Eight_queens_puzzle#Counting_solutions), but the program took a path where it only went through a few.

We can run the problem to some bigger boards, 20, 30, 40... it will take a while but we will find a solution. In my machine a board of size 40 for instance will be solved in about 10 seconds.

But you are probably thinking about some bigger problems right? For instance, what about a board of size 100?

{% highlight shell %}
SCIP Status        : problem is solved [optimal solution found]
Solving Time (sec) : 114.91
Solving Nodes      : 99
Primal Bound       : +1.00000000000000e+02 (1497 solutions)
Dual Bound         : +1.00000000000000e+02
Gap                : 0.00 %
{% endhighlight %}

Well, that was not incredibly slow or anything, but that did take a while. Can we do better?

## Understanding your problem

By looking at the [table of possible solutions](https://en.wikipedia.org/wiki/Eight_queens_puzzle#Counting_solutions), you will see that there are a ridiculous amount of possible solutions for all sizes of _n_ in this problem. A board with size _n_ of 27 has actually 234,907,967,154,122,528 solutions! That is huge!

With an optimization problem, what the integer programming algorithm is trying to do is find the optimal solution. We said we want to maximize the number of queens in our board, so the algorithm will try to find the best possible solution. But integer programming is actually a _NP_ problem, and it can take a lot of time. Internally it will use all sorts of techniques to make the solution space smaller and then it will backtrack and try different "paths" to find the best solution. You can check SCIP features [here](http://scip.zib.de/#features) if you are interested.

But anyways, this particular problem can take a long time to be solved for larger _n_ sizes, is there anything we can do? Why, yes!

## Pure Constraint Programming Problem

The reason I choose the queens puzzle to demonstrate this programming paradigm is because we can try to solve it in a different way. Previously we were trying to maximize the number of queens we can place on the board, this is an optimization problem. But by now you might be thinking already, "wait a minute, I'm pretty sure I can only place at most _n_ queens in an _n_ by _n_ board".

And actually you are correct! [In fact, people have already formalized this before](https://en.wikipedia.org/wiki/Eight_queens_puzzle#History). Now, what we want is to know how we can place these queens on the board, so we can change our problem from an *optimization* problem to a *feasibility* problem.

To do that what we need is to remove our objective, we don't need to maximize anything anymore, and add a new constraint: The sum of all the variables has to be exactly equal to _n_. Since we need to model all constraints as inequations, we will have:

$$n \leq \sum_{i,j} board_{i,j} \leq n$$

With the library we are using we don't even need to set up much for that, we simply remove the objective code and add our new constraint:

{% highlight python %}
# Remove objective
# model.setObjective(quicksum(board[i][j] for j in range(width) for i in range(height)), "maximize")

# New constraint
# n by n board must have n queens
model.addCons(width <= (quicksum(board[i][j] for j in range(width) for i in range(height)) <= width))
{% endhighlight %}

The complete code for the feasibility problem is [here](https://github.com/fvcaputo/optimization-problems/blob/master/queens-feasibility.mip.py). Now, how about we try this new code with a board of size 100 again?

{% highlight shell %}
SCIP Status        : problem is solved [optimal solution found]
Solving Time (sec) : 6.36
Solving Nodes      : 1
Primal Bound       : +0.00000000000000e+00 (1 solutions)
Dual Bound         : +0.00000000000000e+00
Gap                : 0.00 %
{% endhighlight %}

How about that? From over 110 seconds to a little bit over 6 seconds. Incredible performance boost, just by understanding our problem a little better. Now that algorithm simply stops when it finds a solution that fits all our constraints, it is able to run much faster.

## Some final thoughts

The idea of this post was really just to present this different programming paradigm of solving optimization problems. This is a really basic intro, there are a lot of really good sources out there if you want to learn more. There are plenty of problems that can be solved with this kind of technique, from "simpler" problems like Knapsack, Sudoku, etc, to really hard problems like weather forecasting.

If you are interested in learning more one resource that I cannot recommend enough is the [Discrete Optimization](https://www.coursera.org/learn/discrete-optimization/) Coursera course created by The University of Melbourne. It's great course that goes deep in this topic.

Hopefully this will help you along the way! You never know when you will stumble on a problem that could be modeled like that and actually solved really quickly.
