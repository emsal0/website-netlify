---
title: "Logs 2020-05-11: Sequences of Sets"
date: 2020-05-11T17:49:22-07:00
draft: false
---

Here's a quick little math one.

In a probability theory class that I was sitting in on, one of the core concepts taught was the _limit infimum_ and _limit supremum_ of a sequence of sets. If \\( A_n \\) is a sequence of sets (that are all subsets of some larger set \\( E \\)) then the two constructs are also known as \\( A_n \\) _almost always_ and \\( A_n \\) _infinitely often_ respectively. They are formally defined as follows:

$$
    \liminf\limits_{n \rightarrow \infty} = A_n \text{ a.a.} = \bigcup_{n=1}^{\infty} \bigcap_{k=n}^{\infty} A_k
$$

$$
    \limsup\limits_{n \rightarrow \infty} = A_n \text{ i.o.} = \bigcap_{n=1}^{\infty} \bigcup_{k=n}^{\infty} A_k
$$

When I saw these for the first time it was tough for me to connect the semantics of the concepts to the unions and intersections of the formal definition. I'll try to impart a bit of intuition now that I think I've gotten a better grasp of it.

* For a.a.: first, focus on each inner intersection, \\( \bigcap_{k=n}^{\infty} A_k \\). Hell, we can even give each of these a name, say, \\( B_n \\). So, we have, for all elements in \\(e \in E \\), that \\(\exists n, e \in \bigcap_{k=n}^{\infty} A_n \implies e \in A_n \text{ a.a.} \\) That is, if \\( e \\) is in any \\( B_n \\) then it's in \\(A_n \text{ a.a} \\).

    Now, we can see here that due to how we know intersection works, once \\( e \\) is in \\( B_n \\) then it's definitely in every \\( B_m \\) for \\( m \ge n \\). The key thing to consider here for intuition are each of the cases when an element \\( e \in B_m \\) but \\( e \notin B_{m-1} \\). In this case, we know that \\( e \\) is in every \\( A_k, \forall k \ge m \\) but is not in \\( A_{m-1} \\). In fact, \\( A_n \text{ a.a.} \\) is just a collection of all of the elements for which this happens at some point.

    The most memorable intuition for me is that \\( A_n \text{ a.a.} \\) is the set of all elements in \\( E \\) that appear in all but finitely many of the \\( A_n \\).

* For i.o.: I like to see this as the union of all \\( A_n \\) with things sheared off from it as the outer intersection works its course. This outer intersection goes to infinity; all elements in \\( A_n \text{ i.o.} \\) are in every single union \\( \bigcup_{k=n}^{\infty} A_k, \forall n \\) (in the same vein as the above call these \\( C_n \\)). As we're thinking of things being sheared off, the intuition here is to think of the things that are absent from the set: if, at any point, an element \\( e \\) ceases to be present in a \\( C_n \\), then it cannot be in the whole intersection (by the way that our unions work, if \\(e \notin C_n \\) then \\(e \notin C_m \forall m \ge n \\)). What worked for me at this point was to think of \\( A_n \text{ i.o.} \\) as all elements \\( e \\) for which, \\( \forall N \in \mathbb{N}, \exists n \ge N, e \\in A_n \\) &mdash; if you "look past" any particular \\( N \\), one of the sets that exist past that point is bound to contain the element \\( e \\).

* Analogous to how the limsup is greater than the liminf for a sequence of numbers, \\( A_n \text{ i.o.} \supseteq A_n \text{ a.a.} \\).

<!--more-->

Hopefully this was informative! I originally tried to make a visual explanation but it got messy and sort of failed:


![Not sure what this is trying to accomplish...](/images/io_and_aa.png)


