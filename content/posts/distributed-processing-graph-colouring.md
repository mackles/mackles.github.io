---
title: "Distributed Processing: Graph Colouring"
description: "A detour from data indexing into mathematics"
date: 2021-02-21T02:57:19Z
---

# Introduction

Last year I was presented with a rather interesting issue which gave me an opportunity to use some latent mathematics knowledge. This post covers the simplest form of the problem where some of the complexity is removed via constraints. The problem revolves around efficiently processing a data model definiton and can be summarised as follows: 

*Given an entity-relationship(ER) model with only many-to-many relationships and multiple isolated workers, with each worker having the following properties:*  
*- it can only process a single entity and the associated relationships concurrently*  
*- it will fail if it processes a relationship at the same time as another worker*  
*- each entity and its set of relationships take the same time to process.*

*How many workers do we start and what entities does each worker process?*

This problem is a perfect example of why I love maths and computer science - what is a complex distributed processing problem constrained by software limitations can be distilled down to a simple problem a non-technical individual can understand. 

# Simple Example 

Consider this simple example of an entity-relationship model, how would we distribute these entities to multiple workers?  

{{< graph basic `Tip: Select any of the entities and the diagram will highlight which relationships are also processed.`>}}
    var nodes = new vis.DataSet([
        {id: 1, label: 'A'},
        {id: 2, label: 'B'},
        {id: 3, label: 'C'},
        {id: 4, label: 'D'},
    ]);
    var edges = new vis.DataSet([
        {from: 1, to: 2},
        {from: 1, to: 3},
        {from: 4, to: 2},
        {from: 4, to: 3}
    ]);
{{< / graph >}}


Clearly if we select two entities that share a relationship the workers will fail. Therefore, we must choose opposing entities, in this case we select the two pairs (A,D) and (B,C).

We also cannot process the pairs together, we must first process one pair and then process the other pair.

We arrive at the final solution where we have two workers instructed as follows:  
Worker 1 - [ A, B ]  
Worker 2 - [ D, C ]

In my view it is important to explore simple examples when exploring an issue of reasonable complexity. When exploring complex problems often there are multiple insights to be gained and viewing these insights in isolation with simple examples helps to build intuition and understanding.

# Adjacent Entities 

In graph theory, two vertices that share an edge are defined to be adjacent. Our constraint is now that any set of entities we processes at the same time cannot contain any adjacent entities.

Let's assign a colour to each group of entities that are processed, e.g first group processed is blue, second group is red and so on. We then reframe our constraint to be that if we colour all the vertices on the graph, no adjacent vertices can share the same colour (if they were they'd be processed at the same time). To minimize the number of iterations (i.e process the entities faster) we must minimize the number of colours. This is a special case of graph labeling, a topic within graph theory, known as Graph colouring.

# Graph Colouring

Graph colouring is a [famous problem](https://en.wikipedia.org/wiki/Graph_coloring#History) within maths and computer science. It can be explained quite simply as, given a graph (here's one for example):

{{< graph colouring >}}
    var nodes = new vis.DataSet([
        {id: 1, label: 'A'},
        {id: 2, label: 'B'},
        {id: 3, label: 'C'},
        {id: 4, label: 'D'},
        {id: 5, label: 'E'},
        {id: 6, label: 'F'},
        {id: 7, label: 'G'},
        {id: 8, label: 'H'},
        {id: 9, label: 'I'},
        {id: 10, label: 'J'},
    ]);
    var edges = new vis.DataSet([
        {from: 1, to: 2},
        {from: 1, to: 4},
        {from: 1, to: 3},
        {from: 4, to: 2},
        {from: 4, to: 3},
        {from: 5, to: 4},
        {from: 6, to: 5},
        {from: 7, to: 6},
        {from: 7, to: 8},
        {from: 5, to: 8},
        {from: 9, to: 8},
        {from: 5, to: 10},
    ]);
{{< / graph >}}

How many crayons do you need such that no circles that are connected have the same colour?

We then can reframe our initial problem to be:  
*Given a graph, what is the minimum set of colours I need to colour each node such that two adjacent nodes do not share the same colour?*

Generalised solutions to it are not polynomial in execution time, and those that are polynomial in execution time constrain the input graphs they apply to.  

# Greed is good

Let's implement a basic algorithm to start assigning colours to our vertices. One of the simplest is the [greedy algorithm](https://en.wikipedia.org/wiki/Greedy_coloring)). The steps of the greedy algorithm are as follows:

- Iterate over every vertex  
	-  For the current vertex get the set of colours used for its neighbours  
	-  Find the first colour that in our list of colours that is not used by any neighbours and assign it to the current vertex

In python code this would look something like (we use integers starting at 0 to denote colours):
{{< highlight python >}}
def colour(graph):
    colours = {}
    for vertex in graph.keys():
        # Get the colours of the neighbours
        used_colours = set(
            [colours[neighbour] for neighbour in graph[vertex] if neighbour in colours]
        )

        colour = 0
        for used_colour in used_colours:
            # Find the first available colour that hasn't been used
            if colour != used_colour:
                break
            colour = colour + 1
        colours[vertex] = colour
    return colours
{{< / highlight >}}

Greedy algorithms are locally optimised, which means that every step of the algorithm will only use local values (in this case only adjacent nodes to the current vertex). The drawbacks of this algorithm are that the solution depends on the order of the input vertices and that we will receive a sub-optimal solution as the algorithm won't (by definition) necessarily identify global optimizations.

The simple way to mitigate this is to run the algorithm multiple times with a random sort being applied to each iteration and select the result with the fewest colours. Given the dataset is metadata (i.e data types and relationships) scaling issues are not a concern.


# Solution

We've identified a simple approach to solving this problem, what remains is to implement the complete algorithm. 

{{< highlight python >}}

import sys
import random

def colour(graph, order):
  colours = {}
  max_colour = 0
  for vertex in order:
    # Get the colours of the neighbours
    used_colours = set(
        [colours[neighbour] for neighbour in graph[vertex] if neighbour in colours]
    )

    colour = 0
    for used_colour in used_colours:
        # Find the first available colour that hasn't been used
        if colour != used_colour:
            break
        colour = colour + 1
        if colour > max_colour:
            max_colour = colour
    colours[vertex] = colour
  return colours, max_colour

# Graph from second example
graph = {
  "A": ["B", "C", "D"],  
  "B": ["A", "D"],  
  "C": ["A", "D"],  
  "D": ["A", "B", "C", "E"],    
  "E": ["D","F", "J","H"],  
  "F": ["E", "G"],   
  "G": ["F","H"],  
  "H": ["E","G","I"],  
  "I": ["H"], 
  "J": ["E"]
}

number_iterations = 1000
vertices= list(graph.keys())
best_chromatic_number = sys.maxsize
best_colours = {}

for i in range(number_iterations):
  random.shuffle(vertices)
  vertex_colours, max_colour = colour(graph, vertices)
  # Colours are 0 indexed, so the number of colours is max colour + 1
  if max_colour + 1 < best_chromatic_number:
      best_colours = vertex_colours
      best_chromatic_number = max_colour + 1


print("Chromatic number:" + str(best_chromatic_number))
print("Vertex colours:" + str(best_colours))

{{< / highlight >}}

Running this gives us:
{{< highlight plaintext >}}
Chromatic number: 3
Vertex colours: {
    'A': 2, 
    'B': 1, 
    'C': 1
    'D': 0, 
    'E': 1, 
    'F': 2, 
    'G': 0, 
    'H': 2, 
    'I': 0, 
    'J': 0, 
}
{{< / highlight >}}


In this case the best_chromatic_number is number of iterations the best solution will require to complete.




# Limitations

This approach to this problem does come with some caveats. We could construct a data model that only uses two colours like a line of vertices or a [star graph](https://en.wikipedia.org/wiki/Star_(graph_theory)), which this algorithm will work well on. Even with thousands of entities they will still optimally use two colours (and we will find a solution close to that with our greedy algorithm). The worst case is a complete graph, which is defined as a graph where all vertices are adjacent which means every vertex would need a different colour, which means the trivial solution (process one entity at a time) is the only solution. 

Most data models will lie somewhere inbetween this, the number of colours (or groups to process) will decrease with increased similarity to a graph that only requires two colours (namely bipartite graphs).

# Conclusion

This solution lays out a general path for optimizing this set of problems (not just distributed processing). There are further optimizations that can be made, either by constraining the input graph or by providing insight to the greedy algorithm (e.g if we have a lot of vertices with only one edge, we should make sure these are coloured the same). 

If we can instruct the workers to only process a subset of the relationships then the initial problem changes entirely, which may be the topic of a future blog post.

I hope you found this interesting, and it has piqued your interest in graph theory and its application to distributed systems.

# References
[Graph colouring algorithms - Thore Husfeldt](https://thorehusfeldt.files.wordpress.com/2010/08/gca.pdf)


