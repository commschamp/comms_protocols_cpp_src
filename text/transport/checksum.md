# CHECKSUM Layer

This layer is responsible to calculate and validate the checksum information.

- During read operation it remembers the initial value of the read iterator,
then invokes the read operation of the next (wrapped) layer. After the
next layer reports successful completion of its read operation, the expected checksum 
value is read. Then, the real checksum on the read data bytes is calculated and
compered to the expected one. If the values match, the read operation is
reported as successfully complete. If not, the created message object is
deleted and error reported.
- During write operation it lets the next (wrapped) layer to finish its
writing, calculates the checksum value on the written
data bytes, and writes the result into output buffer.

Before jumping into writing the code, there is a need to be aware of couple of
issues:

- The generic code of the **CHECKSUM Layer** mustn't depend on any particular
checksum calculation algorithm. I'd recommend providing the calculator class as a template
parameter, `operator()` of which is responsible to implement the checksum 
calculation logic.
- The checksum calculation after write operation requires the iterator to
go back and calculate the checksum on the written data bytes. It can easily
be done when used iterator is 
[random access](http://en.cppreference.com/w/cpp/concept/RandomAccessIterator) 
one. Sometimes it may not
be the case (for example the output data is written into 
[std::vector](http://en.cppreference.com/w/cpp/container/vector) using
[std::back_insert_iterator](http://en.cppreference.com/w/cpp/iterator/back_insert_iterator)).
There is a need to have a generic way to handle such cases.

## Implementing Read

Let's start with implementing the read first.
```cpp
namespace comms
{
// TField is type of the field used to read/write checksum value
// TCalc is generic class that is responsible to implement checksum calculation logic
// TNext is the next layer this one wraps
template <typename TField, typename TCalc, typename TNext>
class ChecksumLayer
{
public:
    // Type of the field object used to read/write SYNC prefix.
    using Field = TField;
    
    // Take type of the ReadIterator from the next layer
    using ReadIterator = typename TNext::ReadIterator;

    // Take type of the WriteIterator from the next layer
    using WriteIterator = typename TNext::WriteIterator;

    // Take type of the message interface from the next layer
    using Message = typename TNext::Message;
    
    // Take type of the message interface pointer from the next layer
    using MsgPtr = typename TNext::MsgPtr; 
    
    template <typename TMsgPtr>
    ErrorStatus read(TMsgPtr& msgPtr, ReadIterator& iter, std::size_t len)
    {
        Field field;
        if (len < field.length()) {
            return ErrorStatus::NotEnoughData;
        }
        auto fromIter = iter; // The read is expected to use random-access iterator
        auto es = m_next.read(iter, len - field.length());
        if (es != ErrorStatus::Success) {
            return es;
        }
        auto consumedLen = static_cast<std::size_t>(std::distance(fromIter, iter));
        auto remLen = len - consumedLen;
        es = field.read(iter, remLen);
        if (es != ErrorStatus::Success) {
            msgPtr.reset();
            return es;
        }
        auto checksum = TCalc()(fromIter, consumedLen);
        auto expectedValue = field.value();
        if (expectedValue != checksum) {
            msgPtr.reset(); // Delete allocated message
            return ErrorStatus::ProtocolError;
        }
        return ErrorStatus::Success;
    } 
private:
    TNext m_next;
};
} // namespace comms
```

It is clear that `TCalc` class is expected to have `operator()` member
function, which receives the iterator for reading and number of bytes 
in the buffer.

As an example let's implement generic total sum of bytes calculator:
```cpp
namespace comms
{
template <typename TResult = std::uint8_t>
class BasicSumCalc
{
public:
    template <typename TIter>
    TResult operator()(TIter& iter, std::size_t len) const
    {
        using ByteType = typename std::make_unsigned<
            typename std::decay<decltype(*iter)>::type
        >::type;

        auto checksum = TResult(0);
        for (auto idx = 0U; idx < len; ++idx) {
            checksum += static_cast<TResult>(static_cast<ByteType>(*iter));
            ++iter;
        }
        return checksum;
    }
};
} // namespace comms
```

## Implementing Write

Now, let's tackle the write problem. As it was mentioned earlier, there is a need to
recognise the type of the iterator used for writing and behave accordingly. 
If the iterator is properly defined, the 
[std::iterator_traits](http://en.cppreference.com/w/cpp/iterator/iterator_traits) 
class will define `iterator_category` internal type.
```cpp
using WriteIteratorCategoryTag = 
    typename std::iterator_traits<WriteIterator>::iterator_category;
```
For random access iterators the `WriteIteratorCategoryTag` class will be
either [std::random_access_iterator_tag](http://en.cppreference.com/w/cpp/iterator/iterator_tags) 
or any other class that inherits from it. Using this information, we can use
[Tag Dispatch Idiom](http://www.generic-programming.org/languages/cpp/techniques.php#tag_dispatching)
to choose the right writing functionality.
```cpp
namespace comms
{
template <...>
class ChecksumLayer
{
public:
    using WriteIteratorCategoryTag =
        typename std::iterator_traits<WriteIterator>::iterator_category;

    ErrorStatus write(const Message& msg, WriteIterator& iter, std::size_t len) const
    {
        return writeInternal(msg, iter, len, WriteIteratorCategoryTag());
    } 
    
private:
    ErrorStatus writeInternal(
        const Message& msg, 
        WriteIterator& iter, 
        std::size_t len,
        const std::random_access_iterator_tag&) const
    {
        return writeRandomAccess(msg, iter, len);
    } 
    
    ErrorStatus writeInternal(
        const Message& msg, 
        WriteIterator& iter, 
        std::size_t len,
        const std::output_iterator_tag&) const
    {
        return writeOutput(msg, iter, len);
    } 
    
    ErrorStatus writeRandomAccess(const Message& msg, WriteIterator& iter, std::size_t len) const
    {
        auto fromIter = iter;
        auto es = m_next.write(msg, iter, len);
        if (es != ErrorStatus::Success) {
            return es;
        }
        auto consumedLen = static_cast<std::size_t>(std::distance(fromIter, iter));
        auto remLen = len - consumedLen;
        Field field;
        field.value() = TCalc()(fromIter, consumedLen);
        return field.write(iter, remLen);
    } 

    ErrorStatus writeOutput(const Message& msg, WriteIterator& iter, std::size_t len) const
    {
        TField field;
        auto es = m_next.write(msg, iter, len - field.length());
        if (es != ErrorStatus::Success) {
            return es;
        }
        field.write(iter, field.length());
        return ErrorStatus::UpdateRequired;
    } 
    
    TNext m_next;
};
} // namespace comms
```
Please pay attention, that `writeOutput()` function above is unable to
properly calculate the checksum of the written data. There is no way the
iterator can be reversed back and used as input instead of output. As the
result the function returns special error status: `ErrorStatus::UpdateRequired`.
It is an indication that the write operation is not complete and the 
output should be updated using random access iterator.

## Implementing Update

```cpp
namespace comms
{
template <...>
class ChecksumLayer
{
public:

    template <typename TIter>
    ErrorStatus update(TIter& iter, std::size_t len) const
    {
        Field field;
        auto fromIter = iter;
        auto es = m_next.update(iter, len - field.length());
        if (es != ErrorStatus::Success) {
            return es;
        }

        auto consumedLen = static_cast<std::size_t>(std::distance(fromIter, iter));
        auto remLen = len - consumedLen;
        field.value() = TCalc()(fromIter, consumedLen);
        es = field.write(iter, remLen);
        return es;
    }
    
private:
    TNext m_next;
};
} // namespace comms
```

Please note, that every other layer must also implement the `update()` member
function, which will just advance the provided iterator by the number
of bytes required to write its field and invoke `update()` member function
of the next (wrapped) layer.
```cpp
namespace comms
{
template <typename TMessage>
class MsgDataLayer
{
public:
    template <typename TIter>
    ErrorStatus update(TIter& iter, std::size_t len) const
    {
        std::advance(iter, len);
        return ErrorStatus::Success;
    }
};

} // namespace comms
```

```cpp
namespace comms
{
template <...>
class MsgIdLayer
{
public:
    template <typename TIter>
    ErrorStatus update(TIter& iter, std::size_t len) const
    {
        TField field;
        std::advance(iter, field.length());
        return m_next.update(iter, len - field.length());
    }
    
private:
    TNext m_next;
};

} // namespace comms
```

And so on for the rest of the layers. Also note, that the code above will
work, only when the field has the **same serialisation length for any value**.
If this is not the case 
([Base-128](https://en.wikipedia.org/wiki/Variable-length_quantity) encoding
is used), the previously written value needs to be read, instead of
just advancing the iterator, to make sure the iterator is advanced
right amount of bytes:
```cpp
template <typename TIter>
ErrorStatus update(TIter& iter, std::size_t len) const
{
    TField field;
    auto es = field.read(iter);
    if (es != ErrorStatus::Success) {
        return es;
    }
    return m_next.update(iter, len - field.length());
}
```
The variable serialisation length encoding will be forced using some kind of
special option. It can be identified at compile time and 
[Tag Dispatch Idiom](http://www.generic-programming.org/languages/cpp/techniques.php#tag_dispatching)
can be used to select appropriate `update` functionality.

The caller, that requests protocol stack to serialise a message, must check
the error status value returned by the `write()` operation. 
If it is `ErrorStatus::UpdateRequired`,
the caller must create random-access iterator to the already written buffer
and invoke `update()` function with it, to make sure the written information 
is correct.
```cpp
using ProtocolStack = ...;
ProtocolStack protStack;

ErrorStatus sendMessage(const Message& msg)
{
    ProtocolStack::WriteIterator writeIter = ...;
    auto es = protStack.write(msg, writeIter, bufLen);
    if (es == ErrorStatus::UpdateRequired) {
        auto updateIter = ...; // Random access iterator to written data
        es = protStack.update(updateIter, ...);
    }
    return es;
}

```
