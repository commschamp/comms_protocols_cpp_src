# Practical Guide to Implementing Communication Protocols in C++ (for Embedded Systems)

Almost every electronic device/component nowadays has to be able to communicate 
to other devices, components, or outside world over some I/O link. Such 
communication is implemented using various communication protocols.

At first glance the implementation of communication protocols seems to be
quite an easy and straightforward process. Every message has predefined
fields, that need to be serialised and deserialised according to the protocol
specification. Every serialised message is wrapped in a transport data to ensure
a safe delivery to the other end over some I/O link. However, there are multiple
pitfalls and wrong design choices that can lead to a cumbersome, bloated, and
difficult to maintain source code, especially when the development of the
product progresses and initially developed small communication protocol grows to
contain many more messages than initially planned. Adding a new message in
such state can become a tedious, time consuming and error-prone process.

After multiple trials and errors I finally managed to come up with a
flexible, generic and easily extendable architecture as well as develop a
generic C++(11) library which allow quick and easy implementation of many 
binary communication protocols. 

In this e-book I've tried to summarise my knowledge and experience. I hope it's
going to be useful for other developers as well.

As stated in the book's title, my main focus is a development for embedded systems 
(including bare-metal ones). There is no use of RTTI and/or exceptions. 
I also make a significant effort to minimise or exclude altogether 
dynamic memory allocation when possible. All the presented techniques are also
applicable to non-embedded systems which don't have limitations of the latter.

Plan:
* [Messages]
    * [Handling]
    * [Reading and Writing]
    * [Improving Interface]
* [Fields]
* [Transport]    
* [Processing Loop]    
    
