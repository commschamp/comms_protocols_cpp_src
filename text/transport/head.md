[[transport-transport]]
# Transport

In addition to definition of the messages and their contents, every 
communication protocol must ensure that the message is successfully delivered 
over the I/O link to the other side. The serialised message payload must be 
wrapped in some kind of transport information, which usually depends on the
type and reliability of the I/O link being used. For example, protocols that
are designed to be used over TCP/IP connection, such as [MQTT](http://mqtt.org), 
may omit the whole packet synchronisation and checksum logic, because TCP/IP
connection ensures that the data is delivered correctly. Such protocols are
usually defined to use only message ID and remaining size information to wrap
the message payload:
```
ID | SIZE | PAYLOAD
```

Other protocols may be designed to be used over less reliable RS-232 link, 
which may require a bit better protection against data loss or corruption:
```
SYNC | SIZE | ID | PAYLOAD | CHECKSUM
```

The number of most common types of the wrapping "chunks" is quite small. 
However, different protocols may have different rules of how these values are 
serialised. Very similar to [Fields](../fields/head.md).  

The main logic of processing the incoming raw data remains the same for 
all the protocols, though. It is to read and process the transport information "chunks" one
by one:
- SYNC - check the next one or more bytes for an expected predefined value.
If the value is as expected proceed to the next "chunk". If not, drop one
byte from the front of the incoming data queue and try again.
- SIZE - compare the remaining expected data length against actually available. If 
there is enough data, proceed to the next "chunk". If not report, to the 
caller, that more data is required.
- ID - read the message ID value and create appropriate message object, then
proceed to the next "chunk".
- PAYLOAD - let the created message object to read its payload data.
- CHECKSUM - read the expected checksum value and calculate the actual one. If
the checksums don't match, discard the created message and report error.

Each and every "chunk" operates independently, regardless of what information
was processed before and/or after it. When some operation seems to repeat itself
several times, it should be generalised and become part of our 
[Generic Library](../library/head.md).

So, how is it going to be implemented? My advice is to use independent "chunk"
classes, that expose predefined interface, wrap one another, and forward
the requested operation to the next "chunk" when appropriate. As was stated
earlier, the transport information values are very similar to 
[Fields](../fields/head.md), which immediately takes us to the direction of
reusing [Field](../fields/head.md) classes to handle these values:
```cpp
template <typename TField, typename TNextChunk, ... /* some other template parameters */>
class SomeChunk
{
public:
    // Iterator used for reading
    using ReadIterator = typename TNextChunk::ReadIterator;
    
    // Iterator used for writing
    using WriteIterator = typename TNextChunk::WriteIterator;
    
    // Type of the common message interface class
    using Message = typename TNextChunk::Message;
    
    // Smart pointer used to hold newly created message object
    using MsgPtr = typename TNextChunk::MsgPtr;
    
    ErrorStatus read(MsgPtr& msg, ReadIterator& iter, std::size_t len) 
    {
        TField field;
        auto es = field.read(iter, len);
        if (es != ErrorStatus::Success) {
            return es;
        }
        
        ... process field value.
        return m_next.read(msg, iter, len - field.length());
    }
    
    ErrorStatus write(const Message& msg, WriteIterator& iter, std::size_t len)
    {
        TField field;
        field.value() = ...; // set required value
        auto es = field.write(iter, len);
        if (es != ErrorStatus) {
            return es;
        }
        return m_next.write(msg, iter, len - field.length());
    }
    
private:
    TNextChunk m_next;
}
```
Please note that `ReadIterator` and `WriteIterator` are taken from the next
chunk. One of the chunks, which is responsible for processing the `PAYLOAD` will
receive the class of the message interface as a template parameter, will
retrieve the information of the iterators' types, and redefine them as its
internal types. Also, this class will define the type of the message interface as
its internal `Message` type. 
All other wrapping chunk classes will reuse the same information.

Also note, that one of the chunks will have to define pointer to the created
message object (`MsgPtr`). Usually it is the chunk that is responsible to process `ID`
value. 

The sequential processing the transport information "chunks", and stripping
them one by one before proceeding to the next one, may remind of 
[OSI Conceptual Model](https://en.wikipedia.org/wiki/OSI_model), where
a layer serves the layer above it and is served by the layer below it. 

From now on, I will use a term **layer** instead of the **chunk**. 
The combined bundle of such layers will be called
**protocol stack** (of layers).

Let's take a closer look at all the layer types mentioned above.

