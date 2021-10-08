# Generic parameter capture

![stable][]

{{#include ../badges.md}}

When you use an impl trait in return position, there are some limitations on the "hidden type" that may be used. 

XXX document:

* you can capture type parameters
* but not lifetimes, unless they appear in the bounds
