# Vertex Cache Optimised Index Buffer Compression

This is a small proof of concept for compressing and decompressing index buffer triangle lists. It's designed to maintain the order of the triangle list and perform best with a triangle list that has been vertex cache post-transform optimised (a pre-transform cache optimisation is done as part of the compression).

It's also designed to be relatively lightweight, with a decompression throughput in the tens of millions of triangles per core.  It does not achieve state of the art levels of compression levels (which can be less than a bit per triangle, as well as providing good chances for vertex prediction), but it does maintain ordering of triangles and support arbitrary topologies. 

There are some cases where the vertices within a triangle are re-ordered, but the general winding direction is maintained.

## How does it work?

The inspiration was a mix of Fabian Giesen's Simple loss-less index buffer compression http://fgiesen.wordpress.com/2013/12/14/simple-lossless-index-buffer-compression/ and
the higher compression algorithms that make use of shared edges and re-order triangles. The idea was that there is probably a middle ground between them.

The basic goals were:
* Maintain the ordering of triangles, exploiting vertex cache optimal ordering.
* Exploit recent triangle connectivity.
* Make it fast, especially for decompression, without the need to maintain large extra data structures, like winged edge.
* Make it simple enough to be easily understandable. 

The vertex cache optimisation means that there will be quite a few vertices and edges shared between the next triangle in the list and the previous. We exploit this by maintaining two relatively small fixed size FIFOs, an edge FIFO and a vertex FIFO (not unlike the vertex cache itself, except we store recent indices).

The compression relies on 4 codes: 
1. A _new vertex_ code, for vertices that have not yet been seen. 
2. A _cached edge_ code, for edges that have been seen recently. This code is followed by a relative index back into the edge FIFO.
3. A _cached vertex_ code, for vertices that have been seen recently. This code is followed by a relative index back into the vertex FIFO.
4. A _free vertex_ code, for vertices that have been seen, but not recently. This code is followed by a variable length integer encoding of the index relative to the most recent new vertex.

Triangles can either consist of two codes, a cached edge followed by one of the vertex codes, or of 3 of the vertex codes. The most common codes in an optimised mesh are generally the cached edge and new vertex codes.

Cached edges are always the first code in any triangle they appear in and may correspond to any edge in the original triangle (we check all the edges against the FIFO). This means that an individual triangle may have its vertices specified in a different order (but in the same winding direction) than the original uncompressed one.

New vertex codes work because vertices are re-ordered to the order in which they appear in the mesh, meaning whenever we encounter a new vertex, we can just read and an internal counter to get
the current index, incrementing it afterwards. This has the benefit of also meaning vertices are in pre-transform cache optimised order.

