---
layout: post
title: 'How Iterative Deepening Search Combines the Best of DFS and BFS'
date: '2021-07-25 18:05:00 +1000'
categories: artificial-intelligence
---

Breadth-first search(BFS) and Depth-first search(DFS) are the most basic uninformed search strategies used in A.I. In this article, we are going to look at how iterative deepening search combines the best of Breadth-first search(BFS) and Depth-first search(DFS). A basic understanding of graph theory in context of A.I. and "Big O Notation" is necessary to understand the article. We are going to compare the algorithms based on the following criterias:

1. **Completeness:** Is the algorithm guaranteed to find a solution if there is one?
2. **Optimality:** Is the algorithm guaranteed to find a solution which has the lowest path cost?
3. **Time Complexity:** Time required to find a solution. It's usually measured in terms of the number of nodes generated and is represented with the [Big O Notation](https://en.wikipedia.org/wiki/Big_O_notation).
4. **Space Complexity:** Memory required to find a solution. It's usually measure in terms of the maximum number of nodes stored in the memory and is represented with the [Big O Notation](https://en.wikipedia.org/wiki/Big_O_notation).


## Breadth-first search(BFS)
![Diagram of a BFS tree](https://drive.google.com/uc?export=view&id=1QG0byGiB4AwWzQFcRibwsdsZkSaexMqm)
<p style="text-align: center;"><a href="https://commons.wikimedia.org/wiki/File:Breadth-first-tree.svg">Alexander Drichel</a>, <a href="https://creativecommons.org/licenses/by/3.0">CC BY 3.0</a>, via Wikimedia Commons</p>

As the name suggests, BFS strategy expands the root node first, then all of its successors are expanded next and so on. If you look at the image, the numbers inside the nodes represent the order in which the nodes are expanded. In simple words, all the nodes at any given level are expanded first before any node at the next level is expanded.

### Pros of using BFS strategy
1. The BFS strategy is *complete* provided the branching factor *b*(number of children at each node) is finite. It means that it is guaranteed to find the goal node at depth *d*, as it would reach the goal node eventually after expanding all the shallower nodes.
2. The BFS strategy is *optimal* provided the path cost is a nondecreasing function of the depth of the node(for example when all the edges have the same cost)

### Cons of using BFS strategy
**BFS strategy has huge memory requirements.** To give you an example, consider a state space where every state has *b* successors. The search begins from the root node which generates *b* successors at the first node, then those *b* nodes generate $b^2$ nodes a the second level and so on.
Now consider that the goal state is at level *d*, then the total number of nodes generated would be 

$b + b^2 \ + \ . . \ + \ b^d + (b^{d+1} - b) = O(b^{d+1})$

You can clearly see that the number of nodes and space requirement grows exponentially with depth. And you have to keep all those nodes in the memory as each of them is either a fringe(nodes that have been generated but not been expanded) or is an ancestor of a fringe node.

## Depth-first search(DFS)
![Diagram of a BFS tree](https://drive.google.com/uc?export=view&id=1Zrj8o9Oj3iu2eVeKfV5CO6Bg9IzEQ_wn)
<p style="text-align: center;"><a href="https://commons.wikimedia.org/wiki/File:Depth-first-tree.svg">Alexander Drichel</a>, <a href="https://creativecommons.org/licenses/by-sa/3.0">CC BY-SA 3.0</a>, via Wikimedia Commons</p>

DFS strategy starts at the root node and traverses a branch till it reaches the deepest node. If you refer to the image, you can see how only one node’s branch is traversed first before any of its siblings are expanded. After a deepest node with no successor in a path is reached, the search “backs up” and expands the next successor node.

### Pros of using DFS
**DFS doesn’t have high memory requirement.** It needs to store only the path from the root node to a leaf node along with the unexpanded sibling nodes for each node on the path. As soon as all of the descendants of a node are explored, it can be dropped from the memory.

As *b* nodes are generated at every level, the maximum nodes in the memory in a worst case scenario would be *bm + 1*, where m is the maximum depth(Note that *m* could be greater than *d*). Hence the space requirement is bounded by $O(bm)$

### Cons of using DFS
1. DFS can get stuck in the wrong branch even when a different choice of branch could have lead to a solution near the root of the search tree. If the chosen branch is of unbounded depth, DFS would never terminate and hence **its not complete.**
2. **DFS is not optimal** since you can never know if there was any other solution at a shallower level than the found solution(Since DFS doesn’t look at sibling nodes at same level)

## Iterative deepening DFS
![Diagram of an iterative deepening DFS tree](https://drive.google.com/uc?export=view&id=1G6K6hWYzMViGSvCWLk5g48lQCBEiz7MT)

Iterative deepening DFS in simple words is running DFS by gradually increasing the depth limit starting from 0, then repeating DFS from the beginning with depth limit 1, then again repeating DFS from the beginning with depth level 2, and so on, till a solution is found. So how does this combine benefits of DFS and BFS?

1. **It has modest memory requirements like DFS i.e. $O(bm)$.** I think this is pretty straight forward to understand. Since its running a DFS (although by increasing the depth level with every iteration), its memory requirements would be same as DFS.
2. **It is optimal provided the path cost is a nondecreasing function of the depth of the node, like BFS.** Since you are repeating the DFS strategy by increasing depth limit with every iteration, you can be sure that you traversed the siblings of your ancestors in previous iterations which in turn guarantees that no solution state existed on a shallower level.
3. **It is complete like BFS provided the branching factor is finite.** Since its traversing the breadth as well - as you are repeating DFS with increasing depth levels - it cannot get stuck on an infinite path.

Here is a table which gives an objective comparison:

| Criterion | BFS              | DFS     | Iterative deepening DFS
|-----------|------------------|---------|------------------------
| Time      | $O(b^{d+1})$     | $O(bm)$ | $O(b^d)$
| Space     | $O(b^{d+1})$     | $O(bm)$ | $O(bd)$
| Optimal?  | Yes<sup>\*</sup> | No      | Yes<sup>\*</sup>
| Complete? | Yes<sup>+</sup>  | No      | Yes<sup>+</sup>

\* optimal if step costs are identical

\+ complete if *b* is finite

## Conclusion

Iterative deepening DFS combines the memory efficiency of DFS and the completeness and optimality of BFS. You might think that IDS is inefficient since it expands many nodes multiple times in every iteration but that's not the case since the nodes that would be generated the most number of times would be the nodes at the top. And since less number of nodes are generated at the top, it wouldn’t have a significant impact.

## Bibliography
S. Russell and P. Norvig, *Artificial Intelligence: A Modern Approach*, 2nd ed. Prentice Hall, 2003.