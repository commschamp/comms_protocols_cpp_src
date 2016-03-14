# ID Layer

The job of this layer is handle the message ID information. 

- When new message
is received, appropriate message object needs to be created, prior to 
invoking read operation of the next (wrapped) layer. 

- When any message
is about to get sent, just get the ID information from the message object and
serialise it prior to invoking the write operation of the next layer.

The code of the layer is pretty straightforward:
```cpp
namespace comms
{
// TField is type of the field used to read/write message ID
// TNextLayer is the next layer this one wraps
template <typename TField, typename TNextLayer, ... /* other parameters */>
class MsgIdLayer
{
public:
    // Type of the field object used to read/write message ID value.
    typedef TField Field;
    
    // Take type of the ReadIterator from the next layer
    typedef typename TNext::ReadIterator ReadIterator;

    // Take type of the WriteIterator from the next layer
    typedef typename TNext::WriteIterator WriteIterator;

    // Take type of the message interface from the next layer
    typedef typename TNext::Message Message;
    
    // Type of the message ID
    typedef typename Message::MsgIdType MsgIdType;
    
    // Redefine pointer to message type:
    typedef ... MsgPtr;
    
    ErrorStatus read(MsgPtr& msgPtr, ReadIterator& iter, std::size_t len)
    {
        Field field;
        auto es = field.read(iter, size);
        if (es != ErrorStatus::Success) {
            return es;
        }
        
        msgPtr = createMessage(field.value());
        if (!msgPtr) {
            // Unknown ID
            return ErrorStatus::InvalidMsgId;
        } 
        
        es = m_next.read(iter, len - field.length());
        if (es != ErrorStatus::Success) {
            msgPtr.reset(); // Discard allocated message;
        } 
        
        return es;
    }
    
    ErrorStatus write(const Message& msg, WriteIterator& iter, std::size_t len) const
    {
        Field field;
        field.value() = msg.id();
        auto es = field.write(iter, len);
        if (es != ErrorStatus::Success) {
            return es;
        }
        
        return m_next.write(msg, iter, len - field.length());
    }
private:
    MsgPtr createMsg(MsgIdType id)
    {
        ... // TODO: create message object
    }
    TNext m_next;
};
} // namespace comms
```

To properly finalise the implementation above we need to resolve two main challenges:
- Implement `createMsg()` function which receives ID of the message and 
creates the message object.
- Define the `MsgPtr` smart pointer type, which is responsible to hold the
allocated message object. The easiest way is to define it to be 
`std::unique_ptr<Message>`. The main problem with it is usage of dynamic memory
allocation. Bare metal platform may not have such luxury. There must be a 
way to support "in place" allocation instead of usage of dynamic memory.

## Creating Message Object

Let's start with creation of proper message object given the **numeric** message ID. 
Its must be as efficient as possible.

In many cases the IDs of the messages are sequential ones and defined using 
some enumeration type.
```cpp
enum MsgId
{
    MsgId_Message1,
    MsgId_Message2,
    ...
    MsgId_NumOfMessages
};
```

Let's assume that we have `FactoryMethod` class with polymorphic `createMsg()`
function that returns allocated message object wrapped in a `MsgPtr` smart pointer.
```cpp
class FactoryMethod
{
public:
    MsgPtr createMsg() const
    {
        return createMsgImpl();
    }

protected:
    virtual MsgPtr createMsgImpl() const = 0;
};
```

In this case, the most efficient way is to have an array of pointers to
polymorphic class `FactoryMethod`. The index of the array cell corresponds to 
a message ID.

![Image: Message Factory](../image/msg_factory.png)

The code of `MsgIdLayer::createMsg()` function is quite simple:
```cpp
namespace comms
{
template <...>
class MsgIdLayer
{
private:
    MsgPtr createMsg(MsgIdType id)
    {
        auto& registry = ...; // reference to the array of pointers to FactoryMethod-s
        if ((registry.size() <= id) ||
            (registry[id] == nullptr)){
            return MsgPtr();
        }
        
        return registry[id]->createMsg();
    }
};
} // namespace comms
```
The runtime [complexity](https://en.wikipedia.org/wiki/Time_complexity) 
of such code is `O(1)`.

However, there are many protocols that their ID map is quite sparse and it is 
impractical to use an array for direct mapping:
```cpp
enum MsgId
{
    MsgId_Message1 = 0x0101,
    MsgId_Message2 = 0x0205,
    MsgId_Message3 = 0x0308,
    ...
    MsgId_NumOfMessages
};
```
In this case the array of `FactoryMethod`s described earlier must be packed and
[binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm) algorithm
used to find required method. To support the search the `FactoryMethod` must
be able to report ID of the messages it creates.
```cpp
class FactoryMethod
{
public:
    MsgIdType id() const
    {
        return idImpl();
    }
    
    MsgPtr createMsg() const
    {
        return createMsgImpl();
    }

protected:
    virtual MsgIdType idImpl() const = 0;
    virtual MsgPtr createMsgImpl() const = 0;
};
```

Then the code of `MsgIdLayer::createMsg()` needs to apply binary search to find
the required method:
```cpp
namespace comms
{
template <...>
class MsgIdLayer
{
private:
    MsgPtr createMsg(MsgIdType id)
    {
        auto& registry = ...; // reference to the array of pointers to FactoryMethod-s
        auto iter = 
            std::lower_bound(
                registry.begin(), registry.end(), id, 
                [](FactoryMethod* method, MsgIdType idVal) -> bool 
                {
                    return method->id() < id;
                });
                
        if ((iter == registry.end()) ||
            ((*iter)->id() != id)) {
            return MsgPtr();
        }
                
        return (*iter)->createMsg();
    }
};
} // namespace comms
```
Note, that [std::lower_bound](http://en.cppreference.com/w/cpp/algorithm/lower_bound) 
algorithm requires `FactoryMethod`s in the 
"registry" to be sorted by the message ID. The runtime 
[complexity](https://en.wikipedia.org/wiki/Time_complexity) 
of such code is `O(log(n))`, where `n` is size of the registry.

Some communication protocols define multiple variants of the same message, which
are differentiated by some other means, such as serialisation length of the message.
It may be convenient to implement such variants as separate message classes, which
will require separate `FactoryMethod`s to instantiate them. In this case,
the `MsgIdLayer::createMsg()` function may use 
[std::equal_range](http://en.cppreference.com/w/cpp/algorithm/equal_range)
algorithm instead of 
[std::lower_bound](http://en.cppreference.com/w/cpp/algorithm/lower_bound), and
use additional parameter to specify which of the method to pick from the equal
range found:
```cpp
namespace comms
{
template <...>
class MsgIdLayer
{
private:
    MsgPtr createMsg(MsgIdType id, unsigned idx = 0)
    {
        auto& registry = ...; // reference to the array of pointers to FactoryMethod-s
        auto iters = std::equal_range(...);
                
        if ((iters.first == iters.second) ||
            (iters.second < (iters.first + idx))) {
            return MsgPtr();
        }
                
        return (*(iter.first + idx))->createMsg();
    }
};
} // namespace comms
```

Please note, that `MsgIdLayer::read()` message also needs to be modified to
support multiple attempts to create message object with the same id, by just
incrementing the `idx` parameter passed to `createMsg()` member function when
read operation fails before reporting error to the caller. I will leave it as
an exercise to the reader.

To complete the message allocation subject we need to come up with an automatic
way to create the registry of `FactoryMethod`s used earlier. Please remember,
that `FactoryMethod` was just a polymorphic interface. We need to implement
actual method that implement the virtual functionality.
```cpp
template <typename TActualMessage>
class ActualFactoryMethod : public FactoryMethod
{
protected:
    virtual MsgIdType idImpl() const
    {
        return TActualMessage::ImplOptions::MsgId;
    }
    
    virtual MsgPtr createMsgImpl() const
    {
        return MsgPtr(new TActualMessage());
    }
}
```
Note that the code above assumes that `comms::option::StaticNumIdImpl`
option (described in 
[Generalising Message Implementation](../library/impl.md) chapter)
was used to specify numeric message ID
when defining the `ActualMessage*` class.

Also note that the example above uses dynamic memory allocation to allocate
actual message object. This is just for idea demonstration purposes. The
[Allocating Message Object](#allocating-message-object) section below will
describe how to support "in-place" allocation.

The types of the messages that can be received over I/O link are usually known
at compile time. If we bundle them together in `std::tuple`, it is easy to
apply already familiar meta-programming technique of iterating over the provided
types and instantiate proper `ActualFactoryMethod<>` object.
```cpp
using AllMessages = std::tuple<
    ActualMessage1, 
    ActualMessage2,
    ActualMessage3,
    ...
>;
```

The size of the *registry* can easily be identified using 
[std::tuple_size](http://en.cppreference.com/w/cpp/utility/tuple/tuple_size).
```cpp
static const RegistrySize = std::tuple_size<AllMessages>::value;
using Registry = std::array<FactoryMethod*, RegistrySize>;
Registry m_registry; // registry object
```

Now it's time to iterate (at compile time) over all the types defined in 
the `AllMessages` tuple and create separate `ActualFactoryMethod<>` for
each and every one of them. Remember [tupleForEach](../appendix/a.md)? We need
something similar here, but missing the tuple object itself. We are just iterating
over types, not the elements of the tuple object. We'll call it 
`tupleForEachType()`. See [Appendix D](../appendix/d.md) for implementation details.

We also require a functor class that will be invoked for every message type and
will be responsible to fill the provided registry:
```cpp
class MsgFactoryCreator
{
public:
    MsgFactoryCreator(Registry& registry)
      : registry_(registry)
    {
    }

    template <typename TMessage>
    void operator()()
    {
        static const ActualFactoryMethod<TMessage> Factory;
        registry_[idx_] = &Factory;
        ++idx_;
    }

private:
    Registry& registry_;
    unsigned idx_ = 0;
};
```

The initialisation function may be as simple as:
```cpp
void initRegistry()
{
    tupleForEachType<AllMessages>(MsgFactoryCreator(m_registry));
}
```

NOTE, that `ActualFactoryMethod<>` factories do not have any internal state and 
are defined as static objects. It is safe just to store pointers to them in
the *registry* array.

To summarise this section, let's redefine `comms::MsgIdLayer` and add
the message creation functionality.
```cpp
namespace comms
{
// TField is type of the field used to read/write message ID
// TAllMessages is all messages bundled in std::tuple.
// TNextLayer is the next layer this one wraps
template <typename TField, typename TAllMessages, typename TNextLayer>
class MsgIdLayer
{
public:
    // Type of the field object used to read/write message ID value.
    typedef TField Field;
    
    // Take type of the ReadIterator from the next layer
    typedef typename TNext::ReadIterator ReadIterator;

    // Take type of the WriteIterator from the next layer
    typedef typename TNext::WriteIterator WriteIterator;

    // Take type of the message interface from the next layer
    typedef typename TNext::Message Message;
    
    // Type of the message ID
    typedef typename Message::MsgIdType MsgIdType;
    
    // Redefine pointer to message type:
    typedef ... MsgPtr;
    
    // Constructor
    MsgIdLayer()
    {
        tupleForEachType<AllMessages>(MsgFactoryCreator(m_registry));
    }

    // Read operation
    ErrorStatus read(MsgPtr& msgPtr, ReadIterator& iter, std::size_t len) {...}
    
    // Write operation
    ErrorStatus write(const Message& msg, WriteIterator& iter, std::size_t len) const {...}

private:
    class FactoryMethod {...};
    
    template <typename TMessage>
    class ActualFactoryMethod : public FactoryMethod {...};
    
    class MsgFactoryCreator {...};

    // Registry of Factories
    static const auto RegistrySize = std::tuple_size<TAllMessages>::value;
    typedef std::array<FactoryMethod*, RegistrySize> Registry;


    // Create message
    MsgPtr createMsg(MsgIdType id, unsigned idx = 0)
    {
        auto iters = std::equal_range(m_registry.begin(), m_registry.end(), ...);
        ...
    }

    Registry m_registry;
    TNext m_next;
    
};
} // namespace comms
```

## Allocating Message Object

