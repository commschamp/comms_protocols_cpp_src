# Generalising Fields Implementation

The [automation](../fields/automation.md) of read/write operations of the message
required fields to expose predefined minimal interface:
```cpp
class SomeField
{
public:
    // Value storage type definition
    using ValueType = ...;
    
    // Provide an access to the stored value
    ValueType& value();
    const ValueType& value() const;

    // Read (deserialise) and update internal value
    template <typename TIter>
    ErrorStatus read(TIter& iter, std::size_t len);
    
    // Write (serialise) internal value
    template <typename TIter>
    ErrorStatus write(TIter& iter, std::size_t len) const;
    
    // Get the serialisation length
    std::size_t length() const;
    
private:
    ValueType m_value;
}
```

The read/write operations will probably require knowledge about the serialisation
endian used for the protocol. We need to come up
with the way to convey the endian information to the field classes. I would recommend
doing it by having common base class for all the fields:
```cpp
namespace comms
{
template <bool THasLittleEndian>
class Field
{
protected:
    // Read value using appropriate endian
    template <typename T, typename TIter>
    static T readData(TIter& iter) {...} 
    
    // Read partial value using appropriate endian
    template <typename T, std::size_t TSize, typename TIter>
    static T readData(TIter& iter) {...}

    // Write value using appropriate endian
    template <typename T, typename TIter>
    static void writeData(T value, TIter& iter) {...}

    // Write partial value using appropriate endian
    template <std::size_t TSize, typename T, typename TIter>
    static void writeData(T value, TIter& iter)
}
} // namespace comms
```

The choice of the right endian may be implemented using 
[Tag Dispatch Idiom](http://www.generic-programming.org/languages/cpp/techniques.php#tag_dispatching).
```cpp
namespace comms
{
template <bool THasLittleEndian>
class Field
{
protected:
    // Read value using appropriate endian
    template <typename T, typename TIter>
    static T readData(TIter& iter) 
    {
        // Dispatch to appropriate read function
        return readDataInternal<T>(iter, Tag());
    } 
    
    ...
private:
    BigEndianTag {};
    LittleEndianTag {};
    
    using Tag = typename std::conditional<
        THasLittleEndian,
        LittleEndianTag,
        BigEndianTag
    >::type;
    
    // Read value using big endian
    template <typename T, typename TIter>
    static T readBig(TIter& iter) {...} 

    // Read value using little endian
    template <typename T, typename TIter>
    static T readLittle(TIter& iter) {...} 

    // Dispatch to readBig()
    template <typename T, typename TIter>
    static T readDataInternal(TIter& iter, BigEndianTag) 
    {
        return readBig<T>(iter);
    } 

    // Dispatch to readLittle()
    template <typename T, typename TIter>
    static T readDataInternal(TIter& iter, LittleEndianTag) 
    {
        return readLittle<T>(iter);
    } 
};
} // namespace comms
```

Every field class should receive its base class as a template parameter and
may use available `readData()` and `writeData()` static member functions 
when serialising/deserialising internal value in `read()` and `write()` 
member functions.

For example:
```cpp
namespace comms
{
template <typename TBase, typename TValueType>
class IntValueField : public TBase
{
    using Base = TBase;
public:
    using ValueType = TValueType;
    ...
    template <typename TIter>
    ErrorStatus read(TIter& iter, std::size_t len)
    {
        if (len < length()) {
            return ErrorStatus::NotEnoughData;
        }
        
        m_value = Base::template read<ValueType>(iter);
        return ErrorStatus::Success;
    }
    
    template <typename TIter>
    ErrorStatus write(TIter& iter, std::size_t len) const
    {
        if (len < length()) {
            return ErrorStatus::BufferOverflow;
        }
        
        Base::write(m_value, iter);
        return ErrorStatus::Success
    }
    
    static constexpr std::size_t length()
    {
        return sizeof(ValueType);
    }
    
private:
    ValueType m_value
};
} // namespace comms
```

When the endian is known and fixed (for example when implementing third party 
protocol according to provided specifications), and there is little chance it's ever going
to change, the base class for all the fields may be explicitly defined:
```cpp
using MyProjField = comms::Field<false>; // Use big endian for fields serialisation
using MyIntField = comms::IntValueField<MyProjField>;
```

However, there may be the case when the endian information is not known up
front, and the one provided to the message interface definition (`comms::Message`) 
must be used. In this case, the message interface class may define common 
base class for all the fields:
```cpp
namespace comms
{
template <typename... TOptions>
class Message : public typename MessageInterfaceBuilder<TOptions...>::Type
{
    using Base = typename MessageInterfaceBuilder<TOptions...>::Type;
pablic:
    using ParsedOptions = typename Base::ParsedOptions ;
    using Field = comms::Field<ParsedOptions::HasLittleEndian>;
    ...
};
} // namespace comms
```

As the result the definition of the message's fields must receive a template
parameter of the base class for all the fields:
```cpp
template <typename TFieldBase>
using ActualMessage1Fields = std::tuple<
    comms::IntValueField<TFieldBase>,
    comms::IntValueField<TFieldBase>,
    ...
>:

template <typename TMsgInterface>
class ActualMessage1 : public 
    comms::MessageBase<
        comms::option::FieldsImpl<ActualMessage1Fields<typename TMsgInterface::Field> >,
        ...
    >
{
};    
```

## Multiple Ways to Serialise Fields

The [Common Field Types](../fields/common_types.md) chapter described most 
common types of fields with various serialisation and handling nuances,
which can be used to implement a custom communication protocol. 

Let's take the basic integer value field as an example. The most common way
to serialise it is just read/write its internally stored value as is. However,
there may be cases when serialisation takes limited number of bytes. Let's say,
the protocol specification states that some integer value consumes only 3 bytes
in the serialised bytes sequence. In this case the value will probably be be 
stored using `std::int32_t` or `std::uint32_t` type. The field class will also 
require different implementation of read/write/length functionality.

Another possible case is a necessity to add/subtract some predefined offset to/from the value
being serialised and subtracting/adding the same offset when the value is deserialised.
Good example of such case would be the serialisation of a **year** information,
which is serialised as an offset from year 2000 and consumes only 1 byte.
It is possible to store the value as a single byte 
(`comms::IntValueField<..., std::uint8_t>`), but it would be very inconvenient.
It is much better if we could store a normal year value (`2015`, `2016`, etc ...) 
using `std::uint16_t` type, but when serialising, the values that get written
are `15`, `16`, etc... **NOTE**, that such field requires two steps in its 
serialisation logic:

- add required offset (`-2000` in the example above)
- limit the number of bytes when serialising the result

Another popular way to serialise integer value is to use
[Base-128](https://en.wikipedia.org/wiki/Variable-length_quantity)
encoding. In this case the number of bytes in the serialisation sequence is
not fixed. 

What if some protocol decides to serialise the same offset from
year 2000, but using the **Base-128** encoding? It becomes obvious that
having a separate field class for every possible variant is impractical at
least. There must be a way to split the serialisation logic into small 
chunks, which can be applied one on top of another.

Using the same idea of the *options* and adapting the behaviour of the field 
class accordingly, we can generalise all the [fields](../fields/common_types.md) 
into a small subset of classes and make them also part of 
our generic library. 

The options described earlier may be defined using following option classes:
```cpp
namespace comms
{
namespace option
{
// Provide fixed serialisation length
template<std::size_t TLen>
struct FixedLength {};

// Provide numeric offset to be added to the value before serialisation
template<std::intmax_t TOffset>
struct NumValueSerOffset {};

// Force using variable length (base-128 encoding) while providing
// minimal and maximal allowed serialisation lengths.
template<std::size_t TMin, std::size_t TMax>
struct VarLength {};

} // namespace option
} // namespace comms
```

## Parsing the Options

In a very similar way to parsing options of the message interface (`comms::Message`) 
and message implementation (`comms::MessageBase`) described in earlier chapters,
we will create a struct, that will contain all the provided information to be
used later.
```cpp
namespace comms
{
template <typename... TOptions>
struct FieldParsedOptions;

template <>
struct FieldParsedOptions<>
{
    static const bool HasSerOffset = false;
    static const bool HasFixedLengthLimit = false;
    static const bool HasVarLengthLimits = false;
}

template <std::size_t TLen, typename... TOptions>
struct FieldParsedOptions<option::FixedLength<TLen>, TOptions...> : 
    public FieldParsedOptions<TOptions...>
{
    static const bool HasFixedLengthLimit = true;
    static const std::size_t FixedLengthLimit = TLen;
};

template <std::intmax_t TOffset, typename... TOptions>
struct FieldParsedOptions<option::NumValueSerOffset<TOffset>, TOptions...> : 
    public FieldParsedOptions<TOptions...>
{
    static const bool HasSerOffset = true;
    static const auto SerOffset = TOffset;
};

template <std::size_t TMinLen, std::size_t TMaxLen, typename... TOptions>
struct FieldParsedOptions<VarLength<TMinLen, TMaxLen>, TOptions...> : 
    public FieldParsedOptions<TOptions...>
{
    static const bool HasVarLengthLimits = true;
    static const std::size_t MinVarLength = TMinLen;
    static const std::size_t MaxVarLength = TMaxLen;
};
} // namespace comms
```

## Assemble the Required Functionality

Before parsing the options and assembling the right functionality there is a need
to start with basic integer value functionality:
```cpp
namespace comms
{
template <typename TFieldBase, typename TValueType>
class BasicIntValue : public TFieldBase
{
public: 
    using ValueType = TValueType;
    
    ... // rest of the interface
private:
    ValueType m_value;
};
} // namespace comms
```

Such field receives its base class and the type of the value it stores. The 
implementation of read/write/length functionalities are very basic and 
straightforward.

Now, we need to prepare various adaptor classes that will wrap or replace the
existing interface functions:
```cpp
namespace comms
{
template <std::intmax_t TOffset, typename TNext>
class SerOffsetAdaptor
{
public:
    ... // public interface
private:
    TNext m_next;
};

template <std::size_t TLen, typename TNext>
class FixedLengthAdaptor
{
public:
    ... // public interface
private:
    TNext m_next;
};

... // and so on
} // namespace comms
```
**NOTE**, that the adaptor classes above wrap one another (`TNext` template
parameter) and either replace or forward the read/write/length operations to the next
adaptor or final `BasicIntValue` class, instead of using inheritance as it was 
with message interface and implementation chunks. The overall architecture 
presented in this book doesn't require the field classes to exhibit polymorphic
behaviour. That's why using inheritance between adaptors is not necessary, although
not forbidden either. Using inheritance instead of containment has its pros and
cons, and at the end it's a matter of personal taste of what to use.

Now it's time to use the parsed options and wrap the `BasicIntValue` with
required adaptors:

Wrap with `SerOffsetAdaptor` if needed
```cpp
namespace comms
{
template <typename TField, typename TOpts, bool THasSerOffset>
struct AdaptBasicFieldSerOffset;

template <typename TField, typename TOpts>
struct AdaptBasicFieldSerOffset<TField, TOpts, true>
{
    using Type = SerOffsetAdaptor<TOpts::SerOffset, TField>;
};

template <typename TField, typename TOpts>
struct AdaptBasicFieldSerOffset<TField, TOpts, false>
{
    using Type = TField;
};
} // namespace comms
```

Wrap with `FixedLengthAdaptor` if needed
```cpp
namespace comms
{
template <typename TField, typename TOpts, bool THasFixedLength>
struct AdaptBasicFieldFixedLength;

template <typename TField, typename TOpts>
struct AdaptBasicFieldFixedLength<TField, TOpts, true>
{
    using Type = FixedLengthAdaptor<TOpts::FixedLength, TField>;
};

template <typename TField, typename TOpts>
struct AdaptBasicFieldFixedLength<TField, TOpts, false>
{
    using Type = TField;
};
} // namespace comms
```

And so on for all other possible adaptors.

Now, let's bundle all the required adaptors together:
```cpp
namespace comms
{
template <typename TBasic, typename... TOptions>
sturct FieldBuilder
{
    using  ParsedOptions = FieldParsedOptions<TOptions...>;
    
    using Field1 = typename AdaptBasicFieldSerOffset<
        TBasic, ParsedOptions, ParsedOptions::HasSerOffset>::Type;
    
    using Field2 = typename AdaptBasicFieldFixedLength<
        Field1, ParsedOptions, ParsedOptions::HasFixedLengthLimit>::Type;

    using Field3 = ...
    ...
    using FieldN = ...
    using Type = FieldN;
};
} // namespace comms
```

The final stage is to actually define final `IntValueField` type:
```cpp
namespace comms
{
template <typename TBase, typename TValueType, typename... TOptions>
class IntValueField
{
    using Basic = BasicIntValue<TBase, TValueType>;
    using Adapted = typename FieldBuilder<Basic, TOptions...>::Type;
public:
    using ValueType = typename Adapted::ValueType;
    
    // Just forward all the API requests to the adapted field.
    ValueType& value()
    {
        return m_adapted.value();
    }
    
    const ValueType& value() const
    {
        return m_adapted.value();
    }
    
    template <typename TIter>
    ErrorStatus read(TIter& iter, std::size_t len)
    {
        return m_adapted.read(iter, len);
    }
    
    ...
private:
    Adapted m_adapted;
};
} // namespace comms
```

The definition of the **year** field which is serialised using offset from
year `2000` may be defined as:
```cpp
using MyFieldBase = comms::Field<false>; // use big endian
using MyYear = comms::IntValueField<
    MyFieldBase,
    std::uint16_t, // store as 2 bytes unsigned value
    comms::option::NumValueSerOffset<-2000>,
    comms::option::FixedLength<1>
>;
```

## Other Options

In addition to options that regulate the read/write behaviour, there can be 
options which influence how the field is created and/or handled afterwards.

For example, there may be a need to set a specific value when the field object
is created (using default constructor). Let's introduce a new options for
this purpose:
```cpp
namespace comms
{
namespace option
{
template <typename T>
struct DefaultValueInitialiser{};
} // namespace option
} // namespace comms
```
The template parameter provided to this option is expected to be a class/struct
with the following interface:
```cpp
struct DefaultValueSetter
{
    template <typename TField>
    void operator()(TField& field) const
    {
        field.value() = ...; // Set the custom value
    }
}
```

Then the relevant adaptor class may set the default value of the field using the
provided setter class:
```cpp
namespace comms
{
template <typename TSetter, typename TNext>
class DefaultValueInitAdaptor 
{
public:
    using ValueType = typename TNext::ValueType;

    DefaultValueInitAdaptor()
    {
        TSetter()(*this);
    }
    
    ValueType& value()
    {
        return m_next.value();
    }
    
    ...
    
private:
    TNext m_next;
}; 
} // namespace comms
```

Please note, that both `comms::option::DefaultValueInitialiser` option and 
`DefaultValueInitAdaptor` adaptor class are completely generic, and
they can be used with any type of the field.

For numeric fields, such as `IntValueField` defined earlier, the generic library
may provide built-in setter class:
```cpp
namespace comms
{
template<std::intmax_t TVal>
struct DefaultNumValueInitialiser
{
    template <typename TField>
    void operator()(TField& field)
    {
        using FieldType = typename std::decay<TField>::type;
        using ValueType = typename FieldType::ValueType;
        field.value() = static_cast<ValueType>(TVal);
    }
};
} // namespace comms
```
And then, create a convenience alias to `DefaultValueInitialiser` option which
receives a numeric value as its template parameter and insures that the
field's value is initialised accordingly:
```cpp
namespace comms
{
namespace option
{
template<std::intmax_t TVal>
using DefaultNumValue = DefaultValueInitialiser<details::DefaultNumValueInitialiser<TVal> >;
} // namespace option
} // namespace comms
```

As the result, the making the **year** field to be default constructed with
value `2016` may look like this:
```cpp
using MyFieldBase = comms::Field<false>; // use big endian
using MyYear = comms::IntValueField<
    MyFieldBase,
    std::uint16_t, // store as 2 bytes unsigned value
    comms::option::NumValueSerOffset<-2000>,
    comms::option::FixedLength<1>,
    comms::option::DefaultNumValue<2016>
>;
```

## Other Fields

The [Common Field Types](../fields/common_types.md) chapter mentions multiple
other fields and several different ways to serialise them. I'm not going to 
describe each and every one of them here. Instead, I'd recommend taking a look
at the documentation of the 
[COMMS library](https://github.com/arobenko/comms_champion#comms-library) which
was implemented using ideas from this book. It will describe all the fields
it implements and their options.

## Eliminating Dynamic Memory Allocation

Fields like **String** or **List** may contain variable number of characters/elements. 
The default internal value storage type for such fields will probably be 
`std::string` or `std::vector` respectively. It will do the job, mostly. However,
they may not be suitable for bare-metal products that cannot use dynamic 
memory allocation and/or exceptions. In this case there must be a way to 
easily substitute these types with alternatives, such as custom `StaticString` or
`StaticVector` types.

Let's define a new option that will provide fixed storage size and will force
usage of these custom types instead of `std::string` and `std::vector`.
```cpp
namespace comms
{
namespace option
{
template <std::size_t TSize>
struct FixedSizeStorage {};
} // namespace option
} // namespace comms
```

The parsed option structure needs to be extended with new information:
```cpp
namespace comms
{
template <typename... TOptions>
struct FieldParsedOptions;

template <>
struct FieldParsedOptions<>
{
    ...
    static const bool HasFixedSizeStorage = false;
}

template <std::size_t TSize, typename... TOptions>
struct FieldParsedOptions<option::FixedSizeStorage<TSize>, TOptions...> : 
    public FieldParsedOptions<TOptions...>
{
    static const bool HasFixedSizeStorage = true;
    static const std::size_t FixedSizeStorage = TSize;
};

} // namespace comms
```
Now, let's implement the logic of choosing `StaticString` as the value storage
type if the option above is used and choosing `std::string` if not.
```cpp
// TOptions is a final variant of FieldParsedOptions<...>
template <typename TOptions, bool THasFixedStorage>
struct StringStorageType;

template <typename TOptions>
struct StringStorageType<TOptions, true>
{
    typedef comms::util::StaticString<TOptions::FixedSizeStorage> Type;
};

template <typename TOptions>
struct StringStorageType<TOptions, false>
{
    typedef std::string Type;
};

template <typename TOptions>
using StringStorageTypeT =
    typename StringStorageType<TOptions, TOptions::HasFixedSizeStorage>::Type;
```

`comms::util::StaticString` is the implementation of a string management class, 
which exposes the same public interface as `std::string`. It receives the fixed size
of the storage area as a template parameter, uses `std::array` or similar as its private
data member the store the string characters.

The implementation of the **String** field may look like this:
```cpp
template <typename TBase, typename... TOptions>
class StringField
{
public:
    // Parse the option into the struct
    using ParsedOptions = FieldParsedOptions<TOptions...>;
    
    // Identify storage type: StaticString or std::string
    using ValueType = StringStorageTypeT<ParsedOptions>;

    // Use the basic field and wrap it with adapters just like IntValueField earlier
    using Basic = BasicStringValue<TBase, ValueType>;
    using Adapted = typename FieldBuilder<Basic, TOptions...>::Type;
    
    // Just forward all the API requests to the adapted field.
    ValueType& value()
    {
        return m_adapted.value();
    }
    
    const ValueType& value() const
    {
        return m_adapted.value();
    }
    
    template <typename TIter>
    ErrorStatus read(TIter& iter, std::size_t len)
    {
        return m_adapted.read(iter, len);
    }
    
    ...
private:
    Adapted m_adapted;
};
} // namespace comms
```

As the result the definition of the message with a string field that doesn't 
use dynamic memory allocation may look like this:

```cpp
template <typename TFieldBase>
using ActualMessage3Fields = std::tuple<
    comms::StringField<TFieldBase, comms::option::FixedStorageSize<128> >,
    ...
>:

template <typename TMsgInterface>
class ActualMessage3 : public 
    comms::MessageBase<
        comms::option::FieldsImpl<ActualMessage3Fields<typename TMsgInterface::Field> >,
        ...
    >
{
};
```

And what about the case, when there is a need to create a message with a 
string field, but substitute the underlying default `std::string` type with 
`StaticString` **only** when compiling the bare-metal application? In this
case the `ActualMessage3` class may be defined to have additional template
parameter which will determine the necessity to substitute the storage type.
```cpp
template <bool THasFixedSize>
struct StringExtraOptions
{
    using Type = comms::option::EmtpyOption; // doesn't do anything
};

template <>
struct StringExtraOptions<false>
{
    using Type = comms::option::FixedStorageSize<128> >; // forces static storage
};

template <typename TFieldBase, bool THasFixedSize>
using ActualMessage3Fields = std::tuple<
    comms::StringField<TFieldBase, typename StringExtraOptions<THasFixedSize>::Type>,
    ...
>:

template <typename TMsgInterface, bool THasFixedSize = false>
class ActualMessage3 : public 
    comms::MessageBase<
        comms::option::FieldsImpl<ActualMessage3Fields<typename TMsgInterface::Field, THasFixedSize> >,
        ...
    >
{
};
```

Thanks to the fact that `StaticString` and `std::string` classes expose the
same public interface, the message handling function doesn't need to worry about
actual storage type. It just uses public interface of `std::string`:
```cpp
class MsgHandler 
{
public:
    void handle(ActualMessage3& msg)
    {
        auto& fields = msg.fields();
        auto& stringField = std::get<0>(fields);
        
        // The type of the stringVal is either std::string or StaticString
        auto& stringVal = stringField.value();
        if (stringVal == "string1") {
            ... // do something
        }
        else if (stringVal == "string2") {
            ... // do something else
        }
    }
};
```

Choosing internal value storage type for **List** fields to be 
`std::vector` or `StaticVector` is very similar.
