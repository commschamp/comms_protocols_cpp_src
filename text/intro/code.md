# Code Examples

This book contains multiple C++ code examples and snippets. Their main
purpose is to demonstrate ideas expressed in the book as well as guide 
developers into the right direction. There are no huge code listings
(nobody reads them anyway) and no detailed explanations for every line of code. I
expect the readers to understand the demonstrated idea and take it to the next 
level themselves. 

In order to demonstrate the idea I rarely use **production** level code, at 
least not up front. I will start with something simple and non-generic and
gradually increase the complexity and/or genericity.

I'm also a huge fan of 
[Non-Virtual Interface (NVI) Idiom](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-Virtual_Interface)
and often my examples will look like this:
```cpp
class SomeInterface
{
public:
    void someFunction()
    {
        someFunctionImpl();
    }
    
protected:
    virtual void someFunctionImpl() = 0;
};

class Derived : public SomeInterface
{
protected:
    virtual void someFunctionImpl() override {...}
};
```

The non virtual interface function is supposed to check pre- and 
post-conditions of the polymorphic invocation if such exist as well as execute
some common code if such is required. I tend to write the code similar to
above even when there are no pre- and post-conditions to check and no common
code to execute. Please don't be surprised when seeing such constructs throughout
the book.

