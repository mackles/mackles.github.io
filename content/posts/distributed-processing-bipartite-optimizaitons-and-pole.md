---
title: "Distributed Processing: Bipartite Optimizations and POLE"
date: 2021-02-25T19:59:58Z
description: "Now we're cooking with gas"
---

# Introduction

On the first post on graph colouring I touched on bipartite graphs and how we can efficiently colour them with only two colours. This will underpin one of the optimizations we can make on the solution, which currently is only an implementation of the greedy algorithm. 

If you haven't read the first part I'd recommend doing so [here](/posts/distributed-processing-graph-colouring/).

# Bipartite Graphs

[Bipartite graphs](https://en.wikipedia.org/wiki/Bipartite_graph) are a subset of graphs where all vertices can be split into one of two subgraphs such that no two vertices in the same subgraph share an edge. If we want to colour a bipartite graph, we can just assign one colour to each of these subgraphs as, by definition, they will contain no adjacent entities. 

An example of a bipartite graph is a wheel graph:

{{< graph wheel>}}
    var nodes = new vis.DataSet([
        {id: 1, label: 'A', color: { border: 'red' }},
        {id: 2, label: 'B', color: { border: 'blue'}},
        {id: 3, label: 'C', color: { border: 'blue'}},
        {id: 4, label: 'D', color: { border: 'blue'}},
        {id: 5, label: 'E', color: { border: 'blue'}},
        {id: 6, label: 'F', color: { border: 'red'}},
        {id: 7, label: 'G', color: { border: 'red'}},
        {id: 8, label: 'H', color: { border: 'red'}},
        {id: 9, label: 'I', color: { border: 'red'}},
    ]);
    var edges = new vis.DataSet([
        {from: 1, to: 2, color: 'black'},
        {from: 1, to: 3, color: 'black'},
        {from: 1, to: 4, color: 'black'},
        {from: 1, to: 5, color: 'black'},
        {from: 3, to: 6, color: 'black'},
        {from: 4, to: 7, color: 'black'},
        {from: 5, to: 8, color: 'black'},
        {from: 2, to: 9, color: 'black'},
    ]);
{{< / graph >}}

In this example, we alternate which colour we assign as we traverse the graph outwards. If we assign the centre vertex the colour red, then the first vertexes on each spoke are blue, then back to red for the second and so on. The two subgraphs which form the bipartite graph are the same two sets of vertices which are coloured blue and red. 

# POLE Model

When I was first thinking about the problem defined in the first blog posts, the first application that came to mind was a Person, Object, Location, Event (POLE) model.

POLE Models are a common type of data model used in Fraud & other investigative use cases. The main properties of a POLE model are as follows:  
* the person, object, location entities are not directly linked, instead are linked through event entities,
* events can be linked to other events,
* analysis records are linked by a one-many relationship from a POLE entity.

A very basic pole model could be something like:

{{< graph pole>}}
    var nodes = new vis.DataSet([
        {id: 1, label: 'Person'},
        {id: 2, label: 'Object'},
        {id: 3, label: 'Location'},
        {id: 4, label: 'Event One'},
        {id: 5, label: 'Event Two'},
        {id: 6, label: 'Event Three'},
        {id: 7, label: 'Person Analysis'},
    ]);
    var edges = new vis.DataSet([
        {from: 1, to: 4},
        {from: 1, to: 5},
        {from: 1, to: 6},
        {from: 2, to: 5},
        {from: 2, to: 6},
        {from: 3, to: 4},
        {from: 3, to: 6},
        {from: 1, to: 7},
        {from: 5, to: 6},
    ]);
{{< / graph >}}



If we consider only person, object and location entities first, we can see that these can never be adjacent as they must link through event entities (at least in the purest applications of the data model). This sounds like a pretty good candidate to start optimizing our model.

# Passing insight to our algorithm

The greedy algorithm does yield good results, but like any other dilligent, locally optimized algorithm it fails to see the bigger picture.

Using our algorithm we will not reliably identify any graph-wide patterns, as the algorithm only assigns colours to a node based on it's adjacent nodes.  If we have a "global insight", e.g. spot a large group of entities that can be coloured the same colour, how can we pass this knowledge onto our algorithm?

We simply pre-assign the colours - effectively making the choice for the algorithm.

To implement this we only have to initialize the initial colour mappings and pass this to our greedy algorithm. The amended code is as follows (ellipses show removed code for brevity),

{{< highlight python >}}

...

def colour(graph, order, colours=None):
  # Cannot modify an arguments default value in python as it is mutable
  colours = colours or {}
  max_colour = max(colours.values()) 
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

...
colours = {
   'Person': 0,
   'Object': 0,
   'Location': 0
}

# We only need to assign colours to vertices that don't have pre-assigned colours
blank_vertices = [vertex for vertex in vertices where vertex not in colours]
vertex_colours, max_colour = colour(graph, blank_vertices, colours)
... 
{{< / highlight >}}

# Results 

Let's base line against a randomly generated POLE data model with 17500 entities and 130000 relationships (this is orders of magnitude larger than most data models). 

{{< highlight plaintext >}}
Chromatic number: 10
{{< / highlight >}}

We can provide the following insights to our algorithm:
* process person, object and location entities concurrently first
* process event entities which are not linked to other event entities second
* process analysis entities alongside event entities.

Suppose we have functions (*is_pol, is_analysis, is_event*) that tell us if a given vertex is a POL, Event or analysis entity we can then pre-ssign colours as follows:

{{< highlight python >}}


def has_event_neighbours(graph, vertex):
    return len([neighbour for neighbour in graph[vertex] if is_event(neighbour)]) > 0

for vertex in graph.keys():
    # If POL process first
    if is_pol(vertex):
        colours[vertex] = 0
    # If analysis process second
    if is_analysis(vertex):
        colours[vertex] = 1
    # If event and no other event_event relationships process second
    if is_event(vertex) and not has_event_neighbours(graph, vertex): 
        colours[i] = 1

{{< / highlight >}}

If we re-run our algorithm we see a large improvement.

{{< highlight plaintext >}}
Chromatic number: 5
{{< / highlight >}}

We have reduced the number of colours by half!

Pre-colouring our graph also has a performance benefit. We have limited the number of vertices which need to be visited, and therefore our algorithm will take less time to complete. Over 1000 iterations this difference is very significant:

{{< highlight plaintext >}}
Chromatic number: 10, Time to process: 27.858741760253906
Optimized Chromatic Number: 5, Time to process: 15.56061840057373
{{< / highlight >}}

This means in the same amount of time, we can also test ~2x as many random sequences of vertices, improving our chance of finding an optimal, or close to, solution.

# Other benefits

More efficient colourings and performance improvements aren't the only benefits of pre-colouring the graph. If we wish to prioritize some entities being processed earlier we can also do this by pre-colouring these entities. We may however see a performance degradation because of this as the resulting colouring may be suboptimal.

We can also dictate some entities not be processed together as well. For example, if we have multiple data sources providing our entities then we may not want to process too many entities from the same data source at the same time. To do this we can add suplimentary relationships between these entities, thereby making them adjacent and enforcing they will not be processed together.

# Conclusion

For the original problem domain, this is a "good enough" solution. Having a 50% decrease in the number of iterations versus a greedy colouring was an excellent improvement and highlights the importance of domain knowledge in solving technical problems.

There are still a few remaining caveats which intrigue me however, namely that we current process all relationships and that all entities are assumed to take an equal time to process. This however does move away from graph colouring and into optimization of weighted graphs. I may have to dust off a few textbooks...



