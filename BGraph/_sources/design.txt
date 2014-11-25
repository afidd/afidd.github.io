***************************
Design
***************************

This document looks at implementation of the same adjacency list data structure
that is used in Boost.Graph. It is a string over vertex storage, where
each vertex stores a string of neighbors. The vertices aren't necessarily
indexed in any way. An adjacency list knows which entry is a neighbor of
which other entry, but the string of vertices and string of neighbors
may be stored many different ways.

Vertex Descriptor
---------------------

The ``vertex_descriptor`` in Boost.Graph is an opaque handle to a vertex.
It is usually a pointer to the vertex, except in one special case.
For adjacency lists constructed with vector types for both the vertex list
and neighbor lists, the ``vertex_descriptor`` is an integer.

Instead of using a dictionary at vertices and edges to store properties,
it is possible to append those properties to the vertex type and neighbor
type. In the following example, note that the type of the VertexDescriptor
used in the Neighbor class is determined by the Vertex class. It's recursive.::

  immutable type Neighbor{VertexDescriptor,EdgeProperty}
    n::VertexDescriptor
    edge_property::EdgeProperty
  end

  immutable type Vertex{VertexProperty,EdgeProperty}
    # This is the recursive bit, where it makes pointers to itself.
    v::Deque{Neighbor{Vertex{VertexProperty,EdgeProperty},EdgeProperty}
    vertex_property::VertexProperty
  end

  type AdjacencyList{VertexProperty,EdgeProperty}
    vertices::Deque{Vertex{VertexProperty,EdgeProperty}}
    is_directed::Bool
  end

This setup would work fine for the Deque type, but Boost.Graph uses
C++ vector, list, or set types. An array type would use integers for
indexing.::

  immutable type Neighbor{VertexDescriptor,EdgeProperty}
    n::VertexDescriptor
    edge_property::EdgeProperty
  end

  immutable type Vertex{VertexProperty,EdgeProperty}
    # This is the recursive bit, where it makes pointers to itself.
    v::Vector{Neighbor{Int,EdgeProperty}
    vertex_property::VertexProperty
  end

  type AdjacencyList{VertexProperty,EdgeProperty}
    vertices::Vector{Vertex{VertexProperty,EdgeProperty}}
    is_directed::Bool
  end

Parameterizing Storage
------------------------

It is difficult in Julia to create a type which changes
storage class depending on a parameter type. For instance,
this is not allowed::

  type Foo{C,V}
    container::C{V}
  end

That it is not allowed is specified in an issue 3359,
https://github.com/JuliaLang/julia/issues/3359.
Although it is not allowed, this works (for now)::

  cconstruct(x::TypeVar, y)=x
  cconstruct(x::Union(DataType,TypeConstructor), y)=x{y}

  type Foo{C,V}
    container::cconstruct(C,V)
  end

  Foo{Vector,Int}(Array(Int, 0))

Julia does three passes through the type, two
where it calls ``cconstruct(x::TypeVar, y)`` and then
one where it passes in the given type. At that point,
the function can do calculations on the incoming types.

If the code can vary the container type parametrically,
then it helps to abstract across the container types
the exact interface this code requires. This will
look something like the Boost ``property_map`` interface.
There are two containers in this class, one for
vertices and one for neighbors of a vertex,
and they are used slightly differently.
The ``vertex_descriptor`` will always be some sort
of key, whether it is a reference to an object in
a Deque or an Int pointing into a Vector.
On the other hand, the ``edge_descriptor`` in Boost.Graph
isn't just a pair of vertices. It's
a pair of (vertex_descriptor, edge_pointer). In Julia,
that becomes (vertex_descriptor, edge instance), no
matter whether neighbor edges are stored in a Set, Deque,
or Vector.

To this end, there are a set of methods defined for
each type of container. Here are two.::

	container_key{V}(::Type{Vector{V}})=Int
	container_value{V}(::Type{Vector{V}})=V
	container_construct{V}(::Type{Vector{V}})=Array(V, 0)
	container_push{V}(c::Vector{V}, v::V)=begin push!(c, v); length(c) end
	container_add{V}(c::Vector{V}, args...)=begin v=V(args...); push!(c, v); v end
	container_add_key{V}(c::Vector{V}, args...)=begin push!(c, V(args...)); length(c) end
	container_get{V}(c::Vector{V}, k::Int)=c[k]
	container_set{V}(c::Vector{V}, k::Int, v::V)=begin c[k]=v; k end
	container_iter{V}(c::Vector{V})=1:length(c)
	container_value_iter{V}(c::Vector{V})=c

	container_key{V}(::Type{Deque{V}})=V
	container_value{V}(::Type{Deque{V}})=V
	container_construct{V}(::Type{Deque{V}})=Deque{V}()
	container_push{V}(c::Deque{V}, v::V)=begin push!(c, v); v end
	container_add{V}(c::Deque{V}, args...)=begin v=V(args...); push!(c, v); v end
	container_add_key{V}(c::Deque{V}, args...)=container_add(c, args...)
	container_get{V}(c::Deque{V}, k::V)=k
	container_set{V}(c::Deque{V}, k::V, v::V)=begin c[k]=v; v end
	container_iter{V}(c::Deque{V})=c
	container_value_iter{V}(c::Deque{V})=c

These abstract the container type, detecting the
type of the container that was created.

Lastly, Boost.Graph has a special integer ``vertex_descriptor``
when both containers are vectors, but the integer descriptor
is possible when only the vertex container is a vector. As well,
it is possible to use a Dict to store vertices (which would
also be possible in Boost if it weren't already so complicated),
so this container is included in the code, too.

