# Gravity C API
This document is here to serve as a cheat-sheet for the Gravity C API.

Gravity: https://github.com/marcobambini/gravity

## NOTE
Take this documentation with a grain of salt: I do not maintain Gravity I only use it, and dig through the source code a LOT. I may be highly interested in the creation of programming languages and their usage, especially in regards to features, but I actually do not have the know-how to create one myself. So... I just use what I find :)

Scripting languages I have played with before include:
- ObjectScript (seems deprecated - lack of activity): https://github.com/unitpoint/objectscript
  * Lots and lots of syntactic sugar.
  * Allows for some crazy-cool syntax hacks:
    ```js
    var myFunc = function(head) {
      return function(body) {
        // ...
      }
    }

    myFunc("head") "body"
    myFunc("head") { /*...*/ }
    ```
- AngelScript: https://www.angelcode.com/angelscript/
  * I highly recommend this one if you need a typed language. It's binding API is one of a kind!
- Dao: https://github.com/daokoder/dao
- Wren: https://github.com/wren-lang/wren

## TODO

- [ ] `gravity_compiler_serialize_infile`: Find out what it _actually_ does.
- [ ] Decide wether or not to document `gravity_ast.h`, `gravity_codegen.h`, `gravity_lexer.h` and `gravity.parser.h`
  - Those files are only required for modding.
  - The AST and Codegen functions may be used for writing debuggers of some sort.
  - New Syntax requires modifying the lexer and parser and maybe the AST.
- [ ] `vm_transfer_cb`:
  - What does this one do?
  - For what does it actually return a boolean?
- [ ] `gravity_vm_loadclosure`
  - As far as I understand it, this function transfers a closure into the VM.
  - Do I need to use this to utilize `runclosure`?
- [ ] `gravity_vm_filter`: What does it filter?
- [ ] `gravity_vm_initmodule`: Is this meant as part of the Module system?
- [ ] `gravity_vm_memupdate`
  - Does this function increase pre-allocated memory?
  - Why does it take a `gravity_value_t` instead of an integer type as parameter?
- [ ] `gravity_vm_maxmemblock`
  - I bet this is meant as a counterpart to `memupdate`.
  - What does it actually return - what is this value meant to be?
- [ ] `gravity_vm_anonymous`: What does it return?
- [ ] `marray_resize0`: Does it resize the array whilst emptying everything?

# The cheat-sheet
## The structure
Basically, Gravity is split into several parts:
- `cli`: Gravity's CLI executable.
- `compiler`: The code needed to compile Gravity source code into instructions it's VM can use.
- `optionals`: Includes stuff that is...well, optional.
- `runtime`: The code needed to execute Gravity code. Compile it to instructions first, then run it with the VM within this portion of the code.
- `shared`: Code used by the compiler and the runtime.
- `utils`: Small helpers.

## The most important functions
### `compiler/`
#### `gravity_compiler.h`
- `gravity_compiler_t *gravity_compiler_create (gravity_delegate_t *delegate);`
  * Description: This function creates a compiler for the Gravity language.
  * Parameters:
    1. `gravity_delegate_t*`: A delegate. It contains the compiler AND runtime settings.
  * Returns a `gravity_compiler_t`.
- `gravity_closure_t *gravity_compiler_run (gravity_compiler_t *compiler, const char *source, size_t len, uint32_t fileid, bool is_static, bool add_debug);`
  * Description: Execute the compiler on a given source code.
  * Parameters:
    1. `gravity_compiler_t *compiler`: This is the compiler you created previously.
    2. `const char *source`: The source that should be compiled.
    3. `size_t len`: The length of the source code (which means, your code does not need to be `NULL`-terminated).
    4. `uint32_t fileid`: A nummeric value that uniquely identifies sources included via the `#include` directive.
    5. `bool is_static`: This parameter tells the compiler if the buffer passed in the `const char *source` parameter must be freed or not.
    6. `bool add_debug`: Add debugging information to the source (think `gcc -g`).
  * Returns a `gravity_closure_t*` which can now be executed. It essentially contains the compiled source, as a "program".
- `json_t  *gravity_compiler_serialize (gravity_compiler_t *compiler, gravity_closure_t *closure);`
  * Description: Serialize a compiled Gravity program into JSON data.
  * Parameters:
    1. `gravity_compiler_t *compiler`: The compiler that **was used to compile the code**.
    2. `gravity_closure_t *closure`: The closure (program) you would like to serialize.
      - You could also serialize only specific functions, if you wanted to.
  * Returns a JSON object containing the serialized closure.
- `bool gravity_compiler_serialize_infile (gravity_compiler_t *compiler, gravity_closure_t *closure, const char *path);`
  * Description: Serialize a Gravity script. (I think?)
  * Parameters:
    1. `gravity_compiler_t *compiler`: The compiler instance to be used.
    2. `gravity_closure_t *closure`: The closure pointer you want the compiled code to be stored at.
      - I dont think you can pass `NULL` in here, actually.
    3. `const char *path`: Path to the file you'd like to compile.
  * Returns wether the compilation was successful or not.
- `void gravity_compiler_transfer (gravity_compiler_t *compiler, gravity_vm *vm);`
  * Description: Transfer a compiled program to the VM. You have to do this for execution. The instructions on their own won't do anything unless they are transfered to the VM. Think of this as the same process as to when your OS copies a program to RAM to hand it to the CPU.
  * Parameters:
    1. `gravity_compiler_t *compiler`: The compiler used for the compilation.
    2. `gravity_vm_t *vm`: The target VM.
  * Returns nothing.
- `gnode_t *gravity_compiler_ast (gravity_compiler_t *compiler);`
  * Description: Obtain the whole AST currently available in the compiler.
  * Parameters:
    1. `gravity_compiler_t *compiler`: The compiler whose AST you'd like to get.
  * Returns a `gnode_t`. This is part of the lexer and parser. It represents the AST.
- `void gravity_compiler_free (gravity_compiler_t *compiler);`
  * Description: Frees a compiler instance.
  * Parameters:
    1. `gravity_compiler_t *compiler`: The compiler to free.
  * Returns nothing.

#### `gravity_ast.h`
You need this file only for either of these reasons:
- Traversing the AST returned from `gravity_compiler_ast`.
- As part of extending a syntax that isn't at lest partially covered already.
  - Custom macros won't need this, for instance.
  - Custom syntax however, might.

#### `gravity_codegen.h`
The only function in this file is `gravity_codegen`. I am _very_ certain this is an internal function and used within the compiler to generate the code off an AST.

#### `gravity_ircode.h`
"ir" means "Intermediate Representation" - so, "ircode" is "Intermediate Representation Code" - and worded correctly, it is what we commonly know from LLVM and it's "Intermediate Bytecode Representation". This is _definitively_ not meant for the public, and therefore you won't need to touch it. Ever.

#### `gravity_optimization.h`
The function within here, `gravity_optimizer`, optimizes a given `gravity_function_t` and additionaly adds debugging information to it (second parameter, `bool add_Debug`). You can be sure that the compiler already runs this for you. So unless you are recreating the compilation process, you will not need this.

However. Speaking of LLVM. You do realize that, by all functions laid out, you can easily drop all related functions into the various LLVM compilation steps and "teach" it how to compile Gravity, right? :p Just a random thought.

#### `gravity_semacheck1.h` / `gravity_semacheck2.h`
Semantic checks, related to the parser and lexer.

#### `gravity_symboltable.h`
This represents the symbol table used within a compiler. Usually, this is used by a C compiler for picking up adreses and generating instructions with those. So you will not need to use this either.

#### `gravity_token.h` / `gravity_visitor.h`
A token is used to denote a specific piece of syntax for the lexer and parser, whilst a visitor is used to verify and check the order of tokens - or to even modify entire AST trees as well. BabelJS uses a lot of those to implement various syntactic stuff.

### `optionals/`
#### `gravity_math.h`
This file contains functions to represent a `Math` object within Gravity. You can use this and it's `.c` source file to build your own "optionals".

#### `gravity_optionals.h`
This file serves as a nexus for all the optionals. It's macros can be used to add optionals - or even to "silently" disable them. Use the actual functions off the very optionals you want to use, if you want to use them and be sure that they are there.

### `runtime/`
#### `gravity_core.h`
The following functions aren't neccessary to be called. Upon creating a VM, those are done for you already. However, I will document them anyway because there may be a reason to call them directly.
- `void gravity_core_init (void);`: Initialize the core modules.
- `void gravity_core_register (gravity_vm *vm);`: Register the core module to this given VM.
- `bool gravity_iscore_class (gravity_class_t *c);`
  * Description: Check if the given value is actually a Core class.
    - To turn `gravity_value_t` into a class pointer: `gravity_class_t *c = VALUE_AS_CLASS(...myValue...)`
  * Parameters:
    1. `gravity_class_t *c`: The class to inspect.
  * Returns wether the given class is or is not a Core class.
- `void gravity_core_free (void);`: The opposit of `_init(void);`, destruct the core module.
- `const char **gravity_core_identifiers (void);`: Returns the name of all the core identifiers as a `const char*` array.
  - CAUTION: You are not given a size, but the last element is `NULL` instead.
- `gravity_class_t *gravity_core_class_from_name (const char *name);`
  * Description: Obtain a core class by it's name.
  * Parameters:
    1. `const char *name`: The name of the class you want to retrive.
  * Returns the class pointer to the given class - `NULL` otherwise (likely - I actually don't know).

The other functions are internal.

#### `gravity_vm.h`
This header contains a lot of macros and stuff. I will only go over what a user really, really needs here, because this file essentially represents the entire VM.

- `gravity_vm *gravity_vm_new (gravity_delegate_t *delegate);`
  * Description: Create a new VM instance.
  * Parameters:
    1. `gravity_delegate_t *delegate`: Just like creating a compiler, a VM needs settings. The delegate is used to present them.
  * Returns a new VM instance.
- `gravity_vm *gravity_vm_newmini (void);`: Same as above - but creates a most-minimal VM. Therefore, you dont need to pass a delegate, therefore the VM is set up with the most minimal settings.
- `void gravity_vm_set_callbacks (gravity_vm *vm, vm_transfer_cb vm_transfer, vm_cleanup_cb vm_cleanup);`
  * Description: This function allows you to attach some callbacks to the VM. Those are very specific though.
  * Parameters:
    1. `gravity_vm *vm`: The VM to which the callbacks should be applied to.
    2. `vm_transfer_cb vm_transfer`: A function pointer to a callback which should be called when stuff is transfered into the VM.
      - Definition for `vm_transfer_cb`: `bool f(gravity_object_t *obj)`
      - Description: ???
      - Parameters:
        1. `gravity_object_t *obj`: The object being transfered.
      - Returns: ???
    3. `vm_cleanup_cb vm_cleanup`: Callback being called on a cleanup (possibly a GC run)
      - Definition for `vm_cleanup_cb`: `void f(gravity_vm *vm, gravity_object_t *obj)`
      - Description: This callback is called when cleaning is performed.
      - Parameters:
        1. `gravity_vm *vm`: The VM performing the cleanup.
        2. `gravity_object_t *obj`: The object being cleaned.
      - Returns nothing.
  * Returns nothing.
- `void gravity_vm_free (gravity_vm *vm);`
  * Description: Free a VM
  * Parameters:
    1. `gravity_vm *vm`: The VM to be freed.
  * Returns nothing.
- `void gravity_vm_reset (gravity_vm *vm);`
  * Description: Resets a VM to it's original state. Useful for reusing already allocated memory.
  * Parameters:
    1. `gravity_vm *vm`: The VM to be reset.
  * Returns nothing.
- `bool gravity_vm_runclosure (gravity_vm *vm, gravity_closure_t *closure, gravity_value_t sender, gravity_value_t params[], uint16_t nparams);`
  * Description: Run a specific closure in the VM.
    - You can obtain a closure from compiling source code. (See the `compiler/` section)
    - You can load a closure off an object (class method, instance method, variable containing a function, ...)
    - You can also obtain a closure that had been passed as an argument. However, there are better ways to do this.
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the closure is to be executed.
    2. `gravity_closure_t *closure`: The closure that is to be executed.
    3. `gravity_value_t sender`: The value representing the callee.
    4. `gravity_value_t params[]`: The parameters that are to be given to the closure.
    5. `uint16_t nparams`: The number of parameters to be given to the closure (aka. the length of `params[]`).
  * Returns wether the execution HAPPENED or not. You have to check for an error in other ways!
- `bool gravity_vm_runmain (gravity_vm *vm, gravity_closure_t *closure);`
  * Description: Run `function main()`. It won't get any arguments, though!
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the closure should be executed.
    2. `gravity_closure_t *closure`: The closure to be executed.
  * Returns wether the execution HAPPENED or not.
- `void gravity_vm_loadclosure (gravity_vm *vm, gravity_closure_t *closure);`
  * Description: ???
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the closure should be executed.
    2. `gravity_closure_t *closure`: The closure to be loaded.
  * Returns nothing.
- `void gravity_vm_setvalue (gravity_vm *vm, const char *key, gravity_value_t value);`
  * Description: Set a value in the VM (on global scope!)
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the value should be set.
    2. `const char *key`: The key to be used for the value.
    3. `gravity_value_t value`: The actual value to be set.
  * Returns nothing.
- `gravity_value_t gravity_vm_lookup (gravity_vm *vm, gravity_value_t key);`
  * Description: Find a value in the VM (global scope) and return it.
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the look up is to be performed.
    2. `gravity_value_t key`: The key (as a Gravity value) for which the lookup should be performed.
  * Returns the found value - likely `NULL` if none.
    - Mind you, This can either be `(void) 0` kind of value, or a Gravity-wise `null`.
    - `bool isNull = VALUE_ISA_NULL(value);` will help you figure out if the requested value might be a Gravity-style `null`.
- `gravity_value_t gravity_vm_getvalue (gravity_vm *vm, const char *key, uint32_t keylen);`
  * Description: Same as above - but using a regular C string instead.
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the lookup should be performed.
    2. `const char *key`: The key to be looked up as C-string.
    3. `uint32_t keylen`: The length of the key-string.
  * Returns the found value (see above for some notes).
- `double gravity_vm_time (gravity_vm *vm);`
  * Description: Returns the time the VM needed for execution.
  * Parameters:
    1. `gravity_vm *vm`: The VM to be measured.
  * Returns the time, as a `double`.
- `gravity_value_t gravity_vm_result (gravity_vm *vm);`
  * Description: Get the last return value.
    - When using `_runmain(...);`, this will contain the return value of the `main()` function in your script - or `null`.
    - When using the `_runclosure(...);` function, this will contain whatever that closure returned.
  * Parameters:
    1. `gravity_vm *vm`: The VM whose return value should be requested.
  * Returns the last return value (either off `main()` or given closure).
- `gravity_delegate_t *gravity_vm_delegate (gravity_vm *vm);`: Returns the delegate used by the given VM.
- `gravity_fiber_t *gravity_vm_fiber (gravity_vm *vm);`
  * Description: Get the current fiber used by the VM.
    - Fibers are similiar to a thread but not the same. See this: https://marcobambini.github.io/gravity/#/fiber
  * Parameters:
    1. `gravity_vm *vm`: The VM whose fiber should be returned.
  * Returns the fiber used by the given VM.
- `void gravity_vm_setfiber(gravity_vm* vm, gravity_fiber_t *fiber);`
  * Description: Set a fiber for a VM to execute.
  * Parameters:
    1. `gravity_vm *vm`: The VM whose fiber is to be set.
    2. `gravity_fiber_t`: The fiber to be set to the VM.
  * Returns nothing.
- `void gravity_vm_seterror (gravity_vm *vm, const char *format, ...);`
  * Description: Set an error message to the VM. A `printf()` syntax may be used.
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the error should be set.
    2. `const char *format`: The (formatted) string to be used as the error message.
    3. `...`: Like with `printf()`, the values mentioned in the formating can be passed as additional arguments. However, I actually don't know the format...
  * Returns nothing.
- `void gravity_vm_seterror_string (gravity_vm* vm, const char *s);`: Same as above, but with just a simple string instead. It must be `NULL`-terminated though, as no length parameter is passed.
- `bool gravity_vm_ismini (gravity_vm *vm);`: Tells you if the VM is mini. Useful for extensions that may not work on a mini-VM.
- `gravity_value_t gravity_vm_keyindex (gravity_vm *vm, uint32_t index);`
  * Description: Get a value by it's index.
  * Parameters:
    1. `gravity_vm *vm`: The VM on which the lookup should be performed.
    2. `uint32_t index`: The value-index (value ID or key-index, whichever you prefer to call this)
  * Returns the found value.
- `bool gravity_vm_isaborted (gravity_vm *vm);`: Tells you if the VM is aborted.
- `void gravity_vm_setaborted (gravity_vm *vm);`: Set a VM as aborted.
- `gravity_closure_t *gravity_vm_getclosure (gravity_vm *vm);`
  * Description: Get the top-level closure of the VM.
    - When transfering a closure to the VM, it becomes the top-level closure. That is the one this function returns.
  * Parameters:
    1. `gravity_vm *vm`: The VM whose closure is to be returned.
  * Returns the VM's top-level closure.

The following functions are all meant for the garbage collector. Their names are rather self-explaining:
- `void gravity_gray_value (gravity_vm* vm, gravity_value_t v);`
- `void gravity_gray_object (gravity_vm* vm, gravity_object_t *obj);`
- `void gravity_gc_start (gravity_vm* vm);`
- `void gravity_gc_setenabled (gravity_vm* vm, bool enabled);`
- `void gravity_gc_temppush (gravity_vm *vm, gravity_object_t *obj);`
- `void gravity_gc_temppop (gravity_vm *vm);`
- `void gravity_gc_setvalues (gravity_vm *vm, gravity_int_t threshold, gravity_int_t minthreshold, gravity_float_t ratio);`

- `void gravity_vm_transfer (gravity_vm* vm, gravity_object_t *obj);`
  * Description: Transfer an object into a VM
  * Parameters:
    1. `gravity_vm *vm`: The VM an object value is to be transfered into
    2. `gravity_object_t *obj`: The object to be transfered.
  * Returns nothing.
- `void gravity_vm_cleanup (gravity_vm* vm);`
  * Description: Perform a cleanup (which probably implies running the GC).
  * Parameters:
    1. `gravity_vm *vm`: The VM to be cleaned up.
  * Returns nothing.
- `void gravity_vm_filter (gravity_vm* vm, vm_filter_cb cleanup_filter);`
  * Description: ???
  * Parameters:
    1. `gravity_vm *vm`: The VM which should be filtered.
    2. `vm_filter_cb cleanup_filter`: The callback ran during the cleanup.
  * Returns nothing.
- `gravity_closure_t *gravity_vm_loadfile (gravity_vm *vm, const char *path);`
  * Description: Load a file containing bytecode (valid JSON)
  * Parameters:
    1. `gravity_vm *vm`: VM into which the file is to be loaded.
    2. `const char *path`: The path to the file containing the bytecode.
  * Returns the closure resulting from the load (it can be used with `runclosure`).
- `gravity_closure_t *gravity_vm_loadbuffer (gravity_vm *vm, const char *buffer, size_t len);`
  * Description: The same as above, but off a buffer instead of a file.
  * Parameters:
    1. `gravity_vm *vm`: The VM into which the code is to be loaded.
    2. `const char *buffer`: The buffer containing the bytecode.
    3. `size_t len`: The buffer's length.
  * Returns the resulting closure (which can be used with `runmain`).
- `void gravity_vm_initmodule (gravity_vm *vm, gravity_function_t *f);`
  * Description: ???
- `gravity_closure_t *gravity_vm_fastlookup (gravity_vm *vm, gravity_class_t *c, int index);`
  * Description: Perform a fast lookup on a class object.
  * Parameters:
    1. `gravity_vm *vm`: The VM within which the lookup is to be performed.
    2. `gravity_class_t *c`: The class in which the lookup is to be performed.
    3. `int index`: The numeric index of the value to be returned.
  * Returns the value at index `index` of class pointer `*c`.
- ` void gravity_vm_setslot (gravity_vm *vm, gravity_value_t value, uint32_t index);`
  * Description: Set a value in a specific slot.
    - You will use this function _a lot_ when writing native bindings.
    - Setting the return type of a native function is done using this function.
  * Parameters:
    1. `gravity_vm *vm`: The VM in which the slot is to be set.
    2. `gravity_value_t value`: The value to be set.
    3. `uint32_t index`: The index to which the value is to be assigned to.
  * Returns nothing.
- `gravity_value_t gravity_vm_getslot (gravity_vm *vm, uint32_t index);`
  * Description: Retrive a value by it's index.
  * Parameters:
    1. `gravity_vm *vm`: The VM from which the value is to be retrived.
    2. `uint32_t index`: The value's index.
  * Returns the value at the specified index.
- `void gravity_vm_setdata (gravity_vm *vm, void *data);`
  * Description: Set native data to a VM instance.
  * Parameters:
    1. `gravity_vm *vm`: The VM to which the native data is to be assigned to.
    2. `void *xdata`: The native data.
  * Returns nothing.
- `void *gravity_vm_getdata (gravity_vm *vm);`
  * Description: Obtain native data attached to a VM.
  * Parameters:
    1. `gravity_vm *vm`: The VM whose attached native data is to be retrived.
  * Returns a `void*`, pointing to native data.
- `void gravity_vm_memupdate (gravity_vm *vm, gravity_int_t value);`: ???
- `gravity_int_t gravity_vm_maxmemblock (gravity_vm *vm);`: ???
- `gravity_value_t gravity_vm_get (gravity_vm *vm, const char *key);`:
  * Description: Get a value within the VM by it's string-key.
  * Parameters:
    1. `gravity_vm *vm`: The VM from which data is to be retrived.
    2. `const char *key`: The key.
  * Returns the requested value.
- `bool gravity_vm_set (gravity_vm *vm, const char *key, gravity_value_t value);`
  * Description: Set a value in the VM by using a string-key.
    - This is how you ultimatively add classes, functions and values to a VM from within native code.
    - Use the above function to return such values.
  * Parameters:
    1. `gravity_vm *vm`: The VM into which the value is to be inserted.
    2. `const char *key`: The key under which the value is to be inserted.
    3. `gravity_value_t value`: The value.
  * Returns wether setting the value was or was not a success.
- `char *gravity_vm_anonymous (gravity_vm *vm);`: ???
- `bool gravity_isopt_class (gravity_class_t *c);`: Returns wether the given class is an optional class or not.
- `void gravity_opt_register (gravity_vm *vm);`: Register optional classes to the VM.
- `void gravity_opt_free (void);`: Free the optional classes.

### `shared/`
#### `gravity_array.h`
This is an array implementation - no, not your `a[]` arrays you already know, but with a few more functions - making this array implementation a little bit more object-oriented in a C-ish way. And, this is even used to working with array values in Gravity. It may not be obvious at first, but I will explain that later down below.

All of the functions in here, are actually macros and the arrays are actually dynamic in growth.

- `MARRAY_DEFAULT_SIZE`: The default size of an array.
- `marray_t(type)`
  * Use it like this: `typedef marray_t(some_c_type) my_array_type;`
  * This macro generates a struct. So you don't _have_ to use `typedef`, but using it makes things a little bit more readable.
- `marray_init(v)`
  * Initialize an array.
  * Example: `my_array_type arr; marray_init(arr);`
- `marray_decl_init(_t,_v)`
  * Same as above, but unified in one macro.
  * Example: `marray_decl_init(my_array_type, arr);`
- `marray_destroy(v)`
  * Destroy an array. Like `free()`.
- `marray_get(v, i)`: Get the `i`th value in array `v`.
- `marray_pop(v)`: Pop the last value from `v`.
- `marray_last(v)`: Get the last value from `v`.
- `marray_size(v)`: Get the current size of the array.
- `marray_max(v)`: Get the current maximal array size.
  * Note: When pushing elements above the top boundry, the maximum changes.
  * The maximum is changed with a left-shift, so the array grows towards the next bit.
  * `2 << 1 = 4`
- `marray_inc(v)`: Increase the array pointer by one.
- `marray_dec(v)`: Decrease the array pointer by one.
- `marray_nset(v,N)`: Set the current array pointer.
- `marray_push(type, v, x)`: Insert value `x` into array `v` of type `type` at the current array pointer position.
- `marray_resize(type, v, n)`: Resize the array `v` of type `type` by `n` slots.
  * `type` is actually the type defined for the array to hold.
  * `typedef marray_t(int)` would mean that `type` has to be `int`.
- `marray_resize0(type, v, n)`: ???
- `marray_npop(v,k)`: Pop an amount of elements `k` from array `v`.
- `marray_reset(v,k)`: Essentially the same as `marray_nset(v,N)`.
- `marray_reset0(v)`: Reset array `v` to zero elements.
- `marray_set(v,i,x)`: Set value `x` in array `v` at position `i`.

#### `gravity_delegate.h`
This file defines the `gravity_delegate_t` structure, which is passed to the VM and the compiler. In fact, this file holds a couple of very important things:
- Structures to describe errors (`enum error_type_t`, `struct error_desc_t`), which can be used in the error callbacks to generate very well described error messages.
- Callback function definitions.
- Bridge API definitions (to allow dynamic binding of functions/classes/etc. Only really well usable with Objective-C.)
- Unit tests, file handling and optional class loading APIs.
  * Note: `gravity_optclass_callback` does not need to be used to register native types!
  * This callback is probably more intended to utilize `gravity_isopt_class(gravity_class_t*)` by returning a `char*` array of optional class names.
  * This callback can also be used to improve semantic checking, as to make the parser and lexer aware of this optional class, so that symbol checks for defined symbols will pass, when this specific class is referenced.
- The `gravity_delegate_t` struct.
Taking a look at this file itself will explain mostly everything.

#### `gravity_hash.h`
Although it says just `hash` in the name, it actually is a hashtable implementation. Just like `gravity_array`, only it's relevant functions are found in here. Except for a few functions, all of them have a first parameter of `gravity_hash_t *hashtable`, which I will replace with `*self` in the descriptions, just to make reading a bit faster.

- `gravity_hash_t *gravity_hash_create (uint32_t size, gravity_hash_compute_fn compute, gravity_hash_isequal_fn isequal, gravity_hash_iterate_fn free, void *data);`
  * Description: Create a new hash table.
  * Parameters:
    1. `uint32_t`
