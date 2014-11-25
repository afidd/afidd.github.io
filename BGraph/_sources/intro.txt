**************************
Introduction
**************************

This package implements an adjacency list, along the textbook
definitions, using the Boost.Graph interface. Where Boost.Graph
returns an opaque ``vertex_descriptor``, this package returns
an instance of a type which is internal to the package, so it's the
moral equivalent.

This package is slower than Graphs.jl. The data structure is more
compact, but maybe it could use optimization. It is young, yet.

Create an AdjacencyList
-------------------------

Choose what data structure stores the list of
vertices (VC) and what data structure stores the list of neighbors
of each vertex (NC). Choose what extra properties each vertex (VP), edge (EP),
and graph (GP) should store additionally. For instance,::

  adj=AdjacencyList(Vector,Deque,Int,Float,"my graph")
  println(graph_property(adj)) # "my graph"
  v1=add_vertex!(3, adj)
  v2=add_vertex!(7, adj)
  println(vertex_property(v2, adj)) # 7
  e1=add_edge!(v1, v2, 2.6, adj)
  println(edge_property(e1, adj)) # 2.6

Choices for vertex storage are

 * Vector---This makes the vertex_descriptor an integer. Slower for growing
   the graph.

 * Deque---The vertex_descriptor is some internal type. Faster for growing
   the graph.

 * Dict---Specify a dict by creating a TypeConstructor, using a typealias.
   "typealias IntDict{T} Dict{Int,T}" Then pass ``IntDict`` as the argument.
   When using a dict, add a key value when calling ``add_vertex!(key, value, graph)``.

Choices for neighbor storage are

 * Vector---A compact storage choice. Using a Vector here doesn't affect
   naming of vertices and doesn't make the ``edge_descriptor`` type any more
   simple. It is always possible to make an edge with the method
   ``edge(vertex_descriptor, vertex_descriptor)``.

 * Deque---Would grow faster.

 * Set--Using a set guarantees that there is only one edge between any
   two vertices.

The graph can be created as undirected, as well::

  adj=AdjacencyList(Vector,Deque,Int,Float,"my graph", is_directed=false)

This creates a reverse edge for each edge added.


Working with an AdjacencyList
------------------------------

Properties attached to a graph, edge, or vertex, can be any type::

  type EdgeProperty
    weight::Float64
    stoichiometry::Int
  end

  adj=AdjacencyList(Vector,Deque,Int,EdgeProperty,"my graph")
  v1=add_vertex!(3, adj)
  v2=add_vertex!(7, adj)
  e1=add_edge!(v1, v2, EdgeProperty(0.6, -2), adj)
  println(edge_property(e1).stoichiometry, adj) # -2

In order to omit a property, specify it as a tuple type.
For instance add no types with::

  adj=AdjacencyList(Vector,Deque, (), (), ())

Access behaves as with Boost.Graph::

  for v in vertices(adj)
    for out_edge in out_edges(v, adj)
      if edge_property(out_edge, adj).stoichiometry<0
        println(edge_property(out_edge, adj).weight)
      end
    end
  end

  assert(v1==source(e1, adj))
  assert(v2==target(e1, adj))
  assert(e1==edge(v1, v2, adj))
  assert(2==num_vertices(adj))
  assert(1==num_edges(adj))
  assert(1==out_degree(v1, adj))

  for n in out_neighbors(v1, adj)
    assert(n==v2)
  end

There is no way to remove an edge or vertex.
