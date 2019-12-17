# Calling Go Functions from Other Languages using C Shared Libraries

This respository contains source examples for the article [*Calling Go Functions from Other Languages*](https://medium.com/learning-the-go-programming-language/calling-go-functions-from-other-languages-4c7d8bcc69bf#.n73as5d6d) (medium.com).  Using the `-buildmode=c-shared` build flag, the compiler outputs a standard shared object binary file (.so) exposing Go functions as a C-style APIs. This lets programmers create Go libraries that can be called from other languages including C, Python, Ruby, Node, and Java (see contributed example for Lua) as done in this repository.

## The Go Code
First, let us write the Go code. Assume that we have written an `awesome` Go library that we want to make available to other languages. There are four requirements to follow before compiling the code into a shared library: 

* The package must be amain  package. The compiler will build the package and all of its dependencies into a single shared object binary.
* The source must import the pseudo-package “C”.
* Use the //export comment to annotate functions you wish to make accessible to other languages.
* An empty main function must be declared.

The following Go source exports four functions `Add`, `Cosine`, `Sort`, and `Log`. Admittedly, the awesome library is not that impressive. However, its diverse function signatures will help us explore type mapping implications.

File [awesome.go](./awesome.go)
```go
package main

import "C"

import (
	"fmt"
	"math"
	"sort"
	"sync"
)

var count int
var mtx sync.Mutex

//export Add
func Add(a, b int) int {
	return a + b
}

//export Cosine
func Cosine(x float64) float64 {
	return math.Cos(x)
}

//export Sort
func Sort(vals []int) {
	sort.Ints(vals)
}

//export Log
func Log(msg string) int {
	mtx.Lock()
	defer mtx.Unlock()
	fmt.Println(msg)
	count++
	return count
}

func main() {}
```

The package is compiled using the `-buildmode=c-shared` build flag to create the shared object binary:
```shell
go build -o awesome.so -buildmode=c-shared awesome.go
```
Upon completion, the compiler outputs two files: `awesome.h`, a C header file and `awesome.so`, the shared object file, shown below:
```shell
-rw-rw-r —    1362 Feb 11 07:59 awesome.h
-rw-rw-r — 1997880 Feb 11 07:59 awesome.so
```
Notice that the `.so` file is around 2 Mb, relatively large for such a small library. This is because the entire Go runtime machinery and dependent packages are crammed into a single shared object binary (similar to compiling a single static executable binary).

### The header file
The header file defines C types mapped to Go compatible types using cgo semantics. 
```c
/* Created by “go tool cgo” — DO NOT EDIT. */
...
typedef signed char GoInt8;
typedef unsigned char GoUint8;
typedef short GoInt16;
typedef unsigned short GoUint16;
typedef int GoInt32;
typedef unsigned int GoUint32;
typedef long long GoInt64;
typedef unsigned long long GoUint64;
typedef GoInt64 GoInt;
typedef GoUint64 GoUint;
typedef __SIZE_TYPE__ GoUintptr;
typedef float GoFloat32;
typedef double GoFloat64;
typedef float _Complex GoComplex64;
typedef double _Complex GoComplex128;

/*
  static assertion to make sure the file is being used on architecture
  at least with matching size of GoInt.
*/
typedef char _check_for_64_bit_pointer_matching_GoInt[sizeof(void*)==64/8 ? 1:-1];

typedef struct { const char *p; GoInt n; } GoString;
typedef void *GoMap;
typedef void *GoChan;
typedef struct { void *t; void *v; } GoInterface;
typedef struct { void *data; GoInt len; GoInt cap; } GoSlice;

#endif

/* End of boilerplate cgo prologue.  */

#ifdef __cplusplus
extern "C" {
#endif


extern GoInt Add(GoInt p0, GoInt p1);

extern GoFloat64 Cosine(GoFloat64 p0);

extern void Sort(GoSlice p0);

extern GoInt Log(GoString p0);

#ifdef __cplusplus
}
#endif
```
### The shared object file
The other file generated by the compiler is a 64-bit ELF shared object binary file. We can verify its information using the `file` command. 
```shell
$> file awesome.so
awesome.so: ELF 64-bit LSB shared object, x86–64, version 1 (SYSV), dynamically linked, BuildID[sha1]=1fcf29a2779a335371f17219fffbdc47b2ed378a, not stripped
```
Using the `nm` and the `grep` commands, we can ensure our Go functions got exported in the shared object file.
```shell
$> nm awesome.so | grep -e "T Add" -e "T Cosine" -e "T Sort" -e "T Log"
00000000000d0db0 T Add
00000000000d0e30 T Cosine
00000000000d0f30 T Log
00000000000d0eb0 T Sort
```
## From C
There are two ways to use the shared object library to call Go functions from C. First, we can statically bind the shared library at compilation, but dynamically link it at runtime. Or, have the Go function symbols be dynamically loaded and bound at runtime.
### Dynamically linked
In this approach, we use the header file to statically reference types and functions exported in the shared object file. The code is simple and clean as shown below:

File [client1.c](./client1.c)
```c
#include <stdio.h>
#include "awesome.h"

int main() {
    printf("Using awesome lib from C:\n");
   
    //Call Add() - passing integer params, interger result
    GoInt a = 12;
    GoInt b = 99;
    printf("awesome.Add(12,99) = %d\n", Add(a, b)); 

    //Call Cosine() - passing float param, float returned
    printf("awesome.Cosine(1) = %f\n", (float)(Cosine(1.0)));
    
    //Call Sort() - passing an array pointer
    GoInt data[6] = {77, 12, 5, 99, 28, 23};
    GoSlice nums = {data, 6, 6};
    Sort(nums);
    printf("awesome.Sort(77,12,5,99,28,23): ");
    for (int i = 0; i < 6; i++){
        printf("%d,", ((GoInt *)nums.data)[i]);
    }
    printf("\n");

    //Call Log() - passing string value
    GoString msg = {"Hello from C!", 13};
    Log(msg);
}
```
Next we compile the C code, specifying the shared object library:
```shell
$> gcc -o client client1.c ./awesome.so
```
When the resulting binary is executed, it links to the awesome.so library, calling the functions that were exported from Go as the output shows below.
```shell
$> ./client
awesome.Add(12,99) = 111
awesome.Cosine(1) = 0.540302
awesome.Sort(77,12,5,99,28,23): 5,12,23,28,77,99,
Hello from C!
```
### Dynamically Loaded
In this approach, the C code uses the dynamic link loader library (`libdl.so`) to dynamically load and bind exported symbols. It uses functions defined in `dhfcn.h` such as `dlopen` to open the library file, `dlsym` to look up a symbol, `dlerror` to retrieve errors, and `dlclose` to close the shared library file.

Because the binding and linking is done in your source code, this version is lengthier. However, it is doing the same thing as before, as highlighted in the following snippet (some print statements and error handling omitted).

File [client2.c](./client2.c)
```c
#include <stdlib.h>
#include <stdio.h>
#include <dlfcn.h>

// define types needed
typedef long long go_int;
typedef double go_float64;
typedef struct{void *arr; go_int len; go_int cap;} go_slice;
typedef struct{const char *p; go_int len;} go_str;

int main(int argc, char **argv) {
    void *handle;
    char *error;

    // use dlopen to load shared object
    handle = dlopen ("./awesome.so", RTLD_LAZY);
    if (!handle) {
        fputs (dlerror(), stderr);
        exit(1);
    }
    
    // resolve Add symbol and assign to fn ptr
    go_int (*add)(go_int, go_int)  = dlsym(handle, "Add");
    if ((error = dlerror()) != NULL)  {
        fputs(error, stderr);
        exit(1);
    }
    // call Add()
    go_int sum = (*add)(12, 99); 
    printf("awesome.Add(12, 99) = %d\n", sum);

    // resolve Cosine symbol
    go_float64 (*cosine)(go_float64) = dlsym(handle, "Cosine");
    if ((error = dlerror()) != NULL)  {
        fputs(error, stderr);
        exit(1);
    }
    // Call Cosine
    go_float64 cos = (*cosine)(1.0);
    printf("awesome.Cosine(1) = %f\n", cos);

    // resolve Sort symbol
    void (*sort)(go_slice) = dlsym(handle, "Sort");
    if ((error = dlerror()) != NULL)  {
        fputs(error, stderr);
        exit(1);
    }
    // call Sort
    go_int data[5] = {44,23,7,66,2};
    go_slice nums = {data, 5, 5};
    sort(nums);
    printf("awesome.Sort(44,23,7,66,2): ");
    for (int i = 0; i < 5; i++){
        printf("%d,", ((go_int *)data)[i]);
    }
    printf("\n");

    // resolve Log symbol
    go_int (*log)(go_str) = dlsym(handle, "Log");
    if ((error = dlerror()) != NULL)  {
        fputs(error, stderr);
        exit(1);
    }
    // call Log
    go_str msg = {"Hello from C!", 13};
    log(msg);
    
    // close file handle when done
    dlclose(handle);
}
```
In the previous code, we define our own subset of Go compatible C types `go_int`, `go_float`, `go_slice`, and `go_str`. We use `dlsym` to load symbols `Add`, `Cosine`, `Sort`, and `Log` and assign them to their respective function pointers. Next, we compile the code linking it with the `dl` library (not the awesome.so) as follows:
```shell
$> gcc -o client client2.c -ldl
```
When the code is executed, the C binary loads and links to shared library awesome.so producing the following output:
```shell
$> ./client
awesome.Add(12, 99) = 111
awesome.Cosine(1) = 0.540302
awesome.Sort(44,23,7,66,2): 2,7,23,44,66,
Hello from C!
```
## From Python
In Python things get a little easier. We use can use the `ctypes` [foreign function library](https://docs.python.org/3/library/ctypes.html) to call Go functions from the the awesome.so shared library as shown in the following snippet (some print statements are omitted).

File [client.py](./client.py)
```python
from __future__ import print_function
from ctypes import *

lib = cdll.LoadLibrary("./awesome.so")

# describe and invoke Add()
lib.Add.argtypes = [c_longlong, c_longlong]
lib.Add.restype = c_longlong
print("awesome.Add(12,99) = %d" % lib.Add(12,99))

# describe and invoke Cosine()
lib.Cosine.argtypes = [c_double]
lib.Cosine.restype = c_double
print("awesome.Cosine(1) = %f" % lib.Cosine(1))

# define class GoSlice to map to:
# C type struct { void *data; GoInt len; GoInt cap; }
class GoSlice(Structure):
    _fields_ = [("data", POINTER(c_void_p)), ("len", c_longlong), ("cap", c_longlong)]

nums = GoSlice((c_void_p * 5)(74, 4, 122, 9, 12), 5, 5) 

# call Sort
lib.Sort.argtypes = [GoSlice]
lib.Sort.restype = None
lib.Sort(nums)
print("awesome.Sort(74,4,122,9,12) = %s" % [nums.data[i] for i in range(nums.len)])

# define class GoString to map:
# C type struct { const char *p; GoInt n; }
class GoString(Structure):
    _fields_ = [("p", c_char_p), ("n", c_longlong)]

# describe and call Log()
lib.Log.argtypes = [GoString]
lib.Log.restype = c_longlong
msg = GoString(b"Hello Python!", 13)
print("log id %d"% lib.Log(msg))
```
Note the `lib` variable represents the loaded symbols from the shared object file. We also defined Python classes `GoString` and `GoSlice` to map to their respective C struct types. When the Python code is executed, it calls the Go functions in the shared object producing the following output:
```shell
$> python client.py
awesome.Add(12,99) = 111
awesome.Cosine(1) = 0.540302
awesome.Sort(74,4,122,9,12) = [4, 9, 12, 74, 122]
Hello Python!
log id 1
```

### Python CFFI (contributed)
The following example was contributed by [@sbinet](https://github.com/sbinet) (thank you!)

Python also has a portable CFFI library that works with Python2/Python3/pypy unchanged.  The 
following example uses a C-wrapper to defined the exported Go types.  This makes
the python example less opaque and even easier to understand.

File [client-cffi.py](./client-cffi.py)
```python
from __future__ import print_function
import sys
from cffi import FFI

is_64b = sys.maxsize > 2**32

ffi = FFI()
if is_64b: ffi.cdef("typedef long GoInt;\n")
else:      ffi.cdef("typedef int GoInt;\n")

ffi.cdef("""
typedef struct {
    void* data;
    GoInt len;
    GoInt cap;
} GoSlice;

typedef struct {
    const char *data;
    GoInt len;
} GoString;

GoInt Add(GoInt a, GoInt b);
double Cosine(double v);
void Sort(GoSlice values);
GoInt Log(GoString str);
""")

lib = ffi.dlopen("./awesome.so")

print("awesome.Add(12,99) = %d" % lib.Add(12,99))
print("awesome.Cosine(1) = %f" % lib.Cosine(1))

data = ffi.new("GoInt[]", [74,4,122,9,12])
nums = ffi.new("GoSlice*", {'data':data, 'len':5, 'cap':5})
lib.Sort(nums[0])
print("awesome.Sort(74,4,122,9,12) = %s" % [
    ffi.cast("GoInt*", nums.data)[i] 
    for i in range(nums.len)])

data = ffi.new("char[]", b"Hello Python!")
msg = ffi.new("GoString*", {'data':data, 'len':13})
print("log id %d" % lib.Log(msg[0]))
```


## From Ruby
Calling Go functions from Ruby follows a similar pattern as above. We use the the [FFI gem](https://github.com/ffi/ffi) to dynamically load and call exported Go functions in the awesome.so shared object file as shown in the following snippet.

File [client.rb](./client.rb)
```ruby
require 'ffi'

# Module that represents shared lib
module Awesome
  extend FFI::Library
  
  ffi_lib './awesome.so'
  
  # define class GoSlice to map to:
  # C type struct { void *data; GoInt len; GoInt cap; }
  class GoSlice < FFI::Struct
    layout :data,  :pointer,
           :len,   :long_long,
           :cap,   :long_long
  end

  # define class GoString to map:
  # C type struct { const char *p; GoInt n; }
  class GoString < FFI::Struct
    layout :p,     :pointer,
           :len,   :long_long
  end

  # foreign function definitions
  attach_function :Add, [:long_long, :long_long], :long_long
  attach_function :Cosine, [:double], :double
  attach_function :Sort, [GoSlice.by_value], :void
  attach_function :Log, [GoString.by_value], :int
end

# Call Add
print "awesome.Add(12, 99) = ",  Awesome.Add(12, 99), "\n"

# Call Cosine
print "awesome.Cosine(1) = ", Awesome.Cosine(1), "\n"

# call Sort
nums = [92,101,3,44,7]
ptr = FFI::MemoryPointer.new :long_long, nums.size
ptr.write_array_of_long_long  nums
slice = Awesome::GoSlice.new
slice[:data] = ptr
slice[:len] = nums.size
slice[:cap] = nums.size
Awesome.Sort(slice)
sorted = slice[:data].read_array_of_long_long nums.size
print "awesome.Sort(", nums, ") = ", sorted, "\n"

# Call Log
msg = "Hello Ruby!"
gostr = Awesome::GoString.new
gostr[:p] = FFI::MemoryPointer.from_string(msg) 
gostr[:len] = msg.size
print "logid ", Awesome.Log(gostr), "\n"
```
In Ruby, we must extend the `FFI` module to declare the symbols being loaded from the shared library. We use Ruby classes `GoSlice` and `GoString` to map the respective C structs. When we run the code it calls the exported Go functions as shown below:
```shell
$> ruby client.rb
awesome.Add(12, 99) = 111
awesome.Cosine(1) = 0.5403023058681398
awesome.Sort([92, 101, 3, 44, 7]) = [3, 7, 44, 92, 101]
Hello Ruby!
```
## From Node
For Node, we use a foreign function library called [node-ffi](https://github.com/node-ffi/node-ffi) (and a couple dependent packages) to dynamically load and call exported Go functions in the awesome.so shared object file as shown in the following snippet:

File [client.js](./client.js)
```js
var ref = require("ref");
var ffi = require("ffi");
var Struct = require("ref-struct")
var ArrayType = require("ref-array")

var longlong = ref.types.longlong;
var LongArray = ArrayType(longlong);

// define object GoSlice to map to:
// C type struct { void *data; GoInt len; GoInt cap; }
var GoSlice = Struct({
  data: LongArray,
  len:  "longlong",
  cap: "longlong"
});

// define object GoString to map:
// C type struct { const char *p; GoInt n; }
var GoString = Struct({
  p: "string",
  n: "longlong"
});

// define foreign functions
var awesome = ffi.Library("./awesome.so", {
  Add: ["longlong", ["longlong", "longlong"]],
  Cosine: ["double", ["double"]], 
  Sort: ["void", [GoSlice]],
  Log: ["longlong", [GoString]]
});

// call Add
console.log("awesome.Add(12, 99) = ", awesome.Add(12, 99));

// call Cosine
console.log("awesome.Cosine(1) = ", awesome.Cosine(1));

// call Sort
nums = LongArray([12,54,0,423,9]);
var slice = new GoSlice();
slice["data"] = nums;
slice["len"] = 5;
slice["cap"] = 5;
awesome.Sort(slice);
console.log("awesome.Sort([12,54,9,423,9] = ", nums.toArray());

// call Log
str = new GoString();
str["p"] = "Hello Node!";
str["n"] = 11;
awesome.Log(str);
```
Node uses the `ffi` object to declare the loaded symbols from the shared library . We also use Node struct objects `GoSlice` and `GoString` to map to their respective C structs. When we run the code it calls the exported Go functions as shown below:
```shell
awesome.Add(12, 99) =  111
awesome.Cosine(1) =  0.5403023058681398
awesome.Sort([12,54,9,423,9] =  [ 0, 9, 12, 54, 423 ]
Hello Node!
```
## From Java
To call the exported Go functions from Java, we use the [Java Native Access library](https://github.com/java-native-access/jna) or JNA as shown in the following code snippet (with some statements omitted or abbreviated):

File [Client.java](./Client.java)
```java
import com.sun.jna.*;
import java.util.*;
import java.lang.Long;

public class Client {
   public interface Awesome extends Library {
        // GoSlice class maps to:
        // C type struct { void *data; GoInt len; GoInt cap; }
        public class GoSlice extends Structure {
            public static class ByValue extends GoSlice implements Structure.ByValue {}
            public Pointer data;
            public long len;
            public long cap;
            protected List getFieldOrder(){
                return Arrays.asList(new String[]{"data","len","cap"});
            }
        }

        // GoString class maps to:
        // C type struct { const char *p; GoInt n; }
        public class GoString extends Structure {
            public static class ByValue extends GoString implements Structure.ByValue {}
            public String p;
            public long n;
            protected List getFieldOrder(){
                return Arrays.asList(new String[]{"p","n"});
            }

        }

        // Foreign functions
        public long Add(long a, long b);
        public double Cosine(double val);
        public void Sort(GoSlice.ByValue vals);
        public long Log(GoString.ByValue str);
    }
 
   static public void main(String argv[]) {
        Awesome awesome = (Awesome) Native.loadLibrary(
            "./awesome.so", Awesome.class);

        System.out.printf("awesome.Add(12, 99) = %s\n", awesome.Add(12, 99));
        System.out.printf("awesome.Cosine(1.0) = %s\n", awesome.Cosine(1.0));
        
        // Call Sort
        // First, prepare data array 
        long[] nums = new long[]{53,11,5,2,88};
        Memory arr = new Memory(nums.length * Native.getNativeSize(Long.TYPE));
        arr.write(0, nums, 0, nums.length); 
        // fill in the GoSlice class for type mapping
        Awesome.GoSlice.ByValue slice = new Awesome.GoSlice.ByValue();
        slice.data = arr;
        slice.len = nums.length;
        slice.cap = nums.length;
        awesome.Sort(slice);
        System.out.print("awesome.Sort(53,11,5,2,88) = [");
        long[] sorted = slice.data.getLongArray(0,nums.length);
        for(int i = 0; i < sorted.length; i++){
            System.out.print(sorted[i] + " ");
        }
        System.out.println("]");

        // Call Log
        Awesome.GoString.ByValue str = new Awesome.GoString.ByValue();
        str.p = "Hello Java!";
        str.n = str.p.length();
        System.out.printf("msgid %d\n", awesome.Log(str));

    }
}
```
To use JNA, we define Java interface `Awesome` to represents the symbols loaded from the awesome.so shared library file. We also declare classes `GoSlice` and `GoString` to map to their respective C struct representations. When we compile and run the code, it calls the exported Go functions as shown below:
```shell
$> javac -cp jna.jar Client.java
$> java -cp .:jna.jar Client
awesome.Add(12, 99) = 111
awesome.Cosine(1.0) = 0.5403023058681398
awesome.Sort(53,11,5,2,88) = [2 5 11 53 88 ]
Hello Java!
```
## From Lua (contributed)
This example was contributed by [@titpetric](https://github.com/titpetric). See his insightful write up on [*Calling Go functions from LUA*](https://scene-si.org/2017/03/13/calling-go-functions-from-lua/).

The forllowing shows how to invoke exported Go functions from Lua. As before, it uses an FFI library to dynamically load the shared object file and bind to the exported function symbols.

File [client.lua](./client.lua)
```lua
local ffi = require("ffi")
local awesome = ffi.load("./awesome.so")

ffi.cdef([[
typedef long long GoInt64;
typedef unsigned long long GoUint64;
typedef GoInt64 GoInt;
typedef GoUint64 GoUint;
typedef double GoFloat64;

typedef struct { const char *p; GoInt n; } GoString;
typedef struct { void *data; GoInt len; GoInt cap; } GoSlice;

extern GoInt Add(GoInt p0, GoInt p1);
extern GoFloat64 Cosine(GoFloat64 p0);
extern void Sort(GoSlice p0);
extern GoInt Log(GoString p0);
]]);


io.write( string.format("awesome.Add(12, 99) = %f\n", math.floor(tonumber(awesome.Add(12,99)))) )

io.write( string.format("awesome.Cosine(1) = %f\n", tonumber(awesome.Cosine(1))) )

local nums = ffi.new("long long[5]", {12,54,0,423,9})
local numsPointer = ffi.new("void *", nums);
local typeSlice = ffi.metatype("GoSlice", {})
local slice = typeSlice(numsPointer, 5, 5)
awesome.Sort(slice)

io.write("awesome.Sort([12,54,9,423,9] = ")
for i=0,4 do
	if i > 0 then
		io.write(", ")
	end
	io.write(tonumber(nums[i]))
end
io.write("\n");

local typeString = ffi.metatype("GoString", {})
local logString = typeString("Hello LUA!", 10)
awesome.Log(logString)
```
When the example is executed, it produces the following:
```
$> luajit client.lua
awesome.Add(12, 99) = 111.000000
awesome.Cosine(1) = 0.540302
awesome.Sort([12,54,9,423,9] = 0, 9, 12, 54, 423
Hello LUA!
```
## From Julia (Contributed)
The following example was contributed by [@r9y9](https://github.com/r9y9). It shows how to invoke exported Go functions from the Julia language. As [documented here](https://docs.julialang.org/en/stable/manual/calling-c-and-fortran-code/), Julia has the capabilities to invoke exported functions from shared libraries similar to other languages discussed here.

File [client.jl](./client.jl)

```julia
struct GoSlice
    arr::Ptr{Void}
    len::Int64
    cap::Int64
end
GoSlice(a::Vector, cap=10) = GoSlice(pointer(a), length(a), cap)

struct GoStr
    p::Ptr{Cchar}
    len::Int64
end
GoStr(s::String) = GoStr(pointer(s), length(s))

const libawesome = "awesome.so"

Add(x,y) = ccall((:Add, libawesome), Int,(Int,Int), x,y)
Cosine(x) = ccall((:Cosine, libawesome), Float64, (Float64,), x)
function Sort(vals)
    ccall((:Sort, libawesome), Void, (GoSlice,), GoSlice(vals))
    return vals # for convenience
end
Log(msg) = ccall((:Log, libawesome), Int, (GoStr,), GoStr(msg))

for ex in [:(Add(12, 9)),:(Cosine(1)), :(Sort([77,12,5,99,28,23]))]
    println("awesome.$ex = $(eval(ex))")
end
Log("Hello from Julia!")
```
When the example is executed, it produces the following:

```
> julia client.jl
awesome.Add(12, 9) = 21
awesome.Cosine(1) = 0.5403023058681398
awesome.Sort([77, 12, 5, 99, 28, 23]) = [5, 12, 23, 28, 77, 99]
Hello from Julia!
```
From Dart (contributed)
The following example was contributed by @purfield. It shows how to invoke exported Go functions from the Dart language. As documented [here](https://dart.dev/guides/libraries/c-interop), Dart has the capability to invoke exported functions from shared libraries similar to other languages discussed here.

File [client.dart](./client.dart)

```dart
import 'dart:convert';
import 'dart:ffi';
import 'dart:io';

class GoSlice extends Struct<GoSlice> {
  Pointer<Int64> data;

  @Int64()
  int len;

  @Int64()
  int cap;

  List<int> toList() {
    List<int> units = [];
    for (int i = 0; i < len; ++i) {
      units.add(data.elementAt(i).load<int>());
    }
    return units;
  }

  static Pointer<GoSlice> fromList(List<int> units) {
    final ptr = Pointer<Int64>.allocate(count: units.length);
    for (int i =0; i < units.length; ++i) {
      ptr.elementAt(i).store(units[i]);
    }
    final GoSlice slice = Pointer<GoSlice>.allocate().load();
    slice.data = ptr;
    slice.len = units.length;
    slice.cap = units.length;
    return slice.addressOf;
  }
}

class GoString extends Struct<GoString> {
  Pointer<Uint8> string;

  @IntPtr()
  int length;

  String toString() {
    List<int> units = [];
    for (int i = 0; i < length; ++i) {
      units.add(string.elementAt(i).load<int>());
    }
    return Utf8Decoder().convert(units);
  }

  static Pointer<GoString> fromString(String string) {
    List<int> units = Utf8Encoder().convert(string);
    final ptr = Pointer<Uint8>.allocate(count: units.length);
    for (int i = 0; i < units.length; ++i) {
      ptr.elementAt(i).store(units[i]);
    }
    final GoString str = Pointer<GoString>.allocate().load();
    str.length = units.length;
    str.string = ptr;
    return str.addressOf;
  }
}

typedef add_func = Int64 Function(Int64, Int64);
typedef Add = int Function(int, int);
typedef cosine_func = Double Function(Double);
typedef Cosine = double Function(double);
typedef log_func = Int64 Function(Pointer<GoString>);
typedef Log = int Function(Pointer<GoString>);
typedef sort_func = Void Function(Pointer<GoSlice>);
typedef Sort = void Function(Pointer<GoSlice>);

void main(List<String> args) {

  final awesome = DynamicLibrary.open('awesome.so');

  final Add add = awesome.lookup<NativeFunction<add_func>>('Add').asFunction();
  stdout.writeln("awesome.Add(12, 99) = ${add(12, 99)}");

  final Cosine cosine = awesome.lookup<NativeFunction<cosine_func>>('Cosine').asFunction();
  stdout.writeln("awesome.Cosine(1) = ${cosine(1.0)}");

  final Log log = awesome.lookup<NativeFunction<log_func>>('LogPtr').asFunction();
  final Pointer<GoString> message = GoString.fromString("Hello, Dart!");
  try {
    log(message);
  }
  finally {
    message.free();
  }

  final Sort sort = awesome.lookup<NativeFunction<sort_func>>('SortPtr').asFunction();
  var nums = [12,54,0,423,9];
  final Pointer<GoSlice> slice = GoSlice.fromList(nums);
  try {
    sort(slice);
    stdout.writeln(slice.load<GoSlice>().toList());
  } finally {
    slice.free();
  }

  for (int i=0; i < 100000; i++) {
    Pointer<GoString> m = GoString.fromString("Hello, Dart!");
    Pointer<GoSlice> s = GoSlice.fromList(nums);
    print("$m $s");
    m.free();
    s.free();
  }

  stdin.readByteSync();
}
```

## Conclusion
This repo shows how to create a Go library that can be used from C, Python, Ruby, Node, Java, Lua, Julia. By compiling Go packages into C-style shared libraries, Go programmers have a powerful way of integrating their code with any modern language that supports dynamic loading and linking of shared object files.
