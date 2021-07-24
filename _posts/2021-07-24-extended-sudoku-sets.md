---
layout: post
title: Representing extended sudoku rules with sets
categories: Sudoku
---

## What is sudoku with extended rules?

Sudoku with extended rules is just what it sounds like - it's a sudoku puzzle that has some rules added to it, making it more interesting. This allows for crazy and, at first glance, unsolvable puzzles, like [this one](https://logic-masters.de/Raetselportal/Raetsel/zeigen.php?id=00071Y): ![alt text](/images/sudoku_rules/empty_grid.png "This sudoku has a unique solution!")


## What is this post about

This post is about representing extended sudoku rules using sets. It is the first step in translating the rules to computer-understandable ones. After they are translated, I plan on using Z3 to solve these types of sudoku.

## Important notes

Throughout the text, I use sets to define various rules. The names of the sets are shown in bold, and most of them have an index sub-k since more than one instance is possible in a sudoku grid. Where functions are used (e.g. sum()), I italicize the name of the function.

I also use sudokus from [Logic Masters Germany](https://logic-masters.de/), a wonderful portal for all puzzle lovers, and [F-puzzles](https://f-puzzles.com/) to visualize the rules.

# Defining rules

## Standard

Most people know standard sudoku rules. Anyways, here they are in a set notation:

First, we must define what a sudoku grid is:

**Grid** = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9, &vert;Grid&vert; = 81}

As expected, a sudoku grid is just a set of positions, denoted x<sub>ij</sub>, with a cardinality of 81.

Next, all the positions contain digits from 1 to 9:

<code>&forall;x<sub>ij</sub> &isin; **Grid**, 1 &le; *val*(x<sub>ij</sub>) &le; 9</code>

Here, *val*(square) functions returns the value of a square.

There are also some restrictions on the grid. For one, you cannot repeat the same digits in a row or a column:

<code>&forall;x<sub>ij</sub> &isin; **Grid**, j &ne; k &rarr; *val*(x<sub>ij</sub>) &ne; *val*(x<sub>ik</sub>)</code>

<code>&forall;x<sub>ij</sub> &isin; **Grid**, j &ne; k &rarr; *val*(x<sub>ji</sub>) &ne; *val*(x<sub>ki</sub>)</code>

You also cannot repeat digits in a sudoku "box":

<code>&forall;x<sub>ij</sub>,x<sub>mn</sub> &isin; **Grid**, (i &ne; m &or; j &ne; n) &and; &LeftFloor;i/3&RightFloor; = &LeftFloor;m/3&RightFloor; &and; &LeftFloor;j/3&RightFloor; = &LeftFloor;n/3&RightFloor; &rarr; *val*(x<sub>ij</sub>) &ne; *val*(x<sub>mn</sub>)</code>

## Odd, even

Let's start with simple extended rules.

Gray circle means that the digit in it is odd. Gray square means that the digit in it is even.

![alt text](/images/sudoku_rules/odd_even.png "Odd, even rules")

The definitions are pretty easy:

<code>*val*(**Odd<sub>k</sub>**) mod 2 = 1</code>

<code>*val*(**Even<sub>k</sub>**) mod 2 = 0</code>

## Killer Cage

Killer Cages sound spooky, but in reality they are quite easy. They allow defining extra restricted regions in the grid. Here's how they look:

![alt text](/images/sudoku_rules/cages.png "Killer Cages")

And here's how they are defined:

<code>**Cage<sub>k</sub>** = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

First, digits cannot repeat within a Cage:

<code>&forall;x<sub>ij</sub>,x<sub>mn</sub> &isin; **Cage<sub>k</sub>**, (i &ne; m &or; j &ne; n) &rarr; *val*(x<sub>ij</sub>) &ne; *val*(x<sub>mn</sub>)</code>

Also, some cages have their sums shown in the top left corner:

<code>(*sum*(**Cage<sub>k</sub>**) = {i &isin; &#8469;}) &or; (*sum*(**Cage<sub>k</sub>**) = "?")</code>

The *sum*() functions returns the sum of all the elements in a set. I use "?" to signify that the sum is not known.

## Thermo

Thermometers are an interesting concept for sure. Digits along the thermometers increase starting from the bulb (the circle). That's how they look:

![alt text](/images/sudoku_rules/thermo.png "Thermometer")

And that's how they are defined:

<code>**Thermo<sub>k</sub>** = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>&forall;x<sub>ij</sub>,x<sub>mn</sub> &isin; **Thermo<sub>k</sub>**, x<sub>ij</sub> &pr; x<sub>mn</sub> &rarr; *val*(x<sub>ij</sub>) &lt; *val*(x<sub>mn</sub>)</code>

I cheat a little here: I define an ordering of the set, where the set is ordered from the bulb up.

## Arrow

Arrows is another rule that is hard to define with just sets. The rule is simple - digits on the arrow add to the one in the bulb.

![alt text](/images/sudoku_rules/arrow.png "Arrow")

<code>**Arrow<sub>k</sub>** = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>*val*(&top;) = *sum*(**Arrow<sub>k</sub>** - &top;)</code>

I cheat a little here again - I define the bulb of the arrow as the top element of the set.

## Kropki dots

Kropki dots, just like XV rules (defined later) are relations between two squares of the grid. There are two types:

White Kropki, where the difference between two squares is one:

![alt text](/images/sudoku_rules/wkropki.png "White Kropki dots")

<code>**WKropki<sub>k</sub>** =&#12296;one, two&#12297;</code>

<code>&vert;**WKropki<sub>k</sub>**.one - **WKropki<sub>k</sub>**.two&vert; = 1</code>

And Black Kropki, where one of the squares is twice the other:

![alt text](/images/sudoku_rules/bkropki.png "Black Kropki dots")

<code>**BKropki<sub>k</sub>** =&#12296;one, two&#12297;</code>

<code>(**BKropki<sub>k</sub>**.one &gt; **BKropki<sub>k</sub>**.two) &and; (**BKropki<sub>k</sub>**.one = 2(**BKropki<sub>k</sub>**.two))) &or; ((**BKropki<sub>k</sub>**.one &lt; **BKropki<sub>k</sub>**.two) &and; (2(**BKropki<sub>k</sub>**.one) = **BKropki<sub>k</sub>**.two)</code>

I define the Kropki rules using tuples to shorten the notation.

## XV Sudoku

As mentioned previously, XV rules are similar to Kropki rules. There are two different symbols:

V signifies that two squares add up to 5:

![alt text](/images/sudoku_rules/v.png "V")

<code>**V<sub>k</sub>** =&#12296;one, two&#12297;</code>

<code>**V<sub>k</sub>**.one + **V<sub>k</sub>**.two = 5</code>

X signifies that two squares add up to 10:

![alt text](/images/sudoku_rules/x.png "X")

<code>**X<sub>k</sub>** =&#12296;one, two&#12297;</code>

<code>**X<sub>k</sub>**.one + **X<sub>k</sub>**.two = 10</code>

## Renban lines

Renban lines are simple rules that are incredibly hard to define. The rule is that the digits on the line are consequtive and are in any order (e.g. you could have 1-2-3 and 3-1-2).

![alt text](/images/sudoku_rules/renban.png "Renban")

<code>**Renban<sub>k</sub>** = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>&forall;x<sub>ij</sub> &isin; **Renban<sub>k</sub>**, *val*(&top;) - &vert;**Renban<sub>k</sub>**&vert; &lt; *val*(x<sub>ij</sub>) &leq; *val*(&top;)</code>

I have to specify here that the set is ordered by ascending value. The rule above says that all the digits on the Renban line are between top (including) and top minus the line length.

## Palindrome

Palindrome rules are exacly what they sound like - squares opposite on the line are the same. The set is ordered along the line.

![alt text](/images/sudoku_rules/palindrome.png "Palindrome, squares of the same color contain the same digits")

<code>**Palindrome<sub>k</sub>** = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>&forall;x<sub>ij</sub> &isin; **Palindrome<sub>k</sub>**, *val*(x<sub>ij</sub>) = *val*(&perp; + (&top; - x<sub>ij</sub>))</code>

The arithmetic on &top; and &perp; calculates the "offset" of the set element.

## Bishop

Bishop rules are first of the Chess Sudoku rules. The digits along a bishop move from a given digit cannot be the same.

![alt text](/images/sudoku_rules/bishop.png "Bishop rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; **Grid**, 1 &le; n &le; 9, x<sub>ij</sub> &ne; x<sub>(i&plusmn;n)(j&plusmn;n)</sub></code>

A short note on notation: i use x &plusmn; y as a shorthand for (x + y) &and; (x - y). Also, it is assumed that if the offset is out of the grid, the case is not considered.

## King

King rules are analogous to other chess sudoku rules. The digits along a king move from a given digits cannot be the same.

![alt text](/images/sudoku_rules/king.png "King rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; **Grid**, x<sub>ij</sub> &ne; x<sub>(i&plusmn;1)(j&plusmn;1)</sub> &and; x<sub>ij</sub> &ne; x<sub>(i&plusmn;1)j</sub> &and; x<sub>ij</sub> &ne; x<sub>i(j&plusmn;1)</sub></code>

## Queen

Just as in chess, queen rules are like an extention of king rules with bigger offsets.

![alt text](/images/sudoku_rules/queen.png "Queen rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; **Grid**, 1 &le; n &le; 9, x<sub>ij</sub> &ne; x<sub>(i&plusmn;n)(j&plusmn;n)</sub> &and; x<sub>ij</sub> &ne; x<sub>(i&plusmn;n)j</sub> &and; x<sub>ij</sub> &ne; x<sub>i(j&plusmn;n)</sub></code>

## Knight

Again, knight sudoku rules are equivalent to their chess counterparts. Digits knight move away from each other cannot be the same.

![alt text](/images/sudoku_rules/knight.png "Knight rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; **Grid**, x<sub>ij</sub> &ne; x<sub>(i&plusmn;1)(j&plusmn;2)</sub> &and; x<sub>ij</sub> &ne; x<sub>(i&plusmn;2)(j&plusmn;1)</sub></code>

## Sandwich

The sandwich rule is a fun one. It says that in an indicated row/column, the numbers between 1 and 9 must sum to the clue. For example, in the case below, you can deduce that 8 has to go between 1 and 9:

![alt text](/images/sudoku_rules/sandwich.png "Sandwich rules")

<code>**SandRow<sub>i</sub>** &isin; &#8469;</code>

<code>*val*(x<sub>ij</sub>) = 1 &and; *val*(x<sub>ik</sub>) = 9, **SandRow<sub>i</sub>** = *sum*({x<sub>iq</sub> &vert; *min*(j, k) &lt; q &lt; *max*(j, k)})</code>

<code>**SandCol<sub>i</sub>** &isin; &#8469;</code>

<code>*val*(x<sub>ji</sub>) = 1 &and; *val*(x<sub>ki</sub>) = 9, **SandCol<sub>i</sub>** = *sum*({x<sub>qi</sub> &vert; *min*(j, k) &lt; q &lt; *max*(j, k)})</code>

## Killer Sum

Killer sums indicate that, on the marked diagonal, the squares sum up to the given clue. 

![alt text](/images/sudoku_rules/killer.png "Killer Sum rule, highlited squares sum up to the clue")

It is a bit hard to define in terms of sets, so here is my definition.

First, a Killer Sum is a 3-tuple that contains the sum itself, the starting square and the end square.

<code>**Killer<sub>k</sub>** =&#12296;sum, x<sub>ij</sub>, x<sub>mn</sub>&#12297;</code>

Next, there are four possible cases:

<code>(i &lt; m &and; j &lt; n &rarr; *sum*({x<sub>(i+l)(j+l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = **Killer<sub>k</sub>**.sum) &or;</code>

<code>(i &lt; m &and; j &gt; n &rarr; *sum*({x<sub>(i+l)(j-l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = **Killer<sub>k</sub>**.sum) &or;</code>

<code>(i &gt; m &and; j &lt; n &rarr; *sum*({x<sub>(i-l)(j+l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = **Killer<sub>k</sub>**.sum) &or;</code>

<code>(i &gt; m &and; j &gt; n &rarr; *sum*({x<sub>(i-l)(j-l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = **Killer<sub>k</sub>**.sum)</code>

# Foreword

As you can see, extensions to sudoku rules are not hard to implement. There are quite some ways to go from sets to Z3 rules, however. I plan on doing a full Python implementation of the above rules, which I will link to later on.

I would also like to thank Mark and Simon from Cracking the Cryptic YouTube channel for introducing me to the world of extended sudoku.