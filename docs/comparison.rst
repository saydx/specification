*****************************************
Comparions with other transport protocols
*****************************************

There are already several solutions with considerable overlap with the suggested
protocol. They are described and compared with the suggested SAYDX protocol
below in order to get a clearer view, whether it is worth to implement a new
protocol at all. Only the transport layer should be discussed here. Possible
higher level protocols based on SAYDX (e.g. for exchanging data between
atomistic simulation tools) should be described and compared with existing ones
in a different chapter.


HDF5
====

`HDF5 <https://www.hdfgroup.org/solutions/hdf5/>`_ is an open-source library
(partially BSD, partially MIT licensed) using (and defining) its own data
storage format. It is widely spread in the scientific community and bindings
exist for almost all commonly used programming languages.

Similarities:

* SAYDX is probably closest to HDF5 in spirit.

* Basic building blocks are arrays. Type information about elements in an array
  is stored only once (together with the shape of the array).

* Data is stored in binary format.

* Arrays can be arranged in a tree like structure with named nodes.

* HDF5 allows to store attributes (of arbitrary data type) for a node, which
  should be possible in SAYDX as well (although, only string attributes).


Differences:

* HDF5 is pretty much file I/O oriented. Although it is possible to build up a
  tree in memory, it is not clear to me (B.A.), whether this tree can be
  communicated to a routine in a library other than being written to disc and
  being reread again. SAYDX should allow for passing trees between program
  components (even if written in different languages, e.g. Fortran and C).
  
* HDF5 is optimized for handling large amount of data. It had very advanced
  features, like parallel I/O. On the other hand, it is a very complex library
  which is not straightforward to build or link. SAYDX should be optimized for
  more moderate data amounts (few kilobytes to few megabytes, maybe up to few
  hundred megabytes). It should have much less features than HDF5 (e.g. no
  parallel I/O), but hopefully be then much easier to maintain, to build and to
  link.

* HDF5 requires the path to each node in the tree being unique. SAYDX should be
  more like XML, allowing a node to have several children with the same name
  (although open for discussions). Using XML-notation to indicate nodes, in
  SAYDX one could have::

    <simulation>
      <frame>
        <timestep>1</timestep>
        <coordinates>...</coordinates>
        [...]
      </frame>
      <frame>
        <timestep>2</timestep>
        <coordinates>...</coordinates>
        [...]
      </frame>
    </simulation>

  while in HDF5 (using the same notation) one would have to write ::
    
    <simulation>
      <frame1>
        <coordinates>...</coordinates>
        [...]
      </frame1>
      <frame2>
        <coordinates>...</coordinates>
        [...]
      </frame2>
    </simulation>

  

JSON
====

The `JSON protocol <https://www.json.org>`_ became very popular in the last few
years. It allows a standardized data exchange between different
application. Being used a lot in informatics (and also in science), libraries
exist probably in all programing languages used in science.

Similarities:

* Data can be arranged in a structure, which one may interprete as a tree with
  named nodes (in JSON they are actually a combination of lists and
  dictionaries).


Differences:

* JSON was not designed to exchange scientific (numeric) data. Its JavaScript
  implementation has only one numerical data type (double precision float),
  although other implementations may handle numerical data differently. SAYDX
  should offer all important numeric data types (e.g. from half-precision up to
  quadruple precision -- given appropriate compiler support)

* JSON was not designed to exchange array data. Its "array" construct is
  basically a flat list where each element can be of arbitrary data type. Type
  information must be stored for each element separately. Shape information
  (e.g. for a multi-dimensional array) must be stored separately in the tree.

* JSON was not designed to exchange large amount of numeric data. It is a text
  based format, so binary data would have to be converted to text first and then
  back again to binary format.
  


MessagePack
===========

`MessagePack <https://msgpack.org/>`_ is a binary protocol to exchange data
between applications. Implementations seem to be present in nearly all relevant
programming languages, but not in Fortran so far. It uses similar concepts as
JSON, but is binary data based and has a much better handling of numerical data
as JSON. It has support for single and double precision float numbers and allows
to define extended types.


Similarities:

* Data can be arranged in a structure, which one may interpreted as a tree with
  named nodes (actually a combination of lists and dictionaries).

* Data is stored in / serialized to binary format.


Differences:

* Similar to JSON, MessagePacks array concept is a flat list containing objects
  of arbitrary types, with the same disadvantages as above.


* As a consequence of the additional type information stored with each element,
  adding a native (Fortran or C) array to the tree always requires copying and
  reading out the array from the tree into a native array as well. This is
  maybe not a problem if the tree is communicated via sockets, but could raise
  efficiency problems when the tree is passed via an API between various
  components of an application.


CSlib
=====

`CSlib <https://cslib.sandia.gov/>`_ is a client-server library for
interchanging data between applications. It allows for exchanging data via
files, sockets (via ZeroMQ) or MPI. It is already part of LAMMPS, and should
also exist as a separate project under GitHub, but apparently it was not
uploaded yet. Licensing is unclear, probably BSD, although some documents
mention it as GPL-licensed.

Similarities:

* CSlib passes data in binary form. It has data types suited for scientific
  applications.

* It is possible to transmit multi-dimensional arrays with CSlib.


Differences:

* CSlibs messages are composed of fields, each field being assigned to an
  arbitrary data type and having zero or more entries of data type. It does not
  have the concept of a hierarchical tree. However, with an appropriate wrapper,
  it could be probably used to transmit a tree.

* While it is possible to transmit multi-dimensional arrays with CSlib, it seems
  that the array shape is not transmitted explicitely (only the number of
  elements). This would have to be communicated in an extra message.

* CSlib is not designed for passing a tree via an API between parts of an
  application (e.g. caller passes a tree to a library routine and receives an
  other tree as response), but concentrates on sending it via sockets, file I/O
  or MPI-messaging.
