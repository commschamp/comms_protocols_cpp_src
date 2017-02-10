# Code Generation vs C++ Library

The implementation of the binary communication 
protocols can be a tedious, time consuming and error-prone process.
Therefore, there is a growing tendency among developers to use third party code 
generators for data (de)serialisation. Usually such tools receive description
of the protocol data layout in separate source file(s) with a custom grammar, 
and generate appropriate (de)serialisation code and necessary abstractions to 
access the data. 

There are so many of them: 
[ProtoBuf](https://developers.google.com/protocol-buffers/), 
[Cap'n Proto](https://capnproto.org/), [MessagePack](http://msgpack.org/index.html),
[Thrift](https://thrift.apache.org/), [Kaitai Struct](http://kaitai.io/),
[Protlr](https://www.protlr.com/), you-name-it...
All of these tools are capable of generating **C++** code. However,
the generated code quite often is not good enough be used in embedded systems, especially
bare-metal ones. Either the produced **C++** code or the tool itself has 
**at least** one of the following limitations:

- Inability to specify binary data layout. Many of the tools use their own
serialisation format without an ability to provide custom one. It makes them
impossible to use to implement already defined and used binary communication
protocol.
- Inability to customise underlying types. Most (or all) of the mentioned code 
generating tools, which do allow customisation of binary data layout,
choose to use **std::string** for string fields and/or 
**std::vector** for lists, as well as (de)serialisation code is generated to use 
standard streams (**std::istream** and **std::ostream**). Even if such ability
is provided, it is usually "global" one and do not allow substitution of types only for
specific messages / fields.
- Small number of supported data fields or limited number of their serialisation options.
For example, strings can be serialised by being prefixed with their size
(which in turn can have different lengths), or being terminated with '\0', or
having fixed size with '\0' padding if the string is too short. There are 
protocols that use all three variants of strings.
- Poor or weak description grammar without an ability to support conditional
(de)serialisation. For example, having a value
(such as single bit in some bitmask field) which determines whether some other
optional field exists or not. 
- Lack of polymorphic interface to allow implementation of the common code for all the 
defined messages.
- When polymorphic interface with virtual functions is provided, there is no
way to exclude generation of unnecessary virtual functions for a particular embedded application.
All the provided virtual functions will probably remain in the final image even
if they are not used.
- Lack of efficient built-in way of dispatching the deserialised message object into 
its appropriate handling function. There is a need to provide a separate 
dispatch table or map from message ID to some callback function or object.
- Lack of ability to override or complement the generated serialisation code with the manually
written one where extra logic is required.

The generalisation is hard. Especially when the main focus of the tools'
developers is on supporting as many target programming languages as possible, 
rather than allowing multiple configuration variants of a single specific
language. Currently there is no universal "fit all needs" code generation 
solution. 

As the result many embedded C++ developers still have to manually implement
the required binary communication protocol rather than relying on the existing
tools for code generation. There is still a way to help them in such endeavour by
developing a C++ library which will provide highly configurable classes, usage
of which will allow to implement required functionality using simple declarative
statements of types and classes definitions (instead of implementing everything
from scratch). That's what this book is all about. 

Thanks to new language features introduced in **C++11** standard and multiple 
meta-programming techniques, it becomes possible to write simple, clear, but 
highly configurable code, which can be used in multiple applications: embedded
bare-metal with limited resources, Linux based platform, even the GUI analysis
tools. They all can use the same single implementation of the protocol, but
each generate the code suitable for the developed platform / application. The
C++ compiler itself serves as code generation tool.

