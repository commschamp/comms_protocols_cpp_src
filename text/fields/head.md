# Fields

Every message in any communication protocol has zero or more internal fields, which
get serialised in some predefined order and transferred as message payload over
I/O link.

Usually developers implement some 
[boilerplate code](https://en.wikipedia.org/wiki/Boilerplate_code)
of explicitly reading and writing all message fields in appropriate functions,
such as `readImpl()` and `writeImpl()` described in
[Reading and Writing](../message/read_write.md) chapter. The primary 
disadvantage of this approach is an increased development effort when contents 
of some message need to be modified, i.e. some new field is added, 
or existing one removed, even when the type of an existing field changes. 
Having multiple places in the code, that need to be updated, leads to
an increased chance of forgetting to update one of the places, or introducing
some silly error, which will take time to notice and fix.

This chapter describes how to automate basic operations, such as read and write,
i.e. to make it a responsibility of the compiler to generate appropriate code.
All the developer needs to do is to define the list of all the field types
the message contains, and let the compiler do the job.
