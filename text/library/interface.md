# Generalising Interface

The basic generic message interface may include the following operations:

- Retrieve the message ID.
- Read (deserialise) the message contents from raw data in a buffer
- Write (serialise) the message contents into a buffer
- Calculate the serialisation length of the message.
- Dispatch the message to an appropriate handling function.
- Check the validity of the message contents.

There may be multiple cases when not all of the operations stated above are
needed for some specific case. For example, some sensor only reports its
internal data to the outside world over some I/O link, and doesn't listen to
the incoming messages. In this case the `read()` operation is redundant and
its implementation should not take space in the produced binary code.
However, the component that resides on the other end of the I/O link requires
the opposite functionality, it only consumes data, without producing anything, 
i.e. `write()` operation becomes unnecessary.

There must be a way to limit the basic interface to a particular set of
functions, when needed.

Also there must be a way to specify:

- type used to store and report the message ID.
- type of the read/write iterators 
- endian used in data serialisation.
- type of the message handling class, that is used in `dispatch()` functionality.

The best way to support such variety of requirements is to use the 
[variadic templates](http://en.cppreference.com/w/cpp/language/parameter_pack) 
feature of C++11, which allows having non-fixed number of template parameters.

These parameters have to be parsed and used to define all the required 
internal functions and types. The common message interface class is expected
to be defined like this:
```cpp
namespace comms
{
template <typename... TOptions>
class Message
{
    ...
};
} // namespace comms
```
where `TOptions` is a set of classes/structs which can be used to define all the 
required types and functionalities.

Below is an example of such possible option classes:
```cpp
namespace comms
{
namespace option
{
// Define type used to store message ID
template <typename T>
struct MsgIdType{};

// Specify type of iterator used for reading
template <typename T>
struct ReadIterator {};

// Specify type of iterator used for writing
template <typename T>
struct WriteIterator {};

// Use little endian for serialisation (instead of default big)
struct LittleEndian {};

// Include serialisation length retrieval in public interface
struct LengthInfoInterface {};

// Include validity check in public interface
struct ValidCheckInterface {};

// Define handler class
template <typename T>
struct Handler{};
} // namespace option
} // namespace comms
```

Our **PRIMARY OBJECTIVE** for this section is to use meta-programming techniques
and to be able to define a common message interface class with multiple
variations of public interface.

For example, the definition of `MyMessage` interface class below
```cpp
class MyHandler;
using MyMessage = comms::Message<
    comms::option::MsgIdType<std::uint16_t>, // use std::uint16_t
    comms::option::ReadIterator<const std::uint8_t*>, // use const std::uint8_t* as iterator for reading
    comms::option::WriteIterator<std::uint8_t*>, // use std::uint8_t* as iterator for writing
    comms::option::LengthInfoInterface, // add length() member function to interface
    comms::option::Handler<MyHandler> // add dispatch() member function with MyHandler as the handler class
>;
```
should be equivalent to defining:
```cpp
class MyMessage
{
public:
    typedef std::uint16_t MsgIdType;
    typedef const std::uint8_t* ReadIterator;
    typedef std::uint8_t* WriteIterator;
    typedef MyHandler Handler;
    
    MsgIdType id() {...}
    ErrorStatus read(ReadIterator& iter, std::size_t len) {...}
    ErrorStatus write(WriteIterator& iter, std::size_t len) const {...}
    std::size_t length() const {...}
    void dispatch(Handler& handler) {...}
protected:
    template <typename T>
    static T readData(ReadIterator& iter) {...} // use big endian by default

    template <typename T>
    static void writeData(T value, WriteIterator& iter) {...}  // use big endian by default
    ...    
};
```

And the following definition of `MyMessage` interface class
```cpp
using MyMessage = comms::Message<
    comms::option::MsgIdType<std::uint8_t>, // use std::uint8_t
    comms::option::LittleEndian, // use little endian in serialisation
    comms::option::ReadIterator<const std::uint8_t*> // use const std::uint8_t* as iterator for reading
>;
```
will be equivalent to:
```cpp
class MyMessage
{
public:
    typedef std::uint8_t MsgIdType;
    typedef const std::uint8_t* ReadIterator;
    
    MsgIdType id() {...}
    ErrorStatus read(ReadIterator& iter, std::size_t len) {...}
protected:
    template <typename T>
    static T readData(ReadIterator& iter) {...} // use little endian

    template <typename T>
    static void writeData(T value, WriteIterator& iter) {...}  // use little endian

    ...    
};
```

Looks nice, isn't it? So, how are we going to achieve this? Any ideas? 

That's right! We use **MAGIC**!

Sorry, I mean template meta-programming. Let's get started!

## Parsing the Options

First thing, that needs to be done, is to parse the provided options and record
them in some kind of a summary structure, with predefined list of 
`static const bool` variables, which indicate what options have been used, 
such as one below:
```cpp
struct MessageInterfaceParsedOptions
{
    static const bool HasMsgIdType = false;
    static const bool HasLittleEndian = false;
    static const bool HasReadIterator = false;
    static const bool HasWriteIterator = false;
    static const bool HasHandler = false;
    static const bool HasValid = false;
    static const bool HasLength = false;
}
```
If some variable is set to `true`, the *summary structure* may also contain 
some additional relevant types and/or more variables.

For example the definition of
```cpp
class MyHandler;
using MyMessage = comms::Message<
    comms::option::MsgIdType<std::uint16_t>, // use std::uint16_t
    comms::option::ReadIterator<const std::uint8_t*>, // use const std::uint8_t* as iterator for reading
    comms::option::WriteIterator<std::uint8_t*>, // use std::uint8_t* as iterator for writing
    comms::option::LengthInfoInterface, // add length() member function to interface
    comms::option::Handler<MyHandler> // add dispatch() member function with MyHandler as the handler class
>;
```
should result in 
```cpp
struct MessageInterfaceParsedOptions
{
    static const bool HasMsgIdType = true;
    static const bool HasLittleEndian = false;
    static const bool HasReadIterator = true;
    static const bool HasWriteIterator = true;
    static const bool HasHandler = true;
    static const bool HasValid = false;
    static const bool HasLength = true;
    
    typedef std::uint16_t MsgIdType;
    typedef const std::uint8_t* ReadIterator;
    typedef std::uint8_t* WriteIterator;
    typedef MyHandler Handler;
}
```

Here goes the actual code.

First, there is a need to define an initial version of such summary structure:
```cpp
namespace comms
{
template <typename... TOptions>
class MessageInterfaceParsedOptions;

template <>
struct MessageInterfaceParsedOptions<>
{
    static const bool HasMsgIdType = false;
    static const bool HasLittleEndian = false;
    static const bool HasReadIterator = false;
    static const bool HasWriteIterator = false;
    static const bool HasHandler = false;
    static const bool HasValid = false;
    static const bool HasLength = false;
}
} // namespace comms
```

Then, handle the provided options one by one, while replacing the initial values
and defining additional types when needed.
```cpp
namespace comms
{
template <typename T, typename... TOptions>
struct MessageInterfaceParsedOptions<comms::option::MsgIdType<T>, TOptions...> : 
                                        public MessageInterfaceParsedOptions<TOptions...>
{
    static const bool HasMsgIdType = true;
    typedef T MsgIdType;
};

template <typename... TOptions>
struct MessageInterfaceParsedOptions<comms::option::LittleEndian, TOptions...> : 
                                        public MessageInterfaceParsedOptions<TOptions...>
{
    static const bool HasLittleEndian = true;
};

template <typename T, typename... TOptions>
struct MessageInterfaceParsedOptions<comms::option::ReadIterator<T>, TOptions...> : 
                                        public MessageInterfaceParsedOptions<TOptions...>
{
    static const bool HasReadIterator = true;
    typedef T ReadIterator;
};

... // and so on
} // namespace comms
```
Note, that inheritance relationship is used and according to the C++ language specification
the new variables with the same name hide (or replace) the variables defined in 
the base class.

Also note, that the order of the options being used to define the interface
class does NOT really matter. However, it is recommended, to add some
`static_assert()` statements in, to make sure the same options are not used twice,
or no contradictory ones are used together (if such exist).

## Assemble the Required Interface

The next stage in the **defining message interface** process is to define
various chunks of interface functionality and connect them via inheritance.
```cpp
namespace comms
{
// ID retrieval chunk
template <typename TBase, typename TId>
class MessageInterfaceIdTypeBase : public TBase
{
public:
    typedef TId MsgIdType;
    MsgIdType getId() const
    {
        return getIdImpl();
    }

protected:
    virtual MsgIdType getIdImpl() const = 0;
};

// Big endian serialisation chunk
template <typename TBase>
class MessageInterfaceBigEndian : public TBase
{

protected:
    template <typename T>
    static T readData(ReadIterator& iter) {...} // use big endian

    template <typename T>
    static void writeData(T value, WriteIterator& iter) {...}  // use big endian
};

// Little endian serialisation chunk
template <typename TBase>
class MessageInterfaceLittleEndian : public TBase
{

protected:
    template <typename T>
    static T readData(ReadIterator& iter) {...} // use little endian

    template <typename T>
    static void writeData(T value, WriteIterator& iter) {...}  // use little endian
};

// Read functionality chunk
template <typename TBase, typename TReadIter>
class MessageInterfaceReadBase : public TBase
{
public:
    typedef TReadIter ReadIterator;
    ErrorStatus read(ReadIterator& iter, std::size_t size)
    {
        return readImpl(iter, size);
    }

protected:
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t size) = 0;
};

... // and so on
} // namespace comms
```

Note, that the interface chunks receive their base class through template
parameters. It will allow us to connect them together using inheritance.

There is a need for some extra helper classes to to implement such connection 
logic which chooses only requested chunks and skips others.
```cpp
namespace comms
{
template <typename TBase, typename TParsedOptions, bool THasMsgIdType>
struct MessageInterfaceProcessMsgId;

template <typename TBase, typename TParsedOptions>
struct MessageInterfaceProcessMsgId<TBase, TParsedOptions, true>
{
    typedef MessageInterfaceIdTypeBase<TBase, typename TParsedOptions::MsgIdType> Type;
};

template <typename TBase, typename TParsedOptions>
struct MessageInterfaceProcessMsgId<TBase, TParsedOptions, false>
{
    typedef TBase Type;
};
} // namespace comms
```

Let's assume that the interface options were parsed and typedef-ed into some
`ParsedOptions` type:
```cpp
typedef comms::MessageInterfaceParsedOptions<TOptions...> ParsedOptions;
```
Then after the following definition statement
```cpp
using NewBaseClass = 
    comms::MessageInterfaceProcessMsgId<
        SomeBaseClass, 
        ParsedOptions, 
        ParsedOptions::HasMsgIdType
    >::Type;
```
the `NewBaseClass` is the same as `OldBaseClass` if the value of
`ParsedOptions::HasMsgIdType` is `false`, otherwise `NewBaseClass` becomes 
`comms::MessageInterfaceIdTypeBase` which inherits from `OldBaseClass`.

Using the same pattern the other helper wrapping classes must be implemented
also:
```cpp
namespace comms
{
template <typename TBase, bool THasLittleEndian>
struct MessageInterfaceProcessEndian;

template <typename TBase>
struct MessageInterfaceProcessEndian<TBase, true>
{
    typedef MessageInterfaceLittleEndian<TBase> Type;
};

template <typename TBase>
struct MessageInterfaceProcessEndian<TBase, false>
{
    typedef MessageInterfaceBigEndian<TBase> Type;
};

template <typename TBase, typename TParsedOptions, bool THasReadIterator>
struct MessageInterfaceProcessReadIterator;

template <typename TBase, typename TParsedOptions>
struct MessageInterfaceProcessReadIterator<TBase, TParsedOptions, true>
{
    typedef MessageInterfaceReadBase<TBase, typename TParsedOptions::ReadIterator> Type;
};

template <typename TBase, typename TParsedOptions>
struct MessageInterfaceProcessReadIterator<TBase, TParsedOptions, false>
{
    typedef TBase Type;
};

... // and so on
} // namespace comms
```

The interface building code just uses the helper classes in a sequence of
type definitions:
```cpp
namespace comms
{
class EmptyBase {};

template <typename... TOptions>
struct MessageInterfaceBuilder
{
    using ParsedOptions = MessageInterfaceParsedOptions<TOptions...>;
   
    using Base1 = typename MessageInterfaceProcessMsgId<
            EmptyBase, ParsedOptions, ParsedOptions::HasMsgIdType>::Type; 
            
    using Base2 = typename MessageInterfaceProcessEndian<
            Base1, ParsedOptions::HasLittleEndian>::Type;            
            
    using Base3 = typename MessageInterfaceProcessReadIterator<
            Base2, ParsedOptions, ParsedOptions::HasReadIterator>::Type; 

    using Base4 = typename MessageInterfaceProcessWriteIterator<
            Base3, ParsedOptions, ParsedOptions::HasWriteIterator>::Type; 
    
    ...
    
    using BaseN = ...;
    
    using Type = BaseN;
};
} // namespace comms
```

Once all the required definitions are in place the common dynamic message
interface class `comms::Message` may be defined as:
```cpp
namespace comms
{
template <typename... TOptions>
class Message : public typename MessageInterfaceBuilder<TOptions...>::Type
{
};
} // namespace comms
```

As the result, any distinct set of options provided as the template parameters
to `comms::Message` class will cause it to have the required types and 
member functions.

Now, when the interface is in place, it is time to think about providing 
common `comms::MessageBase` class which is responsible to provide 
default implementation for functions, such as `readImpl()`, `writeImpl()`,
`dispatchImpl()`, etc... 

