# FRIDA CHEATSHEET(BASIC BINARIES)
---
**NOTE- For whole tutorial refer frida-boot**
## Setup
### Attaching to a already running executable
```bash=
frida pew -l index.js
```
### Index.js
```javascript=
Interceptor.attach(target, {
    onEnter: function(args) { ... },
    onLeave: function(retval) {... }
});
```
## Basic Hook using Frida
### Extracting exports existed in libc

```javascript=
var libc = Process.getModuleByName("libc-2.31.so"); //libc ver differs
var exports = libc.enumerateExports();

console.log(exports);
```
### Extracting a particular syscall address
```javascript=
Process.getModuleByName("libc-2.30.so")
    .enumerateExports()
        .filter(function(n) {
            return n.name == "sleep";
        });
```
#### OR
```javascript=
Module.getExportByName("libc-2.30.so", "sleep");
```
Output will be the addr
### Attaching to function call

```javascript=
var sleep = Module.getExportByName(null, "sleep");

Interceptor.attach(sleep, {
    onEnter: function(args) {
        console.log("[*] Sleep from Frida!");
    },
    onLeave: function(retval) {
        console.log("[*] Done sleeping from Frida!");
    }
});
```
## Interceptor Arguments
### Extracting args of a function
```javascript=
var sleep = Module.getExportByName(null, "sleep");

Interceptor.attach(sleep, {
    onEnter: function(args) {
        console.log("[*] Argument for sleep() => " + parseInt(args[0]));
        console.log("[*] Sleep from Frida!");
    },
    onLeave: function(retval) {
        console.log("[*] Done sleeping from Frida!");
    }
});
```
### Overriding Arguments
```javascript=
var sleep = Module.getExportByName(null, "sleep");

Interceptor.attach(sleep, {
    onEnter: function(args) {
        console.log("[*] Argument for sleep() => " + parseInt(args[0]));
        console.log("[*] Overriding argument to 1");
        args[0] = ptr("0x1");   // short for new NativePointer("0x1");
    },
    onLeave: function(retval) {
        console.log("[*] Done sleeping from Frida!");
    }
});
```
### Overriding String Arguments
```javascript=
var printf = Module.getExportByName(null, "printf");

// Allocate a new memory region, returning the pointer to the string.
var buf = Memory.allocUtf8String("Frida sleep! :D\n");

Interceptor.attach(printf, {
    onEnter: function(args) {
        console.log("printf(\"" + args[0].readCString().trim() + "\")");
        args[0] = buf;  // update the argument to printf
    }
});
```
### Register Access
```javascript=
var printf = Module.getExportByName(null, "printf");

Interceptor.attach(printf, {
    onEnter: function(args) {
        console.log('Context information:');
        console.log('Context  : ' + JSON.stringify(this.context, null, 4));
        console.log('Return   : ' + this.returnAddress);
        console.log('ThreadId : ' + this.threadId);
        console.log('Errornr  : ' + this.err);
    }
});
```
## Interceptor Return Values
### DebugSymbol API
```
DebugSymbol.getFunctionByName("rand_range");
```
### Stripped Binaries
```javascript=
var b = Process.getModuleByName("pew").base;
var rand_range = b.add(0x1185);

Interceptor.attach(rand_range, {
    onEnter: function(args) {
        console.log("rand_range(" + args[0] + ", " + args[1] +")");
    }
});
```
#### OR
```javascript=
var rand_range = DebugSymbol.getFunctionByName("rand_range");

Interceptor.attach(rand_range, {
    onEnter: function(args) {
        console.log("rand_range(" + args[0] + ", " + args[1] +")");
    }
});
```
### Modifying Return Values
```javascript=
var rand_range = DebugSymbol.getFunctionByName("rand_range");

Interceptor.attach(rand_range, {
    onLeave: function(retval) {
        console.log(retval);
        retval.replace(ptr("0x1"));
    }
});
```
### Data binding in onenter and onleave
```javascript=
var rand_range = DebugSymbol.getFunctionByName("rand_range");

Interceptor.attach(rand_range, {
    onEnter: function(args) {
        this.arg1 = args[0];
    },
    onLeave: function(retval) {
        console.log(retval);
        retval.replace(ptr(this.arg1));
    }
});

```
