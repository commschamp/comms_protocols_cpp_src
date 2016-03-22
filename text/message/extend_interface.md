# Extending Interface

Let's assume the protocol was initially developed for some embedded system
which required very basic message interface of only **read** / **write** / **dispatch**. 

The interface class definition was defined allowing iterators to be specified elsewhere:
```cpp
class Handler;
template <typename TReadIterator, typename TWriteIterator>
class Message
{
public:
    using ReadIterator = TReadIterator;
    using WriteIterator = TWriteIterator;
    
    // Read the message
    ErrorStatus read(ReadIterator& iter, std::size_t len)
    {
        return readImpl(iter, len);
    }

    // Write the message
    ErrorStatus write(WriteIterator& iter, std::size_t len) const
    {
        return writeImpl(iter, len);
    }
    
    // Dispatch to handler
    void dispatch(Handler& handler)
    {
        dispatchImpl(handler);
    }
    
protected:
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) = 0;
    virtual ErrorStatus writeImpl(WriteIterator& iter, std::size_t len) const = 0;
    virtual void dispatchImpl(Handler& handler) = 0;
};
```

The intermediate class allowing common implementation of `dispatchImpl()`:
```cpp
template <typename TReadIterator, typename TWriteIterator, typename TDerived>
class MessageBase : public Message<TReadIterator, TWriteIterator>
{
protected:
    virtual void dispatchImpl(Handler& handler) override {...};
};
```

And the actual message classes:
```cpp
template <typename TReadIterator, typename TWriteIterator>
class ActualMessage1 : public MessageBase<TReadIterator, TWriteIterator, ActualMessage1>
{
    ...
};

template <typename TReadIterator, typename TWriteIterator>
class ActualMessage2 : public MessageBase<TReadIterator, TWriteIterator, ActualMessage2>
{
    ...
};

```

Then, after a while a new application needs to be developed, which monitors
the I/O link and dumps all the message traffic into standard output and/or *.csv file. This
application requires knowledge about names of the messages, and it would be 
convenient to add an appropriate function into the common message interface and
reuse the existing implementation. There is 
one problem though, the code of the protocol is already written and used in
the embedded system, which does not require this additional functionality and 
its binary code should not contain these extra functions.

One of the solutions can be to use preprocessor:
```cpp
template <...>
class Message
{
public:
#ifdef HAS_NAME    
    const char* name() const
    {
        return nameImpl();
    }
#endif    
    
protected:
#ifdef HAS_NAME    
    virtual const char* nameImpl() const = 0;
#endif    
};

template <...>
class MessageBase : public Message<...> {...};

template <>
class ActualMessage1 : public MessageBase<...>
{
protected:
#ifdef HAS_NAME    
    virtual const char* nameImpl() const
    {
        return "ActualMessage1";        
    }
#endif    
};

```

Such approach may work for some products, but not for others, especially ones
that developed by multiple teams. If one team developed a reference implementation
of the communication protocol being used and is an "owner" of the code, 
then it may be difficult and/or impractical for other team to push required changes
upstream.

Another approach is to remove hard coded
inheritance relationship between `Message` interface class and intermediate
`MessageBase` class. Instead, provide the common interface class as a 
template parameter to the latter:
```cpp
template <typename TIternface, typename TDerived>
class MessageBase : public TIternface
{
protected:
    virtual void dispatchImpl(Handler& handler) override {...};
}
```

And the `ActualMessage*` classes will look like this:
```cpp
template <typename TIternface>
class ActualMessage1 : public MessageBase<TIternface, ActualMessage1>
{
    ...
};

template <typename TIternface>
class ActualMessage2 : public MessageBase<TIternface, ActualMessage2>
{
    ...
};
```

Then, the initial embedded system may use the common protocol code like this:
```cpp
using EmbReadIterator = ...;
using EmbWriteIterator = ...;
using EmbMessage = Message<EmbReadIterator, EmbWriteIterator>;
using EmbMessage1 = ActualMessage1<EmbMessage>;
using EmbMessage2 = ActualMessage2<EmbMessage>;
```

The original class hierarchy preserved intact:
![Image: Original class hierarchy](../image/message_old_interface_hierarchy.png)

And when extended interface and functionality are required, just use extra
class inheritances:
```cpp
// Define extended interface
template <typename TReadIterator, typename TWriteIterator> 
class ExtMessage : public Message<TReadIterator, TWriteIterator>
{
public:
    const char* name() const
    {
        return nameImpl();
    }
    
protected:
    virtual const char* nameImpl() const = 0;
}

// Define extended messages
<typename TInterface>
class ExtActualMessage1 : public ActualMessage1<TInterface>
{
protected:
    virtual const char* nameImpl() const
    {
        return "ActualMessage1";        
    }

}
```

The new application that requires extended implementation may still reuse the 
common protocol code like this:
```cpp
using NewReadIterator = ...;
using NewWriteIterator = ...;
using NewMessage = ExtMessage<NewReadIterator, NewWriteIterator>;
using NewMessage1 = ExtActualMessage1<NewMessage>;
using NewMessage2 = ExtActualMessage2<NewMessage>;
```

As a result, no extra modifications to the original source code of the 
protocol implementation is required, and every team achieves their own goal. 
Everyone is happy!!!

The extended class hierarchy becomes:

![Image: Extended class hierarchy](../image/message_ext_interface_hierarchy.png)
