---
title: "Probabilities and Packet Classification I"
date: 2020-03-22T15:24:54+08:00
draft: true
---

I wrote this blog in English just because I want to feel like I was writing a thesis. But this is not a thesis or a academic paper, it is just an essay.

For years, I have been trying to understand the enssential nature of the packet classification. This is all due to the my research experience on these algorithms at my Ph.D undergraduate. During that period, I have implemented almost all the existing Decision-Tree based agorithms, and had my own obervations and invented my own algorithm. I was believing that my algorithm can beat these existing ones, until I met the algorithm, **HyperSplit**. It's so simple, but it is so fast, that I found my invented one is bullshit as well as the other algorithms. It's always 10 times memory utilization smaller and 2x faster, which makes me think there should be some unrevealed nature inside this problem.

The original HyperSplit only describe the method to construct the tree, but does not explain why. Why construct the tree using this method? As a paper, it has no obligation to record how these ideas originate, but this causes a lot of confusion in the area of the packet classification research. Almost of all papers in this area, except the first few ones, have no logics and describe everything like woodoo: "you just do this and this, and tada, the tree is there and it's super good".

I was trying to form a theoritical foundation to understand this, but I donot have too much progress back at my Ph.D. undergraduate. I have some results but sadly I have no ideas how to publish this ones. They are just every preliminary. However this kind of research becomes one of my habits during my spare time.

I will try to write down my results there and if lucky, it might become a paper in the future. At that time, my co-authors can read these blog to understand what I found. If not lucky, it's just a record of my time spending on reveal of the myth in an old and perhaps nobody-care research area.



