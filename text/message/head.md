# Message

Most C++ developers intuitively choose to express every independent message as 
a separate class, which inherit from a common interface. 

![Image: Message class hierarchy](../image/message.png)

This is a step to the **right** direction.
It becomes easy and convenient to write a common code that suites all
possible messages:

```cpp
class Message 
{
public:
    void write(...) const {
        writeImpl(...);
    }
    ...
protected:
    // Implements writing to a buffer functionality
    virtual void writeImpl(...) const = 0;
};

class ActualMessage1 : public Message 
{
    ...
protected:
    virtual void writeImpl(...) const override {...};
};

class ActualMessage2 : public Message 
{
    ...
protected:
    virtual void writeImpl(...) const override {...};
};

// Send any message
void sendMessage(const Message& msg)
{
    ...
    msg.write(...); // write message to a buffer
    ...// send buffer contents over I/O link;
}
```


