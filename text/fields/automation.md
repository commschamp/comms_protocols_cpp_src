# Automating Basic Operations

Let's start with automation of read and write. In most cases the `read()`
operation of the message has to read **all** the fields the message
contains, as well as `write()` operation has to write **all** the fields of 
the message.

In order to make the generation of appropriate read/write code to be a job of 
the compiler we have to:
- Provide the same interface for every message field.
- Introduce a meta-programming friendly structure to hold all the fields, such
as `std::tuple`.
- Use meta-programming techniques to invoke the required read/write function of
every field.

Let's assume that all the message fields provide the following interface:
```cpp
class SomeField
{
public:
    // Value storage type definition
    typedef ... ValueType;
    
    // Provide an access to the stored value
    ValueType& value();
    const ValueType& value() const;

    // Read (deserialise) and update internal value
    template <typename TIter>
    ErrorStatus read(TIter& iter, std::size_t len);
    
    // Write (serialise) internal value
    template <typename TIter>
    ErrorStatus write(TIter& iter, std::size_t len) const;
    
    // Get the serialisation length
    std::size_t length() const;
    
private:
    ValueType m_value;
}
```

The custom message class needs to define its fields bundled in `std::tuple`
```cpp
class ActualMessage1 : public Message
{
public:
    using Field1 = ...
    using Field2 = ...
    using Field3 = ...
    
    using AllFields = std::tuple<
        Field1,
        Field2,
        Field3
    >;
    ...
    
protected:
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) override
    {
        ...// invoke read() member function of every field
    }
    
    virtual ErrorStatus writeImpl(WriteIterator& iter, std::size_t len) const override
    {
        ...// invoke write() member function of every field
    }
    
    
private:
    AllFields m_fields;
};
```

What remains to be done is to implement automatic invocation of `read()` and
`write()` member function for every field in `AllFields` tuple.

Let's take a look at standard algorithm 
[std::for_each](http://en.cppreference.com/w/cpp/algorithm/for_each).
Its last parameter is a functor object, which must define appropriate 
`operator()` member function. This function is invoked for every element being
iterated over. What we need is something similar, but 
instead of receiving iterators, it must receive a full tuple object and the
`operator()` of provided functor must be able to receive any type, i.e. be a template
function.

As the result the signature of such function may look like this:
```cpp
template <typename TTuple, typename TFunc>
void tupleForEach(TTuple&& tuple, TFunc&& func);
```
where `tuple` is l- or r-value reference to any tuple object, and `func` is
l- or r-value reference to a functor object that must define the following 
public interface:
```cpp
struct MyFunc
{
    template <typename TTupleElem>
    void operator()(TTupleElem&& elem) {...}
};
```

Implementation of the`tupleForEach()` function described above can be a nice
exercise for practising meta-programming skills. 
[Appendix A](../appendix/a.md) contains the required code if help is required.

## Implementing Read

In order to implement read functionality there is a need to define proper reading
functor class, which may receive any field:
```cpp
class FieldReader
{
public:
    FieldReader(ErrorStatus& status, ReadIterator& iter, std::size_t& len)
      : m_status(status),
        m_iter(iter),
        m_len(len)
    {
    }
    
    template <typename TField>
    void operator()(TField& field)
    {
        if (m_status != ErrorStatus::Success) {
            // Error occurred earlier, don't continue with read
            return;
        }
            
        m_status = field.read(m_iter, m_len);
        if (m_status == ErrorStatus::Success) {
            m_len -= field.length();
        }
    }
    
private:
    ErrorStatus& m_status;    
    ReadIterator& m_iter;
    std::size_t& m_len;
}
```

Then the body of `readImpl()` member function of the actual message class
may look like this:
```cpp
class ActualMessage1 : public Message
{
public:
    using AllFields = std::tuple<...>;
    ...
    
protected:
    virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) override
    {
        auto status = ErrorStatus::Success;
        tupleForEach(m_fields, FieldReader(status, iter, len));
        return status;
    }
    
    
private:
    AllFields m_fields;
};
```

From now on, any modification to the `AllFields` bundle of fields does NOT require
any additional modifications to the body of `readImpl()` function. It becomes 
a responsibility of the compiler to invoke `read()` member function of all the
fields.

## Implementing Write

Implementation of the write functionality is very similar. Below is the implementation
of the writer functor class:
```cpp
class FieldWriter
{
public:
    FieldWriter(ErrorStatus& status, WriterIterator& iter, std::size_t& len)
      : m_status(status),
        m_iter(iter),
        m_len(len)
    {
    }
    
    template <typename TField>
    void operator()(TField& field)
    {
        if (m_status != ErrorStatus::Success) {
            // Error occurred earlier, don't continue with write
            return;
        }
            
        m_status = field.write(m_iter, m_len);
        if (m_status == ErrorStatus::Success) {
            m_len -= field.length();
        }
    }
    
private:
    ErrorStatus& m_status;    
    WriterIterator& m_iter;
    std::size_t& m_len;
}
```

Then the body of `writeImpl()` member function of the actual message class
may look like this:
```cpp
class ActualMessage1 : public Message
{
public:
    using AllFields = std::tuple<...>;
    ...
    
protected:
    virtual ErrorStatus writeImpl(WriterIterator& iter, std::size_t len) const override
    {
        auto status = ErrorStatus::Success;
        tupleForEach(m_fields, FieldWriter(status, iter, len));
        return status;
    }
    
    
private:
    AllFields m_fields;
};
```
Just like with reading, any modification to the `AllFields` bundle of fields does NOT require
any additional modifications to the body of `writeImpl()` function. It becomes 
a responsibility of the compiler to invoke `write()` member function of all the
fields.

## Eliminating Code Duplication

It is easy to notice that the body of `readImpl()` and `writeImpl()` of every
`ActualMessage*` class looks the same. What differs is the tuple of fields which
get iterated over. 

It is possible to eliminate such duplication of boilerplate code by introducing
additional class in the class hierarchy, which receives a bundle of fields as
a template parameter and implements the required functions:
```cpp
// Common interface class:
class Message {...};

template <typename TFields>
class MessageBase : public Message
{
public:
    using Message::ReadIterator;
    using Message::WriteIterator;
        
    typedef TFields AllFields;
    
    // Access to fields bundle
    AllFields& fields() { return m_fields; }
    const AllFields& fields() const { return m_fields; }
    
protected:    
   virtual ErrorStatus readImpl(ReadIterator& iter, std::size_t len) override
    {
        auto status = ErrorStatus::Success;
        tupleForEach(m_fields, FieldReader(status, iter, len));
        return status;
    }
    
    virtual ErrorStatus writeImpl(WriterIterator& iter, std::size_t len) const override
    {
        auto status = ErrorStatus::Success;
        tupleForEach(m_fields, FieldWriter(status, iter, len));
        return status;
    }

private:
    class FieldReader { ... /* same code as from earlier example */ };
    class FieldWriter { ... /* same code as from earlier example */ };
    
    AllFields m_fields;    
}
```

All the `ActualMessage*` classes need to inherit from `MessageBase` while
providing their own fields. The right implementation of 
`readImpl()` and `writeImpl()` is going to be generated by the compiler automatically for
every custom message.

```cpp
using ActualMessage1Fields = std::tuple<...>;
class ActualMessage1 : public MessageBase<ActualMessage1Fields> {...};

using ActualMessage2Fields = std::tuple<...>;
class ActualMessage2 : public MessageBase<ActualMessage2Fields> {...};

...
```
The class hierarchy looks like this:
![Image: Class hierarchy](../image/message_auto_fields_hierarchy.png)

## Other Basic Operations

In addition to read and write, there are other operations that can be automated.
For example, the serialisation length of the full message is a summary of the 
serialisation lengths of all the fields. If every field can report its serialisation
length, then the implementation may look like this:
```cpp
class Message
{
public:
    std::size_t length() const
    {
        return lengthImpl();
    }

protected:
    virtual std::size_t lengthImpl() const = 0;
};

template <typename TFields>
class MessageBase : public Message
{
protected:
    virtual std::size_t lengthImpl() const override
    {
        return tupleAccumulate(m_fields, 0U, LengthCalc());
    }
    
private:
    struct LengthCalc
    {
        template <typename TField>
        std::size_t operator()(std::size_t size, const TField& field) const
        {
            return size + field.length();
        }
    };
    
    AllFields m_fields;    
}

```
**NOTE**, that example above used `tupleAccumulate()` function, which is 
similar to [std::accumulate](http://en.cppreference.com/w/cpp/algorithm/accumulate).
The main difference is that binary operation function object provided to the
function, must be able to receive any type, just like with `tupleForEach()` 
described earlier. The code of `tupleAccumulate()` function can be found in
[Appendix B](../appendix/b.md)

Another example is an automation of validity check. In most cases the message
is considered to be valid if **all** the fields are valid. Let's assume that
every fields can also provide an information about validity of its data:
```cpp
class SomeField
{
public:
    // Get validity information
    bool valid() const;

    ...
}
```

The implementation of message contents validity check may look like this:
```cpp
class Message
{
public:
    bool valid() const
    {
        return validImpl();
    }

protected:
    virtual bool validImpl() const = 0;
};

template <typename TFields>
class MessageBase : public Message
{
protected:
    virtual bool validImpl() constImpl() const override
    {
        return tupleAccumulate(m_fields, true, ValidityCalc());
    }
    
private:
    struct ValidityCalc
    {
        template <typename TField>
        bool operator()(bool valid, const TField& field) const
        {
            return valid && field.valid();
        }
    };
    
    AllFields m_fields;    
}

```

## Overriding Automated Default Behaviour

It is not uncommon to have some optional fields in the message, the existence
of which depends on some bits in previous fields. In this case the default 
read and/or write behaviour generated by the compiler needs to be modified.
Thanks to the inheritance relationship between the classes, nothing prevents us
to overriding the `readImpl()` and/or `writeImpl()` function and provide the
right behaviour:
```cpp
using ActualMessage1Fields = std::tuple<...>;
class ActualMessage1 : public MessageBase<ActualMessage1Fields> 
{
protected:
    virtual void readImpl(ReadIterator& iter, std::size_t len) override {...}
    virtual void writeImpl(WriteIterator& iter, std::size_t len) const override {...}
}
```

The `MessageBase<...>` class already contains the definition of `FieldReader` and
`FieldWriter` helper classes, it can provide helper functions to read/write only
several fields from the whole bundle. These functions can be reused in the
overriding implementations of `readImpl()` and/or `writeImpl()`:
```cpp
template <typename TFields>
class MessageBase : public Message
{
    ...
protected:
    template <std::size_t TFromIdx, std::size_t TUntilIdx>
    ErrorStatus readFieldsFromUntil(
        ReadIterator& iter,
        std::size_t& size)
    {
        auto status = ErrorStatus::Success;
        tupleForEachFromUntil<TFromIdx, TUntilIdx>(m_fields, FieldReader(status, iter, size));
        return status;
    }
    
    template <std::size_t TFromIdx, std::size_t TUntilIdx>
    ErrorStatus writeFieldsFromUntil(
        WriteIterator& iter,
        std::size_t size) const
    {
        auto status = ErrorStatus::Success;
        tupleForEachFromUntil<TFromIdx, TUntilIdx>(m_fields, FieldWriter(status, iter, size));
        return status;
    }
    
private:
    class FieldReader { ... };
    class FieldWriter { ... };
    
    AllFields m_fields;    
}

```
The provided `readFieldsFromUntil()` and `writeFieldsFromUntil()` protected member
functions use `tupleForEachFromUntil()` function to perform read/write operations
on a group selected fields. It is similar to `tupleForEach()` used earlier, but
receives additional template parameters that specify indices of the fields for
which the provided functor object needs to be invoked. The code of 
`tupleForEachFromUntil()` function can be found in [Appendix C](../appendix/c.md).

