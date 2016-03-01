# Common Field Types

The majority of communication protocols use relatively small set of various
field types. However, the number of various ways used to serialise these fields
as well as handle them in different parts of the code may be significantly bigger.

It would be impractical to create a separate class for each and every variant
of the same type fields. That's why there is a need to use template parameters
when defining a frequently used field type. The basic example would be implementing
**numeric integral value** fields. Different fields of such type may have
different serialisation lengths.
```cpp
template <typename TValueType>
class IntValueField
{
public:
    using ValueType = TValueType;
    
    // Accessed stored value
    ValueType& value() { return m_value; }
    const ValueType& value() const { return m_value; }
    
    template <typename TIter>
    ErrorStatus read(TIter& iter, std::size_t len) {... /* read m_value */ }
    
    template <typename TIter>
    ErrorStatus write(TIter& iter, std::size_t len) const {... /* write m_value */ }
    
private:
    ValueType m_value = 0;    
};
```

Below is a description of most common fields used by majority of 
binary communication protocols with the list of possible variations that can
influence how the field is serialised and/or handled. 

The [Generic Library](../library/head.md) chapter will
concentrate on how to generalise development of any communication protocol by
creating a generic library and reusing it in independent implementations of
various protocols. It will also concentrate on creation of generic field 
classes for the types listed below.

## Numeric Integral Values

Used to operate with simple numeric integer values.

- May have different serialisation length: 1, 2, 3, 4, 5, ... bytes. Having
basic types of `std::uint8_t`, `std::uint16_t`, `std::uint32_t`, ... may be
not enough. Some extra work may be required to support lengths, such as 3, 5, 6,
7 bytes.
- May be signed and unsigned. Some protocols require different serialisation rules
for signed values, such as adding some predefined offset prior
to serialisation to make sure that the value being serialised is non-negative. When
value deserialised, the same offset must be subtracted to get the actual value.
- May have different serialisation length, based on the value being serialised,
such as having [Base-128](https://en.wikipedia.org/wiki/Variable-length_quantity)
encoding.

## Enumeration Values

Similar to [Numeric Integral Values](#numeric-integral-values), but storing
the value as enumeration type for easier access.

## Bitmask Values

Similar to [Numeric Integral Values](#numeric-integral-values), but with 
**unsigned** internal storage type and with each bit having separate meaning. 
The class definition should support having different serialisation lengths as
well as provide a convenient interface to inquire about and update various bits'
values.

## Strings

Some protocol serialise strings by prefixing the string itself with its size,
others have '\0' suffix to mark the end of the string. 
Some strings may be allocated a fixed size and require
'\0' padding if its actual length is shorter.

## Lists

There may be lists of raw bytes, list of other fields, or even a group of fields.
Similar to [Strings](#strings), the serialisation of lists may differ. Lists of
variable size may require a prefix with their size information. Other lists
may have fixed (predefined) size and will not require any additional size
information.

## Bundles

The group of fields sometimes needs to be bundled into a single entity and
be treated as a single field. The good example would be having a [list](#lists)
of complex structures (bundles).

## Bitfields

Similar to [Bundles](#bundles), where every field member takes only limited 
number of bits instead of bytes. Usually the members of the bitfields are
[Numeric Integral Values](#numeric-integral-values), 
[Enumeration Values](#enumeration-values), and
[Bitmask Values](#bitmask-values)

## Common Variations

All the fields stated above may require an ability to:

- set custom default value when the field object is created.
- have custom value validation logic.
- fail the read operation on invalid value.
- ignore the incoming invalid value, i.e. not to fail the read operation, but
preserve the existing value if the value being read is invalid.