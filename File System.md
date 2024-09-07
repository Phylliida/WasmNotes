# WASI

Wasm files frequently import things like `fd_write` `fd_read`. These must be defined in order for us to be able to run the wasm.

These originate from the WASI standard, which let wasm do stuff they couldn't normally do
(connect to network, open files, etc.) by calling outside functions.

These are somewhat undocumented/in progress functions.
Best documentation I've found is [https://wasix.org/docs/api-reference](WASIX) (though that's a superset of WASI and includes extra stuff).

Obviously we do not want to let users write to arbitrary files, or make arbitrary network connections.

WASI shares these goals, and access to any of it's functionalities need to be enabled manually.

So the simplest thing to do is a pop-up (like with networking) where WASI requests to access files before accessing them.

I think there's a better solution, but first:

# stdin/stdout/stderr

The file system calls actually serve two purposes:
1. Reading from stdin (file descriptor 0) and Writing to stdout (file descriptor 1) and stderr (file descriptor 2)
2. Reading and writing files

Many webassembly programs don't ever open files, and just use stdin/stdout/stderr.

Fortunately, Wasmtime dot net includes a `WasiConfiguration` that you can give to `Store`. This provides two options:
- Redirect to a file.
- Inherit stdin/stdout from the calling program (Resonite)

Redirect to a (temp) file seems like a good option.

Vulnerability: Flooding the user's hard drive with large tmp file.
Solution: Checks on file size and terminate the program and delete the tmp file if it gets too big (possibly users can override this setting manually). (question: how fast can these be filled up? Maybe checks not fast enough? TBD)

An even better option would be to have custom stdin/stdout streams. [Wasmtime supports this](https://github.com/bytecodealliance/wasmtime/issues/7581), however Wasmtime dot net does not.

TODO: Make a PR to Wasmtime dot net to support this functionality.

Alternatively, we can just manually attach the various fs_... imports to our implementation. But that seems like overkill when this functionality is supported.

# Alternative: Use an in memory (virtual) file system

Some programs do require reading and writing from files, not just std io.

Two options here:
- In memory file system implemented externally, and attached to wasm
- File system implemented in a wasm file

## In memory file system implemented externally

This is commonly done in javascript, where there is no file system access.

This may have a performance advantage, but comes with extra functionality outside of wasm that needs to be maintained.

If we want to implement that in c#, [https://github.com/xoofx/zio](https://github.com/xoofx/zio) seems like a good choice.

## File system implemented in a wasm file

wasm can allocate memory itself, so it's feasible to make an in memory file system entirely in a wasm file.

Emscripten does this, with its MEMFS. I'm still learning how to make a wasm file that just has MEMFS and nothing else.

Alternatively, there's [https://github.com/bytecodealliance/WASI-Virt](https://github.com/bytecodealliance/WASI-Virt). 
- But it is read-only 
