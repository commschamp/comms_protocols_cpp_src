# Defining Protocol Stack

To summarise the protocol [transport](head.md) wrapping subject,
let's define a custom protocol that wraps the message payload in the following way:
```
SYNC | SIZE | ID | PAYLOAD | CHECKSUM 
```
where:

- All the fields are serialised using BIG endian.
- SYNC - 2 bytes of synchronisation value to indicate beginning of the message, 
must be "0xab 0xcd"
- SIZE - 2 bytes, length of remaining data **including checksum** and not 
including SIZE field itself.
- ID - 1 byte, numeric ID of the message.
- PAYLOAD - any number of bytes, serialised message data
- CHECKSUM - 2 bytes, arithmetic summary of all bytes starting (and including) 
from SIZE field and ending after PAYLOAD field.

The protocol layer should wrap one another in the following way:

![Image: Protcols Stack](../image/protocol_stack.png)

Please note, that `CHECKSUM` layer doesn't wrap `SYNC` because synchronisation
prefix is not included in the checksum calculation.

## Common Protocol Definitions

```cpp
// BIG endian fields serialisation
using MyField = comms::Field<false>; 

// Message IDs
enum MyMsgId : std::uint16_t
{
    MyMsgId_ActualMessage1,
    MyMsgId_ActualMessage2,
    MyMsgId_ActualMessage3,
    ...
};

// Forward declaration of MyHandler handling class
class MyHandler;

// Message interface
using MyMessage = 
    comms::Message<
        comms::option::MsgIdType<MyMsgId>
        comms::option::ReadIterator<const std::uint8_t*>
        comms::option::WriteIterator<std::back_insert_iterator<std::vector<std::uint8_t> > >,
        comms::option::LengthInfoInterface,
        comms::option::Handler<MyHandler>
    >;
    
// Definition of the messages
class ActualMessage1Fields = std::tuple<...>;
template <typename TMessage>
class ActualMessage1 : public
    comms::MessageBase<
        comms::option::StaticNumIdImpl<MyMsgId_ActualMessage1>,
        comms::option::FieldsImpl<ActualMessage1Fields>,
        comms::option::DispatchImpl<ActualMessage1<TMessage>
    >
{
};

class ActualMessage2Fields = std::tuple<...>;
template <typename TMessage>
class ActualMessage1 : public comms::MessageBase<...> {};
...

// Bundling all messages together
template <typename TMessage>
using AllMessages = std::tuple<
    ActualMessage1<TMessage>,
    ActualMessage2<TMessage>,
    ...
>;
```

## PAYLOAD Layer

```cpp
using MyPayloadLayer = comms::MsgDataLayer<MyMessage>;
```

## ID Layer

```cpp
using MyMsgIdField = comms::EnumValueField<MyField, MyMsgId>
using MyIdLayer = comms::MsgIdLayer<MyMsgIdField, AllMessages, MyPayloadLayer>;
```

## SIZE Layer

```cpp
using MySizeField = 
    comms::IntValueField<
        MyField, 
        std::uint16_t,
        comms::option::NumValueSerOffset<sizeof(std::uint16_t)>
    >; 
using MySizeLayer = comms::MsgSizeLayer<MySizeField, MyIdLayer>;
```
Please note, that `SIZE` field definition uses `comms::option::NumValueSerOffset`
option, which effectively adds `2` when size value is serialised and 
subtracts it when remaining length is deserialised. It must be done, because
`SIZE` value specifies number of remaining bytes including the `CHECKSUM` value at the end.

## CHECKSUM Layer

```cpp
using MyChecksumField = comms::IntValueField<MyField, std::uint16_t>;
using MyChecksumLayer = comms::ChecksumLayer<
    MyChecksumField,
    comms::BasicSum<std::uint16_t>
    MySizeLayer
>;
```

## SYNC Layer

```cpp
using MySyncField = comms::IntValueField<
    MyField, 
    std::uint16_t, 
    comms::option::DefaultNumValue<0xabcd> 
>;
using MySyncPrefix = comms::SyncPrefixLayer<SyncField, MyChecksumLayer>;
```

## Processing Loop

The outermost layer defines a full protocol stack. It should be typedef-ed to avoid any confusion: 
```cpp
using MyProtocolStack MySyncPrefix;
```

The processing loop may look like this:
```cpp
// Protocol stack
MyProtocolStack protStack;

// Message handler object
MyHandler handler;

// Input data storage, the data received over I/O link is appended here
std::vector<std::uint8_t> inData;

void processData()
{
    while (!inData.empty()) {
        MyProtocolStack::ReadIterator readIter = &inData[0];
        MyProtocolStack::MsgPtr msg;
        auto es = protStack.read(msg, readIter, inData.size());
        if (es == comms::ErrorStatus::NotEnoughData) {
            // More data is required;
            return;
        }
        if (es == comms::ErrorStatus::Success) {
            assert(msgPtr); // Must hold the valid message object
            msgPtr->dispatch(handler); // Process message, dispatch to handling function
            
            // Erase consumed bytes from the buffer
            auto consumedBytes = 
                std::distance(ProtocolStack::ReadIterator(&inData[0]), readIter);
            inData.erase(inData.begin(), inData.begin() + consumedBytes);
            continue;
        }
        // Some error occurred, pop only one first byte and try to process again
        inData.erase(inData.begin());
    }
}
```
The processing loop above is not the most efficient one, but it demonstrates 
what needs to be done and how our generic library can be used in order to identify 
and process the received message.

## Writing Message

The write logic is even simpler. 
```cpp
void sendMessage(const MyMessage& msg)
{
    // Output buffer
    std::vector<std::uint8_t> outData; 
    // Reserve enough space in output buffer
    outData.reserve(protStack.length(msg)); 
    auto writeIter = std::back_inserter(outData);
    auto es = protStack.write(msg, writeIter, outData.max_size());
    if (es = comms::ErrorStatus::UpdateRequired) {
        auto updateIter = &outData[0];
        es = protStack.update(updateIter, outData.size());
    }
    assert(es == comms::ErrorStatus::Success); // No error is expected
    
    ... // Send written data over I/O link
}
```
