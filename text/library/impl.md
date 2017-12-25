# Generalising Message Implementation

Previous chapters described `MessageBase` class, which provided implementation for
some portions of polymorphic behaviour defined in the interface class `Message`.
Such implementation eliminated common boilerplate code used in every `ActualMessage*`
class.

This chapter is going to generalise the implementation of `MessageBase` into
the generic `comms::MessageBase` class, which is communication protocol independent
and can be re-used in any other development.

The generic `comms::MessageBase` class must be able to:

- provide the ID of the message, i.e. implement the `idImpl()` 
virtual member function, when such ID is known at compile time.
- provide common dispatch functionality, i.e. implement `dispatchImpl()`
virtual member function, described in 
[Message / Dispatching and Handling](../message/dispatch_handle.md) chapter.
- support extension of the default message interface, described in
[Message / Extending Interface](../message/extend_interface.md) chapter.
- automate common operations on fields, i.e. implement `readImpl()`, `writeImpl()`,
`lengthImpl()`, etc..., described in 
[Fields / Automating Basic Operations](../fields/automation.md) chapter.

Just like common `comms::Message` interface class, the `comms::MessageBase`
will also receive options to define its behaviour.
```cpp
namespace comms
{
template <typename TBase, typename... TOptions>
class MessageBase : public TBase
{
    ...
};
} // namespace comms
```
Note, that the `comms::MessageBase` class receives its base class as a 
template parameter. It is expected to be any variant of `comms::Message` or
any extended interface class, which inherits from `comms::Message`.

The supported options may include:
```cpp
namespace comms
{
namespace option
{
// Provide static numeric ID, to facilitate implementation of idImpl()
template <std::intmax_t TId>
struct StaticNumIdImpl {};

// Facilitate implementation of dispatchImpl()
template <typename TActual>
struct DispatchImpl {};

// Provide fields of the message, facilitate implementation of
// readImpl(), writeImpl(), lengthImpl(), validImpl(), etc...
template <typename TFields>
struct FieldsImpl {};

} // namespace option
} // namespace comms
```

## Parsing the Options

The options provided to the `comm::MessageBase` class need to be parsed in a
very similar way as it was with `comms::Message` in the previous chapter.

Starting with initial version of the options struct:
```cpp
namespace comms
{
template <typename... TOptions>
class MessageImplParsedOptions;

template <>
struct MessageImplParsedOptions<>
{
    static const bool HasStaticNumIdImpl = false;
    static const bool HasDispatchImpl = false;
    static const bool HasFieldsImpl = false;
}
} // namespace comms
```
and replacing the initial value of the appropriate variable with new ones, when
appropriate option is discovered:
```cpp
namespace comms
{
template <std::intmax_t TId, typename... TOptions>
struct MessageImplParsedOptions<option::StaticNumIdImpl<TId>, TOptions...> :
        public MessageImplParsedOptions<TOptions...>
{
    static const bool HasStaticNumIdImpl = true;
    static const std::intmax_t MsgId = TID;
};

template <typename TActual, typename... TOptions>
struct MessageImplParsedOptions<option::DispatchImpl<TActual>, TOptions...> :
        public MessageImplParsedOptions<TOptions...>
{
    static const bool HasDispatchImpl = true;
    using ActualMessage = TActual;
};

template <typename TFields, typename... TOptions>
struct MessageImplParsedOptions<option::FieldsImpl<TFields>, TOptions...> :
        public MessageImplParsedOptions<TOptions...>
{
    static const bool HasFieldsImpl = true;
    using Fields = TFields;
};
} // namespace comms
```

## Assemble the Required Implementation

Just like with building custom message interface, there is a need to 
create chunks of implementation parts and connect them using inheritance 
based on used options.
```cpp 
namespace comms
{
// ID information chunk
template <typename TBase, std::intmax_t TId>
class MessageImplStaticNumIdBase : public TBase
{
public:
    // Reuse the message ID type defined in the interface
    using MsgIdType = typename Base::MsgIdType;
    
protected:
    virtual MsgIdType getIdImpl() const override
    {
        return static_cast<MsgIdType>(TId);
    }
};

// Dispatch implementation chunk
template <typename TBase, typename TActual>
class MessageImplDispatchBase : public TBase
{
public:
    // Reuse the Handler type defined in the interface class
    using Handler = typename Base::Handler;
    
protected:
    virtual void dispatchImpl(Handler& handler) const override
    {
        handler.handle(static_cast<TActual&>(*this));
    }
};
} // namespace comms
```

**NOTE**, that single option `comms::option::FieldsImpl<>` may facilitate 
implementation of multiple functions: `readImpl()`, `writeImpl()`, `lengthImpl()`, 
etc... Every such function was declared due to using a separate option when
defining the interface. We'll have to cherry-pick appropriate implementation
parts, based on the interface options. As the result, these implementation
chunks must be split into separate classes.
```cpp
namespace comms
{
template <typename TBase, typename TFields>
class MessageImplFieldsBase : public TBase
{
public:
    using AllFields = TFields;
    
    AllFields& fields() { return m_fields; }
    const AllFields& fields() const { return m_fields; }
private:
    TFields m_fields;
};

template <typename TBase>
class NessageImplFieldsReadBase : public TBase
{
public:
    // Reuse ReadIterator definition from interface class
    using ReadIterator = typename TBase::ReadIterator;
protected:
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) override 
    {
        // Access fields via interface provided in previous chunk
        auto& allFields = TBase::fields(); 
        ... // read all the fields
    }
};

... // and so on
} // namespace comms
```

All these implementation chunks are connected together using extra helper classes
in a very similar way to how the interface chunks where connected:

Add `idImpl()` if needed
```cpp
namespace comms
{
template <typename TBase, typename ParsedImplOptions, bool TImplement>
struct MessageImplProcessStaticNumId;

template <typename TBase, typename ParsedImplOptions>
struct MessageImplProcessStaticNumId<TBase, ParsedImplOptions, true>
{
    using Type = MessageImplStaticNumIdBase<TBase, ParsedImplOptions::MsgId>;
};

template <typename TBase, typename ParsedImplOptions>
struct MessageInterfaceProcessEndian<TBase, false>
{
    using Type = TBase;
};
} // namespace comms
```

Add `dispatchImpl()` if needed
```cpp
namespace comms
{
template <typename TBase, typename ParsedImplOptions, bool TImplement>
struct MessageImplProcessDispatch;

template <typename TBase, typename ParsedImplOptions>
struct MessageImplProcessDispatch<TBase, ParsedImplOptions, true>
{
    using Type = MessageImplDispatchBase<TBase, typename ParsedImplOptions::ActualMessage>;
};

template <typename TBase, typename ParsedImplOptions>
struct MessageImplProcessDispatch<TBase, false>
{
    using Type = TBase;
};
} // namespace comms
```

Add `fields()` access if needed
```cpp
namespace comms
{
template <typename TBase, typename ParsedImplOptions, bool TImplement>
struct MessageImplProcessFields;

template <typename TBase, typename ParsedImplOptions>
struct MessageImplProcessFields<TBase, ParsedImplOptions, true>
{
    using Type = MessageImplFieldsBase<TBase, typename ParsedImplOptions::Fields>;
};

template <typename TBase, typename ParsedImplOptions>
struct MessageImplProcessFields<TBase, false>
{
    using Type = TBase;
};
} // namespace comms
```

Add `readImpl()` if needed
```cpp
namespace comms
{
template <typename TBase, bool TImplement>
struct MessageImplProcessReadFields;

template <typename TBase>
struct MessageImplProcessReadFields<TBase, true>
{
    using Type = NessageImplFieldsReadBase<TBase>;
};

template <typename TBase>
struct MessageImplProcessReadFields<TBase, false>
{
    using Type = TBase;
};

} // namespace comms
```

And so on for all the required implementation chunks: `writeImpl()`, `lengthImpl()`,
`validImpl()`, etc...

The final stage is to connect all the implementation chunks together
via inheritance and derive `comms::MessageBase` class from the result.

**NOTE**, that existence of the implementation chunk depends not only on the
implementation options provided to `comms::MessageBase`, but also on the 
interface options provided to `comms::Message`. For example, `writeImpl()` must
be added only if `comms::Message` interface includes `write()` member function 
(`comms::option::WriteIterator<>` option was used) and implementation option
which adds support for fields (`comms::option::FieldsImpl<>`) was passed to
`comms::MessageBase`.

The implementation builder helper class looks as following:
```cpp
namespace comms
{
// TBase is interface class
// TOptions... are the implementation options
template <typename TBase, typename... TOptions>
struct MessageImplBuilder
{
    // ParsedOptions class is supposed to be defined in comms::Message class
    using InterfaceOptions = typename TBase::ParsedOptions;
    
    // Parse implementation options
    using ImplOptions = MessageImplParsedOptions<TOptions...>;
    
    // Provide idImpl() if possible
    static const bool HasStaticNumIdImpl = 
        InterfaceOptions::HasMsgIdType && ImplOptions::HasStaticNumIdImpl;
    using Base1 = typename MessageImplProcessStaticNumId<
            TBase, ImplOptions, HasStaticNumIdImpl>::Type;
            
    // Provide dispatchImpl() if possible
    static const bool HasDispatchImpl = 
        InterfaceOptions::HasHandler && ImplOptions::HasDispatchImpl;
    using Base2 = typename MessageImplProcessDispatch<
            Base1, ImplOptions, HasDispatchImpl>::Type;

    // Provide access to fields if possible
    using Base3 = typename MessageImplProcessFields<
            Base2, ImplOptions, ImplOptions::HasFieldsImpl>::Type;
            
    // Provide readImpl() if possible
    static const bool HasReadImpl = 
        InterfaceOptions::HasReadIterator && ImplOptions::HasFieldsImpl;
    using Base4 = typename MessageImplProcessReadFields<
            Base3, HasReadImpl>::Type;
       
    // And so on...
    ...
    using BaseN = ...;
    
    // The last BaseN must be taken as final type.
    using Type = BaseN;
};
} // namespace comms
```

Defining the generic `comms::MessageBase`:
```cpp
namespace comms
{
template <typename TBase, typename... TOptions>
class MessageBase : public typename MessageImplBuilder<TBase, TOptions>::Type
{
    ...
};
} // namespace comms
```

Please note, that `TBase` template parameter is passed to `MessageImplBuilder<>`,
which in turn passes it up the chain of possible implementation chunks, and
at the end it turns up to be the base class of the whole hierarchy. 

The full hierarchy of classes presented at the image below. 
![Image: Full class hierarchy](../image/library_full_hierarchy.png)

The total number of used classes may seem scary, but there are only two, which 
are of any particular interest to us when implementing communication protocol.
It's `comms::Message` to specify the interface and `comms::MessageBase` to
provide default implementation of particular functions. All the rest are just
implementation details.

## Summary

After all this work our library contains generic `comms::Message` class, that
defines the interface, as well as generic `comms::MessageBase` class, that provides
default implementation for required polymorphic functionality. 

Let's define a custom communication protocol which uses little endian
for data serialisation and has numeric
message ID type defined with the enumeration below:
```cpp
enum MyMsgId
{
    MyMsgId_Msg1,
    MyMsgId_Msg2, 
    ...
};
```

Assuming we have relevant field classes in place (see [Fields](../fields/head.md)
chapter), let's define custom `ActualMessage1` that contains two integer 
value fields: 2 bytes unsigned value and 1 byte signed value.
```cpp
using ActualMessage1Fields = std::tuple<
    IntValueField<std::uint16_t>,
    IntValueField<std::int8_t>
>;
template <typename TMessageInterface>
class ActualMessage1 : public 
    comms::MessageBase<
        comms::option::StaticNumIdImpl<MyMsgId_Msg1>, // provide idImpl() if needed
        comms::option::DispatchImpl<ActualMessage1>, // provide dispatchImpl() if needed
        comms::option::FieldsImpl<ActualMessage1Fields> // provide access to fields and
                                                        // readImpl(), writeImpl(),
                                                        // lengthImpl(), validImpl() 
                                                        // functions if needed
    >
{
};
```

That's it, no extra member functions are needed to be implemented, unless the
message interface class is [extended one](../message/extend_interface.md).
Note, that the implementation of the `ActualMessage1` is completely generic
and doesn't depend on the actual message interface. It can be reused in any
application with any runtime environment that uses our custom protocol.

The interface class is defined according to the requirements of the application, that
uses the implementation of the defined protocol.
```cpp
class MyHandler; // forward declaration of the handler class.
using MyMessage = comms::Message<
    comms::option::MsgIdType<MyMsgId>, // add id() operation
    comms::option::ReadIterator<const std::uint8_t*>, // add read() operation
    comms::option::WriteIterator<std::uint8_t*> // add write() operation
    comms::option::Handler<MyHandler>, // add dispatch() operation
    comms::option::LengthInfoInterface, // add length() operation
    comms::option::ValidCheckInterface, // add valid() operation
    comms::option::LittleEndian // use little endian for serialisation
>;
```

For convenience the protocol messages should be redefined with appropriate
interface:
```cpp
using MyActualMessage1 = ActualMessage1<MyMessage>;
using MyActualMessage2 = ActualMessage2<MyMessage>;
...
```

