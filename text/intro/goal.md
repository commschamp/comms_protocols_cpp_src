# Goal

Our primary goal is to come up with an architecture that:

- does NOT depend or make any assumptions on the system it is running on.
- does NOT make any assumptions on the resources available to the system, such
as dynamic memory allocation, exceptions, RTTI, etc...
- has an efficient way to parse the incoming message in dispatch it to an
appropriate handler. The complexity shouldn't exceed O(log(n)), where n is a total
number of messages in the protocol.
- provides quick, easy and straightforward way of adding new messages to the 
protocol.
- has as little connection as possible between the application level messages
and wrapping transport data, which allows easy substitution of the latter if
need arises.

Our ultimate goal would be to create a generic library that can be used to implement
any communication protocol with simple declarative statements of types and 
classes. Such library will significantly boost the development process and
allow to avoid the code duplication and *deja-vu* feeling mentioned in
previous section.
