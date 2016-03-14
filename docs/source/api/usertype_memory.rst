usertype memory
===============

The userdata generated by Sol has a specific layout, depending on how Sol recognizes userdata passed into it. All of the referred to metatable names are generated from :ref:`usertype_traits\<T><usertype-traits>`

In general, we always insert a T* in the first `sizeof(T*)` bytes, so the any framework that pulls out those first bytes expecting a pointer will work. The rest of the data has some different alignments and contents based on what it's used for and how it's used.

For ``T``
---------

These are classified with the metatable from ``

The data layout for references is as follows::

	|        T*        |               T              |
	^-sizeof(T*) bytes-^-sizeof(T) bytes, actual data-^

Lua will clean up the memory itself but does not know about any destruction semantics T may have imposed, so when we destroy this data we simply call the destructor to destroy the object and leave the memory changes to for lua to handle after the "__gc" method exits.


For ``T*``
----------

These are classified as a separate ``T*`` metatable, essentially the "reference" table. Things passed to Sol as a pointer or as a ``std::reference<T>`` are considered to be references, and thusly do not have a ``__gc`` (garbage collection) method by default. All raw pointers are non-owning pointers in C++. If you're working with a C API, provide a wrapper around pointers that are supposed to own data and use the constructor/destructor idioms (e.g., with an internal ``std::unique_ptr``) to keep things clean.

The data layout for data that only refers is as follows::

	|        T*        |
	^-sizeof(T*) bytes-^

That is it. No destruction semantics need to be called.

For ``std::unique_ptr<T, D>`` and ``std::shared_ptr<T>``
--------------------------------------------------------

These are classified as :ref:`"unique usertypes"<unique-usertype>`, and have a special metatable for them as well. The special metatable is either generated when you add the usertype to Lua using :ref:`set_usertype<set-usertype>` or when you first push one of these special types. In addition to the data, a deleter function that understands the following layout is injected into the usertype.

The data layout for these kinds of types is as follows::

	|        T*        |    void(*)(void*) function_pointer    |               T               |
	^-sizeof(T*) bytes-^-sizeof(void(*)(void*)) bytes, deleter-^- sizeof(T) bytes, actal data -^

Note that we put a special deleter function before the actual data. This is because the custom deleter must know where the offset to the data is, not the rest of the library. Sol just needs to know about ``T*`` and the userdata (and userdata metatable) to work, everything else is for preserving construction / destruction syntax.