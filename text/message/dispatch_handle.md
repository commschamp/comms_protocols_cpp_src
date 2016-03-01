# Dispatching and Handling

When a new message arrives, its appropriate object is created and the contents
are deserialised using `read()` member function described in previous section. 
It is time to dispatch it to an appropriate handling function. Many developers
use the `switch` statement or even a sequence of `dynamic_cast`s to identify
the real type of the message object and call appropriate handling function. 

As you may have guessed, this is pretty inefficient, especially when there are
more than 7-10 messages to handle. There is a much better way of doing a dispatch
operation by using a C++ ability to differentiate between functions with the
same name but with different parameter types. It is called
[Double Dispatch Idiom](https://en.wikipedia.org/wiki/Double_dispatch).

Let's assume we have a handling class `Handler` that is capable of handling
all possible messages:
```cpp
class Handler
{
public:
    void handle(ActualMessage1& msg);
    void handle(ActualMessage2& msg);
    ...
}
```

Then the definition of the messages may look like this:
```cpp
class Message 
{
public:
    void dispatch(Handler& handler)
    {
        dispatchImpl(handler);
    }
    ...
    
protected:
    virtual void dispatchImpl(Handler& handler) = 0; 
};

class ActualMessage1 : public Message 
{
    ...
protected:
    virtual void dispatchImpl(Handler& handler) override
    {
        handler.handle(*this); // invokes handle(ActualMessage1&);
    }
};

class ActualMessage2 : public Message 
{
    ...
protected:
    virtual void dispatchImpl(Handler& handler) override
    {
        handler.handle(*this); // invokes handle(ActualMessage2&);
    }
};
```
Then the following code will invoke appropriate handling function
in the `Handler` object:
```cpp
typedef std::unique_ptr<Message> MsgPtr;

MsgPtr msg = ... // any custom message object;
Handler handler;

msg->dispatch(handler); // will invoke right handling function.
```

Please note, that the `Message` interface class doesn't require the definition 
of the `Handler` class, the forward declaration
of the latter is enough. The `Handler` also doesn't require
the definitions of all the actual messages being available, forward declarations of all 
the message classes will suffice. Only the implementation part of the `Handler`
class will require knowledge about the interface of the messages being handled.
However, the public interface of the `Handler` class must be known when
compiling `dispatchImpl()` member function of any `ActualMessageX` class.

## Eliminating Code Duplication

You may also notice that the body of all `dispatchImpl()` member functions in
all the `ActualMessageX` classes is going to be the same:
```cpp
virtual void dispatchImpl(Handler& handler) override
{
    handler.handle(*this); 
}
```
The problem is that `*this` expression in every function evaluates to the
object of different type.

The apperent code duplication may be eliminated using 
[Curiously Recurring Template Pattern](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Curiously_Recurring_Template_Pattern) idiom.

```cpp
class Message 
{
public:
    void dispatch(Handler& handler)
    {
        dispatchImpl(handler);
    }
    ...
    
protected:
    virtual void dispatchImpl(Handler& handler) = 0; 
};

template <typename TDerived>
class MessageBase : public Message
{
protected:
    virtual void dispatchImpl(Handler& handler) override
    {
        auto& derived = static_cast<Derived&>(*this);
        handler.handle(derived); 
    }
    
}

class ActualMessage1 : public MessageBase<ActualMessage1>
{
    ...
}; 

class ActualMessage2 : public MessageBase<ActualMessage2>
{
    ...
};

```

Please note, that `ActualMessageX` provide their own type as a template
parameter to their base class `MessageBase` and do not require to implement
`dispatchImpl()` any more. The class hierarchy looks like this:

![Image: Class hierarchy](../image/message_dispatch_hierarchy.png)

## Handling Limited Number of Messages

What if there is a need to handle only limited number of messages, all the rest
just need to be ignored. Let's assume the protocol defines 10 messages: 
`ActualMessage1`, `ActualMessage2`, ..., `ActualMessage10`. The messages that
need to be handled are just `ActualMessage2` and `ActualMessage5`, all the rest
ignored. Then the definition of the `Handler` class will look like this:
```cpp
class Handler
{
public:
    void handle(ActualMessage2& msg) {...}
    void handle(ActualMessage5& msg) {...}
    
    void handle(Message& msg) {} // empty body
}
```
In this case, when compiling `dispatchImpl()` member function of `ActualMessage2`
and `ActualMessage5`, the compiler will generate invocation code for appropriate
`handle()` function. For the rest of the message classes, the best matching option
will be invocation of `handle(Message&)`.

## Polymorphic Handling

There may be a need to have multiple handlers for the same set of messages. It
can easily be achieved by making the `Handler` an abstract interface class and
defining its `handle()` member functions as virtual. 
```cpp
class Handler
{
public:
    virtual void handle(ActualMessage1& msg) = 0;
    virtual void handle(ActualMessage2& msg) = 0;    
    ...
}

class ActualHandler1 : public Handler
{
public:
    virtual void handle(ActualMessage1& msg) override;
    virtual void handle(ActualMessage2& msg) override;    
    ...
}

class ActualHandler2 : public Handler
{
public:
    virtual void handle(ActualMessage1& msg) override;
    virtual void handle(ActualMessage2& msg) override;    
    ...
}

```
No other changes to dispatch functionality is required:
```cpp
typedef std::unique_ptr<Message> MsgPtr;

MsgPtr msg = ... // any custom message object;
AtualHandler1 handler1;
AtualHandler2 handler2;

// Works for any handler
msg->dispatch(handler1);
msg->dispatch(handler2);
```

## Generic Handler

Now it's time to think about the required future effort of extending the 
handling functionality when new messages are added to the protocol and their
respective classes are implemented. It is especially relevant when
[Polymorphic Handling](#polymorphic-handling) is involved. There is a need
to introduce new `virtual handle(...)` member function for every new message
that is being added. 

There is a way to delegate this job to the compiler using template specialisation. 
Let's assume that all the message types that need
to be handled are bundled into a simple declarative statement of `std::tuple`
definition:
```cpp
using AllMessages = std::tuple<
    ActualMessage1,
    ActualMessage2,
    ...
>;
```

Then the definition of the generic handling class will be as following:
```cpp
/// @tparam TCommon Common interface class for all the messages
/// @tparam TAll All the message types that need to be handled bundled in std::tuple
template <typename TCommon, typename TAll>
class GenericHandler;

template <typename TCommon, typename TFirst, typename... TRest>
class GenericHandler<TCommon, std::tuple<TFirst, TRest...> > :
                        public GenericHandler<TCommon, std::tuple<TRest...> >
{
    typedef GenericHandler<TCommon, std::tuple<TRest...> > Base;
public:

    virtual ~GenericHandler() {}

    using Base::handle;
    virtual void handle(TFirst& msg)
    {
        // By default call handle(TCommon&)
        this->handle(static_cast<TCommon&>(msg));
    }
};

template <typename TCommon>
class GenericHandler<TCommon, std::tuple<> >
{
public:

    virtual ~GenericHandler() {}

    virtual void handle(TCommon&)
    {
        // Nothing to do
    }
};

```

The code above generates `virtual handle(TCommon&)` function for the common
interface class, which does nothing by default. It also creates a separate 
`virtual handle(...)` function for every message type provided in `TAll` 
tuple. Every such function upcasts the message type to its interface class 
`TCommon` and invokes the `handle(TCommon&)`.

As the result simple declaration of 
```cpp
class Handler : public GenericHandler<Message, AllMessages> {};
```
is equivalent to having the following class defined:
```cpp
class Handler
{
public:
    virtual void handle(ActualMessage1& msg)
    {
        this->handle(static_cast<Message&>(msg));
    }
    
    virtual void handle(ActualMessage2& msg)
    {
        this->handle(static_cast<Message&>(msg));
    }

    ...
    
    virtual void handle(Message& msg)
    {
        // do nothing
    }
```

From now on, when new message class is defined, just add it to the `AllMessages`
tuple definition. If there is a need to override the default behaviour for 
specific message, override the appropriate message in the handling class:
```cpp
class ActualHandler1 : public Handler
{
public:
    virtual void handle(ActualMessage2& msg) override
    {
        std::cout << "Handling ActualMessage2" << std::endl;
    }
    
    virtual void handle(Message& msg) override
    {
        std::cout << "Common handling function is invoked" << std::endl;
    }
}
```

**REMARK**: Remember that the `Handler` class was forward declared when 
defining the `Message` interface class? Usually it looks like this:
```cpp
class Handler;
class Message
{
public:
    void dispatch(Handler& handler) {...}    
};    
```
Note, that `Handler` is declared to be a `class`, which prevents it from being
a simple `typedef` of `GenericHandler`. Usage of `typedef` will cause compilation
to fail.

**CAUTION**: The implementation of the `GenericHandler` presented above creates
a chain of **N + 1** inheritances for **N** messages defined in `AllMessages`
tuple. Every new class adds a single virtual function. Many compilers will 
create a separate `vtable` for every such class. The size of every new `vtable`
is greater by one entry than a previous one. Depending on total number of 
messages in that tuple, the code size may grow quite big due to growing number
of `vtable`s generated by the compiler. It may be not suitable for some
systems, especially bare-metal. To mitigate this impact it is possible to
significantly reduce number of inheritances using more template specialisation
classes. Below is an example of adding up to 3 virtual functions in a single
class at once. You may easily extend the example to say 10 functions or more.
```cpp
template <typename TCommon, typename TAll>
class GenericHandler;

template <typename TCommon, typename TFirst, TSecond, TThird, typename... TRest>
class GenericHandler<TCommon, std::tuple<TFirst, TSecond, TThird, TRest...> > :
                        public GenericHandler<TCommon, std::tuple<TRest...> >
{
    typedef GenericHandler<TCommon, std::tuple<TRest...> > Base;
public:

    virtual ~GenericHandler() {}

    using Base::handle;
    virtual void handle(TFirst& msg)
    {
        this->handle(static_cast<TCommon&>(msg));
    }
    
    virtual void handle(TSecond& msg)
    {
        this->handle(static_cast<TCommon&>(msg));
    }
    
    virtual void handle(TThird& msg)
    {
        this->handle(static_cast<TCommon&>(msg));
    }    
};

template <typename TCommon, typename TFirst, typename TSecond>
class GenericHandler<TCommon, std::tuple<TFirst, TSecond> >
{
public:

    virtual ~GenericHandler() {}
    
    virtual void handle(TFirst& msg)
    {
        this->handle(static_cast<TCommon&>(msg));
    }
    
    virtual void handle(TSecond& msg)
    {
        this->handle(static_cast<TCommon&>(msg));
    }

    virtual void handle(TCommon&)
    {
        // Nothing to do
    }
};

template <typename TCommon, typename TFirst>
class GenericHandler<TCommon, std::tuple<TFirst> >
{
public:

    virtual ~GenericHandler() {}
    
    virtual void handle(TFirst& msg)
    {
        this->handle(static_cast<TCommon&>(msg));
    }

    virtual void handle(TCommon&)
    {
        // Nothing to do
    }
};

template <typename TCommon>
class GenericHandler<TCommon, std::tuple<> >
{
public:

    virtual ~GenericHandler() {}

    virtual void handle(TCommon&)
    {
        // Nothing to do
    }
};
```