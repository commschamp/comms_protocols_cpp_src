# PAYLOAD Layer

Processing of the `PAYLOAD` is always the last stage in the protocol stack, when
all the other transport data was successfully handled, message object was
created and ready to read its field encoded in the PAYLOAD.

Such layer must receive type of the message interface class as a template
parameter and redefine read/write iterator types.
```cpp
namespace comms
{
template <typename TMessage>
class MsgDataLayer
{
public:

    // Define type of the message interface
    typedef TMessage Message;
    
    // Type of the pointer to message is not defined yet, will be defined in
    // the layer that processes message ID
    typedef void MsgPtr;

    // ReadIterator is the same as Message::ReadIterator if such exists, void 
    // otherwise.
    typedef typename std::conditional<
            Message::InterfaceOptions::HasReadIterator,
            typename TMessage::ReadIterator,
            void
        >::type ReadIterator;

    // WriteIterator is the same as Message::WriteIterator if such exists, void 
    // otherwise.
    typedef typename std::conditional<
            Message::InterfaceOptions::HasWriteIterator,
            typename TMessage::WriteIterator,
            void
        >::type WriteIterator;

    ...
};
} // namespace comms
```

The read/write operations just forward the request the message object.
```cpp
namespace comms
{
template <typename TMessage>
class MsgDataLayer
{
public:

    template <typename TMsgPtr>
    static ErrorStatus read(
        TMsgPtr& msgPtr,
        ReadIterator& iter,
        std::size_t len)
    {
        return msgPtr->read(iter, len);
    }

    static ErrorStatus write(
        const Message& msg,
        WriteIterator& iter,
        std::size_t len)
    {
        return msg.write(iter, len);
    }
};
} // namespace comms
```
Please note that `read()` member function expects to receive a reference to 
the smart pointer, which holds allocated message object, as the first parameter. 
The type of the pointer is not known yet. 
As the result type of such pointer is provided via
template parameter.
