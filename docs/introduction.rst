.. highlight:: none

************
Introduction
************

There is a clear need for a general and robust transport protocol to enable data
exchange and communication between scientific software packages that need to
interact. To suport this and demonstrate its use, a library implementation of
the protocol should be developed and offered to the general scientific computing
community. The proof of concept application for the library should be
demonstrated for atomistic simulation packages, but the protocol and library
need to be general enough to satisfy the needs of other kind of scientific
software as well.

There are a number of methods for communication between codes. However are
either special purpose implementations or do not abstract this task for the
developers of the communicating codes.

Main goals
==========

* Data exchange should be robust, guaranteeing reliable transmission.

* One-to-one and eventually many-to-one and many-to-many communication scenarios
  should be supported.

* Exchange of complex data (e.g. all the information needed to initialize and
  start a simulation) should be straightforward.

* The protocol should allow communication through different communcation
  channels.

* Cross language support for Fortran, C/C++ and Python family langauages with
  cross-platform numerical model support.
  
Communication layers
====================

In order to ensure flexibility, the data exchange protocol needs (probably) three
layers of implementation:

#. Transport layer: deals with the technicalities of the communication. It
   should allow multiple transport channels, e.g. file I/O, socket
   communication, loadable code modules, etc. It should be extensible for future
   channels and guarantee communication reliability.

#. Message layer: Provides a flexible message format, which can be transmitted
   through the low-level layer betwen the applications.

#. High-level communication layer: Domain specific protocol composed of
   messages, as customised for the scientific codes' using the library.


Transport layer
---------------

It would be good if many different transport channels can be supported and
they are treated on the same footing. It would be desirable, if data could be
exchanged via

* File I/O (text and binary)

* Socket communication

* MPI-messaging

* Binary API (enabling direct communication between C, Fortran, Python,
  etc. programs)

For the socket and MPI channels, we could probably directly use Steve Plimptons
`cslib library <https://cslib.sandia.gov/>`_, which uses the quite robust ZeroMQ
framework for the socket communication. It is already part of LAMMPS but could
also be used as a stand-alone library without LAMMPS. Alternatively, one could
write something similar.


Message layer
-------------

One should use messages that are flexible enough to carry complex
information. For scientific applications the exchange of array data seems to be
enough, provided several arrays can be sent as one message and different data
types are supported within a message.

Example: Driver (a molecular-mechanics (MM)-program) sends data to a calculator
(quantum-mechanical (QM)-program) to initialize it. This can be quite complex,
as QM-programs usually require a lot of initialization parameters (Hamiltonian
settings, basis set settings, various control settings, etc.). The message
format needs to be flexible enough to allow for optional components, so that the
driver has to specify only required settings and also optional ones which it
wishes to override.

This could be easily realized by using a data tree as a message. A possible
structure could be like a simplified XML DOM-tree with following specification:

* Each node of the tree is named (like in XML).

* Each node of the tree can either contain further nodes (a container node) or
  data (data nodes), but never both. Consequently, data nodes were the leaves of
  the tree and have only container nodes as parents.

* Each data node contains a single array of a given type and shape or a scalar
  as data. The data is in native binary format.

* Optional: the nodes should contain attributes to store additional information
  (e.g. the unit of the data in the node, etc.). To make things simple, the
  attributes should be text attributes, like in XML.

The sender would assemble a tree with the necessary information and transmit it via
the transport layer. The receiver would then query the received tree, look
for the presence / absence of given nodes and extract the necessary information
from the tree.

I have already started a small C-library with this functionality, the `saydx
library <https://github.com/saydx/saydx>`_. Although not finished yet, it could
be used as the message layer. It would provide the basic infrastructure for tree
manipulation, as well as routines to read and write trees to file or to pass
them from C to Fortran and vice-versa. Combined with cslib, it could cover the
functionality of the first two layers.


Protocol layer
--------------

In contrast to the other two layers, the protocol layer must be domain specific,
as different scientific applications need different data to be communicated.

As a proof of concept, communication between atomistic simulation packages could
be implemented. One could start from the i-Pi protocol, as several packages are
using it already, base it on the new message format and extend it with
additional components.

As an example, the transmitted data for passing the geometry between driver and
client could look like the structure sketched below. The XML-notation is used to
indicate nodes and the ``@`` symbols indicate (binary) scalars or arrays of a
given type and shape in the leaves (e.g., ``@s`` is scalar string, ``@r8(3,2)``
is a rank two array of 64 bit reals with shape (3, 2), etc.)::

  <ipi-message>
    <command>
      @s ::
      POSDATA
    </command>
    <data>
      <atom_positions>
        @r8(3, 2) ::
        0.0   0.0   0.0
        0.0   0.0   1.0
     </atom_positions>
     <lattice_vectors>
        @r8(3,3) ::
        10.0  0.0  0.0
         0.0 10.0  0.0
         0.0  0.0 10.0
      </lattice_vectors>
   </data>
  <ipi-message>

The receiver could then query the transmitted tree using following Fortran
pseudo code::

  call receive_tree(root_node)
  if (root_node%get_name() /= "ipi-message") then
    call error("Invalid message protocol")
  end if
  
  call get_child_data(root_node, "command", commandstr)
  if (.not. allocated(commandstr)) then
    call error("Could not find command node or it contains wrong data type")
  end if
  
  select case (commandstr)
  
  case ("POSDATA")
  
    call get_child(root_node, "data", data_node)
    if (.not. data_node%is_associated()) then
      call error("Data node not found")
    end if
    
    call get_child_data(data_node, "atom_positions", atom_positions)
    if (.not. allocated(atom_positions)) then
      call error("Node 'atom_positions' not found or it contains wrong data type")
    end if
    if (all(shape(atom_positions) /= (3, natom)) then
      call error("Array in node 'atom_positions' has invalid shape")
    end if
    
    ! Only query tree for lattice vectors if the system is periodic
    if (periodic) then
    
      call get_child_data(data_node, "lattice_vectors", lattice_vectors)
      if (.not. allocated(lattice_vectors)) then
        call error("Node 'lattice_vectors' not found or has wrong data type)
      end if
      if (shape(lattice_vectors) /= (3, 3)) then
        call error("Array in node 'lattice_vectors' has invalid shape')
      end if

    end if
    
    [...]
    
  end select case
    
The lower lying layers warranty that the entire data tree (as sent by the
sender) gets trasmitted before the receiver can start to read it. The receiver,
therefore, can be sure that it has all the data the sender wanted
communicating. It does not need to assume the shape / size of the transmitted
data when receiving the message and hope for the best (as it is the case with
the bare socket based i-Pi protocol). The arrays in the tree have type and shape
information. The receiver can check whether they match its expectations and
handle the error gracefully if not.

Debugging communication problems (e.g. sender and receiver implement the
high-level protocol differently) should be also straightforward, as the saydx-library
contains routines to write the trees from memory to disk.
