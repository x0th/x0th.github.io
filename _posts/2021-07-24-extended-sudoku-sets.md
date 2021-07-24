---
layout: post
title: Representing extended sudoku rules with sets
categories: Sudoku
---

<style>
	code {
		background-color:#282A36;
		padding: 10px;
	}
	sub {
		vertical-align: sub;
		font-size: medium;
	}
</style>

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

<code><b>Grid</b> = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9, &vert;Grid&vert; = 81}</code>

As expected, a sudoku grid is just a set of positions, denoted x<sub>ij</sub>, with a cardinality of 81.

Next, all the positions contain digits from 1 to 9:

<code>&forall;x<sub>ij</sub> &isin; <b>Grid</b>, 1 &le; <i>val</i>(x<sub>ij</sub>) &le; 9</code>

Here, <i>val</i>(square) functions returns the value of a square.

There are also some restrictions on the grid. For one, you cannot repeat the same digits in a row or a column:

<code>&forall;x<sub>ij</sub> &isin; <b>Grid</b>, j &ne; k &rarr; <i>val</i>(x<sub>ij</sub>) &ne; <i>val</i>(x<sub>ik</sub>)</code>

<code>&forall;x<sub>ij</sub> &isin; <b>Grid</b>, j &ne; k &rarr; <i>val</i>(x<sub>ji</sub>) &ne; <i>val</i>(x<sub>ki</sub>)</code>

You also cannot repeat digits in a sudoku "box":

<code>&forall;x<sub>ij</sub>,x<sub>mn</sub> &isin; <b>Grid</b>, (i &ne; m &or; j &ne; n) &and; &LeftFloor;i/3&RightFloor; = &LeftFloor;m/3&RightFloor; &and; &LeftFloor;j/3&RightFloor; = &LeftFloor;n/3&RightFloor; &rarr; <i>val</i>(x<sub>ij</sub>) &ne; <i>val</i>(x<sub>mn</sub>)</code>

## Odd, even

Let's start with simple extended rules.

Gray circle means that the digit in it is odd. Gray square means that the digit in it is even.

![alt text](/images/sudoku_rules/odd_even.png "Odd, even rules")

The definitions are pretty easy:

<code><i>val</i>(<b>Odd<sub>k</sub></b>) mod 2 = 1</code>

<code><i>val</i>(<b>Even<sub>k</sub></b>) mod 2 = 0</code>

## Killer Cage

Killer Cages sound spooky, but in reality they are quite easy. They allow defining extra restricted regions in the grid. Here's how they look:

![alt text](/images/sudoku_rules/cages.png "Killer Cages")

And here's how they are defined:

<code><b>Cage<sub>k</sub></b> = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

First, digits cannot repeat within a Cage:

<code>&forall;x<sub>ij</sub>,x<sub>mn</sub> &isin; <b>Cage<sub>k</sub></b>, (i &ne; m &or; j &ne; n) &rarr; <i>val</i>(x<sub>ij</sub>) &ne; <i>val</i>(x<sub>mn</sub>)</code>

Also, some cages have their sums shown in the top left corner:

<code>(<i>sum</i>(<b>Cage<sub>k</sub></b>) = {i &isin; &#8469;}) &or; (<i>sum</i>(<b>Cage<sub>k</sub></b>) = "?")</code>

The <i>sum</i>() functions returns the sum of all the elements in a set. I use "?" to signify that the sum is not known.

## Thermo

Thermometers are an interesting concept for sure. Digits along the thermometers increase starting from the bulb (the circle). That's how they look:

![alt text](/images/sudoku_rules/thermo.png "Thermometer")

And that's how they are defined:

<code><b>Thermo<sub>k</sub></b> = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>&forall;x<sub>ij</sub>,x<sub>mn</sub> &isin; <b>Thermo<sub>k</sub></b>, x<sub>ij</sub> &pr; x<sub>mn</sub> &rarr; <i>val</i>(x<sub>ij</sub>) &lt; <i>val</i>(x<sub>mn</sub>)</code>

I cheat a little here: I define an ordering of the set, where the set is ordered from the bulb up.

## Arrow

Arrows is another rule that is hard to define with just sets. The rule is simple - digits on the arrow add to the one in the bulb.

![alt text](/images/sudoku_rules/arrow.png "Arrow")

<code><b>Arrow<sub>k</sub></b> = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code><i>val</i>(&top;) = <i>sum</i>(<b>Arrow<sub>k</sub></b> - &top;)</code>

I cheat a little here again - I define the bulb of the arrow as the top element of the set.

## Kropki dots

Kropki dots, just like XV rules (defined later) are relations between two squares of the grid. There are two types:

White Kropki, where the difference between two squares is one:

![alt text](/images/sudoku_rules/wkropki.png "White Kropki dots")

<code><b>WKropki<sub>k</sub></b> =&#12296;one, two&#12297;</code>

<code>&vert;<b>WKropki<sub>k</sub></b>.one - <b>WKropki<sub>k</sub></b>.two&vert; = 1</code>

And Black Kropki, where one of the squares is twice the other:

![alt text](/images/sudoku_rules/bkropki.png "Black Kropki dots")

<code><b>BKropki<sub>k</sub></b> =&#12296;one, two&#12297;</code>

<code>(<b>BKropki<sub>k</sub></b>.one &gt; <b>BKropki<sub>k</sub></b>.two) &and; (<b>BKropki<sub>k</sub></b>.one = 2(<b>BKropki<sub>k</sub></b>.two))) &or; ((<b>BKropki<sub>k</sub></b>.one &lt; <b>BKropki<sub>k</sub></b>.two) &and; (2(<b>BKropki<sub>k</sub></b>.one) = <b>BKropki<sub>k</sub></b>.two)</code>

I define the Kropki rules using tuples to shorten the notation.

## XV Sudoku

As mentioned previously, XV rules are similar to Kropki rules. There are two different symbols:

V signifies that two squares add up to 5:

![alt text](/images/sudoku_rules/v.png "V")

<code><b>V<sub>k</sub></b> =&#12296;one, two&#12297;</code>

<code><b>V<sub>k</sub></b>.one + <b>V<sub>k</sub></b>.two = 5</code>

X signifies that two squares add up to 10:

![alt text](/images/sudoku_rules/x.png "X")

<code><b>X<sub>k</sub></b> =&#12296;one, two&#12297;</code>

<code><b>X<sub>k</sub></b>.one + <b>X<sub>k</sub></b>.two = 10</code>

## Renban lines

Renban lines are simple rules that are incredibly hard to define. The rule is that the digits on the line are consequtive and are in any order (e.g. you could have 1-2-3 and 3-1-2).

![alt text](/images/sudoku_rules/renban.png "Renban")

<code><b>Renban<sub>k</sub></b> = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>&forall;x<sub>ij</sub> &isin; <b>Renban<sub>k</sub></b>, <i>val</i>(&top;) - &vert;<b>Renban<sub>k</sub></b>&vert; &lt; <i>val</i>(x<sub>ij</sub>) &leq; <i>val</i>(&top;)</code>

I have to specify here that the set is ordered by ascending value. The rule above says that all the digits on the Renban line are between top (including) and top minus the line length.

## Palindrome

Palindrome rules are exacly what they sound like - squares opposite on the line are the same. The set is ordered along the line.

![alt text](/images/sudoku_rules/palindrome.png "Palindrome, squares of the same color contain the same digits")

<code><b>Palindrome<sub>k</sub></b> = {x<sub>ij</sub> &vert; 1 &le; i,j &le; 9}</code>

<code>&forall;x<sub>ij</sub> &isin; <b>Palindrome<sub>k</sub></b>, <i>val</i>(x<sub>ij</sub>) = <i>val</i>(&perp; + (&top; - x<sub>ij</sub>))</code>

The arithmetic on &top; and &perp; calculates the "offset" of the set element.

## Bishop

Bishop rules are first of the Chess Sudoku rules. The digits along a bishop move from a given digit cannot be the same.

![alt text](/images/sudoku_rules/bishop.png "Bishop rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; <b>Grid</b>, 1 &le; n &le; 9, x<sub>ij</sub> &ne; x<sub>(i&plusmn;n)(j&plusmn;n)</sub></code>

A short note on notation: i use x &plusmn; y as a shorthand for (x + y) &and; (x - y). Also, it is assumed that if the offset is out of the grid, the case is not considered.

## King

King rules are analogous to other chess sudoku rules. The digits along a king move from a given digits cannot be the same.

![alt text](/images/sudoku_rules/king.png "King rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; <b>Grid</b>, x<sub>ij</sub> &ne; x<sub>(i&plusmn;1)(j&plusmn;1)</sub> &and; x<sub>ij</sub> &ne; x<sub>(i&plusmn;1)j</sub> &and; x<sub>ij</sub> &ne; x<sub>i(j&plusmn;1)</sub></code>

## Queen

Just as in chess, queen rules are like an extention of king rules with bigger offsets.

![alt text](/images/sudoku_rules/queen.png "Queen rules, highlited squares cannot be 2")

<code">&forall;x<sub>ij</sub> &isin; <b>Grid</b>, 1 &le; n &le; 9, x<sub>ij</sub> &ne; x<sub>(i&plusmn;n)(j&plusmn;n)</sub> &and; x<sub>ij</sub> &ne; x<sub>(i&plusmn;n)j</sub> &and; x<sub>ij</sub> &ne; x<sub>i(j&plusmn;n)</sub></code>

## Knight

Again, knight sudoku rules are equivalent to their chess counterparts. Digits knight move away from each other cannot be the same.

![alt text](/images/sudoku_rules/knight.png "Knight rules, highlited squares cannot be 2")

<code>&forall;x<sub>ij</sub> &isin; <b>Grid</b>, x<sub>ij</sub> &ne; x<sub>(i&plusmn;1)(j&plusmn;2)</sub> &and; x<sub>ij</sub> &ne; x<sub>(i&plusmn;2)(j&plusmn;1)</sub></code>

## Sandwich

The sandwich rule is a fun one. It says that in an indicated row/column, the numbers between 1 and 9 must sum to the clue. For example, in the case below, you can deduce that 8 has to go between 1 and 9:

![alt text](/images/sudoku_rules/sandwich.png "Sandwich rules")

<code><b>SandRow<sub>i</sub></b> &isin; &#8469;</code>

<code><i>val</i>(x<sub>ij</sub>) = 1 &and; <i>val</i>(x<sub>ik</sub>) = 9, <b>SandRow<sub>i</sub></b> = <i>sum</i>({x<sub>iq</sub> &vert; <i>min</i>(j, k) &lt; q &lt; <i>max</i>(j, k)})</code>

<code><b>SandCol<sub>i</sub></b> &isin; &#8469;</code>

<code><i>val</i>(x<sub>ji</sub>) = 1 &and; <i>val</i>(x<sub>ki</sub>) = 9, <b>SandCol<sub>i</sub></b> = <i>sum</i>({x<sub>qi</sub> &vert; <i>min</i>(j, k) &lt; q &lt; <i>max</i>(j, k)})</code>

## Killer Sum

Killer sums indicate that, on the marked diagonal, the squares sum up to the given clue. 

![alt text](/images/sudoku_rules/killer.png "Killer Sum rule, highlited squares sum up to the clue")

It is a bit hard to define in terms of sets, so here is my definition.

First, a Killer Sum is a 3-tuple that contains the sum itself, the starting square and the end square.

<code><b>Killer<sub>k</sub></b> =&#12296;sum, x<sub>ij</sub>, x<sub>mn</sub>&#12297;</code>

Next, there are four possible cases:

<code>(i &lt; m &and; j &lt; n &rarr; <i>sum</i>({x<sub>(i+l)(j+l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = <b>Killer<sub>k</sub></b>.sum) &or;</code>

<code>(i &lt; m &and; j &gt; n &rarr; <i>sum</i>({x<sub>(i+l)(j-l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = <b>Killer<sub>k</sub></b>.sum) &or;</code>

<code>(i &gt; m &and; j &lt; n &rarr; <i>sum</i>({x<sub>(i-l)(j+l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = <b>Killer<sub>k</sub></b>.sum) &or;</code>

<code>(i &gt; m &and; j &gt; n &rarr; <i>sum</i>({x<sub>(i-l)(j-l)</sub> &vert; 0 &le; l &le; &vert;i - m&vert;}) = <b>Killer<sub>k</sub></b>.sum)</code>

# Foreword

As you can see, extensions to sudoku rules are not hard to implement. There are quite some ways to go from sets to Z3 rules, however. I plan on doing a full Python implementation of the above rules, which I will link to later on.

I would also like to thank Mark and Simon from Cracking the Cryptic YouTube channel for introducing me to the world of extended sudoku.