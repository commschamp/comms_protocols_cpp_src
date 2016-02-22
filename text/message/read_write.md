# Reading and Writing

When new raw data bytes are received over some I/O link, they need to be 
deserialised into the custom message object, then dispatched to an appropriate
handling function. When continuing **message as an object** concept, expressed in
previous section, it becomes convenient to make a reading/writing functionality
a responsibility of the message object itself.

```cpp
class Message 
{
public:
    ErrorStatus read(...) {
        return readImpl(...);
    }
    
    ErrorStatus write(...) const {
        return writeImpl(...);
    }
    ...
    
protected:
    // Implements reading from the buffer functionality
    virtual ErrorStatus readImpl(...) = 0; 
    
    // Implements writing to a buffer functionality
    virtual ErrorStatus writeImpl(...) const = 0;
};

class ActualMessage1 : public Message 
{
    ...
protected:
    virtual ErrorStatus readImpl(...) override {...}; 
    virtual ErrorStatus writeImpl(...) const override {...};
};

class ActualMessage2 : public Message 
{
    ...
protected:
    virtual ErrorStatus readImpl(...) override {...}; 
    virtual ErrorStatus writeImpl(...) const override {...};
};
```

There is obviously a need to know success/failure status of the read/write operation. 
The `ErrorStatus` return value may be defined for example like this:
```cpp
enum class ErrorStatus
{
    Success,
    NotEnoughData, 
    BufferOverflow,
    InvalidMsgData,
    ...
};
```

Let's assume that at the stage of parsing transport wrapping information, the
ID of the message was retrieved and appropriate actual message object was
created in an **efficient** way. This whole process will be described later in
the [Transport](../transport/head.md) chapter.

Once the appropriate message object was created and returned in some kind of
smart pointer, just call the `read(...)` member function of the message object:

```cpp
using MsgPtr = std::unique_ptr<Message>;

MsgPtr msg = processTransportData(...)
auto es = msg.read(...); // read message payload
if (es != ErrorStatus::Success) {
    ... // handle error
}

```

## Data Structures Independence

One of our [goals](../intro/goal.md) was to make the implementation
of the communication protocol to be system independent. 
In order to make the code as generic as possible we have to eliminate any dependency
on specific data structures, where the incoming raw bytes are stored before being
processed, as well as outgoing data before being sent out. 

The best way to achieve such independence is to use 
[iterators](http://en.cppreference.com/w/cpp/concept/Iterator) instead of
specific data structures and make it a responsibility of the caller to maintain
appropriate buffers:
```cpp
template <typename TReadIter, typename TWriteIter>
class Message
{
public:
    typedef TReadIter ReadIterator;
    typedef TWriteIter WriteIterator;
    
    ErrorStatus read(ReadIterator& iter, std::size_t len) {
        return readImpl(iter, len);
    }
    
    ErrorStatus write(WriteIterator& iter, std::size_t len) const {
        return writeImpl(iter, len);
    }
    ...
    
protected:
    // Implements reading from the buffer functionality
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) = 0; 
    
    // Implements writing to a buffer functionality
    virtual ErrorStatus writeImpl(WriteIterator& iter, std::size_t len) const = 0;
};

template <typename TReadIter, typename TWriteIter>
class ActualMessage1 : public Message<TReadIter, TWriteIter>
{
    typedef Message<TReadIter, TWriteIter> Base;
public:
    using Base::ReadIterator;
    using Base::WriteIterator;
    ...
    
protected:
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) override {...}; 
    virtual ErrorStatus writeImpl(WriteIterator& iter, std::size_t len) const override {...};
};

```
Please note, that iterators are passed by reference, which allows the increment
and assignment operations required to implement serialisation/deserialisation
functionality.

Also note that the same implementation of the read/write operations can be used
in any system with any restrictions. For example, the bare-metal embedded system
cannot use dynamic memory allocation and must serialise the outgoing messages
into a static array, which forces the definition of the write iterator to be
`std::uint8_t*`. 
```cpp
using EmbReadIter = const std::uint8_t*;
using EmbWriteIter = std::uint8_t*; 
using EmbMessage = Message<EmbReadIter, EmbWriteIter>
using EmbActualMessage1 = ActualMessage1<EmbReadIter, EmbWriteIter>
using EmbActualMessage2 = ActualMessage2<EmbReadIter, EmbWriteIter>

std::array<std::uint8_t, 1024> outBuf;
EmbWriteIter iter = &outBuf[0];

EmbActualMessage1 msg;
msg.write(iter, outBuf.size());
auto writtenCount = std::distance(&outBuf[0], iter); // iter was incremented
```

The Linux server system which resides on the other end of
the I/O link doesn't have such limitation and uses `std::vector<std::uint8_t>`
to store outgoing serialised messages. The generic and data structures independent 
implementation above makes it possible to be reused:

```cpp
using LinReadIter = const std::uint8_t*;
using LinWriteIter = std::back_insert_iterator<std::vector<std::uint8_t> >; 
using LinMessage = Message<LinReadIter, LinWriteIter>
using LinActualMessage1 = ActualMessage1<LinReadIter, LinWriteIter>
using LinActualMessage2 = ActualMessage2<LinReadIter, LinWriteIter>

std::vector<std::uint8_t> outBuf;
LinWriteIter iter = std::back_inserter(outBuf);

LinActualMessage1 msg;
Lin.write(iter, outBuf.max_size());
auto writtenCount = outBuf.size();

```

## Data Serialisation

The `readImpl()` and `writeImpl()` member functions of the actual message
class are supposed to properly serialise and deserialise message fields. It is
a good idea to provide some common serialisation functions accessible by the
actual message classes.
```cpp
template <typename TReadIter, typename TWriteIter>
class Message
{
public:
    typedef TReadIter ReadIterator;
    typedef TWriteIter WriteIterator;
    
    ...
    
protected:

    template <typename T>
    static T readData(ReadIterator& iter) {...}
    
    template <typename T>
    static void writeData(T value, WriteIterator& iter) {...}
};
```

The `readData()` and `writeData()` static member functions above are responsible
to implement the serialisation and deserialisation of the values using the
right endian. 

Depending on a communication protocol there may be a need to serialise only part 
of the value. For example, some field of communication protocol is defined 
to have only 3 bytes. In this case the value will probably be stored in a 
variable of `std::uint32_t` type. There must be similar set of functions, 
but with additional template parameter that specifies how many bytes to read/write:

```cpp
template <typename TReadIter, typename TWriteIter>
class Message
{
public:
    typedef TReadIter ReadIterator;
    typedef TWriteIter WriteIterator;
    
    ...
    
protected:
    ...
    
    template <std::size_t TSize, typename T>
    static T readData(ReadIterator& iter) {...}
    
    template <std::size_t TSize, typename T>
    static void writeData(T value, WriteIterator& iter) {...}
};
```

**CAUTION**: The interface described above is very easy and convenient to use 
and quite easy to implement using straightforward approach. However, any 
variation of template parameters create an instantiation of new binary code, 
which may create significant code bloat if not used carefully. 
Consider the following:

- Read/write of signed vs unsigned integer values. The serialisation/deserialisation 
code is identical for both cases, but won't be considered as such when 
instantiating the functions. To optimise this case, there is a need to 
implement read/write operations only for unsigned value, while the “signed” 
functions become wrappers around the former. Don't forget a sign extension 
operation when retrieving partial signed value.
- The read/write operations are more or less the same for any length of the 
values, i.e of any types: (unsigned) char, (unsigned) short, (unsigned) int, etc... 
To optimise this case, there is a need for internal function that receives 
length of serialised value as a run time parameter, while the functions 
described above are mere wrappers around it.
- Usage of the iterators also require caution. For example reading values may 
be performed using regular `iterator` as well as `const_iterator`, i.e. 
iterator pointing to `const` values. These are two different iterator types 
that will duplicate the “read” functionality if both of them are used.

All the consideration points stated above require quite complex implementation 
of the serialisation/deserialisation functionality with multiple levels of 
abstraction which is beyond the scope of this book. 
It would be a nice exercise to try and implement them yourself. 
You may take a look at 
[util/access.h](https://github.com/arobenko/comms_champion/blob/master/src/lib/comms/include/comms/util/access.h)
file in the
[COMMS Library](https://github.com/arobenko/comms_champion#comms-library)
of the [comms_champion](https://github.com/arobenko/comms_champion) project
for reference.

