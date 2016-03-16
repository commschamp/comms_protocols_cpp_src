# SYNC Layer

This layer is responsible to find and validate the synchronisation prefix.

```cpp
namespace comms
{
// TField is type of the field used to read/write SYNC prefix
// TNext is the next layer this one wraps
template <typename TField, typename TNext>
class SyncPrefixLayer
{
public:
    // Type of the field object used to read/write SYNC prefix.
    typedef TField Field;
    
    // Take type of the ReadIterator from the next layer
    typedef typename TNext::ReadIterator ReadIterator;

    // Take type of the WriteIterator from the next layer
    typedef typename TNext::WriteIterator WriteIterator;

    // Take type of the message interface from the next layer
    typedef typename TNext::Message Message;
    
    // Take type of the message interface pointer from the next layer
    typedef typename TNext::MsgPtr MsgPtr; 
    
    template <typename TMsgPtr>
    ErrorStatus read(TMsgPtr& msgPtr, ReadIterator& iter, std::size_t len)
    {
        Field field;
        auto es = field.read(iter, size);
        if (es != ErrorStatus::Success) {
            return es;
        }

        if (field.value() != Field().value()) {
            // doesn't match expected
            return ErrorStatus::ProtocolError;
        }

        return m_next.read(msgPtr, iter, size - field.length());
    } 
    
    ErrorStatus write(const Message& msg, WriteIterator& iter, std::size_t len) const
    {
        Field field;
        auto es = field.write(iter, size);
        if (es != ErrorStatus::Success) {
            return es;
        }
        return m_next.write(msg, iter, size - field.length());
    }   
    
    std::size_t length(const TMessage& msg) const
    {
        return Field().length() + m_next.length(msg);
    }
    
private:
    TNext m_next;
};
} // namespace comms
```

Note, that the value of the `SYNC` prefix is expected to be equal to the value
of the default constructed `TField` field type. The default construction value
may be set using `comms::option::DefaultNumValue` option described in
[Generalising Fields Implementation](../library/fields.md) chapter.

For example, 2 bytes synchronisation prefix `0xab 0xcd` with big endian
serialisation may be defined as:
```cpp
using CommonFieldBase = comms::Field<false>; // big endian serialisation base
using SyncPrefixField = 
    comms::IntValueField<
        CommonFieldBase,
        std::uint16_t,
        comms::option::DefaultNumValue<0xabcd>
    >;
```
