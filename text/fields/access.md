# Working With Fields

In order to [automate](automation.md) some basic operations, all the fields
had to provide the same basic interface. As the result the actual field values
had to be wrapped in a class that defines the required public interface.
Such class must also provide means to access/update the wrapped value.
For example:
```cpp
class SomeField
{
public:
    // Value storage type definition
    typedef ... ValueType;
    
    // Provide an access to the stored value
    ValueType& value() { return m_value; }
    const ValueType& value() const { return m_value; }

    ...
private:
    ValueType m_value;
}
```

Let's assume the `ActualMessage1` defines 3 integer value fields with 1, 2, and
4 serialisation lengths respectively.
```cpp
using ActualMessage1Fields = std::tuple<
    IntValue<std::int8_t>,
    IntValue<std::int16_t>
    IntValue<std::int32_t>
>;
class ActualMessage1 : public MessageBase<ActualMessage1Fields> {...};
```

The [Dispatching and Handling](../message/dispatch_handle.md) chapter described
the efficient way to dispatch message object to its handler. The appropriate
handling function may access its field's value using the following code 
flow:
```cpp
class Handler
{
public:
    void handle(ActualMessage1& msg)
    {
        // Get access to the field's bundle of type std::tuple
        auto& allFields = msg.fields();
        
        // Get access to the field abstractions.
        auto& field1 = std::get<0>(allFields);
        auto& field2 = std::get<1>(allFields);
        auto& field3 = std::get<2>(allFields);
        
        // Get access to the values themselves:
        std::int8_t val1 = field1.value();
        std::int16_t val2 = field2.value();
        std::int32_t val3 = field3.value();
        
        ... Do something with retrieved values
    }
};
```

When preparing message to send the similar code sequence may be applied to
update the values:
```cpp
ActualMessage1 msg;

// Get access to the field's bundle of type std::tuple
auto& allFields = msg.fields();

// Get access to the field abstractions.
auto& field1 = std::get<0>(allFields);
auto& field2 = std::get<1>(allFields);
auto& field3 = std::get<2>(allFields);

// Update the values themselves:
field1.value() = ...;
field2.value() = ...;
field3.value() = ...;

// Serialise and send the message:
sendMessage(msg);
```
