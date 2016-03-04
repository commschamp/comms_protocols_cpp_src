# Generic Library

All the generalisation techniques described so far are applicable to most
binary communication protocols. It is time to think about something generic - 
a library that can be reused between independent projects and facilitate a
development of any binary communication protocol.

From now on, every generic, protocol independent class and/or function
is going to reside in `comms` namespace in order to differentiate it from a
protocol specific code.
