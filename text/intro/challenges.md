# Main Challenges

There are multiple challenges that need to be considered prior to starting 
implementation of any communication protocol. It will guide us into the 
right direction when designing an architecture. Let's take a careful look at
them one by one.

## Code Duplication

As a whole, most of the communication protocols are very similar, they define 
various messages with their internal fields, define serialisation rules for all 
the fields and wrap them in some kind of transport information to ensure safe 
delivery of the message over the I/O link. 

When serialising any message, all its fields must be serialised in predefined
order. There is also very limited number of field types that is usually used:

- **numeric values** - may differ in sizes, being signed or unsigned, have a different
ranges of valid values, etc...
- **enumeration values** - similar to numeric values, but have a very limited range
of valid values, and `enum` type is usually used to operate the values, just for
convenience.
- **bitmask values** - similar to numeric values, but each bit has a different 
meaning
- **strings** - may differ in the way they are serialised (zero-suffixed or size-prefixed)
- **lists** of raw bytes or other fields - may have fixed (predefined) or 
variable size.
- **bundles of multiple fields** - may be used as a single element of a **list**.
- **bitfields** - similar to bundles, but internal member fields have a length of
several bits (instead of bytes).

The number of field types is quite small, but the number of different nuances when 
serialising or using a single field is much bigger. It is very difficult to
generalise such use and most developers don't even bother to come up with 
something generic. As the result they experience a *deja-vu* feeling every time
they have to implement a new message or add a new field into an existing message.
There is a strong feeling that the code is being duplicated, but there is no
obvious and/or easy way to minimise it.

## Runtime Efficiency
In most cases the messages are differentiated by some numeric value ID. When
the new message is received over some I/O link, it needs to be identified and
dispatched to appropriate handling function. Many developers implement this logic
using a `switch` statement. However, after about 7 - 10 `case`-s such dispatch
mechanism becomes quite inefficient and its inefficiency grows with number of
new messages being introduced. When not having a limitation of inability to
use dynamic memory allocation and/or exception, some developers resort to
standard collections (`std::map` for example) of pointer to functions or
`std::function` objects. Bare-metal developers usually stick to the `switch`
statement option incurring certain performance penalties when the implemented
communication protocol grows.

## Protocol Extension Effort
Also keep in mind the development effort that will be required to introduce
a new message to the protocol being implemented. The number of different places 
in the existing code base, that need to be modified/updated, must obviously be
kept at a minimum. Ideally no more than 2 or 3, but most implementations I've
seen significantly bypass these numbers. In many cases developers forget
to introduce compile time checks, such as `static_assert` statements to
verify that all the required places in the code have been updated after new
message was introduced. Failure to do so results in unexpected bugs and extended
development effort to find and fix them.

What about extending an existing message by adding an extra field at the end or
even in the middle? How easy it's going to be and what development time needs
to be spent? How error-prone it's going to be? 

## Inter-System Reuse
Quite often the implementation of the same protocol needs to be reused between
different systems. For example, some embedded sensor device needs to communicate its 
data to a management server (both implemented in C++) and it would be wise to
share the same implementation of the communication protocol on both ends. 
However, managing the I/O link and usage of various data structures may
be different for both of them. Making the implementation of the communication
protocol system dependent may make such reuse impossible.

Sometimes different teams are responsible for implementation of different components
that reside on different ends of the communication link. 
Even in this case, making the implementation of the
communication protocol system dependent is a bad idea. It may be necessary to
develop some additional protocol testing tools because the other team is
not ready yet.

## Intra-System Reuse
It is not uncommon for various embedded systems to add extra I/O interfaces
in the next generations of the device hardware which can be used to communicate with
other devices using the same protocol. For example, the first generation of
some embedded sensor communicates its data over TCP/IP network link to 
some data management server. The second generation adds a Bluetooth interface
that allows to communicate the same data to a tablet of the person working nearby.
The application level messages, used to communicate the data, 
are the same for the server and the tablet.
However, the transport wrapping information for TCP/IP and Bluetooth will 
obviously differ. If initial implementation of the communication protocol 
hasn't properly separated the application level messages and wrapping transport
data, it's going to be difficult, time consuming and error-prone to introduce
a new Bluetooth I/O link.
