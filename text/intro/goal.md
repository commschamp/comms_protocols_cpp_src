# Goal

Our primary goal is to come up with an architecture that:

- does NOT depend or make any assumptions on the system it is running on.
- does NOT make any hard-coded assumptions on the resources available to the system, such
as dynamic memory allocation, exceptions, RTTI, etc...
- has an **efficient** way to parse the incoming message and dispatch it to an
appropriate handler. The runtime complexity shouldn't exceed `O(log(n))`, 
where `n` is a total number of messages in the protocol.
- provides quick, easy and straightforward way of adding new messages to the 
protocol.
- has as little connection as possible between the application level messages
and wrapping transport data, which allows easy substitution of the latter if
need arises.

Our ultimate goal would be creation of a generic C++(11) library, that can 
assist in implementation of many binary communication protocols. Such 
library will provide all the necessary types and classes, usage of which will make the
implementation of the required communication protocol easy, quick and
straightforward process of using simple declarative statements. 
It will significantly reduce the amount of boilerplate code and boost the 
development process.
