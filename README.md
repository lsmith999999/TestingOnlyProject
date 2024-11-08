# FunctionTraits
## C++ function traits library (single header-only) for retrieving info about any function (arg types, arg count, return type, etc.). Clean and easy-to-use, the most "complete" implementation on the web.

See [here](https://godbolt.org/z/KjGvef1x3) for a complete working example (demo that displays all available traits for various sample functions - for those who want to get right to the code). Also see usage examples further below.

"FunctionTraits" is a lightweight C++ traits struct (template) that allows you to quickly and easily determine the traits of any function at compile-time, such as argument types, number of arguments, return type, etc. (for C++17 and later). It's a natural extension to the C++ standard itself, whose [<type\_traits>](https://en.cppreference.com/w/cpp/header/type_traits) header offers almost no support for handling function traits, save for [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function) and [std::is\_member\_function\_pointer](https://en.cppreference.com/w/cpp/types/is_member_function_pointer) (and one or two other borderline cases). It's also a "complete" implementation in the sense that it handles (detects) all mainstream function syntax unlike any other implementation you'll normally find at this writing, which usually fail to address one or more issues (including the Boost implementation - read on). Many (actually most) of these implementations are just hacks or solutions quickly developed on-the-fly to answer someone's question on [stackoverflow.com](https://stackoverflow.com/) for instance. Only a small handful are more complete and far fewer are reasonably complete, though still missing at least one or two features, in particular calling convention support (again, read on).

Some have even written articles on the subject but still don't address various issues. Some of these issues are obvious (or should be), such as failing to detect functions declared as "*noexcept*" (part of a function's type since C++17), or "*const*" and/or "*volatile*" non-static member functions (usually "*const*" in practice), or variadic functions (those whose arguments end with "..."), or function pointers with cv-qualifiers (the pointers themselves are "*const*" and/or "*volatile*"), among other things (often resulting in a lot of cryptic compiler errors).

Some are less obvious, like failing to detect non-static [Member functions with ref-qualifiers](https://en.cppreference.com/w/cpp/language/member_functions#Member_functions_with_ref-qualifier) (rarely used but still required for completeness), or much more prominently (and important usually), functions with calling conventions other than the default (usually "*cdecl*"), which isn't addressed in the C++ standard itself (and a bit more tricky to implement than it first appears as far as creating a functions traits class is concerned). Without (mainstream) calling convention support however, all functions in the entire Microsoft Windows API would fail to be detected for instance, since it's almost entirely based on the "*stdcall*" calling convention. Only the one from Boost ("*boost::callable\_traits*") made a half-hearted attempt to handle calling conventions but it's not officially active (more on this later), it targets the Microsoft C++ compiler only (and in a limited capacity only - again, more on this later), and the feature is even discouraged by the author's own internal comments (referring to it to as "*likely broken*" and "*too much work to maintain*"). For all intents and purposes it's therefore not supported and no other implementation I'm aware of addresses calling conventions at all, even in the (very) few other (mostly) "complete" solutions for handling function traits that are out there (that I've ever come across). Again, the result is usually a lot of cryptic compiler errors or possibly incorrect behavior (including in the Boost version since its limited support for calling conventions is officially inactive).

"*FunctionTraits*" does address all these issues however. It provides traits info about every (mainstream) aspect of any function within the limitations of what C++ currently supports (or easily supports), and allows you to change every trait as well, where applicable (including function parameter types, which will be rarely used by most but no other implementation I've seen at this writing supports it, including the one from Boost).

<a name="CppFunctionTypes"></a>
## C++ function types

> [!NOTE]
> *This section provides a short tutorial on C++ function types, and may be skipped by those who are already comfortable with the subject. It may be informative to even experienced C++ developers however so is generally recommended reading. If you wish to bypass it however then you may optionally proceed to the [Function Type Templates](#functiontypetemplates) section for a brief summary of the library's main templates (for targeting each component of a function's type seen under "*C++ Function Type Format*" just below - see [Helper templates](#helpertemplates) for the library's complete list of templates), or directly to the [Usage](#usage) section if you wish to immediately get started.*

<a name="CppFunctionTypeFormat"></a>
#### C++ Function Type Format:

<pre><a href="#returntype">ReturnType</a> [<<a href="#callingconvention">CallingConvention</a>>] (<a href="#args">Args</a> [, <a href="#variadic">...</a>]) [<a href="#const">const</a>] [<a href="#volatile">volatile</a>] [<a href="#refqualifiers">&|&&</a>] [<a href="#noexcept">noexcept</a>[(true|false)]]</pre>

Note that function types in C++ are always of the general format seen just above, where items in square brackets are optional. This is what [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function) in the C++ standard itself looks for to determine if a type qualifies as a function (though the function's "*CallingConvention*" seen above isn't addressed by the C++ standard itself - it's a platform-specific extension in most C++ implementations, but still part of a function's type in every mainstream C++ compiler). Note that the above format is somewhat imprecise but sufficient for most purposes (the precise legalese is beyond the scope of this documentation). Each part of a function's type seen above will be informally be referred to as a "*function type component*" throughout this documentation. You may click each "*component*" above to see the available templates in this library for targeting that specific component, but consult the [Usage](#usage) section later on for full usage details (and the [Helper templates](#helpertemplates) section for the complete, fully-documented list of all templates in the library).

The format above is what actually qualifies as a function type in C++. In particular, pointers to functions, which most experienced developers are already very familiar with (including non-static member function pointers), references to functions though they are rarely used by most in practice (noting that references to *non-static* member functions aren't legal in C++), functors (classes with a function call operator "*operator()"* including lambdas which are just syntactic sugar for functors), and [std::function](https://en.cppreference.com/w/cpp/utility/functional/function) (just a wrapper for a function), are *not* actual functions in their own right. They therefore don't qualify as functions and [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function) will therefore return false for them. A pointer to a function for instance is still a pointer, not a function, and its type is therefore a pointer type in the C++ type system, not a function type (though *non-static* member function pointers don't always qualify as mainstream pointers in the language - [std::is\_pointer](https://en.cppreference.com/w/cpp/types/is_pointer) for instance will return false for them - you need to rely on [std::is\_member\_function\_pointer](https://en.cppreference.com/w/cpp/types/is_member_function_pointer) instead). Actual C++ functions have their own type in the C++ type system and that type is the format seen above (not pointer types or any of the others mentioned above).

Many developers may not know that a function has its own type however, or don't normally think about it this way. When you declare a C++ function such as this for instance:

```C++
// Free function
int SomeFunc(std::string_view, float) noexcept;
```

or the same function as a non-static member of some class:

```C++
class Test
{
public:
    // Non-static member function
    int SomeFunc(std::string_view, float) noexcept;
}
```

In effect you are actually declaring an "*object*" no different than any other, such as an "*int*", though functions are a different breed of of object, and the term "*object*" is even evaluated somewhat differently for functions in the language (though the details are beyond the scope of this documentation). For all intents and purposes it's still an "*object*" however, just not an object in the conventional sense (both in the language itself and what most think of as being an "*object*"). Given this for instance:

```C++
int SomeInt;
```

It obviously declares an "*object*" of type "*int*", so going back to first principles, an object (effectively a chunk of memory) will be allocated of the correct size for an "*int*" on the target platform (its size can vary on different platforms), and that memory will obviously be interpreted as an "*int*" according to the rules for what constitutes an "*int*" on your particular platform (such as a 4 byte blob of memory interpreted using the rules of "*two's complement*" integers if this is how an "*int*" is handled on your particular platform). That's what the term "*type*" means for all intents and purposes, how a chunk of memory is interpreted (what's actually stored there), and keywords such as "*int*" are just the names of the language's built-in types (and classes are just user-defined types of course). Names like "*SomeInt*" above are how we refer to that blob in our code but the blob itself is interpreted as an "*int*" (its "*type*"). That's what most think of when they refer to an "*object*", a data object like an "*int*".

The same applies to functions as well however though programmers don't usually think of it that way (compared to data types like an "*int*"). In effect however, when you write a function you're allocating a chunk of memory to store the function no different than an "*int*", except what's stored at that location is no longer interpreted as an "*int*", but as a "*function*" instead (in simplest terms, program code is now stored at that location instead of a data object like an integer).

A function therefore has its own type in C++, just like an integer ("*int*") does, and that's what the ```int (std::string_view, float) noexcept``` refers to in the examples above, the function's C++ type (whether it's a free function or a non-static member function). The name "*SomeFunc*" is just its name (not its type), just like "*SomeInt*" is the name of an "*int*", not its type. This becomes a lot more clear if the function is declared as follows instead, using the more conventional syntax for data types. First let's declare a function alias:

```C++
using FuncType = int (std::string_view, float) noexcept;
```

Now the same function shown previously can be declared as an "*object*" of "*FuncType*" instead of using the usual syntax for functions. It appears as an object that is, since we're now using the traditional syntax for declaring an "*object*":

```C++
// Free function
FuncType SomeFunc;
```

An of course we can also declare the same function as a non-static member of some class:

```C++
class Test
{
public:
    // Non-static member function
    FuncType SomeFunc;
}
```

Though the language does still force you to *define* "*SomeFunc*" in the usual C++ way (the language's syntax doesn't allow you to rely on "*FuncType*" to *define* the function, only *declare* it), nevertheless, the declaration of "*SomeFunc*" is declared as a "*FuncType*" object, where "*FuncType*" is the function's C++ type. It must therefore conform to the format of a C++ function type (see [here](#cppfunctiontypeformat)). The name "*SomeFunc*" itself is just the name for that function's blob in memory (how we refer to it in code), but the type of the blob itself is interpreted as a "*FuncType*" (an ```int (std::string_view, float) noexcept```object), the same way that "*SomeInt*" is the name of its own blob in memory except that blob is interpreted as an "*int*" type (an integer object). In both cases, other than the language's syntax for declaring a function vs an integer, each is just declaring a named blob and the blob's type. The integer's type is "*int*" and the function's type is "*FuncType*" (an ```int (std::string_view, float) noexcept```). Whether we formally categorize an "*int*" as an actual *object* and a function as something else (not really an "*object*" in the minds of many) is immaterial. The point is they both have a "*type*" in C++.

The syntax for a C++ function type is therefore just the [format](#cppfunctiontypeformat) shown earlier, whether we declare a function in the usual way or rely on an alias for it like "*FuncType*" (usually not but as demonstrated we can). Pointers to such types are therefore *not* function types, but pointer types as noted earlier (their own blob of memory is interpreted as a pointer, i.e., the address of a function, not the actual function itself). It's no different than a pointer to an "*int* which is also interpreted as a pointer type, i.e., the address of an "*int*", not the actual "*int*" itself (in spite of language rules that are sometimes applied a bit differently to functions vs other types). Once you remove the pointer then you're dealing with the actual type itself (of whatever that pointer points to), and in the case of functions, that type is the [format](#cppfunctiontypeformat) seen earlier (which is clearly not a pointer). The same applies to other types that deal with functions, such as functors. They are classes in their own right, not actual functions, and therefore have their own type (the functor's class). That type is clearly not the format of a C++ function type (again, see [here](#cppfunctiontypeformat)), even though the "*operator()*" member of a functor's class *is* a C++ function (and therefore has a type that conforms to the [format](#cppfunctiontypeformat) of a C++ function). The functor itself is just an ordinary class however and its type is therefore not a function type.

The upshot is that in C++, functions have a type like any other C++ "*object*" ("*int*", etc.), and that type conforms to the [C++ function type format](#cppfunctiontypeformat). They are not pointer types (```int (*)(std::string_view, float)``` and ```int (Test::*)(std::string_view, float)``` are both pointers to functions, free and non-static functions respectively, but not the actual function itself), they are not functor types including lambdas which are just functors written on-the-fly (a class that contains "*operator()*" is not a function, it's a class whose "*operator()*" member is a function but the class itself isn't), and they are not wrappers for functions like [std::function](https://en.cppreference.com/w/cpp/utility/functional/function) (which is just a template for a class that stores a function - its type is the class itself which is not a function type even though the class wraps a function).

However, this leads to the following issue. If you add the cv-qualifiers "*const*" and/or "*volatile*" to a *non-static* member function (usually "*const*" only in practice), or the ref-qualifiers "*&*" or "*&&*", though these are rarely used by most (many have likely never heard of them being applied to functions before), then because these qualifiers aren't allowed on *free* functions in C++ (they apply to *non-static* member functions only), the function's type seems to become problematic. For instance, if we add "*const*" to the "*Test::SomeFunc*" function above we wind up with this:

```C++
class Test
{
public:
    int SomeFunc(std::string_view, float) const noexcept;
}
```

What's this function's type? It's not the following because as previously described, the following is a pointer to the function, so it's a pointer type, not the function's actual C++ type (even though the following is what programmers sometimes work with):

```C++
int (Test::*)(std::string_view, float) const noexcept;
```

Its actual function type however is simply the function after the pointer's been removed:

```C++
int (std::string_view, float) const noexcept;
```

This now looks like an ordinary "*free*" function however, because the function's raw type in C++ has no concept of its class anymore (class "*Test*" itself is nowhere to be found in the above type). How can it therefore be a "*free*" function when it originated from a *non-static* member function and moreover, its type has "*const*" in it. Free functions aren't allowed to be "*const*", only non-static member functions are so the above type seems to be illegal.

It's not illegal however because this is not a "*free*" function, but just the actual C++ function type of a *non-static* member function. In effect, function types in C++ aren't associated with any class. The above syntax is just the pure (raw) C++ function type itself. There is therefore no class involved in a function's actual C++ type. If the function's type contains the qualifiers "*const*", "*volatile*", "*&*" or "*&&*", then they are not "*free*" functions. They are informally known as [abominable functions](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) instead (coined by the author of the latter link but not official C++ terminology). In fact, given an apparent "*free*" function type like so, with the "*const*" now removed again:

```C++
int (std::string_view, float) noexcept;
```

From a purely logical perspective, there's no way to know if it's an actual "*free*" function because it could actually be a *non-static* member function that simply doesn't include any of the qualifiers "*const*", "*volatile*", "*&*" or "*&&*". While it's normally interpreted by developers as a "*free*" function because it looks like one, it's only in the context of actually being declared as one does it become possible to know (by creating an actual "*free*" function with that type). Before that it's just a type only, neither *free* nor *non-static*, but just a neutral function type, which has no concept of "*free*" or "*non-static*". In the language itself its classification as "*free*" or "*non-static*" only becomes known when actually declared as one or the other. Nevertheless, as described later in this document (see template [Template arg "F"](#templateargF)), the "*FunctionTraits*" library itself does in fact treat such types as "*free*" functions. With one caveat discussed momentarily, its [IsFreeFunction\_v](#isfreefunction_v) template always returns true for native C++ function types. *Non-static* member functions are handled exclusively by pointers to non-static member functions only, never native C++ function types which are always assumed to be "*free*" functions (noting that functors including lambdas also qualify as *non-static* member functions by the library, in addition to qualifying as functors, because in the former case they're effectively just referring to "*operator()*" which of course is a *non-static* member function).

There's an important caveat for "*free*" functions however, as noted above. If a C++ function type contains the qualifiers "*const*", "*volatile*", "*&*" or "*&&*", i.e., it's an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) type (mentioned earlier), then a "*free*" function clearly can't be declared with that type, only a *non-static* member function can. It's therefore not considered to be a "*free*" function, and the library's [IsFreeFunction\_v](#isfreefunction_v) template will therefore return false for such functions. Its [IsAbominableFunction\_v](#isabominablefunction_v) template will return true instead.

The above discussion is intended to emphasize that functions have their own type in the C++ type system which is really no different than any other type in the language (notwithstanding differences in how functions are treated vs other types of "objects"), and this type is distinct from pointers to function types (pointer types), references to function types (reference types), functors (class types) and [std::function](https://en.cppreference.com/w/cpp/utility/functional/function) (also a class type). Now that you've been introduced to C++ function types (for those who may not have already fully been exposed to them before), then based on the format (syntax) of a [C++ function type](#cppfunctiontypeformat), you can obtain and/or change every component of a function's type by simply passing your function's type to the templates in this library (and other useful features also exist, like the ability to [determine if a member function exists](#determiningifamemberfunctionexists)). When doing so, the library also supports pointers and references to functions (their types), as well as *non-overloaded* functor types (including lambdas), so you're free to pass your function to the library's templates using any convenient syntax you wish (you're not restricted to the actual [C++ function type](#cppfunctiontypeformat) itself - see [Template arg "F"](#templateargf) for details). You simply need to bear in mind that the types you pass to the library, whether actual function types or pointers to functions, or references to functions, or functors, always refer to a function whose whose actual C++ type itself assumes the [format](#cppfunctiontypeformat) shown earlier, once pointers and references are removed if any (yielding the actual function type), or in the case of functors including lambdas, the function type of member "*operator()*" (for lambdas, the compiler just wraps your lambda's function in a functor behind the scenes whose "*operator*" member matches your lambda's function).

The term "*raw*" will therefore be applied throughout this documentation to refer to the native C++ function type [format](#cppfunctiontypeformat) shown earlier (to distinguish them from function pointers, function references, functors, etc.), and you can obtain the "*raw*" type of any function if required using templates [FunctionRawType\_t](#functionrawtype_t) and its string equivalent [FunctionRawTypeName\_v](#functionrawtypename_v) (whether you pass function pointers, function references or functors to these templates, including lambda types, or simply the "*raw*" type itself). The templates in the following section are then available to target each component comprising a [C++ function type](#cppfunctiontypeformat) as described in the section below (regardless of whether you pass the raw C++ function type itself or a pointer to a function type, reference to a function type, or functor type including lambdas).

<a name="FunctionTypeTemplates"></a>
## Function Type Templates

The following provides a brief summary of the library's main templates corresponding to each component of a C++ function type, in left-to-right order as they appear in a C++ function type (see [here](#cppfunctiontypeformat)). Note that several other useful templates also exist in the library, but those below specifically target the actual components comprising a C++ function type (again, see [here](#cppfunctiontypeformat)). They comprise the bulk of the library's templates. For the library's complete (fully-documented) list see [Helper Templates](#helpertemplates). See [Usage](#usage) section to get started.

1. <a name="ReturnType"></a>***ReturnType***

    1. [ReturnType\_t](#returntype_t) - Type alias for a function's return type
    2. [ReturnTypeName\_v](#returntypename_v) - Same as template just above but returns this as a WYSIWYG string (so a float return type for instance will literally be returned as "float")
    3. [IsReturnTypeSame\_v](#isreturntypesame_v) - Compares the return type in a function with the type you pass (returning "*true*" if equal or "*false*" otherwise)
    4. [IsVoidReturnType\_v](#isvoidreturntype_v) - Returns "*true*" if a function's return type is "*void*" or "*false*" otherwise
    5. [ReturnTypesMatch\_v](#returntypesmatch_v) - Compares the return types of the two function types you pass, returning "*true*" if identical or "*false*" otherwise.
    6. [ReplaceReturnType\_t](#replacereturntype_t) - Type alias for a function after replacing its return type with the type you pass

2. <a name="CallingConvention"></a>***CallingConvention***

    Note that as briefly described earlier, all other function traits libraries (at the time of this writing) can detect functions with the default calling convention only (usually "*cdecl*" or possibly "*thiscall*" for non-static member functions depending on the platform). Only "*callable\_traits*" from Boost was partially designed to handle calling conventions but for the Microsoft C++ compiler only, though again, it's officially unavailable (the documentation makes this clear and doesn't even describe how to turn it on), its support is limited even if it is turned on (by intrepid developers who wish to inspect the code - it won't handle certain syntax, it won't compile for 64 bit builds, etc.), and its use is discouraged by the author's own internal comments (see [Why choose this library](#whychoosethislibrary)). Note that calling conventions aren't addressed in the C++ standard itself and are platform specific. As a result, there are numerous calling conventions in the wild that are specific to one platform only and it's therefore inherently difficult for any function traits library to support them all. For now "*FunctionTraits*" currently supports the most mainstream calling conventions which will usually meet the needs of most users ("*cdecl*", "*stdcall*", "*fastcall*", "*vectorcall*", though GCC itself doesn't support "*vectorcall*", "*thiscall*", and on Clang and Intel platforms only, "*regcall*"). The following templates are available for working with calling conventions:

    1. [CallingConvention\_v](#callingconvention_v) - Variable of enumerator type "*CallingConvention*" (declared in "*FunctionTraits.h*" - see [Usage](#usage) section). Returns the calling convention for any function ("*Cdecl*", "*Stdcall*", etc.)
    2. [CallingConventionName\_v](#callingconventionname_v) - Same as template just above but returns this as a WYSIWYG string ("*cdecl*", "*stdcall*", etc. - always in lowercase)
    3. [DefaultCallingConvention\_v](#defaultCallingConvention_v) - Variable of enumerator type "*CallingConvention*" (declared in "*FunctionTraits.h*" - see [Usage](#usage) section). Returns the calling convention assigned to a function by the compiler if none is explicitly specified in the function's type.
    4. [DefaultCallingConventionName\_v](#DefaultCallingConventionName_v) - Same as template just above but returns this as a WYSIWYG string ("*cdecl*", "*stdcall*", etc. - always in lowercase)
    5. [ReplaceCallingConvention\_t](#replacecallingconvention_t) - Type alias for a function after replacing its calling convention with the calling convention you pass

3. <a name="Args"></a>***Args***

    1. [ArgType\_t](#argtype_t) - Type of the (zero-based) "*I*th" argument in a function's argument list
    2. [ArgTypeName\_v](#argtypename_v) - Same as template just above but returns this as a WYSIWYG string (so a float arg for instance will literally be returned as "float")
    3. [IsArgTypeSame\_v](#isargtypesame_v) - Compares the type of the (zero-based) "*I*th" argument in any function with the type you pass (returning "*true*" if equal or "*false*" otherwise)
    4. [ReplaceNthArg\_t](#replacentharg_t) - Type alias of any function after replacing its (zero-based) "*I*th" argument type with the type you pass (effectively replacing the existing type)
    5. [ArgCount\_v](#argcount_v) - Variable template returning the number of (non-variadic) arguments in a function (formally called "*arity*"), but to determine if the argument list is empty, don't check this for zero (0). Use [IsEmptyArgList\_v](#isemptyarglist_v) instead (see this for details).
    6. [ArgTypes\_t](#argtypes_t) - Returns a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) containing all  argument types in a function (so effectively the function's argument list)
    7. [AreArgTypesSame\_v](#areargtypessame_v) - Compares all arguments in any function with the types (parameter pack) you pass (returning "*true*" if equal or "*false*" otherwise)
    8. [AreArgTypesSameTuple\_v](#areargtypessametuple_v) - Identical to template just above but allows you to pass the types to compare in a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) instead of using a parameter pack
    9. [ArgTypesMatch\_v](#argtypesmatch_v) - Compares the arguments types of the two function types you pass, returning "*true*" if they're all identical or "*false*" otherwise.
    10. [ArgTypeMatches\_v](#argtypematches_v) - Compares the type of the (zero-based) "*I*th" argument in one function with the corresponding argument in another, returning "*true*" if equal or "*false*" otherwise
    11. [ReplaceArgs\_t](#replaceargs_t) - Type alias for a function after replacing all its *non-variadic* argument types with the argument types given by the parameter pack you pass (which become the new argument list)
    12. [ReplaceArgsTuple\_t](#replaceargstuple_t) - Identical to template just above but allows you to pass the types to replace using a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) instead of a parameter pack
    13. [ForEachArg](#foreacharg) - Iterates all function arguments and allows you to invoke a callback template functor for each (effectively allowing you to process each argument's type)
    14. [IsEmptyArgList\_v](#isemptyarglist_v) - Checks any function for an empty argument list, returning "*true*" if so or "*false*" otherwise

4. <a name="Variadic"></a>***Variadic args (...)***

    1. [IsVariadic\_v](#isvariadic_v) - "*bool*" variable set to "*true*" if a function has variadic args, i.e., its last arg is "...", or "*false*" otherwise.
    2. [AddVariadicArgs\_t](#addvariadicargs_t) - Type alias for a function after adding "..." to the function, hence converting it to a variadic function
    3. [RemoveVariadicArgs\_t](#removevariadicargs_t) - Type alias for a function after removing "..." from the function if present, hence reverting its type to a *non-variadic* function

5. <a name="Const"></a>***const***

    1. [IsFunctionConst\_v](#isfunctionconst_v) - "*bool*" variable set to "*true*" if a function is declared with the "*const*" cv-qualifier or "*false*" otherwise
    2. [FunctionAddConst\_t](#functionaddconst_t) - Type alias for a function after adding the "*const*" cv-qualifier to the function if not already present (also see [FunctionAddCV\_t](#functionaddcv_t) to add both cv-qualifiers)
    3. [FunctionRemoveConst\_t](#functionremoveconst_t) - Type alias for a function after removing the "*const*" cv-qualifier from the function if present (also see [FunctionRemoveCV\_t](#functionremovecv_t) to remove both cv-qualifiers)

6. <a name="Volatile"></a>***volatile***

    1. [IsFunctionVolatile\_v](#isfunctionvolatile_v) - "*bool*" variable set to "*true*" if a function is declared with the "*volatile*" cv-qualifier or "*false*" otherwise
    2. [FunctionAddVolatile\_t](#functionaddvolatile_t) - Type alias for a function after adding the "*volatile*" cv-qualifier to the function if not already present (also see [FunctionAddCV\_t](#functionaddcv_t) to add both cv-qualifiers)
    3. [FunctionRemoveVolatile\_t](#functionremovevolatile_t) - Type alias for a function after removing the "*volatile*" cv-qualifier from the function if present (also see [FunctionRemoveCV\_t](#functionremovecv_t) to remove both cv-qualifiers)

7. <a name="RefQualifiers"></a>***&|&&***

    1. [FunctionRefQualifier\_v](#functionrefqualifier_v) - Variable of enumerator type "*RefQualifier*" (declared in "FunctionTraits.h" - see Usage section). Returns a function's reference qualifier if any (&, && or none).
    2. [FunctionRefQualifierName\_v](#functionrefqualifiername_v) - Same as template just above but returns this as a WYSIWYG string
    3. [FunctionAddLValueReference\_t](#functionaddlvaluereference_t) - Type alias for a function after adding the "*&*" ref-qualifier to the function if not already present
    4. [FunctionAddRValueReference\_t](#functionaddrvaluereference_t) - Type alias for a function after adding the "*&&*" ref-qualifier to the function if not already present
    5. [FunctionRemoveReference\_t](#functionremovereference_t) - Type alias for a function after removing the "&" or "&&" ref-qualifier from the function if present

8. <a name="Noexcept"></a>***noexcept***

    1. [IsNoExcept\_v](#isnoexcept_v) - "*bool*" variable set to "*true*" if a function is declared as "*noexcept*" or "*false*" otherwise
    2. [AddNoexcept\_t](#addnoexcept_t) - Type alias for a function after adding the "*noexcept*" specifier to the function if not already present
    3. [RemoveNoexcept\_t](#removenoexcept_t) - Type alias for a function after removing the "*noexcept*" specifier from the function if not already present

9. <a name="FunctionClassification"></a>***Function's classification***

    1. [IsFreeFunction\_v](#isfreefunction_v) - "*bool*" variable set to "*true*" if a function is a free function (which includes static member functions) or "*false*" otherwise. Note that "*free*" functions are simply non-member or *static* member functions including pointers and/or references to such functions (for the purposes of this library), but when stripped of their pointer and/or reference (if any), i.e., when dealing with such functions in their "*raw*" (native C++) format described earlier, can't contain the "*const*" and/or "*volatile*" [cv-qualifiers]( https://en.cppreference.com/w/cpp/language/member_functions#Member_functions_with_cv-qualifiers), nor the "*&*" or "*&&*" [ref-qualifiers](https://en.cppreference.com/w/cpp/language/member_functions#Member_functions_with_ref-qualifier) (i.e., their "*raw*" type isn't an [abominable functions](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) described earlier).
    2. [IsMemberFunction\_v](#ismemberfunction_v) - "*bool*" variable set to "*true*" if the function type you pass is a *non-static* member function pointer or "*false*" otherwise (noting that this also returns true for functors in addition to [IsFunctor_v](#isfunctor_v) just below)
    3. [IsFunctor\_v](#isfunctor_v) - "*bool*" variable set to "*true*" if the type you pass is a functor (a class with a *non-overloaded*, *non-static* "*operator()*" member),  or *false*" otherwise. Note that [IsMemberFunction\_v](#ismemberfunction_v) just above also returns true for functors.
    4. [IsAbominableFunction\_v](#isabominablefunction_v) - "*bool*" variable set to "*true*" if the function type you pass is an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (described earlier), or "*false*" otherwise.

10. ***Class type***

    While not part of a C++ function type itself (see format [here](#cppfunctiontypeformat)), the class associated with a *non-static* member function including functors:
    1. [MemberFunctionClass\_t](#memberfunctionclass_t) - Class type for any *non-static* member function pointer or functor (always "*void*" otherwise)
    2. [MemberFunctionClassName\_v](#memberfunctionclassname_v) - Same as template just above but returns this as a WYSIWYG string
    3. [FunctionReplaceClass\_t](#functionreplaceclass_t) - Type alias for a function after replacing its class with the one you pass (usually targeting non-static member functions, though passing "*void*" for the class will convert it to a free function - if targeting a free function, will convert it to a non-static member function pointer for the class you pass)

The above covers almost all function traits most programmers will ever be interested in. Other possible traits such as whether a *non-static* member function is "*virtual*", and/or the "*override*" keyword is in effect, etc., are not available in this release, normally due to limitations in the language itself (either impossible or difficult/unwieldy to implement in current releases of C++). They may be available in a future version however if C++ better supports it (C++ reflection is scheduled in C++26 so will be revisited at that time).
<a name="Usage"></a>
## Usage (C++17 and later: GCC[^1], Microsoft[^2], Clang[^3] and Intel[^4] compilers only)
To use "*FunctionTraits*", simply add both "*FunctionTraits.h*" and "*CompilerVersions.h*" to your code and then *#include "FunctionTraits.h"* wherever you require it (an experimental module version is also now available - see [Module support in C++20 or later](#moduleusage)). All code is declared in namespace "*StdExt*". Note that you need not explicitly *#include "CompilerVersions.h"* unless you wish to use it independently of "*FunctionTraits.h*", since "*FunctionTraits.h*" itself #includes it as a dependency ("*CompilerVersions.h*" simply declares various #defined constants used to identify the version of C++ you're using, and a few other compiler-related declarations - you're free to use these in your own code as well if you wish). The struct (template) "*FunctionTraits*" is then immediately available for use (see [Technique 1 of 2](#technique1of2) below), though you'll normally rely on its [Helper templates](#helpertemplates) instead (see [Technique 2 of 2](#technique2of2) below). Note that both files above have no platform-specific dependencies except when targeting Microsoft, where the native Microsoft header *<tchar.h>* is expected to be in the usual #include search path (and it normally will be on Microsoft platforms). Otherwise they rely on the C++ standard headers only which are therefore (also) expected to be in the usual search path on your platform.

<a name="TemplateArgF"></a>
### Template arg "F"
Note that template arg "*F*" is the first (and usually only) template arg of "*FunctionTraits*" and all its [Helper templates](#helpertemplates), and refers to the function's type which can be any of the following:

1. Free functions which also includes static member functions, in either case referring to the function's actual C++ type (which are always considered to be free functions - non-static member functions are always handled by 4 or 5 below). Note that [abominable function types](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (raw C++ function types with the qualifiers "*const*" and/or "*volatile*" and/or "*&*" or "*&&*" - see [C++ function types](#cppfunctiontypes) for details), are also supported but these are not considered "*free*" functions for the purposes of this library (since "*free*" functions by definition cannot have such qualifiers in C++ - such functions are still legal C++ types however and therefore supported by this library but are not considered to be "*free*" functions - again, see [C++ function types](#cppfunctiontypes) for details).
2. Pointers and references to free functions
3. References to pointers to free functions
4. Pointers to non-static member functions
5. References to pointers to non-static member functions (note that C++ doesn't support references to non-static member functions, only pointers)
6. Non-overloaded functors (i.e., functors or references to functors with a single "*operator()*" member only, otherwise which overload to target becomes ambiguous - if present then you'll need to target the specific overload you're interested in using 4 or 5 above instead). Includes lambdas as well (just syntactic sugar for creating functors on-the-fly). Simply apply "*decltype*" to your lambda to retrieve its compiler-generated class type (officially known as its "*closure type*"), which you can then pass to "*FunctionTraits*" or any of its [Helper templates](#helpertemplates). Please note however that *generic* lambdas are *not* supported by passing their closure type for technical reasons beyond the scope of this documentation (their class type can't be passed for "*F*"). They can still be accessed via 4 and 5 above however using some unconventional syntax described in "*FunctionTraits.h*" itself. Search the latter file for "_Example 3 (generic lambda)_" (quotes not included), which you'll find in the comments preceding the "*FunctionTraits*" specialization for functors (the comments for this example also provides additional technical details on this issue).

Note that the code also incorporates [concepts](https://en.cppreference.com/w/cpp/language/constraints) when targeting C++20 or later. This will trap invalid function types with cleaner error messages at compile-time (if anything other than the above is passed for "*F*"). In C++17 however (again, earlier versions aren't supported), if "*F"* is invalid then a compiler-specific error message indicating that "*FunctionTraits*" isn't defined will occur instead (note that a "*static\_assert*" was once employed to provide a more specific message in C++17 but was removed in later versions since "*static\_assert*" isn't [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae)-friendly - it doesn't respect [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae) contexts so will trigger instead of quietly failing - omitting it therefore allows the "*FunctionTraits*" templates to properly support [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae), and invalid values of "*F*" outside of [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae) are normally just programmer mistakes that rarely occur in practice).

Once you've #included "*FunctionTraits.h*", there are two ways to use the "*FunctionTraits*" class. You can either use the "*FunctionTraits*" class directly as described in the following section ("*Technique 1 of 2*" just below), but doing often results in verbose syntax due to C++ syntax rules, so most users should rely on the library's helper templates instead. These templates wrap the "*FunctionTraits*" class itself, cleanly eliminating the (C++) clutter of using it directly (see [Technique 2 Of 2](#technique2of2)). Most will normally always rely on them so feel free to jump there now if you wish (normally recommended).

<a name="Technique1Of2"></a>
## Technique 1 of 2 - Using "FunctionTraits" directly (not usually recommended)

```C++
// Only file you need to explicitly #include (see "Usage" section just above)
#include "FunctionTraits.h"

int main()
{
    // Everything declared in this namespace
    using namespace StdExt;

    ////////////////////////////////////////////////////////////////////
    // Free function whose traits you wish to retrieve (note that
    // static member functions are also considered "free" functions).
    // Pointers and references to free functions are also supported
    // (including references to such pointers), pointers to non-static
    // member functions (including references to such pointers, though
    // references to non-static member functions aren't supported in
    // C++ itself), and functors (including lambdas).
    ////////////////////////////////////////////////////////////////////
    float SomeFunc(const std::string &, double, int);

    // Type of the above function (but see alternate syntax just below)
    using F = decltype(SomeFunc);

    ///////////////////////////////////////////////
    // Same as above but using a function pointer
    // (function references also supported)
    ///////////////////////////////////////////////
    // using F = decltype(&SomeFunc);

    ///////////////////////////////////////////////////////////
    // Also works but using a reference to a function pointer
    // (though the following syntax is rare in practice but
    // more mainstream syntax may exist in your own code,
    // though references to pointers are usually rare outside
    // of function templates taking reference args to arbitrary
    // types, so if passed a function pointer then the function
    // template winds up taking a reference to a function
    // pointer)
    ///////////////////////////////////////////////////////////
    // using F = decltype(&SomeFunc) &;

    ///////////////////////////////////////////////////
    // And this works too (manually passing the type,
    // just to show we can)
    ///////////////////////////////////////////////////
    // using F = float (const std::string &, double, int);

    // Apply "FunctionTraits" to above function
    using SomeFuncTraits = FunctionTraits<F>;

    ////////////////////////////////////////////////////
    // Retrieve the function's return type (a "float").
    // Note that this (mandatory C++) syntax is ugly
    // however. A cleaner alias for this exists as
    // described in the next section.
    ////////////////////////////////////////////////////
    using SomeFuncReturnType_t = typename SomeFuncTraits::ReturnType;

    // Number of arguments (3)
    constexpr std::size_t SomeFuncArgCount = SomeFuncTraits::ArgCount;

    /////////////////////////////////////////////////////////
    // Retrieve type of the function's 3rd arg (an "int").
    // Arg's index is zero-based, so passing 2 here (to
    // target the 3rd arg). Note that this (mandatory C++)
    // syntax is ugly however. A much cleaner alias for this
    // exists as described in the next section.
    /////////////////////////////////////////////////////////
    using SomeFuncArg3Type_t = typename SomeFuncTraits::template Args<2>::Type;

    ////////////////////////////////////////////////////////////////
    // Type of the function's 3rd arg (same as above) but returned
    // as a "std::basic_string_view" (so literally returns "int",
    // quotes not included). Again, a cleaner template exists for
    // this as described in the next section. Note that the
    // following variable template, "TypeName_v", can be used to
    // get the (WYSIWYG) name for any type, not just types
    // associated with "FunctionTraits". See its entry in the
    // "Helper templates" section later.
    ////////////////////////////////////////////////////////////////
    constexpr auto SomeFuncArg3TypeName = TypeName_v<SomeFuncArg3Type_t>;

    /////////////////////////////////////////////////////////////////
    // Though few will rarely need to update a function trait (most
    // need to read them only), you can also modify them as well
    // (every available trait is supported). The following adds
    // "noexcept" to the function (see its string representation
    // just below).
    /////////////////////////////////////////////////////////////////
    using SomeFuncWithNoexceptAdded_t = typename SomeFuncTraits::AddNoexcept;

    ////////////////////////////////////////////////////////////////////////
    // Above type as a string so returns the following (same as "SomeFunc"
    // but with "noexcept" added - note the format of the following may
    // change slightly depending on the target compiler).
    //
    //     float (const std::basic_string<char>&, double, int) noexcept
    ////////////////////////////////////////////////////////////////////////
    constexpr auto SomeFuncWithNoexceptAdded_v = TypeName_v<SomeFuncWithNoexceptAdded_t>;

    /////////////////////////////////////////////////////////
    // And yet a different change, replacing the function's
    // 3rd arg (an "int") with a "char", passing 2 here as
    // the zero-based index of the arg we're targeting (see
    // its string representation just below).
    /////////////////////////////////////////////////////////
    using SomeFuncReplace3rdArgWithChar_t = typename SomeFuncTraits::template ReplaceNthArg<2, char>;

    ////////////////////////////////////////////////////////////
    // Above type as a string so returns the following (same
    // as function "SomeFunc" but the 3rd arg has now been
    // changed to a "char" - note the format of the following
    // may change slightly depending on the target compiler).
    //
    //    float (const std::basic_string<char>&, double, char)
    ////////////////////////////////////////////////////////////
    constexpr auto SomeFuncReplace3rdArgWithChar_v = TypeName_v<SomeFuncReplace3rdArgWithChar_t>;

    // Etc. (see "Helper templates" further below for the complete list)

    return 0;
}
```

<a name="Technique2Of2"></a>
## Technique 2 of 2 - Using the helper templates instead of "FunctionTraits" directly (recommended)

Alternatively, instead of using the "*FunctionTraits*" class directly ([Technique 1 of 2](#technique1of2) above), you can rely on the second technique just below instead, which is normally much cleaner (and you should normally use it). As seen in the first technique above, relying on "*FunctionTraits*" directly can result in verbose syntax. For instance, due to the syntax of C++ itself, accessing the type of a given function arg using the "*Args*" member (template) of "*FunctionTraits*" is ugly because you have to apply both the "*typename*" and "*template*" keywords, as well as access the "*Type*" member (alias) of "*Args*" itself (see "*SomeFuncArg3Type\_t*" alias in the [Technique 1 of 2](#technique1of2) example). A helper template to eliminate all clutter therefore exists not only for the latter example ([ArgType\_t](#argtype_t)), but for every member of "*FunctionTraits*". Therefore, instead of relying on "*FunctionTraits*" directly as seen in the [Technique 1 of 2](#technique1of2) examples, you can rely on the [Helper templates](#helpertemplates) instead. They're easier and cleaner, making the job of extracting or modifying a function's traits a breeze.

Note that two sets of helper templates exist, one where each template takes a "*FunctionTraits*" template arg, and a corresponding set taking a function type template arg instead (just pass your function's type "*F*" as described [here](#templateargf)). The latter (second) set of helper templates simply create a "*FunctionTraits*" for you from the function type you pass (by passing "*F*" itself as the template arg to "*FunctionTraits*"), and then defer to the corresponding "*FunctionTraits*" helper template in the first set. The "*FunctionTraits*" helper templates (the first set) are rarely used directly in practice however so they're not documented in this README file. Only the [Helper templates](#helpertemplates) taking a function template arg "*F*" (the second set) are therefore documented below. The two sets are functionally identical however other than their respective template args ("*FunctionTraits*" and "*F*"), and template names (the first set is normally named the same as the second but just adds the prefix "*FunctionTraits*" to the name), so the documentation below effectively applies to both sets. Most users will exclusively rely on the set taking template arg "*F*" however (so all remaining documentation below specifically targets it), though you can always quickly inspect the code itself if you ever want to use the "*FunctionTraits*" helper templates directly (since the second set taking template arg "*F*" just defers to them as described). Note that the "*FunctionTraits*" helper templates are just thin wrappers around the members of struct "*FunctionTraits*" itself, which inherits all its members from its base classes. Users normally never need to access these classes directly however. The helper templates documented in this file are all you normally ever need to rely on.

Here's the same example seen in [Technique 1 of 2](#technique1of2) above but using these easier [Helper templates](#helpertemplates) instead (the second set taking template arg "*F*"):

```C++
// Only file you need to explicitly #include (see "Usage" section earlier)
#include "FunctionTraits.h"

int main()
{
    // Everything declared in this namespace
    using namespace StdExt;

    ////////////////////////////////////////////////////////////////////
    // Free function whose traits you wish to retrieve (note that
    // static member functions are also considered "free" functions).
    // Pointers and references to free functions are also supported
    // (including references to such pointers), pointers to non-static
    // member functions (including references to such pointers, though
    // references to non-static member functions aren't supported in
    // C++ itself), and functors (including lambdas).
    ////////////////////////////////////////////////////////////////////
    float SomeFunc(const std::string &, double, int);

    // Type of the above function (but see alternate syntax just below)
    using F = decltype(SomeFunc);

    ///////////////////////////////////////////////
    // Same as above but using a function pointer
    // (function references also supported)
    ///////////////////////////////////////////////
    // using F = decltype(&SomeFunc);

    ///////////////////////////////////////////////////////////
    // Also works but using a reference to a function pointer
    // (though the following syntax is rare in practice but
    // more mainstream syntax may exist in your own code,
    // though references to pointers are usually rare outside
    // of function templates taking reference args to arbitrary
    // types, so if passed a function pointer then the function
    // template winds up taking a reference to a function
    // pointer)
    ///////////////////////////////////////////////////////////
    // using F = decltype(&SomeFunc) &;

    ///////////////////////////////////////////////////
    // And this works too (manually passing the type,
    // just to show we can)
    ///////////////////////////////////////////////////
    // using F = float (const std::string &, double, int);

    ////////////////////////////////////////////////////
    // Retrieve the function's return type (a "float")
    // but using the cleaner helper alias this time
    // (instead of "FunctionTraits" directly)
    ////////////////////////////////////////////////////
    using SomeFuncReturnType_t = ReturnType_t<F>;

    ///////////////////////////////////////////////////////
    // Number of arguments (3) but using the helper alias
    // this time (instead of "FunctionTraits" directly)
    ///////////////////////////////////////////////////////
    constexpr std::size_t SomeFuncArgCount = ArgCount_v<F>;

    ////////////////////////////////////////////////////////
    // Retrieve type of the function's 3rd arg (an "int").
    // Arg's index is zero-based, so passing 2 here (to
    // target the 3rd arg). Again, uses the cleaner helper
    // alias this time (instead of "FunctionTraits"
    // directly).
    ////////////////////////////////////////////////////////
    using SomeFuncArg3Type_t = ArgType_t<F, 2>;

    ////////////////////////////////////////////////////////////////
    // Type of the function's 3rd arg (same as above) but returned
    // as a "std::basic_string_view" (so literally returns "int",
    // quotes not included). Note that the following variable,
    // template "ArgTypeName_v", is just a thin wrapper around the
    // "TypeName_v" helper template, which can be used to get the
    // (WYSIWYG) name for any type, not just types associated with
    // "FunctionTraits". See its entry in the "Helper templates"
    // section later.
    ////////////////////////////////////////////////////////////////
    constexpr auto SomeFuncArg3TypeName = ArgTypeName_v<F, 2>;

    /////////////////////////////////////////////////////////////////
    // Though few will rarely need to update a function trait (most
    // need to read them only), you can also modify them as well
    // (every available trait is supported). The following adds
    // "noexcept" to the function (see its string representation
    // just below). Again, uses the easier helper template this
    // time instead of "FunctionTraits" directly.
    /////////////////////////////////////////////////////////////////
    using SomeFuncWithNoexceptAdded_t = AddNoexcept_t<F>;

    ////////////////////////////////////////////////////////////////////////
    // Above type as a string so returns the following (same as "SomeFunc"
    // but with "noexcept" added - note the format of the following may
    // change slightly depending on the target compiler).
    //
    //     float (const std::basic_string<char>&, double, int) noexcept
    ////////////////////////////////////////////////////////////////////////
    constexpr auto SomeFuncWithNoexceptAdded_v = TypeName_v<SomeFuncWithNoexceptAdded_t>;

    /////////////////////////////////////////////////////////
    // And yet a different change, replacing the function's
    // 3rd arg (an "int") with a "char", passing 2 here as
    // the zero-based index of the arg we're targeting (see
    // its string representation just below). Again, uses
    // the cleaner helper template this time instead of
    // "FunctionTraits" directly.
    /////////////////////////////////////////////////////////
    using SomeFuncReplace3rdArgWithChar_t = ReplaceNthArg_t<F, 2, char>;

    ////////////////////////////////////////////////////////////
    // Above type as a string so returns the following (same
    // as function "SomeFunc" but the 3rd arg has now been
    // changed to a "char" - note the format of the following
    // may change slightly depending on the target compiler).
    //
    //    float (const std::basic_string<char>&, double, char)
    ////////////////////////////////////////////////////////////
    constexpr auto SomeFuncReplace3rdArgWithChar_v = TypeName_v<SomeFuncReplace3rdArgWithChar_t>;

    // Etc. (see "Helper templates" further below for the complete list)

    return 0;
}
```
<a name="LoopingThroughAllFunctionArguments"></a>
## Looping through all function arguments

You can even loop through all arguments using the helper function template [ForEachArg](#foreacharg). The following example assumes C++20 or later for the lambda template seen below (lambda templates aren't available until C++20 or later), though if targeting C++17 you can easily replace it with your own functor instead (the "*operator()*" member in your functor needs to be a template however, with the same template args seen in the lambda below and the same code). See [ForEachArg](#foreacharg) for further details.

```C++
// Standard C++ header (to support "tcout" below)
#include <iostream>

////////////////////////////////////////////////////////
// Only file in this repository you need to explicitly
// #include (see "Usage" section earlier)
////////////////////////////////////////////////////////
#include "FunctionTraits.h"

int main()
{
    // Everything declared in this namespace
    using namespace StdExt;

    // Free function whose arg types you wish to iterate
    float SomeFunc(const std::string &, double, int);

    // Type of the above function
    using F = decltype(SomeFunc);

    //////////////////////////////////////////////////////////////
    // Lambda that will be invoked just below, once for each arg
    // in function "F" (where template arg "I" is the zero-based
    // "Ith" argument in function "F", and "ArgTypeT" is its type).
    // Note that lambda templates are supported in C++20 and later
    // only. For C++17 (earlier versions aren't supported), you
    // need to roll your own functor instead (with a template
    // version of "operator()" equivalent to this lambda)
    //////////////////////////////////////////////////////////////
    const auto displayArgType = []<std::size_t I, typename ArgTypeT>()
                                {
                                    //////////////////////////////////////////////////////////
                                    // Display the type of the (zero-based) "Ith" arg in
                                    // function "F" (where template arg "I" just above stores
                                    // this index though we don't use it for anything in this
                                    // example, and "ArgTypeT" is that arg's type). Note that
                                    // we never come through here if function "F" has no args
                                    // (or just variadic args only - variadic args aren't
                                    // processed). Also note that the following call to
                                    // "TypeName_v" simply returns "ArgTypeT" as a compile-time
                                    // string ("std::basic_string_view"). See "TypeName_v" in
                                    // the "Helper templates" section just below (consult its
                                    // entry there - it can be passed any C++ type).
                                    //
                                    // Finally, note that on non-Microsoft platforms, "tcout"
                                    // and "_T" (both automatically declared in namespace
                                    // "StdExt" when you #include "FunctionTraits.h"), always
                                    // resolve to "std::cout" and (for _T) simply the arg
                                    // you pass it (_T effectively does nothing). On
                                    // Microsoft platforms however they resolve to
                                    // "std::wcout" and L##x (for the given macro arg "x")
                                    // respectively when compiling for Unicode (normally the
                                    // case), or "std::cout" and (for _T) the arg you pass
                                    // it otherwise (_T effectively does nothing in the
                                    // latter case). For further details in general, see the
                                    // comments preceding the definitions of "_T", "tcout"
                                    // and "tchar" in "CompilerVersions.h" (for non-Microsoft
                                    // platforms but those targeting Microsoft may wish to
                                    // review this as well).
                                    //////////////////////////////////////////////////////////
                                    tcout << TypeName_v<ArgTypeT> << _T("\n");

                                    //////////////////////////////////////////////
                                    // Return true to continue iterating (false
                                    // would stop iterating, equivalent to a
                                    // "break" statement in a regular "for" loop)
                                    //////////////////////////////////////////////
                                    return true;
                                 };

    ////////////////////////////////////////////////
    // Iterate all argument types in function "F",
    // invoking "displayArgType" above for each.
    ////////////////////////////////////////////////
    ForEachArg<F>(displayArgType);

    return 0;
}
```
<a name="DeterminingIfAMemberFunctionExists"></a>
## Determining if a member function exists

> [!IMPORTANT]
> If you rely on the member function detection templates described in the following section, please see the following footnotes at the end of this document for issues that may affect you (though usually rare in practice):<br><br>***GCC and Clang users only:*** See footnote[^5]<br>***Microsoft users only:*** See footnote[^6]

Determining if a member function with a specific name and type (signature) exists in an arbitrary class is a very common requirement for many users. In practice most users can usually just rely on [std::is\_invocable](https://en.cppreference.com/w/cpp/types/is_invocable) and cousins and/or in C++20 and later, its (concept) cousin [std::invocable](https://en.cppreference.com/w/cpp/concepts/invocable). However, those templates simply determine if a function can be called with a given set of arguments, and the arguments you pass don't need to exactly match the parameter types of the function you're checking (nor does the return type have to exactly match when using the "*std::is\_invocable\_r*" variant). The templates will still return true that is so long as the arguments you pass can be implicitly converted to the function's actual parameter types (say, if the function you're checking takes an "*int*" and you pass a "*double*" to [std::is\_invocable](https://en.cppreference.com/w/cpp/types/is_invocable) - it will return true since a "*double*" can be implicitly converted to an "*int*"). The details of how to use the latter templates to detect if a member function exists are beyond the scope of this documentation, but the point is that it's not sufficient if you need to detect if a function exists with *exact* parameter types (again, because of the implicit argument conversion issue). Moreover, you may need to check other function traits as well, not just its arguments and/or return type (though such granular checks are supported for *non-overloaded* functions only in this release, due to C++ language limitations - more on this shortly). You may want to check a particular parameter type only for instance (not all of them), and/or the function's return type (for an *exact* type which "*std::is\_invocable\_r*" doesn't do), and/or whether the function is "*const*", and/or any of the other possible components that make up a function (see [C++ function types](#cppfunctiontypes)).

The "FunctionTraits" library provides a set of templates to do this that will usually meet the needs of most users in practice. Unfortunately, due to lack of support in the language itself (which will hopefully be rectified when reflection becomes available in C++26), the library is currently restrained in both what it can offer, and how it's able to implement the templates it does offer. The implementation is therefore a (tiny) bit more verbose than it could be, primarily because function names can't be passed as template args in C++, although the library still relies on a reasonably clean approach given the situation. Moreover, if the function you wish to detect is overloaded, the library can detect if it exists based on its exact type but is restrained from detecting if it exists based on its individual traits, since the language itself doesn't support it (since the only way to disambiguate a specific function among its overloads is to "*static\_cast*" it to the function's exact type so the entire type must match - granular checks of the function's individual traits therefore aren't possible). If a function is not overloaded however, it can be detected based on the precise traits you're interested in (since disambiguation using a "*static\_cast*" is no longer required).

Lastly, with the exception of the C++ function call operator "*operator()*" (see [ClassHasOperator\_FunctionCall](#classhasoperator_functioncall)), no templates exist out-of-the-box to detect C++ operators. If you need to do so then you'll have to roll your own technique for the particular operator(s) you're interested in, and in practice for the specific class[es] as well (that you wish to inspect for a given operator). The detection of operators isn't supported in the library itself (out-of-the-box) because the lack of reflection in the language makes it impossible to provide anything other than a very messy and half-baked solution (for reasons beyond the scope of this documentation). Fortunately, since most users normally (typically) need to check for *non-operator* functions only (based on the function's name), or the function call operator, it's unlikely most will be impacted by the situation (lack of support for the operator functions for now). Operator detection will likely be revisited in a future release however, once reflection becomes available in C++26 (where presumably a clean and full solution will then become possible). For now, the library provides templates to detect either non-operator functions only (again, based their name), or the function call operator.

> [!IMPORTANT]
> *Note that in all cases, whether overloaded or not, the normal C++ member "name hiding" rules remain in effect, so if a function exists in a derived class with the same name as a function in a base class, the base class functions are effectively hidden (they won't be detected when using the library's function detection templates on the derived class). This is often surprising to new C++ developers trying to call a member function in general. The C++ name lookup rules first search the class a function is being called from and if not found there, will then search up the base class hierarchy until it is found (the short story). Searching stops as soon as the first base class with that function is encountered so if a function with the same name exists in another base class even further up the class hierarchy, it's effectively ignored (hidden by the one that was first found in a derived class - hidden functions don't participate in overloaded resolution). The library's function detection templates are subject to the same C++ rules (enforced by the language itself).*

If you need to determine whether a function, say,```int Whatever(std::string_view, float) const noexcept``` exists in an arbitrary class (public functions only normally - again, Clang and GCC users should also consult footnote[^5] and Microsoft users footnote [^6]), and [std::is\_invocable](https://en.cppreference.com/w/cpp/types/is_invocable) and cousins aren't sufficient for your needs (again, if you need to check for *exact* parameter types or other traits, otherwise use the latter "*std*" templates instead), you'll rely on one of six possible macros in the library to do so (but if targeting the function call operator "*operator()*" see [ClassHasOperator\_FunctionCall](#classHasOperator_functionCall) and cousins - they require no macros though you'll still need to read the following section to acquire an understanding). While relying on macros may seem a dirty technique to experienced developers (for your own named functions, not "*operator()*"), and six of them to choose from no less, the language itself currently leaves no other choice but the situation isn't as bad as it first seems.

First, each macro merely declares a C++ (struct) template targeting the function you're interested in by name (a [std::bool_constant](https://en.cppreference.com/w/cpp/types/integral_constant#Helper_alias_templates) derivative that resolves to "*std::true\_type*" if the function exists or "*std::false\_type*" otherwise). A *bool* helper variable is also declared with the usual "*\_v*" suffix that just defers to it. C++ provides no other way to do it besides using macros at this writing (read on), which do nothing more than provide the necessary boilerplate code to create the templates. You'll therefore be relying on the declared templates themselves to do the actual work, not on the macros themselves (which just declare the templates as noted). Again, no other way to do it currently exists. Until C++ supports reflection, again, which will be available in C++26 (and which will hopefully provide improved techniques), a function name can't be passed as a template argument in C++, so dedicated templates are required to target each particular function name you're interested in. The six macros simply declare slightly different flavors of these templates targeting the particular function by name, such as "*Whatever*" in the earlier example, so the macros simply declare these templates with the hardcoded name. You'll normally just apply one of these macros for your particular needs however (the flavor you require), not all six of them, so the overhead isn't a significant issue.

Second, the six macros are really comprised of just two complementary sets of three macros each, where the first set targets (detects) *non-static* member functions and the second set targets (detects) *static* member functions. The two sets are effectively identical otherwise. There are therefore really just three macros to choose from, each serving slightly different needs, so you just need to choose the one you require (from the three), and then select either the non-static or static version based on the function you wish to detect. The usage of the non-static and static macros is effectively the same otherwise. These macros are briefly described in the table just below with full examples in the last column (note that the table is somewhat lengthy but can be horizontally scrolled in a desktop browser via ```<Shift+MouseWheel>``` or less conveniently, the horizontal scrollbar beneath the table). For now the table provides a summary only but provides enough detail that most users can start using these macros and the templates they create after reading the table (again, with full examples in the last column). Consult the macros themselves documented later in this README.md file for complete details (see the links to each of these macros repeated in each cell of the table). The macro documentation also includes some additional (useful) techniques in some cases (not shown in the table below), so it's recommended you read it before proceeding.

### Member function detection templates (comparative summary table)

*Using a desktop browser? Click table and use ```<Shift+MouseWheel>``` to scroll horizontally (easier than scrollbar at bottom of table).*

<table>
  <thead>
    <tr span style="font-size: x-large">
      <th><i>Macro</i></th>
      <th><i>Templates created by macro<br>(implementation details omitted or obscured)</i></th>
      <!--Trailing spaces required in following to prevent
          column from being too narrow prior to user clicking
          our example <details> element to open them (creating
          an ergonomically bad experience). Dirty technique but
          couldn't find any work-around for GitHub ".md" files.
          Using arbitrary number of spaces here (workable for
          our needs - pinning down an accurate number a pain). -->
      <th style="text-align:left;"><i>Example</i>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
    </tr>
  </thead>
  <tbody>
    <tr valign="top">
      <td><b><a href="#declare_class_has_function">DECLARE_CLASS_HAS_FUNCTION(NAME)</a></b><br><br>Use when you need to check a class for an <i>exact</i> type match of a <i>non-static</i> (public) member function given by the <i>NAME</i> (macro) arg (among all overloads of NAME if any). Just pass the target class as the 1st arg to the templates created by this macro (see next column), and the <a href="#cppfunctiontypes">function type</a> itself as the 2nd arg (using any function type supported by this library - see <a href="#templateargf">Template arg &quot;F&quot;</a>). The function you're targeting (<i>NAME</i>) must exist in the class (whether overloaded or not), and have the <i>exact</i> <a href="#cppfunctiontypes">function type</a> you pass for a match to occur (i.e., for the templates to return true). See adjacent columns for details.</td>
      <td><b><a href="#declare_class_has_function">DECLARE_CLASS_HAS_FUNCTION(NAME)</a> </b>declares:<pre><b>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>struct ClassHasFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>inline constexpr bool ClassHasFunction_##NAME##_v;</b></pre>First template above inherits from <a href="https://en.cppreference.com/w/cpp/types/integral_constant#Helper_alias_templates">std::true_type</a> or <a href="https://en.cppreference.com/w/cpp/types/integral_constant#Helper_alias_templates">std::false_type</a> based on whether the <i>non-static</i> (public) member function given by the macro's NAME arg exists in the class passed as the template's 1st arg (&quot;<i>T</i>&quot;, which must conform to <a href="https://en.cppreference.com/w/cpp/types/is_class">std::is_class</a>), and exactly matches the <a href="#cppfunctiontypes">function type</a> given by the 2nd arg (&quot;<i>F</i>&quot;, which can be any type supported by this library, such as <code style="white-space: pre;">int (std::string_view, float) const noexcept</code> - see <a href="#templateargf">Template arg &quot;F&quot;</a>). The second template above is just a bool helper variable you'll usually rely on instead (that just defers to the first template). Note that the NAME macro arg is embedded in each template's name, so if you pass, say, &quot;<i>Whatever</i>&quot; to the macro, the templates will be called &quot;<i>ClassHasFunction_Whatever</i>&quot; and &quot;<i>ClassHasFunction_Whatever_v</i>&quot;. The function will be detected whether overloaded or not.</td>
      <td><b><a href="#declare_class_has_function">DECLARE_CLASS_HAS_FUNCTION(NAME)</a></b><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers (only req'd for this example)<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;FunctionTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>/////////////////////////////////////////////////////////////<br>// Class you wish to check for <i>non-static</i> (public) function<br>// &quot;<i>Whatever</i>&quot; with the exact type of the first overload<br>// below (&quot;<i>ClassHasFunction_Whatever_v</i>&quot; called further below<br>// will detect it regardless of overloads).<br>/////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    int Whatever(std::string_view, float) const noexcept;<br><br>    //////////////////////////////////////////////////////////////<br>    // Overload but above function will still be found. Template<br>    // &quot;<i>ClassHasFunction_Whatever_v</i>&quot; works even when &quot;<i>Whatever</i>&quot;<br>    // is overloaded.<br>    //////////////////////////////////////////////////////////////<br>    int Whatever(char) const noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasFunction_Whatever_v</i>&quot; called below<br>// which is set to true if the <i>non-static</i> (public) function<br>// &quot;<i>Whatever</i>&quot; exists in the class you pass it (via the 1st<br>// template arg), and exactly matches the <a href="#cppfunctiontypes">function type</a> you<br>// pass it (via the 2nd template arg). False returned otherwise.<br>/////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a <i>non-static</i> (public) function called &quot;<i>Whatever</i>&quot;<br>    // with the exact type being passed via the 2nd template<br>    // arg, &quot;<i>int (std::string_view, float) const noexcept</i>&quot; in<br>    // this case.<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasFunction_Whatever = ClassHasFunction_Whatever_v&lt;Test, int (std::string_view, float) const noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr valign="top">
      <td><b><a href="#declare_class_has_static_function">DECLARE_CLASS_HAS_STATIC_FUNCTION(NAME)</a></b><br><br>Identical to macro just above but detects a <i>static</i> function instead of non-static.</td>
      <td><b><a href="#declare_class_has_static_function">DECLARE_CLASS_HAS_STATIC_FUNCTION(NAME)</a> </b>declares:<pre><b>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>struct ClassHasStaticFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>inline constexpr bool ClassHasStaticFunction_##NAME##_v;</b></pre>Functionally equivalent to the templates created by the <i>DECLARE_CLASS_HAS_FUNCTION</i> macro on the first row just above but detects a <i>static</i> function instead of non-static (so the function that the &quot;<i>F</i>&quot; template arg targets must conform to a <a href="#functionclassification">free function</a> or false will always be returned). The templates will therefore return true only if the target function is <i>static</i>, but are otherwise identical to the templates on the row just above (see this for details). Note that the name of each template based on a NAME macro arg of, say, &quot;<i>Whatever</i>&quot;, will be &quot;<i>ClassHasStaticFunction_Whatever</i>&quot; and &quot;<i>ClassHasStaticFunction_Whatever_v</i>&quot;. The function will be detected whether overloaded or not.</td>
      <td><b><a href="#declare_class_has_static_function">DECLARE_CLASS_HAS_STATIC_FUNCTION(NAME)</a></b><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers (only req'd for this example)<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;FunctionTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>//////////////////////////////////////////////////////////<br>// Class you wish to check for <i>static</i> (public) function<br>// &quot;<i>Whatever</i>&quot; with the exact type of the first overload<br>// below (&quot;<i>ClassHasStaticFunction_Whatever_v</i>&quot; called<br>// further below will detect it regardless of overloads).<br>//////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    static int Whatever(std::string_view, float) noexcept;<br><br>    //////////////////////////////////////////////////////////////<br>    // Overload but above function will still be found. Template<br>    // &quot;<i>ClassHasStaticFunction_Whatever_v</i>&quot; works even when<br>    // &quot;<i>Whatever</i>&quot; is overloaded.<br>    //////////////////////////////////////////////////////////////<br>    static int Whatever(char) noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasStaticFunction_Whatever_v</i>&quot; called<br>// below which is set to true if the <i>static</i> (public) function<br>// &quot;<i>Whatever</i>&quot; exists in the class you pass it (via the 1st<br>// template arg), and exactly matches the <a href="#cppfunctiontypes">function type</a> you pass<br>// it (via the 2nd template arg). False returned otherwise.<br>/////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_STATIC_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a <i>static</i> (public) function called &quot;<i>Whatever</i>&quot; with<br>    // the exact type being passed via the 2nd template arg,<br>    // &quot;<i>int (std::string_view, float) noexcept</i>&quot; in this case.<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasStaticFunction_Whatever = ClassHasStaticFunction_Whatever_v&lt;Test, int (std::string_view, float) noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr valign="top">
      <td><b><a href="#declare_class_has_non_overloaded_function">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)</a></b><br><br>Functionally equivalent to macro <i>DECLARE_CLASS_HAS_FUNCTION</i> on the first row above but used when you need to detect a <i>non-overloaded</i> member function instead. That is, the templates created by the latter macro will check for a (non-static) member function's existence whether it's overloaded or not, while the templates created by the following macro will only detect (non-static) member functions that are *not* overloaded (always returning false if they are overloaded). The templates are functionally identical otherwise. See adjacent column for details.</td>
      <td><b><a href="#declare_class_has_non_overloaded_function">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)</a> </b>declares:<pre><b>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>struct ClassHasNonOverloadedFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>inline constexpr bool ClassHasNonOverloadedFunction_##NAME##_v;</b></pre>Functionally equivalent to the templates created by the <i>DECLARE_CLASS_HAS_FUNCTION</i> macro on the first row of this table but unlike those templates, the templates just above will only return true if the target function is not overloaded (unlike the templates created by the latter macro which detect the target function whether overloaded or not). See latter macro for further details. Note that the name of each template based on a NAME macro arg of, say, &quot;<i>Whatever</i>&quot;, will be &quot;<i>ClassHasNonOverloadedFunction_Whatever</i>&quot; and &quot;<i>ClassHasNonOverloadedFunction_Whatever_v</i>&quot;.</td>
      <td><b><a href="#declare_class_has_non_overloaded_function">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)</a></b><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers (only req'd for this example)<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;FunctionTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>//////////////////////////////////////////////////////////////<br>// Class you wish to check for <i>non-overloaded</i>, <i>non-static</i><br>// (public) function &quot;<i>Whatever</i>&quot; with the exact type seen<br>// below (&quot;<i>ClassHasNonOverloadedFunction_Whatever_v</i>&quot; called<br>// further below will detect it only if it's not overloaded)<br>//////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    int Whatever(std::string_view, float) const noexcept;<br><br>    /////////////////////////////////////////////////////////////<br>    // Commented out. If uncommented then &quot;<i>Whatever</i>&quot; becomes<br>    // overloaded so &quot;<i>ClassHasNonOverloadedFunction_Whatever_v</i>&quot;<br>    // below will then return false instead (since it only<br>    // returns true for <i>non-overloaded</i> functions)<br>    /////////////////////////////////////////////////////////////<br>    // int Whatever(char) const noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedFunction_Whatever_v</i>&quot;<br>// called below which is set to true if the <i>non-overloaded</i>,<br>// <i>non-static</i> (public) function &quot;<i>Whatever</i>&quot; exists in the class<br>// you pass it (via the 1st template arg), is not overloaded<br>// (unlike the example on the first row above which detects the<br>// target function even if overloaded), and exactly matches the<br>// <a href="#cppfunctiontypes">function type</a> you pass it (via the 2nd template arg). False<br>// returned otherwise.<br>/////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ////////////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg) has<br>    // a <i>non-overloaded</i>, <i>non-static</i> (public) function called<br>    // &quot;<i>Whatever</i>&quot; with the exact type being passed via the 2nd<br>    // template arg, &quot;<i>int (std::string_view, float) const noexcept</i>&quot;<br>    // in this case.<br>    ////////////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedFunction_Whatever = ClassHasNonOverloadedFunction_Whatever_v&lt;Test, int (std::string_view, float) const noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr valign="top">
      <td><b><a href="#declare_class_has_non_overloaded_static_function">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(NAME)</a></b><br><br>Identical to macro just above but detects a <i>static</i> (<i>non-overloaded</i>) function instead of non-static.</td>
      <td><b><a href="#declare_class_has_non_overloaded_static_function">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(NAME)</a> </b>declares:<pre><b>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>struct ClassHasNonOverloadedStaticFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, TRAITS_FUNCTION_C F&gt;<br>inline constexpr bool ClassHasNonOverloadedStaticFunction_##NAME##_v;</b></pre>Functionally equivalent to the templates created by the <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</i> macro on the row just above but detects a <i>static</i> (non-overloaded) function instead of non-static (so the function that the &quot;<i>F</i>&quot; template arg targets must conform to a <a href="#functionclassification">free function</a> or false will always be returned). The templates will therefore return true only if the target function is <i>static</i> but are otherwise identical to the templates on the row just above (see this for details). Note that the name of each template based on a NAME macro arg of, say, &quot;<i>Whatever</i>&quot;, will be &quot;<i>ClassHasNonOverloadedStaticFunction_Whatever</i>&quot; and &quot;<i>ClassHasNonOverloadedStaticFunction_Whatever_v</i>&quot;.</td>
      <td><b><a href="#declare_class_has_non_overloaded_static_function">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(NAME)</a></b><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers (only req'd for this example)<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;FunctionTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>///////////////////////////////////////////////////////////<br>// Class you wish to check for <i>non-overloaded</i>, <i>static</i><br>// (public) function &quot;<i>Whatever</i>&quot; with the exact type seen<br>// below (&quot;<i>ClassHasNonOverloadedStaticFunction_Whatever_v</i>&quot;<br>// called further below will detect it only if it's not<br>// overloaded)<br>///////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    static int Whatever(std::string_view, float) noexcept;<br><br>    /////////////////////////////////////////////////////////////////////<br>    // Commented out. If uncommented then &quot;<i>Whatever</i>&quot; becomes overloaded<br>    // so &quot;<i>ClassHasNonOverloadedStaticFunction_Whatever_v</i>&quot; below will<br>    // then return false instead (since it only returns true for<br>    // <i>non-overloaded</i> functions)<br>    /////////////////////////////////////////////////////////////////////<br>    // static int Whatever(char) noexcept;<br>};<br><br>///////////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedStaticFunction_Whatever_v</i>&quot;<br>// called below which is set to true if the <i>non-overloaded</i>, <i>static</i><br>// (public) function &quot;<i>Whatever</i>&quot; exists in the class you pass it (via<br>// the 1st template arg), is not overloaded (unlike the example on<br>// the second row above which detects the target function even if<br>// overloaded), and exactly matches the <a href="#cppfunctiontypes">function type</a> you pass it<br>// (via the 2nd template arg). False returned otherwise.<br>///////////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ////////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a <i>non-overloaded</i>, <i>static</i> (public) function called<br>    // &quot;<i>Whatever</i>&quot; with the exact type being passed via the 2nd<br>    // template arg, &quot;<i>int (std::string_view, float) noexcept</i>&quot;<br>    // in this case.<br>    ////////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedStaticFunction_Whatever = ClassHasNonOverloadedStaticFunction_Whatever_v&lt;Test, int (std::string_view, float) noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr valign="top">
      <td><b><a href="#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(NAME)</a></b><br><br>Use instead of <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</i> (see third row above) when the latter macro doesn't suffice because you need to do a more granular check for a (<i>non-overloaded</i>) function (say, if you want to ignore the function's return type, its <i>noexcept</i> specification, check a specific arg type only, etc.). You have complete control over what to check for opposed to the latter macro whose templates always look for an exact match of the <a href="#cppfunctiontypes">function type</a> you pass (so instead of passing a <a href="#cppfunctiontypes">function type</a> via the template's 2nd arg, you'll effectively pass the exact function traits you wish to check instead). See adjacent column for details.</td>
      <td><b><a href="#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(NAME)</a> </b>declares:<pre><b>template &lt;IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void&gt;<br>REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)<br>struct ClassHasNonOverloadedFunctionTraits_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void&gt;<br>REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)<br>inline constexpr bool ClassHasNonOverloadedFunctionTraits_##NAME##_v;</b></pre>Similar to the templates created by the <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</i> macro on row 3 above except the 2nd template arg is no longer the function's type you wish to match, but a customized struct you&#39;ll write instead (template arg &quot;<i>HasFunctionTraitsT</i>&quot;), used to determine if the function exists based on the exact function traits you're interested in. Your struct will always declare the template-based function call operator seen in the example below (the struct or class itself can be named anything you wish), or a variation of this call operator taking a 2nd template arg of type &quot;<i>UserTypeT</i>&quot;, as seen seen in the template declaration above (for passing your own user-defined type but see <a href="#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</a> for details). In C++20 or later however it's normally easier to just rely on a lambda template instead (which automatically creates the following template for you behind the scenes - see example in next column and latter link for details). Note that <i>REQUIRES_IS_HAS_FUNCTION_TRAITS_C</i> above simply resolves to a concept in C++20 or later, ensuring that &quot;<i>HasFunctionTraitsT</i>&quot; is a conforming (valid) class.<br><pre>struct HasWhateverTraits<br>{<br>    template &lt;TRAITS_FUNCTION F&gt;<br>    constexpr bool operator()() const noexcept<br>    {<br>        /////////////////////////////////////////////////////<br>        // See explanation in next (example) column and<br>        // <a href="#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</a><br>        // for complete details<br>        /////////////////////////////////////////////////////<br>        return (IsReturnTypeSame_v&lt;F, int&gt; &&<br>                CallingConvention_v&lt;F&gt; == DefaultCallingConvention_v&lt;false&gt; &&<br>                AreArgTypesSame_v&lt;F, std::string_view, float&gt; &&<br>                !IsVariadic_v&lt;F&gt; &&<br>                IsFunctionConst_v&lt;F&gt; &&<br>                !IsFunctionVolatile_v&lt;F&gt; &&<br>                FunctionRefQualifier_v&lt;F&gt; == RefQualifier::None &&<br>                IsNoexcept_v&lt;F&gt;);</pre>&quot;<i>operator()</i>&quot; above will be specialized by the library on the <a href="#cppfunctiontypes">function type</a> of <i>&amp;T::NAME</i> (assuming a <i>non-overloaded</i>, <i>non-static</i> member function called NAME exists in class &quot;<i>T</i>&quot;), and your &quot;<i>operator()</i>&quot; member will simply return <i>true</i> or <i>false</i> to indicate if the function (<a href="#templateargf">Template arg &quot;F&quot;</a>) matches your requirements (by testing the function's traits using the templates in this library). This gives you precise control over what to check for a match in the <a href="#cppfunctiontypes">function's type</a>, opposed to an <i>exact</i> match of the entire type that the <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</i> templates on row 3 require. Note that the name of each template based on a NAME macro arg of, say, &quot;<i>Whatever</i>&quot;, will be &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever</i>&quot; and &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot;.</td>
      <td><b><a href="#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(NAME)</a></b><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers (only req'd for this example)<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;FunctionTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>/////////////////////////////////////////////////////////////<br>// Class you wish to check for <i>non-overloaded</i>, <i>non-static</i><br>// (public) function &quot;<i>Whatever</i>&quot; with the particular traits<br>// seen further below (class &quot;<i>HasWhateverTraits</i>&quot;).<br>// &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot; called<br>// further below will detect it only if it's not overloaded.<br>/////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    int Whatever(std::string_view, float) const noexcept;<br><br>    /////////////////////////////////////////////////////////////////////<br>    // Commented out. If uncommented then &quot;<i>Whatever</i>&quot; becomes overloaded<br>    // so &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot; below will<br>    // then return false instead (since it only returns true for<br>    // non-overloaded functions)<br>    /////////////////////////////////////////////////////////////////////<br>    // int Whatever(char) const noexcept;<br>};<br><br>///////////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot;<br>// called below which is set to true if the <i>non-overloaded</i>, <i>non-static</i><br>// (public) function &quot;<i>Whatever</i>&quot; exists in the class you pass it (via<br>// the 1st template arg), is not overloaded, and has the traits passed <br>// via the 2nd template arg (&quot;<i>HasWhateverTraits</i>&quot; declared in function<br>// &quot;<i>main</i>&quot; below). False returned otherwise.<br>///////////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(Whatever)<br><br>int main()<br>{<br>    //////////////////////////////////////////////////////////////////////////////<br>    // This example assumes C++20 or later (checked for just below) which most<br>    // will likely be using at some point if not already, but see #else comments<br>    // below if you're targeting C++17 (earlier versions aren't supported by the<br>    // &quot;<i>FunctionTraits</i>&quot; library). The following simply creates a lambda template<br>    // to do the work (read on). However, you can even eliminate this &quot;<i>using</i>&quot;<br>    // statement if you wish, and simply pass the &quot;<i>decltype</i>&quot; statement below<br>    // directly to &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot; via its<br>    // 2nd arg. No additional overhead of this &quot;<i>using</i>&quot; statement is therefore<br>    // required, minimal as it is (if you prefer to just invoke the lambda<br>    // on-the-fly).<br>    //<br>    // Note that the lambda (template) itself creates the condition used to<br>    // determine if the <i>non-overloaded</i>, <i>non-static</i> (public) function<br>    // &quot;<i>Test::Whatever</i>&quot; exists in the call to<br>    // &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot; just below (by testing<br>    // the specific function traits you're interested in). The lambda (template)<br>    // will be specialized on the type of &quot;<i>Test::Whatever</i>&quot; in this particular<br>    // example (assuming a <i>non-static</i>, <i>non-overloaded</i> (public) function called<br>    // &quot;<i>Whatever</i>&quot; exists in class &quot;<i>Test</i>&quot;), so its &quot;<i>F</i>&quot; template arg will be<br>    // &quot;<i>int (Test::*)(std::string_view, float) const noexcept</i>&quot; in this example<br>    // (the type of &quot;<i>Test::Whatever</i>&quot;). The same lambda can be reused for other<br>    // functions you wish to check for the same traits however, in any class you<br>    // wish (if ever required). The lambda itself simply checks &quot;<i>F</i>&quot; for whatever<br>    // traits you require using the templates in this library normally (but you<br>    // can conduct any check you want), and returns true if a match is found or<br>    // false otherwise. This is then simply returned by<br>    // &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot; below.<br>    //<br>    // In this particular example however the following lambda effectively<br>    // performs the equivalent of the example on the third row of this table<br>    // above (see <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)</i>), but does so<br>    // by checking the function's individual traits instead (using the templates<br>    // in this library). That is, every trait comprising a <a href="#cppfunctiontypes">C++ function type</a> is<br>    // tested for in the following lambda so it's equivalent to the example on<br>    // the third row above (but the individual traits are tested for here instead<br>    // of the function's actual C++ type). The lambda is therefore passed as the<br>    // 2nd template arg to &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot;<br>    // below, unlike the example on the third row of this table where the<br>    // function's actual C++ type is passed instead. The example on the third row<br>    // would therefore normally be more suitable in this particular case since<br>    // its interface is more natural and less verbose. However, if you need to<br>    // customize your check then the example on the third row can't be used since<br>    // it always checks the entire <a href="#cppfunctiontypes">function type</a> for a match (i.e., every<br>    // possible component of a function as described under <a href="#cppfunctiontypes">C++ function types</a>).<br>    // You'll therefore rely on the following technique instead and just modify<br>    // it for your particular needs (say, if you don't want to check the calling<br>    // convention for instance, which most users normally don't care about - you<br>    // can just remove the call to &quot;<i>CallingConvention_v</i>&quot; check seen in the lambda<br>    // below and it will then be ignored). The following technique therefore<br>    // gives you complete granular control over what to check compared to the<br>    // technique used on the third row of this table. For complete details see<br>    // <a href="#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</a>.<br>    //////////////////////////////////////////////////////////////////////////////<br>    &#35;if CPP20_OR_LATER<br>        using HasWhateverTraits = decltype([]&lt;TRAITS_FUNCTION_C F&gt;() noexcept<br>                                           {<br>                                               return (IsReturnTypeSame_v&lt;F, int&gt; && // Must have &quot;<i>int</i>&quot; return type<br>                                                       CallingConvention_v&lt;F&gt; == DefaultCallingConvention_v&lt;false&gt; && // Must have the default calling convention for a <i>non-static</i> member function<br>                                                       AreArgTypesSame_v&lt;F, std::string_view, float&gt; && // Must have these 2 args (only): &quot;<i>std::string_view</i>&quot; and &quot;<i>float</i>&quot;<br>                                                       !IsVariadic_v&lt;F&gt; && // Must not be variadic (can't end with &quot;<i>...</i>&quot;)<br>                                                       IsFunctionConst_v&lt;F&gt; && // Must be &quot;<i>const</i>&quot;<br>                                                       !IsFunctionVolatile_v&lt;F&gt; && // Must not be &quot;<i>volatile</i>&quot;<br>                                                       FunctionRefQualifier_v&lt;F&gt; == RefQualifier::None && // Must not have any ref-qualifiers (&quot;<i>&amp;</i>&quot; or &quot;<i>&amp;&amp;</i>&quot; though rare in practice)<br>                                                       IsNoexcept_v&lt;F&gt;); // Must be &quot;<i>noexcept</i>&quot;<br>                                           });<br>    &#35;else<br>        //////////////////////////////////////////////////////////////////////<br>        // If targeting C++17 which doesn't support lambda templates, or if<br>        // you simply don't wish to use lambda templates though there's<br>        // normally no reason not to (since they're less verbose), then<br>        // simply replace the &quot;<i>HasWhateverTraits</i>&quot; lambda template above with<br>        // your own equivalent template-based functor instead, like so<br>        // (declared at namespace scope since C++ doesn't support templates<br>        // at function scope - note that the function call operator below is<br>        // equivalent to the lambda template above):<br>        //<br>        //    struct HasWhateverTraits<br>        //    {<br>        //        template &lt;typename F&gt;<br>        //        constexpr bool operator()() const noexcept<br>        //        {<br>        //            return (IsReturnTypeSame_v&lt;F, int&gt; && // Must have &quot;<i>int</i>&quot; return type<br>        //                    CallingConvention_v&lt;F&gt; == DefaultCallingConvention_v&lt;false&gt; && // Must have the default calling convention for a <i>non-static</i> member function<br>        //                    AreArgTypesSame_v&lt;F, std::string_view, float&gt; && // Must have these 2 args (only): &quot;<i>std::string_view</i>&quot; and &quot;<i>float</i>&quot;<br>        //                    !IsVariadic_v&lt;F&gt; && // Must not be variadic (can't end with &quot;<i>...</i>&quot;)<br>        //                    IsFunctionConst_v&lt;F&gt; && // Must be &quot;<i>const</i>&quot;<br>        //                    !IsFunctionVolatile_v&lt;F&gt; && // Must not be &quot;<i>volatile</i>&quot;<br>        //                    FunctionRefQualifier_v&lt;F&gt; == RefQualifier::None && // Must not have any ref-qualifiers (&quot;<i>&amp;</i>&quot; or &quot;<i>&amp;&amp;</i>&quot; though rare in practice)<br>        //                    IsNoexcept_v&lt;F&gt;); // Must be &quot;<i>noexcept</i>&quot;<br>        //        }<br>        //    };<br>        //////////////////////////////////////////////////////////////////////<br>        #error &quot;This example program is designed for C++20 or later (so it can rely on lambda templates which aren't available until C++20), but can be quickly and easily converted to target C++17 or later if required. See comments just above this error message (in the source code) for details&quot;.<br>    &#35;endif<br><br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a <i>non-overloaded</i>, <i>non-static</i> (public) function<br>    // called &quot;<i>Whatever</i>&quot; matching the condition just above<br>    // (passed as the 2nd template arg in the following call).<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedFunctionTraits_Whatever = ClassHasNonOverloadedFunctionTraits_Whatever_v&lt;Test, HasWhateverTraits&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr valign="top">
      <td><b><a href="#declare_class_has_non_overloaded_static_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(NAME)</a></b><br><br>Identical to macro just above but detects a <i>static</i> (<i>non-overloaded</i>) function instead of non-static.</td>
      <td><b><a href="#declare_class_has_non_overloaded_static_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(NAME)</a> </b>declares:<pre><b>template &lt;IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void&gt;<br>REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)<br>struct ClassHasNonOverloadedStaticFunctionTraits_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void&gt;<br>REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)<br>inline constexpr bool ClassHasNonOverloadedStaticFunctionTraits_##NAME##_v;</b></pre>Functionally equivalent to the templates created by the <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</i> macro on the row just above but detects a <i>static</i> function instead of non-static. The templates will therefore return true only if the target (non-overloaded) member function is <i>static</i> (but are otherwise identical to the templates on the row just above). Note that the name of each template based on a NAME macro arg of, say, &quot;<i>Whatever</i>&quot;, will be &quot;<i>ClassHasNonOverloadedStaticFunctionTraits_Whatever</i>&quot; and &quot;<i>ClassHasNonOverloadedStaticFunctionTraits_Whatever_v</i>&quot;.</td>
      <td><b><a href="#declare_class_has_non_overloaded_static_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(NAME)</a></b><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers (only req'd for this example)<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;FunctionTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>////////////////////////////////////////////////////////////<br>// Class you wish to check for <i>non-overloaded</i>, <i>static</i><br>// (public) function &quot;<i>Whatever</i>&quot; with the particular<br>// traits seen further below (class &quot;<i>HasWhateverTraits</i>&quot;).<br>// &quot;<i>ClassHasNonOverloadedStaticFunctionTraits_Whatever_v</i>&quot;<br>// called further below will detect it only if it's not<br>// overloaded.<br>////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    static int Whatever(std::string_view, float) noexcept;<br><br>    /////////////////////////////////////////////////////////////////////<br>    // Commented out. If uncommented then &quot;<i>Whatever</i>&quot; becomes overloaded<br>    // so &quot;<i>ClassHasNonOverloadedStaticFunctionTraits_Whatever_v</i>&quot; below<br>    // will then return false instead (since it only returns true for<br>    // non-overloaded functions)<br>    /////////////////////////////////////////////////////////////////////<br>    // static int Whatever(char) noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedStaticFunctionTraits_Whatever_v</i>&quot;<br>// called below which is set to true if the <i>non-overloaded</i>, <i>static</i> (<i>public</i>)<br>// function &quot;<i>Whatever</i>&quot; exists in the class you pass it (via the 1st template<br>// arg), is not overloaded, and has the traits passed via the 2nd template<br>// arg (&quot;<i>HasWhateverTraits</i>&quot; declared in function &quot;<i>main</i>&quot; below). False<br>// returned otherwise.<br>/////////////////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(Whatever)<br><br>int main()<br>{<br>    ///////////////////////////////////////////////////////////////////<br>    // See the same lambda template in the example on the row just<br>    // above. The same long comments there apply here as well but the<br>    // following detects a <i>static</i> function instead. The check for<br>    // those traits that apply to <i>non-static</i> functions only (as seen<br>    // in the &quot;<i>HasWhateverTraits</i>&quot; lambda for the latter example) have<br>    // therefore been removed from the following &quot;<i>HasWhateverTraits</i>&quot;<br>    // declaration (namely, the checks for cv-qualifiers &quot;<i>const</i>&quot; and<br>    // &quot;<i>volatile</i>&quot; and ref-qualifiers &quot;<i>&amp;</i>&quot; and &quot;<i>&amp;&amp;</i>&quot;).<br>    ///////////////////////////////////////////////////////////////////<br>    &#35;if CPP20_OR_LATER<br>        using HasWhateverTraits = decltype([]&lt;TRAITS_FUNCTION_C F&gt;() noexcept<br>                                           {<br>                                               return (IsReturnTypeSame_v&lt;F, int&gt; && // Must have &quot;<i>int</i>&quot; return type<br>                                                       CallingConvention_v&lt;F&gt; == DefaultCallingConvention_v&lt;true&gt; && // Must have the default calling convention for a <i>free</i> function (since it's static)<br>                                                       AreArgTypesSame_v&lt;F, std::string_view, float&gt; && // Must have these 2 args (only): &quot;<i>std::string_view</i>&quot; and &quot;<i>float</i>&quot;<br>                                                       !IsVariadic_v&lt;F&gt; && // Must not be variadic (can't end with &quot;<i>...</i>&quot;)<br>                                                       IsNoexcept_v&lt;F&gt;); // Must be &quot;<i>noexcept</i>&quot;<br>                                           });<br>    &#35;else<br>        //////////////////////////////////////////////////////////////////////<br>        // Again, see the same comments in the example on the row just above.<br>        // The same comments there apply here as well. Simply replace the<br>        // lambda above with the following template declared at namespace<br>        // scope.<br>        //<br>        //    struct HasWhateverTraits<br>        //    {<br>        //        template &lt;typename F&gt;<br>        //        constexpr bool operator()() const noexcept<br>        //        {<br>        //            return (IsReturnTypeSame_v&lt;F, int&gt; && // Must have &quot;<i>int</i>&quot; return type<br>        //                    CallingConvention_v&lt;F&gt; == DefaultCallingConvention_v&lt;true&gt; && // Must have the default calling convention for a <i>free</i> function (since it's static)<br>        //                    AreArgTypesSame_v&lt;F, std::string_view, float&gt; && // Must have these 2 args (only): &quot;<i>std::string_view</i>&quot; and &quot;<i>float</i>&quot;<br>        //                    !IsVariadic_v&lt;F&gt; && // Must not be variadic (can't end with &quot;<i>...</i>&quot;)<br>        //                    IsNoexcept_v&lt;F&gt;); // Must be &quot;<i>noexcept</i>&quot;<br>        //        }<br>        //    };<br>        //////////////////////////////////////////////////////////////////////<br>        #error &quot;This example program is designed for C++20 or later (so it can rely on lambda templates which aren't available until C++20), but can be quickly and easily converted to target C++17 or later if required. See comments just above this error message (in the source code) for details&quot;.<br>    &#35;endif<br><br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a <i>non-overloaded</i>, <i>static</i> (public) function called<br>    // &quot;<i>Whatever</i>&quot; matching the condition just above (passed<br>    // as the 2nd template arg in the following call).<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedStaticFunctionTraits_Whatever = ClassHasNonOverloadedStaticFunctionTraits_Whatever_v&lt;Test, HasWhateverTraits&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
  </tbody>
</table>

<a name="HelperTemplates"></a>
## Helper templates (complete, alphabetical list)

The following provides a complete (alphabetical) list of all helper templates (see [Usage](#usage) section to get started). Two separate sections exist, the first for [Read traits](#readtraits), allowing you to read any part up a function's type, and the second for [Write traits](#writetraits), allowing you to update any part of a function's type. Note that the first template arg of every template is the function you're targeting, whose name is always "*F*" (see [here](#templateargf) for details). IMPORTANT: When "*F*" is a functor, note that all traits implicitly refer (apply) to the non-static member function "*F::operator()*" unless noted otherwise (so if "*F*" is the type of a lambda for instance then the traits target "*operator()*" of the compiler-generated class for that lambda). It should therefore be understood that whenever "*F*" is a functor and it's cited in the description of each template below, the description is normally referring to member function "*F::operator()*", not class "*F*" itself.

Please note that for all traits, a *TRAITS\_FUNCTION\_C* [concept](https://en.cppreference.com/w/cpp/language/constraints) (declared in "*FunctionTraits.h*") will kick in for illegal values of "*F*" in C++20 or later ("*F*" is declared a *TRAITS\_FUNCTION\_C* in all templates below), or a (compiler-specific) "*FunctionTraits template not defined*" error of some kind in C++17 otherwise (again, earlier versions aren't supported). Note that *TRAITS\_FUNCTION\_C* is just a #defined macro that resolves to the [concept](https://en.cppreference.com/w/cpp/language/constraints) "*StdExt::TraitsFunction\_c*" in C++20 or later (ensuring "*F*" is legal), or simply the "*typename*" keyword in C++17. In either case the "*FunctionTraits*" template is declared but not defined for illegal types of "*F*" (allowing it to be used in SFINAE contexts). For most of the traits, "*F*" is the only template arg (again, see [here](#templateargf)). Only a small handful of templates take additional (template) args which are described on a case-by-case basis below. Lastly, note that the "=" sign is omitted after the template name in each template declaration below. The "=" sign is removed since the implementation isn't normally relevant for most users. You can inspect the actual declaration in "*FunctionTraits.h*" if you wish to see it.

Finally, please note that each template below simply wraps the corresponding member of the "*FunctionTraits*" struct itself
(or indirectly targets it). As previously described, you can access "*FunctionTraits*" directly (see this class and its various base classes in "*FunctionTraits.h*" - you can access all public members), but the helper templates below normally make it unnecessary . You can also access any of its own helper templates *not* documented in this README file (each takes a "*FunctionTraits*" template arg as described in [Technique 2 of 2](#technique2of2)). The following helper templates however taking a function template arg "*F*" instead are normally much easier (again, as described in [Technique 2 of 2](#technique2of2)). The syntax for accessing "*FunctionTraits*" directly (or using any of the helper templates taking a "*FunctionTraits*" template arg) is therefore not shown in this document (just the earlier examples in [Technique 1 of 2](#technique1of2) only). You can simply review the public members of "*FunctionTraits*" inherited from its base classes in "*FunctionTraits.h*" in the unlikely event you ever need to work with them directly (or alternatively review the helper templates in "*FunctionTraits.h*" taking a "*FunctionTraits*" template arg). However, every public member of "*FunctionTraits*" inherited from its base classes has a helper template taking a function template arg "*F*" as seen in each template declaration below, including some additional helper templates not available when accessing "*FunctionTraits*" directly. These helper templates should therefore normally be relied on since they're easier and cleaner, so the following documentation is normally all you ever need to consult.

<a name="ReadTraits"></a>
### _Read traits_

<a name="AreArgTypesSame_v"></a><details><summary>AreArgTypesSame_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          TestIsVariadic TestIsVariadicT,
          typename... ArgsT>
inline constexpr bool AreArgTypesSame_v;
```

"*bool*" variable set to "*true*" if the arg types of function "*F*" match the types passed via the "*ArgsT*" template args (parameter pack) or false otherwise (assuming the variadic status of "*F*" also matches the "*TestIsVariadicT*" template arg - details to follow). This template is effectively the same as calling [std::is\_same\_v](https://en.cppreference.com/w/cpp/types/is_same) for each arg in "*F*", passing the type of each arg in "*F*" as the 1st template arg to [std::is\_same\_v](https://en.cppreference.com/w/cpp/types/is_same), and the corresponding arg in "*ArgsT*" as the second template arg. It therefore simply compares the type of each arg in function "*F*" with each corresponding arg in "*ArgsT*" (in argument order as would be expected). If the number of args in "*F*" doesn't match the number of types in "*ArgsT*" then false is always returned (i.e., it's the equivalent of comparing two function argument lists of different sizes). Note that if "*ArgsT*" is an empty parameter pack then true is returned only if "*F*" has an empty argument list.

Note that in addition to the check for matching *non-variadic* args just described, which is always carried out (so false is guaranteed to be returned if they don't match as just described), the variadic status of "*F*" can also be included in the comparison via the "*TestIsVariadicT*" enumerator arg. This enumerator is declared as follows:

```C++
enum class TestIsVariadic
{
    Ignore = -1, // Rarely used - see below
    False,       // Normally used - see below
    True         // Rarely used - see below
};
```

If you explicitly pass "*TestIsVariadic::Ignore*" which is rarely required (read on), then the variadic status of "*F*" is completely ignored, so it can be variadic or not (it's irrelevant). If you explicitly pass "*TestIsVariadic::False*" however then "*F*" must be non-variadic (i.e., it must not end with "...") or false is guaranteed to be returned (which applies to most functions normally so you'll normally set it to this - note that it doesn't default to it however since "*ArgsT*" is a parameter pack so it must be the last template arg, making it impossible to apply a default value to "*TestIsVariadicT*" in a viable way - rely on [AreArgTypesSameTuple\_v](#areargtypessametuple_v) instead if you wish to do so). Lastly, if you explicitly pass "*TestIsVariadic::True*" which is also rarely required, then "*F*" must be variadic (end with "...") or false is guaranteed to be returned. Again, regardless of what you pass for the "*TestIsVariadicT*" arg, the function's *non-variadic* args must always match "*ArgsT*" as previously described or false is guaranteed to be returned (noting that if "*F*" is in fact a variadic function then its *variadic* args are never compared to "*ArgsT*" - only whether it's variadic or not is checked, as controlled by "*TestIsVariadicT*").

```C++

// Function whose arg list we wish to check
using F = int (std::string_view, float);

////////////////////////////////////////////////////////////
// Returns true since "F" has arg types "std::string_view"
// and "float" (an exact match for the types we're passing
// via the template's "ArgsT" parameter pack), and it's
// also not variadic (as per the "TestIsVariadic::False"
// template arg)
////////////////////////////////////////////////////////////
constexpr bool areArgTypesSame_v = AreArgTypesSame_v<F,
                                                     TestIsVariadic::False, // "F" must be non-variadic (not end with "...")
                                                     std::string_view, float>; // "F" must have these args (exact match)

```

</blockquote></details>

<a name="AreArgTypesSameTuple_v"></a><details><summary>AreArgTypesSameTuple_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          TUPLE_C TupleT,
          TestIsVariadic TestIsVariadicT = TestIsVariadic::False>
inline constexpr bool AreArgTypesSameTuple_v;
```

Identical to [AreArgTypesSame\_v](#areargtypessame_v) just above except the argument list is passed as a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) instead of a parameter pack (via the 2nd template arg). The types in the [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) are therefore compared with those in function "*F*". This template is otherwise identical to [AreArgTypesSame\_v](#areargtypessame_v). See this for details (but please note that the order of the "*TestIsVariadicT*" arg in the latter template and this one is reversed, allowing the "*TestIsVariadic::False*" default to be supported in this template since it's usually required, but not in [AreArgTypesSame\_v](#areargtypessame_v) where it must be explicitly passed - see this for details).

```C++

// Function whose arg list we wish to check
using F = int (std::string_view, float);

////////////////////////////////////////////////////////////
// Returns true since "F" has arg types "std::string_view"
// and "float" (an exact match for the types we're passing
// via the template's "TupleT" template arg), and it's also
// not variadic (as per the "TestIsVariadic::False"
// default template arg)
////////////////////////////////////////////////////////////
constexpr bool areArgTypesSameTuple_v = AreArgTypesSameTuple_v<F,
                                                               std::tuple<std::string_view, float> // "std::tuple" of arg types to check "F" for
                                                              >;

```

</blockquote></details>

<a name="ArgCount_v"></a><details><summary>ArgCount_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr std::size_t ArgCount_v;
```

"*std::size\_t*" variable storing the number of arguments in "*F*" *not* including variadic arguments if any. Will be zero if function "*F*" has no (non-variadic) args. Note that this count is formally called "*arity*" but this variable is given a more user-friendly name.<br /><br /><ins>IMPORTANT</ins>:<br />Please note that if you wish to check if a function's argument list is completely empty, then inspecting this helper template for zero (0) is not sufficient, since it may return zero but still contain variadic args. To check for a completely empty argument list, call [IsEmptyArgList\_v](#isemptyarglist_v) instead.
</blockquote></details>

<a name="ArgType_t"></a><details><summary>ArgType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t I
          bool AssertIfInvalidIndex = true>
using ArgType_t;
```

Type alias for the type of the "*I*th" arg in function "*F*", where "*I*" is in the range 0 to the number of (non-variadic) arguments in "*F*" minus 1 (see [ArgCount\_v](#argcount_v) just above). Pass "*I*" as the (zero-based) 2nd template arg. Note that if "*I*" is greater than or equal to the number of args in "*F*" (again, see [ArgCount\_v](#argcount_v) just above), then a "*static\_assert*" will trigger if "*AssertIfInvalidIndex*" is true (the default), or "*void*" will be returned otherwise (note that "*void*" function arg types are invalid in C++ so if returned then it's guaranteed that "*I*" isn't valid). Note that if "*F*" has no non-variadic args whatsoever then all values of "*I*" are invalid even if passing zero (so it's always guaranteed that either a "*static\_assert*" will trigger or "*void*" will be returned, as controlled by "*AssertIfInvalidIndex*"). Note that passing false for the "*AssertIfInvalidIndex*" arg is often useful in [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae) situations, where you don't want a "*static\_assert*" to trigger when "*I*" is illegal, but for template substitution to simply fail instead. For instance, when using the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS](#declare_class_has_non_overloaded_function_traits) macro (see macro for details), if the "*HasFunctionTraitsT*" arg you pass to the templates created by this macro needs to call [IsArgTypeSame\_v](#isargtypesame_v), which ultimately relies on "*ArgType\_t*", you'll normally want to pass false for the "*AssertIfInvalidIndex*" arg of [IsArgTypeSame\_v](#isargtypesame_v) (which just forwards it along to "*ArgType\_t*"). Otherwise the call to "*HasFunctionTraitsT::operator()*" might trigger a "*static\_assert*" instead of just returning false (again, see [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS](#declare_class_has_non_overloaded_function_traits) for details).
</blockquote></details>

<a name="ArgTypeMatches_v"></a><details><summary>ArgTypeMatches_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F1,
          TRAITS_FUNCTION_C F2,
          std::size_t I>
inline constexpr bool ArgTypeMatches_v;
```

"*bool*" variable set to "*true*" if the arg type at (zero-based) index "*I*" of function "*F1*" is the same as the corresponding arg of function "*F2*" or "*false*" otherwise. This template is effectively the same as calling [std::is\_same_v](https://en.cppreference.com/w/cpp/types/is_same), passing the arg types at index "*I*" of each function to the latter template. Note that if index "*I*" is greater or equal to the number of arguments of either function but not both, then "*false*" is guaranteed to be returned. In this case since arg "*I*" doesn't exist in one function but does in the other, it's the equivalent of substituting "*void*" for the arg that doesn't exist and comparing it with the one that does (the latter can never be "*void*" however since it's not permitted for function arguments in C++ so "*false*" is guaranteed to be returned). Note that if "*I*" is greater or equal to the number of arguments in *both* functions however, then a "*static\_assert*" is guaranteed to trigger (since no such argument exists in either function).

<ins>IMPORTANT</ins>:<br /> Note that this template targets non-variadic arguments only. If either function is a variadic function, i.e., [IsVariadic\_v](#isvariadic_v) returns true, then the template completely ignores this ("*I*" cannot be used to target variadic arguments which are treated as if they don't exist). Caution advised.
</blockquote></details>

<a name="ArgTypeName_v"></a><details><summary>ArgTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t I,
          bool AssertIfInvalidIndex = true>
inline constexpr tstring_view ArgTypeName_v;
```

Same as [ArgType\_t](#argtype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details). A *float* would therefore be (literally) returned as "*float*" for instance (quotes not included). Note that if "*I*" is greater than or equal to the number of args in "*F*", then a "*static\_assert*" will trigger if "*AssertIfInvalidIndex*" is true (the default), or "*void*" will be returned otherwise. See this arg in [ArgType\_t](#argtype_t) for further details.
</blockquote></details>

<a name="ArgTypes_t"></a><details><summary>ArgTypes_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using ArgTypes_t;
```

Type alias for a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) representing all non-variadic argument types in "*F*". Rarely required in practice however since you'll usually rely on [ArgType\_t](#argtype_t) or [ArgTypeName\_v](#argtypename_v) to retrieve the type of a specific argument (see these above). If you require the [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) that stores all (non-variadic) argument types, then it's typically (usually) because you want to iterate all of them (say, to process the type of every argument in a loop). If you require this, then you can use the [ForEachArg](#foreacharg) helper function (template) further below. See this for details.
</blockquote></details>

<a name="ArgTypesMatch_v"></a><details><summary>ArgTypesMatch_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F1,
          TRAITS_FUNCTION_C F2,
          TestIsVariadic TestIsVariadicT = TestIsVariadic::False>
inline constexpr bool ArgTypesMatch_v;
```

"*bool*" variable set to "*true*" if the arg types of function "*F1*" match the arg types of function "*F2*" or false otherwise. This function is equivalent to calling [std::is\_same_v](https://en.cppreference.com/w/cpp/types/is_same) for each corresponding arg type in both functions, returning true if all of them match or false otherwise (assuming the variadic status of both functions also match via the "*TestIsVariadicT*" template arg - read on). If the number of arguments aren't the same in both functions however then "*false*" is guaranteed to be returned (since their arg lists obviously aren't the same).

Note that in addition to the check for matching non-variadic args just described, which is always carried out (so false is guaranteed to be returned if they don't match as just described), the variadic status of both functions can also be included in the comparison via the "*TestIsVariadicT*" enumerator arg. This enumerator is declared as follows:

```C++
enum class TestIsVariadic
{
    Ignore = -1, // Rarely used - see below
    False,       // Normally used - see below
    True         // Rarely used - see below
};
```

The "*TestIsVariadicT*" arg defaults to "*TestIsVariadic::False*" meaning that both functions must both be non-variadic (i.e., must not end with "...") or false is guaranteed to be returned (and since most functions are non-variadic the template arg defaults to "*TestIsVariadic::False*" as noted). If you explicitly pass "*TestIsVariadic::True*" however then both functions must be variadic (i.e., end with "...") or false is guaranteed to be returned. If you explicitly pass "*TestIsVariadic::Ignore*" however then the variadic status of both functions is completely ignored, so either can be variadic or not (it's irrelevant). Again, regardless of what you pass for the "*TestIsVariadicT*" arg, both function's *non-variadic* args must always match as previously described or false is guaranteed to be returned (noting that if both functions are in fact variadic then their *variadic* args are never compared to each other - only whether both functions are variadic or not is checked, as controlled by "*TestIsVariadicT*")

```C++

/////////////////////////////////////////////
// Lambda whose arg list we wish to compare
// with "Test::Whatever" below
/////////////////////////////////////////////
constexpr auto myLambda = [](std::string_view, float);
                            {
                            };

class Test
{
public:
    ///////////////////////////////////////
    // Function whose arg list we wish to
    // compare with the lambda's above
    ///////////////////////////////////////
    int Whatever(std::string_view, float) const noexcept;
};
                  
using F1 = decltype(myLambda);
using F2 = decltype(&Test::Whatever);

////////////////////////////////////////////////////
// Returns true since "F1" and "F2" have matching
// args and both are also not variadic (as per the
// templates's "TestIsVariadic::False" default arg)
////////////////////////////////////////////////////
constexpr bool argTypesMatch_v = ArgTypesMatch_v<F1, F2>;

```

</blockquote></details>

<a name="CallingConvention_v"></a><details><summary>CallingConvention_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr CallingConvention CallingConvention_v;
```

Variable storing the calling convention of "*F*" returned as a "*CallingConvention*" enumerator (declared in "*FunctionTraits.h*"):

```C++
enum class CallingConvention
{
    Cdecl,
    Stdcall,
    Fastcall,
    Vectorcall,
    Thiscall,
    Regcall,
    Variadic = Cdecl // Variadic functions are always "Cdecl" on all supported compilers
};

```

 Note that in all cases only those calling conventions that are supported by a given compiler can be returned by "*CallingConvention\_v*" (so, for instance, "*Vectorcall*" will never be returned on GCC since it doesn't support this calling convention). Also please note that compilers will sometimes change the calling convention declared on your functions to the "*Cdecl*" calling convention depending on the compiler options in effect at the time (in particular when compiling for 64 bits opposed to 32 bits, though some calling conventions like "*Vectorcall*" *are* supported on 64 bits though again, GCC doesn't support "*Vectorcall*"). In this case the calling convention on your function is ignored and "*CallingConvention\_v*" will correctly return the "*Cdecl*" calling convention (if that's what the compiler actually uses). For a function's default calling convention see [DefaultCallingConvention\_v](#defaultcallingconvention_v).
</blockquote></details>

<a name="CallingConventionName_v"></a><details><summary>CallingConventionName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view CallingConventionName_v;
```

Same as [CallingConvention\_v](#callingconvention_v) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details). Note that unlike the "*CallingConvention*" enumerators returned by [CallingConvention\_v](#callingconvention_v) however (which are always uppercase), these strings are always returned in lowercase ("*cdecl*", "*stdcall*", "*fastcall*", "*vectorcall*", "*thiscall*" or "*regcall*").
</blockquote></details>

<a name="DefaultCallingConvention_v"></a><details><summary>DefaultCallingConvention_v</summary>

<blockquote>

```C++
template <bool IsFreeFuncT>
inline constexpr CallingConvention DefaultCallingConvention_v;
```
Variable storing the default calling convention for a function (for a function's actual calling convention see [CallingConvention\_v](#callingconvention_v)). Pass true for "*IsFreeFuncT*" if you require the default calling convention for a *free* function (which includes static member functions), or false for a non-static member function (since the calling convention of free and non-static member functions might differ depending on the compiler and/or compiler options and/or target architecture - the Microsoft compiler in particular defaults to the *\_\_cdecl* calling convention for free functions when targeting x86 but *\_\_thiscall* for non-static member functions - see the [Gd, /Gr, /Gv, /Gz](https://learn.microsoft.com/en-us/cpp/build/reference/gd-gr-gv-gz-calling-convention?view=msvc-170) compiler options - no such option exists for all other supported compilers which normally just default to "*\_\_attribute\_\_((cdecl))*" for both free and non-static functions).

Note that the default calling convention refers to the calling convention applied to a function by the compiler when not explicitly specified in the function itself. The calling convention is returned as a "*CallingConvention*" enumerator declared in "*FunctionTraits.h*":

```C++
enum class CallingConvention
{
    Cdecl,
    Stdcall,
    Fastcall,
    Vectorcall,
    Thiscall,
    Regcall,
    Variadic = Cdecl // Variadic functions are always "Cdecl" on all supported compilers
};
```

Note that in all cases above only those calling conventions that are supported by a given compiler can be returned by "*DefaultCallingConvention\_v*" (so for instance, "*Vectorcall*" will never be returned on GCC since it doesn't support this calling convention).

</blockquote></details>

<a name="DefaultCallingConventionName_v"></a><details><summary>DefaultCallingConventionName_v</summary>

<blockquote>

```C++
template <bool IsFreeFuncT>
inline constexpr tstring_view DefaultCallingConventionName_v;
```

Same as [DefaultCallingConvention\_v](#defaultcallingconvention_v) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details). Note that unlike the "*CallingConvention*" enumerators returned by [DefaultCallingConvention\_v](#defaultcallingconvention_v) however (which are always uppercase), these strings are always returned in lowercase ("*cdecl*", "*stdcall*", "*fastcall*", "*vectorcall*", "*thiscall*" or "*regcall*").
</blockquote></details>

<a name="ForEachArg"></a><details><summary>ForEachArg</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,>
          FOR_EACH_TUPLE_FUNCTOR_C ForEachTupleFunctorT>
inline constexpr bool ForEachArg(ForEachTupleFunctorT &&);
```

Not a traits template (unlike most others), but a helper function template you can use to iterate all arguments for function "*F*" if required (though rare in practice since you'll usually rely on [ArgType\_t](#argtype_t) or [ArgTypeName\_v](#argtypename_v) to retrieve the type of a specific argument - see these above). See [Looping through all function arguments](#loopingthroughallfunctionarguments) earlier for an example, as well as the declaration of "*ForEachArg()*" in "*FunctionTraits.h*" for full details (or for a complete program that also uses it, see the [demo](https://godbolt.org/z/KjGvef1x3) program, also available in the repository itself).
</blockquote></details>

<a name="FunctionRawType_t"></a><details><summary>FunctionRawType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionRawType_t;
```

Type alias of the "raw" (native) C++ function type represented by "*F*" so just "*F*" itself after removing any pointer and/or reference that may exist in "*F*", if any (or if "*F*" is a functor then the "raw" type of "*operator()*" for the functor - read on). Note that this library supports pointers and references to functions (and references to pointers to functions) as well as functors, in addition to "raw" (native) C++ function types (see [Template arg "F"](#templateargf)), so if "*F*" is a pointer or reference to a function (or "*F*" is a functor - again, read on), then this alias simply yields the actual (native) C++ function type of "*F*" after removing the pointer and/or reference (see [C++ function types](#cppfunctiontypes)). The resulting alias therefore always satisfies [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function). Note that if "*F*" is a functor (i.e., [IsFunctor\_v](#isfunctor_v) returns true), then the alias refers to the "raw" type of the functor's "*operator()*" member as previously noted. Also see [FunctionType\_t](#functiontype_t) if you wish to simply return "*F*" as-is (unaltered so any pointer and/or reference in "*F*" remains intact).
</blockquote></details>

<a name="FunctionRawTypeName_v"></a><details><summary>FunctionRawTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view FunctionRawTypeName_v;
```

Same as [FunctionRawType\_t](#functionrawtype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details).
</blockquote></details>

<a name="FunctionRefQualifier_v"></a><details><summary>FunctionRefQualifier_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr RefQualifier FunctionRefQualifier_v;
```

Variable storing the ref-qualifier of "*F*" returned as a "*RefQualifier*" enumerator declared in "*FunctionTraits.h*":

```C++
enum class RefQualifier
{
    None,
    LValue,
    RValue
};
```

 Set to "*RefQualifier::None*" if the function isn't declared with any ref-qualifiers ("*&*" or "*&&*", and in practice neither is declared usually), "*RefQualifier::LValue*" if the function is declared with the "*&*" ref-qualifier, or "*RefQualifier::RValue*" if the function is declared with the "*&&*" ref-qualifier. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then this variable always returns "*RefQualifier::None*" (since free functions can't be declared with ref-qualifiers in C++).
</blockquote></details>

<a name="FunctionRefQualifierName_v"></a><details><summary>FunctionRefQualifierName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          bool UseAmpersands = true>
inline constexpr tstring_view FunctionRefQualifierName_v;
```

Same as [FunctionRefQualifier\_v](#functionrefqualifier_v) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details). Note that this template also takes an extra template arg besides function "*F*", a "*bool*" called "*UseAmpersands*", indicating whether the returned string should be returned as "*&*" or "*&&*" (if the function is declared with an "*&*" or "*&&*" ref-qualifier respectively), or "*lvalue*" or "*rvalue*" otherwise (always in lowercase as seen, which differs from the "*RefQualifier*" enumerator names themselves, as returned by [FunctionRefQualifier\_v](#functionrefqualifier_v)). Defaults to "*true*" if not specified (so returns "*&*" or "*&&*" by default). Not applicable however if no ref-qualifiers are present in "*F*" (i.e., [FunctionRefQualifier\_v](#functionrefqualifier_v) returns "*RefQualifier::None*"), in which case an empty string ("") is always returned (which is usually the case - ref-qualifiers are rarely used by most in practice).
</blockquote></details>

<a name="FunctionType_t"></a><details><summary>FunctionType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionType_t;
```

Type alias identical to "*F*" itself unless "*F*" is a functor (i.e., [IsFunctor\_v](#isfunctor_v) returns true), in which case it's a type alias for the "*operator()*" member of the functor (to retrieve the functor (class) type itself in this case see [MemberFunctionClass\_t](#memberfunctionclass_t)). To retrieve the raw (native) C++ type represented by "*F*" (just "*F*" itself after removing any pointer and/or reference in "*F*" if any, again, referring to "*F::operator()*" if "*F*" is a functor), then see [FunctionRawType\_t](#functionrawtype_t).
</blockquote></details>

<a name="FunctionTypeName_v"></a><details><summary>FunctionTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool FunctionTypeName_v;
```

Same as [FunctionType\_t](#functiontype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details).
</blockquote></details>

<a name="IsAbominableFunction_v"></a><details><summary>IsAbominableFunction_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsAbominableFunction_v;
```

"*bool*" variable set to "*true*" if "*F*" is an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) or "*false*" otherwise. An abominable function (see latter link) is simply any raw (native) C++ function type with the cv-qualifiers "*const*" and/or "*volatile*" and/or the ref-qualifiers "*&*" or "*&&*" (so not a pointer to a non-static member function with these qualifiers but the actual (native) C++ function type itself). Note that while [abominable functions](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) resemble "*free*" functions based on their syntactic appearance, they are not considered "*free*" functions by this library. If this variable returns true then [IsFreeFunction\_v](#isfreefunction_v) is therefore guaranteed to return false (since the qualifiers "*const*", "*volatile*", "*&*" and "*&&*" aren't legal for "*free*" functions in C++). See [C++ function types](#cppfunctiontypes) for complete details.
</blockquote></details>

<a name="IsArgTypeSame_v"></a><details><summary>IsArgTypeSame_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t I,
          typename T,
          bool AssertIfInvalidIndex = true>
inline constexpr bool IsArgTypeSame_v;
```

"*bool*" variable set to "*true*" if the (zero-based) "*Ith*" arg of function "*F*" is the same as the given type "*T*" or false otherwise. This template is just a thin wrapper around [std::is\_same\_v](https://en.cppreference.com/w/cpp/types/is_same), where the latter template is passed the type of the "*Ith*" arg in function "*F*" and type "*T*". It therefore simply compares the type of the "*Ith*" arg in function "*F*" with type "*T*". This is a common requirement for many users (possibly in conjunction with [IsReturnTypeSame\_v](#isreturntypesame_v)), so this template provides a convenient wrapper. Note that if "*I*" is greater than or equal to the number of args in "*F*" (i.e., it refers to a non-existent arg - variadic args if any are ignored), then a "*static\_assert*" will trigger if "*AssertIfInvalidIndex*" is true (the default), or "*void*" will be compared to template arg "*T*" otherwise. In the latter case false is therefore guaranteed to be returned so long as you don't pass "*void*" itself for template arg "*T*". This is usually the case unless you're specifically checking for this situation (an illegal value for "*I*"), since function arg types can't be "*void*" in C++ (so you won't normally pass "*void*" for template arg "*T*" - it will always result in a false return unless "*I*" itself is illegal). See the "*AssertIfInvalidIndex*" arg in [ArgType\_t](#argtype_t) for further details on this parameter.
</blockquote></details>

<a name="IsEmptyArgList_v"></a><details><summary>IsEmptyArgList_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsEmptyArgList_v;
```

"*bool*" variable set to "*true*" if the function represented by "*F*" has an empty arg list (it has no args whatsoever including variadic args), or "*false*" otherwise. If "*true*" then note that [ArgCount\_v](#argcount_v) is guaranteed to return zero (0), and [IsVariadic\_v](#isvariadic_v) is guaranteed to return false.<br /><br /><ins>IMPORTANT</ins>:<br />Note that you should rely on this helper to determine if a function's argument list is completely empty opposed to checking the [ArgCount\_v](#argcount_v) helper for zero (0), since the latter returns zero only if "*F*" has no non-variadic args. If it has variadic args but no others, i.e., its argument list is "(...)", then the argument list isn't empty even though [ArgCount\_v](#argcount_v) returns zero (since it still has variadic args). Caution advised.
</blockquote></details>

<a name="IsFreeFunction_v"></a><details><summary>IsFreeFunction_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsFreeFunction_v;
```

"*bool*" variable set to "*true*" if "*F*" is a free function (which includes static member functions), or "*false*" otherwise. Note that [abominable functions](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) are not considered "*free*" functions by this library so if this variable returns true then [IsAbominableFunction\_v](#isabominablefunction_v) is guaranteed to return false. See the latter link for further details.
</blockquote></details>

<a name="IsFunctionConst_v"></a><details><summary>IsFunctionConst_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsFunctionConst_v;
```

"*bool*" variable set to "*true*" if the function is declared with the "*const*" cv-qualifier, or "*false*" otherwise. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then this variable always returns false (since free functions can't be declared with the "*const*" cv-qualifier in C++).
</blockquote></details>

<a name="IsFunctionVolatile_v"></a><details><summary>IsFunctionVolatile_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsFunctionVolatile_v;
```

"*bool*" variable set to "*true*" if the function is declared with the "*volatile*" cv-qualifier, or "*false*" otherwise. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then this variable always returns false (since free functions can't be declared with the "*volatile*" cv-qualifier in C++).
</blockquote></details>

<a name="IsFunctor_v"></a><details><summary>IsFunctor_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsFunctor_v;
```

"*bool*" variable set to "*true*" if "*F*" is a functor (the functor's class was passed for "*F*") or "*false*" otherwise. Note that when true, [IsMemberFunction\_v](#ismemberfunction_v) is also guaranteed to be true.
</blockquote></details>

<a name="IsMemberFunction_v"></a><details><summary>IsMemberFunction_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsMemberFunction_v;
```

"*bool*" variable set to "*true*" if "*F*" is a non-static member function (including when "*F*" is a functor, i.e., [IsFunctor\_v](#isfunctor_v) returns true), or "*false*" otherwise. Note that [abominable functions](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) aren't non-static member functions so if "*F*" refers to one (i.e., [IsAbominableFunction\_v](#isabominablefunction_v) returns true), then this variable will always returns false.
</blockquote></details>

<a name="IsNoexcept_v"></a><details><summary>IsNoexcept_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsNoexcept_v;
```

"*bool*" variable set to "*true*" if "*F*" is declared as "*noexcept*" or "*false*" otherwise (always false if the "*noexcept*" specifier is absent in the function, otherwise if present then it evaluates to "*true*" if no bool expression is present in the "*noexcept*" specifier (the expression has been omitted), or the result of the bool expression otherwise - WYSIWYG).
</blockquote></details>

<a name="IsReturnTypeSame_v"></a><details><summary>IsReturnTypeSame_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename T>
inline constexpr bool IsReturnTypeSame_v;
```

"*bool*" variable set to "*true*" if the return type of function "*F*" is the same as the given type "*T*" or false otherwise. This template is just a thin wrapper around [std::is\_same\_v](https://en.cppreference.com/w/cpp/types/is_same), where the latter template is passed the return type of function "*F*" and type "*T*". It therefore simply compares the return type of function "*F*" with type "*T*". This is sometimes required when testing function signatures (often in conjunction with the library's various [Arg](#args) templates), so this template provides a convenient wrapper. Also see [ReturnTypesMatch\_v](#returntypesmatch_v) if you wish to compare the return types of two functions, or [IsVoidReturnType\_v](#isvoidreturntype_v) to check a return type for "*void*".
</blockquote></details>

<a name="IsVariadic_v"></a><details><summary>IsVariadic_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsVariadic_v;
```

"*bool*" variable set to "*true*" if "*F*" is a variadic function (last arg of "*F*" is "...") or "*false*" otherwise.
</blockquote></details>

<a name="IsVoidReturnType_v"></a><details><summary>IsVoidReturnType_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsVoidReturnType_v;
```

"*bool*" variable set to "*true*" if the return type of "*F*" is "*void*" (ignoring cv-qualifiers if any), or "*false*" otherwise. Note that this variable is identical to passing [ReturnType\_t](#returntype_t) to [std::is\_void](https://en.cppreference.com/w/cpp/types/is_void). The variable simply provides a convenient wrapper.
</blockquote></details>

<a name="MemberFunctionClass_t"></a><details><summary>MemberFunctionClass_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionClass_t;
```

If "*F*" is a non-static member function (or a functor), yields a type alias for the type of the class (or struct) that declares that function (same as "*F*" itself if "*F*" is a functor). Always "*void*" otherwise (for free functions which includes static member functions, or abominable functions). You may therefore wish to invoke [IsMemberFunction\_v](#ismemberfunction_v) to detect if "*F*" is in fact a non-static member function (or functor) before applying this trait.
</blockquote></details>

<a name="MemberFunctionClassName_v"></a><details><summary>MemberFunctionClassName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view MemberFunctionClassName_v;
```

Same as [MemberFunctionClass\_t](#memberfunctionclass_t) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details).
</blockquote></details>

<a name="ReturnType_t"></a><details><summary>ReturnType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using ReturnType_t;
```

Type alias for the return type of function "*F*".
</blockquote></details>

<a name="ReturnTypeName_v"></a><details><summary>ReturnTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view ReturnTypeName_v;
```

Same as [ReturnType\_t](#returntype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring\_view*" - see [TypeName\_v](#typename_v) for details). A *float* would therefore be (literally) returned as "*float*" for instance (quotes not included).
</blockquote></details>

<a name="ReturnTypesMatch_v"></a><details><summary>ReturnTypesMatch_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F1,
          TRAITS_FUNCTION_C F2>
inline constexpr bool ReturnTypesMatch_v;
```

"*bool*" variable set to "*true*" if the return type of function "*F1*" matches the return type of function "*F2*" or "*false*" otherwise. This template is the same as calling [std::is\_same\_v](https://en.cppreference.com/w/cpp/types/is_same), passing the return type of each function. Also see [IsReturnTypesSame\_v](#isreturntypesame_v) if you wish to check the return type of a function for a specific type, or [IsVoidReturnType\_v](#isvoidreturntype_v) to check a return type for "*void*".
</blockquote></details>

---

<a name="WriteTraits"></a>
### _Write traits_[^8]

<a name="AddNoexcept_t"></a><details><summary>AddNoexcept_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using AddNoexcept_t;
```

Type alias for "*F*" after adding "*noexcept*" to "*F*" if not already present.
</blockquote></details>

<a name="AddVariadicArgs_t"></a><details><summary>AddVariadicArgs_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using AddVariadicArgs_t;
```

Type alias for "*F*" after adding "..." to the end of its argument list if not already present. Note that the calling convention is also changed to the "*Cdecl*" calling convention for the given compiler. This is the only supported calling convention for variadic functions in this release, but most platforms require this calling convention for variadic functions. It ensures that the calling function (opposed to the called function) pops the stack of arguments after the function is called, which is required by variadic functions. Other calling conventions that also do this are possible though none are currently supported in this release (since none of the currently supported compilers support this - such calling conventions are rare in practice).
</blockquote></details>

<a name="FunctionAddConst_t"></a><details><summary>FunctionAddConst_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionAddConst_t;
```

Type alias for "*F*" after adding the "*const*" cv-qualifier to "*F*" if not already present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then the resulting type alias becomes an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (so [IsAbominableFunction\_v](#isabominablefunction_v) returns true for the resulting type and [IsFreeFunction\_v](#isfreefunction_v) returns false). In this case note that if "*F*" is a pointer or reference to a free function (or a reference to a pointer), then the pointer and/or reference is removed (since pointers and/or references to abominable functions are illegal in C++).
</blockquote></details>

<a name="FunctionAddCV_t"></a><details><summary>FunctionAddCV_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionAddCV_t;
```

Type alias for "*F*" after adding both the "*const*" AND "*volatile*" cv-qualifiers to "*F*" if not already present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then the resulting type alias becomes an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (so [IsAbominableFunction\_v](#isabominablefunction_v) returns true for the resulting type and [IsFreeFunction\_v](#isfreefunction_v) returns false). In this case note that if "*F*" is a pointer or reference to a free function (or a reference to a pointer), then the pointer and/or reference is removed (since pointers and/or references to abominable functions are illegal in C++).
</blockquote></details>

<a name="FunctionAddLValueReference_t"></a><details><summary>FunctionAddLValueReference_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionAddLValueReference_t;
```

Type alias for "*F*" after adding the "*&*" ref-qualifier to "*F*" if not already present (replacing the "*&&*" ref-qualifier if present). Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then the resulting type alias becomes an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (so [IsAbominableFunction\_v](#isabominablefunction_v) returns true for the resulting type and [IsFreeFunction\_v](#isfreefunction_v) returns false). In this case note that if "*F*" is a pointer or reference to a free function (or a reference to a pointer), then the pointer and/or reference is removed (since pointers and/or references to abominable functions are illegal in C++).
</blockquote></details>

<a name="FunctionAddRValueReference_t"></a><details><summary>FunctionAddRValueReference_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionAddRValueReference_t;
```

Type alias for "*F*" after adding the "*&&*" ref-qualifier to "*F*" if not already present (replacing the "*&*" ref-qualifier if present). Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then the resulting type alias becomes an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (so [IsAbominableFunction\_v](#isabominablefunction_v) returns true for the resulting type and [IsFreeFunction\_v](#isfreefunction_v) returns false). In this case note that if "*F*" is a pointer or reference to a free function (or a reference to a pointer), then the pointer and/or reference is removed (since pointers and/or references to abominable functions are illegal in C++).
</blockquote></details>

<a name="FunctionAddVolatile_t"></a><details><summary>FunctionAddVolatile_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionAddVolatile_t;
```

Type alias for "*F*" after adding the "*volatile*" cv-qualifier to "*F*" if not already present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), then the resulting type alias becomes an [abominable function](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html) (so [IsAbominableFunction\_v](#isabominablefunction_v) returns true for the resulting type and [IsFreeFunction\_v](#isfreefunction_v) returns false). In this case note that if "*F*" is a pointer or reference to a free function (or a reference to a pointer), then the pointer and/or reference is removed (since pointers and/or references to abominable functions are illegal in C++).
</blockquote></details>

<a name="FunctionRemoveConst_t"></a><details><summary>FunctionRemoveConst_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionRemoveConst_t;
```

Type alias for "*F*" after removing the "*const*" cv-qualifier from "*F*" if present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), yields "*F*" itself (effectively does nothing since "*const*" doesn't apply to free functions so will never be present).
</blockquote></details>

<a name="FunctionRemoveCV_t"></a><details><summary>FunctionRemoveCV_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionRemoveCV_t;
```

Type alias for "*F*" after removing both the "*const*" and "*volatile*" cv-qualifiers from "*F*" if present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), yields "*F*" itself (effectively does nothing since "*const*" and "*volatile*" don't apply to free functions so will never be present).
</blockquote></details>

<a name="FunctionRemoveReference_t"></a><details><summary>FunctionRemoveReference_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionRemoveReference_t;
```

Type alias for "*F*" after removing the "*&*" or "*&&*" ref-qualifier from "*F*" if present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), yields "*F*" itself (effectively does nothing since "*&*" and "*&&*" don't apply to free functions so will never be present).
</blockquote></details>

<a name="FunctionRemoveVolatile_t"></a><details><summary>FunctionRemoveVolatile_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionRemoveVolatile_t;
```

Type alias for "*F*" after removing the "*volatile*" cv-qualifier from "*F*" if present. Note that if "*F*" is a free function (i.e., [IsFreeFunction\_v](#isfreefunction_v) returns true), yields "*F*" itself (effectively does nothing since "*volatile*" doesn't apply to free functions so will never be present).
</blockquote></details>

<a name="FunctionReplaceClass_t"></a><details><summary>FunctionReplaceClass_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          IS_CLASS_OR_VOID_C NewClassT>
using FunctionReplaceClass_t;
```

If "*F*" is a non-static member function which includes functors ([IsMemberFunction\_v](#ismemberfunction_v) returns true for either), yields a type alias for "*F*" after replacing the class it belongs to with "*NewClassT*" (so the resulting type is identical to "*F*" except its class has been replaced with "*NewClassT*" - for functors this refers not to "*F*" however but to "*F::operator()*"). Passing "*void*" for "*NewClassT*" however will remove the class from "*F*", effectively converting the function to a "*raw*" (native) C++ function type (identical to [FunctionRawType\_t](#functionrawtype_t)\<F\> and therefore satisfying [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function) - see [C++ function types](#cppfunctiontypes)). If "*F*" is already a "*raw*" (native) C++ function type however (again, it satisfies [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function)), then converts "*F*" to a *non-static* member function pointer of type ```F NewClassT::*```. Passing "*void*" for "*NewClassT*" however has no effect on "*raw*" (native) C++ functions so the resulting type is identical to "*F*" (since "*void*" means remove the existing class but "*raw*" (native) C++ functions have none - for all intents and purposes it's effectively already been removed).

*Examples:*

```C++
// Non-static member function pointer
using F1 = int (Test::*)(std::string_view, float) const noexcept;

//////////////////////////////////////////////////////////////
// Replaces class "Test" in "F1" with class "Whatever" so
// the resulting type is:
//
// int (Whatever::*)(std::string_view, float) const noexcept
//////////////////////////////////////////////////////////////
using F1ReplacedClassWithWhatever = FunctionReplaceClass_t<F1, Whatever>;

///////////////////////////////////////////////////////////
// Replaces class "Test" in "F1" with class "void" but
// because it's "void" and therefore not an actual class,
// class "Test" is removed from "F1" instead, effectively
// converting it to a "raw" (native C++) function type
// (in this case an "abominable" function because it
// contains "const" - see "C++ function types" earlier in
// this document for details on abominable functions but
// it's immaterial here - qualifiers like "const" in "F1"
// aren't relevant to "FunctionReplaceClass_t", they are
// just migrated to the resulting type as would be
// expected):
//
// int (std::string_view, float) const noexcept
///////////////////////////////////////////////////////////
using F1ConvertedToRawFunction = FunctionReplaceClass_t<F1, void>;

///////////////////////////////////////////////////////
// Raw (native C++) function type (in this case an
// abominable function because it contains "const" -
// again, see "C++ function types" earlier in this
// document for details on abominable functions but
// it's immaterial here - qualifiers like "const" on
// "F1" aren't relevant to "FunctionReplaceClass_t",
// they are just migrated to the resulting type as
// would be expected):
///////////////////////////////////////////////////////
using F2 = int (std::string_view, float) const noexcept;

//////////////////////////////////////////////////////////////
// Replaces the existing class in "F2" with class "Whatever"
// but "F2" is a "raw" (native C++) function (again, an
// "abominable" function in this example because it contains
// "const"), so it has no existing class (since only member
// functions belong to classes). It's therefore assigned one
// ("Whatever"), effectively converting "F2" to a non-static
// member function pointer:
//
// int (Whatever::*)(std::string_view, float) const noexcept
//////////////////////////////////////////////////////////////
using F2ConvertedToNonStaticMemberFunctionPtr = FunctionReplaceClass_t<F2, Whatever>;

///////////////////////////////////////////////////////////
// Replaces the existing class in "F2" with class "void"
// but "F2" is a "raw" (native C++) function so it has no
// existing class (since only member functions belong to
// classes), nor is "void" a class. Because it's "void"
// however the call will therefore remove the existing
// class but since it has none (since it's a "raw"
// function type), there is nothing to do. The resulting
// type is therefore just "F2" unaltered:
//
// int (std::string_view, float) const noexcept
///////////////////////////////////////////////////////////
using F2ReplacedClassWithVoidSoNoChange = FunctionReplaceClass_t<F2, void>;

```
</blockquote></details>

<a name="RemoveNoexcept_t"></a><details><summary>RemoveNoexcept_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using RemoveNoexcept_t;
```

Type alias for "*F*" after removing "*noexcept*" from "*F*" if present.
</blockquote></details>

<a name="RemoveVariadicArgs_t"></a><details><summary>RemoveVariadicArgs_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using RemoveVariadicArgs_t;
```

If "*F*" is a variadic function (its last arg is "..."), yields a type alias for "*F*" after removing the "..." from the argument list. All non-variadic arguments (if any) remain intact (only the "..." is removed).
</blockquote></details>

<a name="ReplaceArgs_t"></a><details><summary>ReplaceArgs_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename... NewArgsT>
using ReplaceArgs_t;
```

Type alias for "*F*" after replacing all its existing non-variadic arguments with the args given by "*NewArgsT*" (a parameter pack of the types that become the new argument list). If none are passed then an empty argument list results instead, though if variadic args are present in "*F*" then they still remain intact (the "..." remains - read on). The resulting alias is identical to "*F*" itself except that the non-variadic arguments in "*F*" are completely replaced with "*NewArgsT*". Note that if "*F*" is a variadic function (its last parameter is "..."), then it remains a variadic function after the call (the "..." remains in place). If you wish to explicitly add or remove the "..." as well then pass the resulting type to [AddVariadicArgs\_t](#addvariadicargs_t) or [RemoveVariadicArgs\_t](#removevariadicargs_t) respectively (either before or after the call to "*ReplaceArgs\_t*"). Note that if you wish to replace specific arguments instead of all of them, then call [ReplaceNthArg\_t](#replacentharg_t) instead. Lastly, you can alternatively use [ReplaceArgsTuple\_t](#replaceargstuple_t) instead of "*ReplaceArgs\_t*" if you have a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) of types you wish to use for the argument list instead of a parameter pack. [ReplaceArgsTuple\_t](#replaceargstuple_t) is identical to "*ReplaceArgs\_t*" otherwise (it ultimately defers to it).
</blockquote></details>

<a name="ReplaceArgsTuple_t"></a><details><summary>ReplaceArgsTuple_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          TUPLE_C NewArgsTupleT>
using ReplaceArgsTuple_t;
```

Identical to [ReplaceArgs\_t](#replaceargs_t) just above except the argument list is passed as a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) instead of a parameter pack (via the 2nd template arg). The types in the [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) are therefore used for the resulting argument list. "*ReplaceArgsTuple\_t*" is otherwise identical to [ReplaceArgs\_t](#replaceargs_t) (it ultimately defers to it).
</blockquote></details>

<a name="ReplaceCallingConvention_t"></a><details><summary>ReplaceCallingConvention_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          CallingConvention NewCallingConventionT>
using ReplaceCallingConvention_t;
```

Type alias for "*F*" after replacing its calling convention with the platform-specific calling convention corresponding to "*NewCallingConventionT*" (a "*CallingConvention*" enumerator declared in "*FunctionTraits.h*"). For instance, if you pass  "*CallingConvention::FastCall*" then the calling convention for "*F*" is replaced with "*\_\_attribute\_\_((fastcall))*" on GCC and Clang, but "*\_\_fastcall*" on Microsoft platforms. Note however that the calling convention for variadic functions (those whose last arg is "...") can't be changed in this release. Variadic functions require that the calling function pop the stack to clean up passed arguments and only the "*Cdecl*" calling convention supports that in this release (on all supported compilers at this writing). Attempts to change it are therefore ignored. Note that you also can't change the calling convention to an unsupported calling convention. For instance, passing "*CallingConvention::Thiscall*" for a free function (which includes static member functions) is ignored since the latter calling convention applies to non-static member functions only. Similarly, passing a calling convention that's not supported by a given compiler is also ignored, such as passing "*CallingConvention::VectorCall*" on GCC (since it doesn't support this calling convention).

Lastly, please note that compilers will sometimes change the calling convention declared on your functions to the "*Cdecl*" calling convention depending on the compiler options in effect at the time (in particular when compiling for 64 bits opposed to 32 bits, though some calling conventions such as "*Vectorcall*" *are* supported on 64 bits, assuming the compiler supports that particular calling convention). Therefore, if you specify a calling convention that the compiler changes to "*Cdecl*" based on the compiler options currently in effect, then "*ReplaceCallingConvention\_t*" will also ignore your calling convention and apply "*Cdecl*" instead (since that's what the compiler actually uses).
</blockquote></details>

<a name="ReplaceNthArg_t"></a><details><summary>ReplaceNthArg_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t N,
          typename NewArgT>
using ReplaceNthArg_t;
```

Type alias for "*F*" after replacing its (zero-based) "*N*th" argument with "*NewArgT*". Pass "*N*" via the 2nd template arg (i.e., the zero-based index of the arg you're targeting), and the type you wish to replace it with via the 3rd template arg ("*NewArgT*"). The resulting alias is therefore identical to "*F*" except its "*N*th" argument is replaced by "*NewArgT*" (so passing, say, zero-based "*2*" for "*N*" and "*int*" for "*NewArgT*" would replace the 3rd function argument with an "*int*"). Note that "*N*" must be less than the number of arguments in the function or a "*static\_assert*" will occur (new argument types can't be added using this trait, only existing argument types replaced). If you need to replace multiple arguments then recursively call "*ReplaceNthArg\_t*" again, passing the result as the "*F*" template arg of "*ReplaceNthArg\_t*" as many times as you need to (each time specifying a new "*N*" and "*NewArgT*"). If you wish to replace all arguments at once then call [ReplaceArgs\_t](#replaceargs_t) or [ReplaceArgsTuple\_t](#replaceargstuple_t) instead. Lastly, note that if "*F*" has variadic arguments (it ends with "..."), then these remain intact. If you need to remove them then call [RemoveVariadicArgs\_t](#removevariadicargs_t) before or after the call to "*ReplaceNthArg\_t*".
</blockquote></details>

<a name="ReplaceReturnType_t"></a><details><summary>ReplaceReturnType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename NewReturnTypeT>
using ReplaceReturnType_t;
```

Type alias for "*F*" after replacing its return type with "*NewReturnTypeT*".
</blockquote></details>

---
<a name="AdditonalHelperTemplates"></a>
### _Additional Helper Templates_

<a name="DECLARE_CLASS_HAS_FUNCTION"></a><details><summary>DECLARE_CLASS_HAS_FUNCTION</summary>

<blockquote>

```C++
#define DECLARE_CLASS_HAS_FUNCTION(NAME)
```

This macro and its cousins declare templates allowing you to determine if an arbitrary class has a (public) member function with a given name and function type. For an overview and comparative summary of these macros and the templates they create (with full examples), please see [Determining if a member function exists](#determiningifamemberfunctionexists). The *DECLARE\_CLASS\_HAS\_FUNCTION* macro specifically detects *non-static* (public) member functions only, whether overloaded or not (the other macros documented further below provide slightly different flavors to target *static* member functions and/or non-overloaded functions). *DECLARE\_CLASS\_HAS\_FUNCTION* allows you to do this by declaring templates taking the class type you're targeting as the 1st template arg, and a *non-static* function type as the 2nd template arg, such as ```int (std::string_view, float) const noexcept``` (passed using any format supported by this library - see [Template arg "F"](#templateargf)). The templates will then determine if the given class (the 1st template arg) has a *non-static* (public) member function with the name you're targeting (read on), whose raw [C++ function type](#cppfunctiontypes) exactly matches that of the latter type (the 2nd template arg). The target function name, "*Whatever*" for instance, is passed to the macro itself and embedded in the template's name. The templates created by the macro therefore always target functions with that specific name only. To this end, the *DECLARE\_CLASS\_HAS\_FUNCTION* macro is used to declare template "*ClassHasFunction\_NAME*" and a *bool* helper variable "*ClassHasFunction\_NAME\_v*" you'll normally use instead (the latter template just defers to the former template). The "*ClassHasFunction\_*" prefix and "*\_v*" suffix in the templates names are hardcoded and never change. The "*NAME*" portion of the templates however is the name of the *non-static* member function you're interested in which you pass via the macro's NAME arg like so (function name "*Whatever*" in this example). You'll always declare this at namespace scope:

```C++
DECLARE_CLASS_HAS_FUNCTION(Whatever)
```

This declares the following two templates used to check for the *non-static* (public) member function "*Whatever*" in any arbitrary class (that matches the function type you're interested in). Again, the 2nd template below is just a helper (bool) variable template with the usual "*\_v*" suffix that defers to the 1st template in the usual C++ way (as seen in its declaration below):

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasFunction_Whatever : std::bool_constant<> // Base class described below
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasFunction_Whatever_v = ClassHasFunction_Whatever<T, F>::value;
```

These templates (again, you'll usually use the "*\_v*" version just above), can then be used to determine if the class given by template arg "*T*" has a *non-static* (public) member function named "*Whatever*" with the exact function type passed via template arg "*F*" (which must conform to any function type supported by this library - see [Template arg "F"](#templateargf)). The comparison itself occurs between the "*raw*" function type of function "*Whatever*", assuming a *non-static* (public) member function with this name exists in class "*T*", and the "*raw*" function type of template arg "*F*". See [C++ function types](#cppfunctiontypes) for details on "*raw*" (native) C++ function types. The "*ClassHasFunction\_Whatever*" template above always inherits from "*std::true\_type*" if the "*raw*" types are identical (every component of their [C++ function types](#cppfunctiontypes) must match exactly including return type, calling convention, etc.), or "*std::false\_type*" otherwise (which are just specializations of [std::bool\_constant](https://en.cppreference.com/w/cpp/types/integral_constant#Helper_alias_templates) seen in the template's declaration above - its *true* or *false* template arg is omitted in the declaration above however since it's an implementation detail handled by the library itself). Again, the second template above is just the "*bool*" equivalent.

For instance, after calling this macro via *DECLARE\_CLASS\_HAS\_FUNCTION(Whatever)*, you can then check any class to determine if it has a *non-static* (public) member function called "*Whatever*" with the given function type you're interested in, such as the following:

```C++
////////////////////////////////////////////////////////
// Only file in this repository you need to explicitly
// #include (see "Usage" section earlier)
////////////////////////////////////////////////////////
#include "FunctionTraits.h"

// Always declared at namespace scope
DECLARE_CLASS_HAS_FUNCTION(Whatever)

///////////////////////////////////////////////////////////
// Class you wish to check for non-static (public) member
// function "Whatever" with the exact type of the first
// overload below ("ClassHasFunction_Whatever_v" called
// further below will detect it regardless of overloads).
///////////////////////////////////////////////////////////
class Test
{
public:
    int Whatever(std::string_view, float) const noexcept;

    /////////////////////////////////////////////////////
    // Overload but above function will still be found.
    // Template "ClassHasFunction_Whatever_v" below
    // works even when "Whatever" is overloaded.
    /////////////////////////////////////////////////////
    int Whatever(char) const noexcept;
};

int main()
{
    ///////////////////////////////////////////////////////////
    // Returns true since class "Test" (the 1st template arg)
    // has a non-static (public) member function called
    // "Whatever" with the exact type being passed via the
    // 2nd template arg (so we're passing the actual C++
    // function type matching "Whatever" above in this case,
    // though you can pass this using any function type
    // supported by this library, since only the raw function
    // type of the function you pass is relevant - see "C++
    // function types" earlier in this document).
    ///////////////////////////////////////////////////////////
    constexpr bool classHasFunction_Whatever = ClassHasFunction_Whatever_v<Test,
                                                                           int (std::string_view, float) const noexcept>;
                                                                           
    return 0;
}    
```

The above variable, "*classHasFunction\_Whatever*", will be set to true because class "*Test*" (the 1st template arg passed in the above call) has a *non-static* (public) member function called "*Whatever*" (the *NAME* arg passed to this macro), whose type exactly matches the 2nd template arg being passed. Again, note that the 2nd template arg "*F*" can be any function type supported by this library (see [Template arg "F"](#templateargf)), but only the raw (native) type of "*F*" is relevant (see [C++ function types](#cppfunctiontypes)). In this example we're passing the actual raw function type itself opposed to a pointer and/or reference to the function (or even a functor, whose "*operator()*" member would then be used for "*F*"), so ```int (std::string_view, float) const noexcept``` in this case (exactly matching the type of function "*Test::Whatever*"). A match will occur if the raw (native) C++ type of "*Whatever*" matches the raw (native) C++ type of "*F*", so any pointers and/or references that may exist in "*F*" (if it's a function pointer and/or reference) are immaterial (as well as when "*F*" is a *non-overloaded* functor - again, the raw function type of its "*operator*()" member will be used for the comparison).

 "*true*" is therefore returned by the template only if the class you pass for the 1st template arg has a *non-static* (public) member function named "*Whatever*" with the exact (raw) [C++ function type](#cppfunctiontypes) of "*F*". This means that every component comprising a C++ function type is tested for in function "*Whatever*" and must match your 2nd template arg *exactly* for these templates to return true (including the return type, calling convention, etc.). The comparison therefore looks for an exact function match of the 2nd template arg (of its "*raw*" type) so caution advised. In the above example for instance, if the return type of "*Test::Whatever*" is changed from "*int*" to, say, "*char*", then the call to "*ClassHasFunction\_Whatever\_v*" will no longer return true (unlike how function signatures are usually handled in most C++ contexts, overloaded functions in particular, where the return type doesn't factor in - it does here however). It will (therefore) now return false unless you also change "*int*" to "*char*" in the 2nd template arg so that the return types match again. Also note that parameter conversions aren't relevant, so if you pass, say, ```int (std::string_view, int) const noexcept``` as the 2nd template arg instead, where the "*float*" parameter in the original call has now been changed to "*int*", then again, the template will now return false even though "*int*" can be implicitly converted to "*float*" (the actual type of the 2nd arg of "*Test::Whatever*"). Again, these templates look for an exact match of the [function type](#cppfunctiontypes) you pass as the 2nd template arg so every component comprising a C++ function (again, see [C++ function types](#cppfunctiontypes)) must match *exactly* for true to be returned (so it's effectively the equivalent of passing the raw function type of "*Test::Whatever*" and the raw function type of the 2nd template arg of "*ClassHasFunction\_Whatever\_v*" to [std::is\_same](https://en.cppreference.com/w/cpp/types/is_same) - the latter targets *every* component of a [C++ function type](#cppfunctiontypes) when passed a function type, including return type, calling convention, etc.).
</blockquote></details>

<a name="DECLARE_CLASS_HAS_STATIC_FUNCTION"></a><details><summary>DECLARE_CLASS_HAS_STATIC_FUNCTION</summary>

<blockquote>

```C++
#define DECLARE_CLASS_HAS_STATIC_FUNCTION(NAME)
```

This macro is functionally equivalent to macro [DECLARE\_CLASS\_HAS\_FUNCTION](#declare_class_has_function) just above but targets *static* (public) member functions instead of non-static. See the latter macro for details. The following templates therefore won't detect functions unless they're *static*. The macro creates the following two templates for this (assuming "*Whatever*" is passed to the macro):

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasStaticFunction_Whatever : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasStaticFunction_Whatever_v = ClassHasStaticFunction_Whatever<T, F>::value;
```

These templates are just the *static* counterparts of the templates targeting *non-static* functions created by [DECLARE\_CLASS\_HAS\_FUNCTION](#declare_class_has_function). Again, see the latter macro for details. Please note however that unlike the templates created by the latter macro, the above templates should normally be passed a [free function](#functionclassification) instead (for template arg "*F*"). Since the templates target *static* functions only and such functions are always [free functions](#functionclassification), if template arg "*F*" contains the "*const*" and/or "*volatile*" cv-qualifiers, and/or the ref-qualifier "*&*" or "*&&*", which don't apply to [free functions](#functionclassification), the templates will always return false (since a *static* function will never have these qualifiers).
</blockquote></details>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION"></a><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</summary>

<blockquote>

```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)
```

This macro is functionally equivalent to macro [DECLARE\_CLASS\_HAS\_FUNCTION](#declare_class_has_function) but targets *non-overloaded* member functions only. These templates will therefore return true only if the function you target exists *and* is not overloaded (unlike the latter macro whose templates will detect functions whether overloaded or not). The two macros are functionally equivalent otherwise. See [DECLARE\_CLASS\_HAS\_FUNCTION](#declare_class_has_function) for details.

The macro creates the following two templates for this (assuming "*Whatever*" is passed to the macro):

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasNonOverloadedFunction_Whatever : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasNonOverloadedFunction_Whatever_v = ClassHasNonOverloadedFunction_Whatever<T, F>::value;
```

These templates are identical to those created by [DECLARE\_CLASS\_HAS\_FUNCTION](#declare_class_has_function) but again, only return true if the target function is not overloaded. See the latter macro for complete details.
</blockquote></details>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION"></a><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION</summary>

<blockquote>

```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(NAME)
```

This macro is functionally equivalent to macro [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) just above but targets *static* (public) member functions instead of non-static. See the latter macro for details. The following templates therefore won't detect functions unless they're *static* as well as *non-overloaded* (since both macros detect non-overloaded functions only). The macro creates the following two templates for this (assuming "*Whatever*" is passed to the macro):

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasNonOverloadedStaticFunction_Whatever : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasNonOverloadedStaticFunction_Whatever_v = ClassHasNonOverloadedStaticFunction_Whatever<T, F>::value;
```

These templates are just the *static* counterparts of the templates targeting *non-static* functions created by [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function).  Again, see the latter macro for details. Please note however that unlike the templates created by the latter macro, the above templates should normally be passed a [free function](#functionclassification) instead (for template arg "*F*"). Since the templates target *static* functions only and such functions are always [free functions](#functionclassification), if template arg "*F*" contains the "*const*" and/or "*volatile*" cv-qualifiers, and/or the ref-qualifier "*&*" or "*&&*", which don't apply to [free functions](#functionclassification), the templates will always return false (since a *static* function will never have these qualifiers).
</blockquote></details>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS"></a><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</summary>

<blockquote>

```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(NAME)
```

This macro is similar to [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function), each allowing you to determine if a class or struct has a *non-overloaded*, *non-static* (public) member function with a given name and function type. Unlike the latter macro however, whose templates always perform an *exact* match of the function type you pass (via the 2nd template arg), the following macro allows you to perform a customized match instead. You therefore have complete control over which parts of the [function's type](#cppfunctiontypes) you wish to inspect for a match, unlike the latter macro whose templates always check the entire type (i.e., every component of the [function's type](#cppfunctiontypes)).

Before proceeding further however, please consult [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) first so you'll better be able to understand the difference. An overview and comparative summary of all the function detection macros and the templates they create can also be found under [Determining if a member function exists](#determiningifamemberfunctionexists).

As described under [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function), the latter macro creates templates used to perform an *exact* type match of the *non-overloaded*, *non-static* (public) member function you're checking for with the function type you pass as the 2nd template arg (say, ```int (std::string_view, float) const noexcept```). The following macro however, *DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS*, creates templates allowing you to perform a customized match instead, so instead of passing a function type to compare with, as required by the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) templates (via the 2nd template arg), you'll effectively pass the specific function traits you're interested in instead (as the 2nd template arg). You'll typically do this if you need to ignore certain components of the function like its return type, its calling convention, its *noexcept* specification, etc. (again, see [C++ function types](#cppfunctiontypes)), but you can check for anything required. You can check for specific parameters only for instance (using [IsArgTypeSame\_v](#isargtypesame_v) usually), assuming you don't need to check them all. You have complete control over the matching process. Therefore, choose [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) if you wish to check if a *non-overloaded*, *non-static* (public) member function precisely matches the exact type you pass (every component described under [C++ function types](#cppfunctiontypes) must precisely match), or *DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS* if you need more fine-grained control.

The DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS macro is called the same way as [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) (and all other function detection macros in the library). You simply pass the name of the *non-overloaded*, *non-static* (public) member function you're interested in via the macro's NAME arg ("*Whatever*" in the following example). Like the other macros you'll always declare this at namespace scope:

```C++
DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(Whatever)
```

This declares the following two templates, where the name of each template always starts with "*ClassHasNonOverloadedFunctionTraits\_*" followed by the *NAME* arg passed to the macro (representing the name of the member function you're targeting, "*Whatever*" in this case). Like all the other member function detection templates in this library, the 1st template below inherits from "*std::true\_type*" if a match is found (described below), or "*std::false\_type*" otherwise. The 2nd template below is just a (bool) helper variable template with the usual "*\_v*" suffix that defers to the 1st template in the usual C++ way:

```C++
template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
struct ClassHasNonOverloadedFunctionTraits_Whatever : std::bool_constant<>
{
};

template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
inline constexpr bool ClassHasNonOverloadedFunctionTraits_Whatever_v = ClassHasNonOverloadedFunctionTraits_Whatever<T, HasFunctionTraitsT, UserTypeT>::value;
```

These templates (you'll usually use the "*\_v*" version just above) can then be used to determine if any class or struct passed via the 1st template arg (*T*), has a *non-overloaded*, *non-static* (public) member function called "*Whatever*" whose traits match those passed via the 2nd template arg ("*HasFunctionTraitsT*", described shortly). Again, the struct itself (the first template above) always inherits from "*std::true\_type*" if so or "*std::false\_type*" otherwise (and the second template above is just the "*bool*" equivalent). Note that unlike the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) templates however, where the 2nd template arg is the actual function type you wish to match, the 2nd arg of both templates above, "*HasFunctionTraitsT*", must be a functor whose "*operator()*" member is a template in its own right, opposed to a non-template function normally associated with "*operator()*" (which isn't normally a template but for our purposes it must be). Your "*operator()*" function (template) should simply return true if its template arg "*F*" (the type of the function given by the macro's NAME arg, "*Whatever*" in this case - the library will pass this in if "*Whatever*" exists), matches your search criteria or false otherwise. Note that your "*operator*()" member can also be optionally declared with a 2nd template parameter for passing your own user-defined type if you require it, but more on this later.

For instance, after calling the DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS macro as seen above (again, passing "*Whatever*" in this case), you can then do the following:

Given any class or struct like so:

```C++
class Test
{
public:
    int Whatever(std::string_view, float) const noexcept;

    /////////////////////////////////////////////////////////////////////
    // Commented out. If uncommented then "Whatever" becomes overloaded
    // so "ClassHasNonOverloadedFunctionTraits_Whatever_v" will then
    // return false instead (since it only returns true for
    // non-overloaded functions)
    /////////////////////////////////////////////////////////////////////
    // int Whatever(char) const noexcept;
};
```

You can determine if it has a *non-overloaded*, *non-static* (public) member function called "*Whatever*" with the specific traits you're interested in. First you'll declare your own (default constructible) struct or class to specify these traits as per the following example (but there's another similar technique you can also rely on and in C++20 or later you can also use an equivalent lambda template instead, which is often more convenient - both techniques will be shown shortly):

```C++
///////////////////////////////////////////////////////
// Note: If targeting C++20 or later, the following
// struct can be replaced with a much more convenient
// lambda template instead. More on this later.
///////////////////////////////////////////////////////
struct HasWhateverTraits
{
    template <TRAITS_FUNCTION_C F>
    bool operator()() const noexcept
    {
        return IsReturnTypeSame_v<F, int> && // Must have "int" return type
               AreArgTypesSame_v<F, std::string_view, float> && // Must have "(std::string_view, float)" arg list
               !IsVariadic_v<F>; // Must not be variadic (can't end with "...")
    }
};
```

Now you can simply pass the class or struct you wish to test for *non-overloaded*, *non-static* (public) member function "*Whatever*" via the 1st template arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*" (so "*Test*" in this case), and your customized "*HasWhateverTraits*" struct (template-based functor) above as the 2nd template arg, which will be used to test for the traits you're interested in:

```C++
constexpr bool classHasNonOverloadedFunctionTraits_Whatever = ClassHasNonOverloadedFunctionTraits_Whatever_v<Test, HasWhateverTraits>;
```

In this case the "*operator()*" (template) member of "*HasWhateverTraits*" will be specialized on the type of *non-overloaded*, *non-static* (public) member function "*&Test::Whatever*" (only if it actually exists), so its "*F*" template arg will resolve to ```int (Test::*)(std::string_view, float) const noexcept``` in this example (the type of "*&Test::Whatever*"). The "*operator()*" (template) member in the above example is then simply testing that function for an "*int*" return type and an argument list matching ```(std::string_view, float)``` (and explicitly no variadic args), but you can test for anything you require here. Contrast this with the templates created by [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function), which always match every component of the function instead (again, see [C++ function types](#cppfunctiontypes)).

Note that your customized class template, "*HasWhateverTraits*" in this example, is simply a functor whose "*operator()*" member is a template. The "*operator()*" member takes the usual TRAITS\_FUNCTION\_C template arg "*F*" used throughout this library (see [Template arg "F"](#templateargf) - the library will specialize "*F*" on the function you're checking for if it exists), but the "*operator()*" member itself takes no actual function parameters (its function parameter list is empty). Note that the functor's class must be default constructible.

As seen in the above example, your "*operator()*" template will normally just invoke whatever traits you're interested in from this library, returning *true* or *false* to indicate if function "*F*" exists or not (again, "*F*" resolves to "*&Test::Whatever*" in this example). In the above example, we're only interested in the function's return type and argument (parameter) types, but it also excludes variadic functions (those ending with "..."), hence we passed ```IsReturnTypeSame_v<F, int>```, ```AreArgTypesSame_v<F, std::string_view, float>``` and ```!IsVariadic_v<F>```. Therefore, when "*HasWhateverTraits*" is passed as the 2nd template arg to "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*" above, its "*operator()*" member will return true if the 1st template arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*" is a class or struct with a *non-overloaded*, *non-static* (public) member function called "*Whatever*" (i.e., the *NAME* arg passed to the *DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS* macro), but only if the return type of "*Whatever*" is declared an "*int*" and the function takes 2 args of type "*std::string\_view*" and *float*" (and exactly those types only - conversions to those types don't factor in because the "*HasWhateverTraits*" struct in this example is relying on *[IsReturnTypeSame\_v](#isreturntypesame_v)\<F, int\>* and *[AreArgTypesSame\_v](#areargtypessame_v)\<F, std::string\_view, float\>* which both perform an *exact* type match - it's also ensuring your function isn't variadic via the call to *\![IsVariadic\_v](#isvariadic_v)\<F\>*). If you wish to loosen the check to support conversions as well, or anything else then you must modify your "*HasWhateverTraits*" struct to do so.

However, there's often a better approach to the above technique. Instead of explicitly testing for the types seen in the above example, you can rely on the following variation instead by using the optional "*UserTypeT*" template arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever*":

```C++
struct HasWhateverTraits
{
    template <TRAITS_FUNCTION_C F, typename CompareF>
    bool operator()() const noexcept
    {
        return ReturnTypesMatch_v<F, CompareF> && // "F" must have same return type as "CompareF" (an "int" in this example)
               ArgTypesMatch_v<F, CompareF> && // "F" must have the same args as "CompareF" ("std::string_view" and "float" in this example)
               IsVariadic_v<F> == IsVariadic_v<CompareF>; // "F" must be variadic only if "CompareF" is variadic
    }
};
```
As in the previous example, you can pass this class when you invoke "*ClassHasNonOverloadedFunctionTraits\_Whatever*" but note that "*operator()*" above has a 2nd template arg called "*CompareF*", unlike the previous example which only had one template arg. This template arg can be optionally passed any type you require which you'll do via the optional "*UserTypeT*" template arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*". It defaults to "*void*" however so if you don't explicitly pass it then the "*operator()*" template of your "*HasWhateverTraits*" must only have one template arg, as seen in the earlier example (though you're also free to declare it with 2 template args so long as the 2nd template arg is assigned a default value). If you explicitly pass a *non-void* template arg however for the "*UserTypeT*" parameter of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*", then the "*operator()*" member of your "*HasWhateverTraits*" class *must* have 2 template args, as seen in the above example. The "*UserTypeT*" parameter (type) you pass to "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*" is then simply passed as the 2nd template arg to your "*operator()*" function but in the above implementation of "*operator()*" we've called it "*CompareF*" instead. That's because in this particular example, we're going to pass a function type for the "*UserTypeT*" parameter of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*", whose traits we expect member function "*Whatever*" to match (i.e., a comparison function). We've therefore given it a more appropriate name than "*UserTypeT*" in the actual "*operator()*" member above, "*CompareF*". You can pass anything you want for the "*UserTypeT*" template parameter however and simply name it accordingly in your "*operator()*" function. In practice however most users will normally use it for the purpose we're using it for in this particular example, so you'll normally want to stick with the name "*CompareF*" or any other similar name you prefer instead.

Note that using this technique (relying on "*UserTypeT*"), we can now check for the presence of member function "*Test::Whatever*" like so:

```C++
constexpr bool classHasNonOverloadedFunctionTraits_Whatever = ClassHasNonOverloadedFunctionTraits_Whatever_v<Test, HasWhateverTraits, int (std::string_view, float) const noexcept>;
```

This call is identical to the previous example except we're now passing our user-defined type via the 3rd template arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*", i.e., the "*UserTypeT*" template arg, which we didn't explicitly pass in the previous example (so it defaulted to "*void*" and was therefore ignored). Because we're now explicitly passing it however, it's no longer "*void*" so your "*operator()*" member must have an extra template arg to accommodate it. Again, we've called this "*CompareF*" in the above example and as you can see, we're passing the function to compare "*F*" with, i.e., ```int (std::string_view, float) const noexcept``` in our call to "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v* (via its "*UserTypeT*" template arg). Our implementation of "*operator()*" now performs the same check as it did in the previous example, only this time it checks the return type, argument types and variadic status of "*Test::Whatever*" against those in the function type that we passed via "*UserTypeT*", again, ```int (std::string_view, float) const noexcept``` (which "*CompareF*" will now be set to). If they're an *exact* match then "*operator()*" returns true as expected or false otherwise. This version of "*operator()*" is therefore identical to the previous example except now it does its job by comparing the traits of "*Test::Whatever*" with those of the function we passed via the "*UserTypeT*" arg (which we named "*CompareF*" in "*operator()*"), instead of the previous example which directly tested "*Test::Whatever*" for those traits instead. Whether you'ld prefer to use the technique of the previous example or the one just described is entirely up to you however (and your particular needs). The example just described does allow you to dynamically change the traits you wish to compare however, by effectively extracting them from a function you pass on-the-fly using the "*UserTypeT*" parameter.

The upshot in both examples above is that the "*HasFunctionTraitsT*" arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*" can be customized to check for a *non-overloaded*, *non-static* (public) member function called "*Whatever* by simply customizing the "*operator()*" template member of "*HasFunctionTraitsT*" to test for "*Whatever*" using any required criteria. The optional "*UserTypeT*" arg of "*ClassHasNonOverloadedFunctionTraits\_Whatever\_v*" can also be used if you wish to pass your own user-defined type to "*operator()*" as well, for any required purpose but typically to support the "*CompareF*" technique just demonstrated. In either case, whether you use the "*UserTypeT* template arg or not, note that as described under [C++ function types](#cppfunctiontypes) but repeated here for your convenience, function types in C++ are always of the following form, where items in square brackets are optional (but see the latter link for details):

<pre><a href="#returntype">ReturnType</a> [<<a href="#callingconvention">CallingConvention</a>>] (<a href="#args">Args</a> [, <a href="#variadic">...</a>]) [<a href="#const">const</a>] [<a href="#volatile">volatile</a>] [<a href="#refqualifiers">&|&&</a>] [<a href="#noexcept">noexcept</a>[(true|false)]]</pre>

You're therefore free to check for any of the above traits by writing a template analogous to "*HasWhateverTraits*" and checking only those requirements you're interested in. To this end, if "*Test::Whatever*" were explicitly declared with *every* possible trait of a C++ function as seen just above, such as the following (just to emphasize the situation):

```C++
class Test
{
public:
    int STDEXT_CC_STDCALL Whatever(std::string_view, float, ...) const volatile & noexcept;
};
```

... then a struct to check for *every* possible function trait in "*Whatever*" above would be declared as follows (but if checking *every* trait then you can just rely on [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) instead - that's its purpose and it's both easier and cleaner):

```C++
struct HasAllTraits
{
    template <TRAITS_FUNCTION_C F>
    bool operator()() const noexcept
    {
        return IsReturnTypeSame_v<F, int> && // Must have "int" return type
               CallingConvention_v<F> == CallingConvention::Stdcall && // Must have the "Stdcall" calling convention (though usually you'll rely on "*DefaultCallingConvention_v*" here instead)
               AreArgTypesSame_v<F, std::string_view, float> && // Must have 2 args "std::string_view" and "float"
               IsVariadic_v<F> && // Must be variadic (must end with "...")
               IsFunctionConst_v<F> && // Must be "const"
               IsFunctionVolatile_v<F> && // Must be "volatile"
               FunctionRefQualifier_v<F> == RefQualifier::LValue && // Must have the ref-qualifier "&" (though rare in practice)
               IsNoexcept_v<F>;
    }
};
```

This struct doesn't rely on the "*UserTypeT*" template arg technique described previously (though it easily can and some may prefer it), but notwithstanding that, in practice you can customize your own version of the above struct by simply copying it into your code and removing any of the conditions that don't matter to you (and renaming the struct to something appropriate for whatever it's checking). For instance, removing the call to [CallingConvention\_v](#callingconvention_v) seen above means the calling convention will no longer be checked so it can be anything (i.e., the calling convention of "*Whatever*" isn't relevant to you). The same applies to the other traits seen above as well. Simply remove those you don't care about and keep the rest (and again, give your struct a meaningful name). Note that for the call to [AreArgTypesSame\_v](#areargtypessame_v) above, you can even replace this with calls to [IsArgTypeSame\_v](#isargtypesame_v) if you wish to drill down into individual arguments (if you don't need to check them all), or perform any other type of argument checks you may require. Otherwise just rely on [AreArgTypesSame\_v](#areargtypessame_v) as seen above.

Finally, as briefly mentioned earlier, if targeting C++20 or later then it's often much more convenient to pass a lambda template instead of the structs seen in the above examples (lambda templates aren't available until C++20). For instance, you can replace the "*HasAllTraits*" struct just above with the following equivalent lambda template instead (which can be conveniently declared at function scope opposed to the "*HasAllTraits*" struct template itself):

```C++
using HasAllTraits = decltype([]<TRAITS_FUNCTION_C F>() noexcept
                              {
                                  return IsReturnTypeSame_v<F, int> && // Must have "int" return type
                                         CallingConvention_v<F> == CallingConvention::Stdcall && // Must have the "Stdcall" calling convention (though usually you'll rely on the "DefaultCallingConvention_v" template here instead or just ignore the calling convention altogether)
                                         AreArgTypesSame_v<F, std::string_view, float> && // Must have 2 args "std::string_view" and "float"
                                         IsVariadic_v<F> && // Must be variadic (must end with "...")
                                         IsFunctionConst_v<F> && // Must be "const"
                                         IsFunctionVolatile_v<F> && // Must be "volatile"
                                         FunctionRefQualifier_v<F> == RefQualifier::LValue && // Must have any ref-qualifier "&" (though rare in practice)
                                         IsNoexcept_v<F>;
                             });

constexpr bool classHasNonOverloadedFunctionTraits_Whatever = ClassHasNonOverloadedFunctionTraits_Whatever_v<Test, HasAllTraits>;
```

Note that you can even just call it on-the-fly if you prefer like so (the following is quite verbose however especially given the long names, so the linefeed after the '=' sign is by design):

```C++
constexpr bool classHasNonOverloadedFunctionTraits_Whatever =
    ClassHasNonOverloadedFunctionTraits_Whatever_v<Test, decltype([]<TRAITS_FUNCTION_C F>() noexcept
                                                                  {
                                                                      return IsReturnTypeSame_v<F, int> && // Must have "int" return type
                                                                             CallingConvention_v<F> == CallingConvention::Stdcall && // Must have the "Stdcall" calling convention (though usually you'll rely on the "DefaultCallingConvention_v" template here instead or just ignore the calling convention altogether)
                                                                             AreArgTypesSame_v<F, std::string_view, float> && // Must have 2 args "std::string_view" and "float"
                                                                             IsVariadic_v<F> && // Must be variadic (must end with "...")
                                                                             IsFunctionConst_v<F> && // Must be "const"
                                                                             IsFunctionVolatile_v<F> && // Must be "volatile"
                                                                             FunctionRefQualifier_v<F> == RefQualifier::LValue && // Must have any ref-qualifier "&" (though rare in practice)
                                                                             IsNoexcept_v<F>;
                                                                   }
                                                                  )>;
```

Behind the scenes, lambda templates like these simply resolve to the equivalent of the (previous) "*HasAllTraits*" struct template (the compiler generates a struct with a template version of "*operator()*"), so it's effectively identical to the "*HasAllTraits*" struct template. It also supports the optional "*UserTypeT*" template arg described earlier. Just add this as an extra template arg to your lambda if you wish, no different than you would for a non-lambda struct.
</blockquote></details>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS"></a><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS</summary>

<blockquote>

```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(NAME)
```

This macro is functionally equivalent to macro [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS](#declare_class_has_non_overloaded_function_traits) just above but targets *static* (public) member functions instead of non-static. See the latter macro for details. The following templates therefore won't detect functions unless they're *static* as well as non-overloaded (since both macros detect non-overloaded functions only). The macro creates the following two templates for this (assuming "*Whatever*" is passed to the macro):

```C++
template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
struct ClassHasNonOverloadedStaticFunctionTraits_Whatever : std::bool_constant<>
{
};

template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
inline constexpr bool ClassHasNonOverloadedStaticFunctionTraits_Whatever_v = ClassHasNonOverloadedStaticFunctionTraits_Whatever<T, HasFunctionTraitsT, UserTypeT>::value;
```

These templates are just the *static* counterparts of the templates targeting *non-static* functions created by [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS](#declare_class_has_non_overloaded_function_traits). Again see the latter macro for details.
</blockquote></details>

<a name="ClassHasOperator_FunctionCall"></a><details><summary>ClassHasOperator_FunctionCall</summary>

<blockquote>

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasOperator_FunctionCall : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasOperator_FunctionCall_v = ClassHasOperator_FunctionCall<T, F>::value;
```

These templates are functionally equivalent to the templates created by the [DECLARE\_CLASS\_HAS\_FUNCTION](#declare_class_has_function) macro but specifically target the function call operator "*operator()*" (so they detect if a class has a *non-static* "*operator()*" member matching the specific type you pass, whether overloaded or not). Unlike the latter macro however, no macro needs to be called to create these templates. They are already declared for you. Other than the names of these templates, they are identical to those created by the latter macro. See this for complete details.
</blockquote></details>

<a name="ClassHasStaticOperator_FunctionCall"></a><details><summary>ClassHasStaticOperator_FunctionCall (C++23 or later)</summary>

<blockquote>

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasStaticOperator_FunctionCall : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasStaticOperator_FunctionCall_v = ClassHasStaticOperator_FunctionCall<T, F>::value;
```

Available in C++23 or later only (since the function call operator was made optionally static in that version), these templates are functionally equivalent to the templates created by the [DECLARE\_CLASS\_HAS\_STATIC\_FUNCTION](#declare_class_has_static_function) macro but specifically target the [static function call operator](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1169r4.html) "*operator()*" (so they detect if a class has a *static* "*operator()*" member matching the specific type you pass, whether overloaded or not). Unlike the latter macro however, no macro needs to be called to create these templates. They are already declared for you. Other than the names of these templates, they are identical to those created by the latter macro. See this for complete details.
</blockquote></details>

<a name="ClassHasNonOverloadeOperator_FunctionCall"></a><details><summary>ClassHasNonOverloadedOperator_FunctionCall</summary>

<blockquote>

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasNonOverloadedOperator_FunctionCall : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasNonOverloadedOperator_FunctionCall_v = ClassHasNonOverloadedOperator_FunctionCall<T, F>::value;
```

These templates are functionally equivalent to the templates created by the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) macro but specifically target the function call operator "*operator()*" (so they detect if a class has a *non-overloaded*, *non-static* "*operator()*" member matching the specific type you pass). Unlike the latter macro however, no macro needs to be called to create these templates. They are already declared for you. Other than the names of these templates, they are identical to those created by the latter macro. See this for complete details.
</details></blockquote>

<a name="ClassHasNonOverloadedStaticOperator_FunctionCall"></a><details><summary>ClassHasNonOverloadedStaticOperator_FunctionCall (C++23 or later)</summary>

<blockquote>

```C++
template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
struct ClassHasNonOverloadedStaticOperator_FunctionCall : std::bool_constant<>
{
};

template <IS_CLASS_C T, TRAITS_FUNCTION_C F>
inline constexpr bool ClassHasNonOverloadedStaticOperator_FunctionCall_v = ClassHasNonOverloadedStaticOperator_FunctionCall<T, F>::value;
```

Available in C++23 or later only (since the function call operator was made optionally static in that version), these templates are functionally equivalent to the templates created by the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_STATIC\_FUNCTION](#declare_class_has_non_overloaded_static_function) macro but specifically target the [static function call operator](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1169r4.html) "*operator()*" (so they detect if a class has a *non-overloaded*, *static* "*operator()*" member matching the specific type you pass). Unlike the latter macro however, no macro needs to be called to create these templates. They are already declared for you. Other than the names of these templates, they are identical to those created by the latter macro. See this for complete details.
</blockquote></details>

<a name="ClassHasNonOverloadedOperatorTraits_FunctionCall"></a><details><summary>ClassHasNonOverloadedOperatorTraits_FunctionCall</summary>

<blockquote>

```C++
template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
struct ClassHasNonOverloadedOperatorTraits_FunctionCall : std::bool_constant<>
{
};

template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
inline constexpr bool ClassHasNonOverloadedOperatorTraits_FunctionCall_v = ClassHasNonOverloadedOperatorTraits_FunctionCall<T, HasFunctionTraitsT, UserTypeT>::value;
```

These templates are functionally equivalent to the templates created by the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION\_TRAITS](#declare_class_has_non_overloaded_function_traits) macro but specifically target the function call operator "*operator()*" (so they detect if a class has a *non-overloaded*, *non-static* "*operator()*" member with the specific traits you pass). Unlike the latter macro however, no macro needs to be called to create these templates. They are already declared for you. Other than the names of these templates, they are identical to those created by the latter macro. See this for complete details.
</blockquote></details>

<a name="ClassHasNonOverloadedStaticOperatorTraits_FunctionCall"></a><details><summary>ClassHasNonOverloadedStaticOperatorTraits_FunctionCall (C++23 or later)</summary>

<blockquote>

```C++
template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
struct ClassHasNonOverloadedStaticOperatorTraits_FunctionCall : std::bool_constant<>
{
};

template <IS_CLASS_C T, IS_CLASS_C HasFunctionTraitsT, typename UserTypeT = void>
REQUIRES_IS_HAS_FUNCTION_TRAITS_C(HasFunctionTraitsT, UserTypeT)
inline constexpr bool ClassHasNonOverloadedStaticOperatorTraits_FunctionCall_v = ClassHasNonOverloadedStaticOperatorTraits_FunctionCall<T, HasFunctionTraitsT, UserTypeT>::value;
```

Available in C++23 or later only (since the function call operator was made optionally static in that version), these templates are functionally equivalent to the templates created by the [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_STATIC\_FUNCTION\_TRAITS](#declare_class_has_non_overloaded_static_function_traits) macro but specifically target the [static function call operator](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1169r4.html) "*operator()*" (so they detect if a class has a *non-overloaded*, *static* "*operator()*" member with the specific traits you pass). Unlike the latter macro however, no macro needs to be called to create these templates. They are already declared for you. Other than the names of these templates, they are identical to those created by the latter macro. See this for complete details.
</blockquote></details>

<a name="DisplayAllFunctionTraits"></a><details><summary>DisplayAllFunctionTraits</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename CharT,
          typename CharTraitsT = std::char_traits<CharT>>
          std::basic_ostream<CharT, CharTraitsT>& DisplayAllFunctionTraits(std::basic_ostream<CharT, CharTraitsT> &stream);
```

Not a traits template (unlike most others), but a helper function template you can use to display all function traits for function "*F*" to the given "*stream*". The traits are displayed in a user-friendly format seen in the example below (all traits in the library that apply to "*F*" are displayed). Note that the function is typically used in a debug or test environment if you wish to inspect all traits for a given function in a user-friendly (human readable) format. This can be useful for different purposes, such as when you simply want to test the library by dumping all traits for a given function at once. Or perhaps you may have a function with a complicated declaration you're having trouble understanding (some can be notoriously complex in C++), since the function's output might make it easier to decipher. Note that the function's output is in English only (the labels seen in the example below are not localized based on the current [std::locale](https://en.cppreference.com/w/cpp/locale/locale)), and the function itself isn't designed to override the format seen below (with your own customized format). This was a design decision since the format will normally serve the needs of most users. If you require a different format then you'll need to roll your own but this would be very rare.

Note that the format of the displayed traits is best demonstrated using an example:
```
// Standard C++ headers
#include <iostream>
#include <string>

////////////////////////////////////////////////////////
// Only file in this repository you need to explicitly
// #include (see "Usage" section earlier)
////////////////////////////////////////////////////////
#include "FunctionTraits.h"

/////////////////////////////////////////////////////////
// Namespace with sample function whose traits you wish
// to display (all of them)
/////////////////////////////////////////////////////////
namespace Test
{
    class SomeClass
    {
    public:
        int DoSomething(const std::basic_string<wchar_t>&,
                        const char*,
                        short int,
                        int,
                        float,
                        long int,
                        double,
                        ...) const volatile && noexcept;
    };
}

int main()
{
    // Everything in the library declared in this namespace
    using namespace StdExt;

    // Above member function whose traits we wish to display
    using F = decltype(&Test::SomeClass::DoSomething);

    //////////////////////////////////////////
    // Display all traits for "F" to "tcout"
    // (or pass any other required stream)
    //////////////////////////////////////////
    DisplayAllFunctionTraits<F>(tcout);

    return 0;
}
```
This will stream the traits for the non-static member function "*Test::SomeClass::DoSomething()*" above to "*tcout*" in the following format (note that the format may vary slightly depending on the function and the target compiler):

      1) Type: int (Test::SomeClass::*)(const std::basic_string<wchar_t>&, const char*, short int, int, float, long int, double, ...) const volatile && noexcept
      2) Classification: Non-static member (class="Test::SomeClass")
      3) Calling convention: cdecl
      4) Return: int
      5) Arguments (7 + variadic):
              1) const std::basic_string<wchar_t>&
              2) const char*
              3) short int
              4) int
              5) float
              6) long int
              7) double
              8) ...
      6) const: true
      7) volatile: true
      8) Ref-qualifier: &&
      9) noexcept: true

Each trait is sequentially numbered as seen (the numbers aren't relevant otherwise), and the "*Arguments*" section is also independently numbered but in argument order (so the numbers *are* relevant - each indicates the 1-based "Nth" arg).

Note that for free functions (which includes static member functions), which don't support "*const*", "*volatile*" or ref-qualifiers, items 6, 7 and 8 above will therefore be removed. Instead, item 9 above, "*noexcept*", will simply be renumbered. The "*Classification*" above will also indicate "*Free or static member*".

Lastly, note that if "*F*" is a functor type (including a lambda type), then the format will be identical to the above except that the "*Classification*" will now indicate "*Functor*" instead of "*Non-static member*", and the type itself in item 1 above will reflect the function type of "*F::operator()*"
</blockquote></details>

<a name="TypeName_v"></a><details><summary>TypeName_v</summary>

<blockquote>

```C++
template <typename T>
inline constexpr tstring_view TypeName_v;
```

Not a template associated with "*FunctionTraits*" per se, but a helper template you can use to return the user-friendly name of any C++ type as a "*tstring\_view*" (more on this shortly). The user-friendly name is normally WYSIWYG, such as "int", "float", etc., though the actual name of some types can vary slightly depending on the target platform (but the names usually match or closely match the actual C++ type name). Just pass the type you're interested in as the template's only template arg. Note however that all helper aliases in the library such as [ArgType\_t](#argtype_t) have a corresponding helper "*Name*" template ([ArgTypeName\_v](#argtypename_v) in the latter case) that simply rely on "*TypeName\_v*" to return the type's user-friendly name (by simply passing the alias itself to "*TypeName\_v*"). You therefore don't have to call "*TypeName\_v*" directly for any of the type aliases in this library since a helper variable template already exists that does this for you (again, one for every alias template in the library, where the name of the variable template returning the type's name is the same as the name of the alias template itself but with the "*\_t*" suffix in the alias' name replaced with "*Name\_v*", e.g., [ArgType\_t](#argtype_t) and [ArgTypeName\_v](#argtypename_v)). The only time you might (typically) need to call "*TypeName\_v*" directly when using "*FunctionTraits*" is when you use [ForEachArg](#foreacharg) as seen in the [Looping through all function arguments](#loopingthroughallfunctionarguments) section above. See the sample code in that section for an example (specifically the call to "*TypeName\_v*" in the "*displayArgType*" lambda of the example).<br/><br/>Note that "*TypeName\_v*" can be passed any C++ type however, not just types associated with "*FunctionTraits*". You can therefore use it for your own purposes whenever you need the user-friendly name of a C++ type as a compile-time string. Note that "*TypeName\_v*" returns a "*tstring\_view*" (in the "*StdExt*" namespace) which always resolves to "*std::string\_view*" on non-Microsoft platforms, and on Microsoft platforms, to "*std::wstring\_view*" when compiling for Unicode (usually the case - strings are normally stored in UTF-16 in modern-day Windows), or "*std::string\_view*" otherwise (when compiling for ANSI but this is very rare these days). Note that the returned "*tstring\_view*" is always guaranteed to be null-terminated[^7], so you can pass its "*data()*" member to a function that expects this for instance. Its "*size()*" member does not include the null-terminator however, as would normally be expected.
</blockquote></details>

<a name="ModuleUsage"></a>
## Module support in C++20 or later (experimental)

Note that you can also optionally use the module version of "*FunctionTraits*" if you're targeting C++20 or later (modules aren't available in C++ before that). Module support is still experimental however since C++ modules are still relatively new at this writing and not all compilers support them yet (or fully support them). GCC for instance has a compiler [bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109679) at this writing that until fixed, results in a compiler error (note that this bug has now been flagged as fixed but no release has been made available by GCC at this writing). The Intel compiler may also fail to compile depending on the platform (informally confirmed by Intel [here](https://community.intel.com/t5/Intel-oneAPI-Data-Parallel-C/Moving-existing-code-with-modules-to-Intel-c/m-p/1550610/thread-id/3448)), even though its documentation states that modules are partially supported at this writing. Only Microsoft and Clang therefore support the module version of "*FunctionTraits*" for now though GCC and Intel may be fixed before too long (and this documentation will be updated as soon as they are).

In addition to supporting modules in C++20, the "*std*" library is also available as a module in C++23 (see the official document for this feature [here](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf)). This occurs in the form of "*import std*" and/or "*import std.compat*" statements (see latter link). Support for this is also very new and not available on most compilers yet (nor supported by CMake at this writing - see CMake [Limitations](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#limitations)). Microsoft recently started supporting it however as of Visual Studio 2022 V17.5 (see [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170#enable-modules-in-the-microsoft-c-compiler)), though this version doesn't yet support mixing #include statements from the "*std*" library and the "*import std*" statement in C++23. Until available you must therefore always exclusively rely on one or the other only (i.e., for now you can't mix #includes from the "*std*" library and the "*import std*" statement in C++23 - Microsoft is on record about this). Also note that not all supported compilers may #define the C++23 feature macro [\_\_cpp_lib_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) yet, indicating that "*import std*" and "*import std.compat*" are available. Where they are available however they're still considered experimental so "*FunctionTraits*" won't rely on them without your explicit permission first (via the transitional *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* macro described shortly).

As a result of all these issues, module support for "*FunctionTraits*" is therefore considered experimental until all compilers fully support both C++ modules (a C++20 feature), and (though less critically), the importing of the standard library in C++23 (again, via "*import std*" and "*import std.compat*"). The instructions below therefore reflect the incomplete state of module support in all compilers at this writing, and are therefore (unavoidably) longer they should be (for now). Don't be discouraged however, as they're much simpler than they first appear.

Note that the following instructions will be condensed and simplified in a future release however, once all compilers are fully compliant with the C++20 and C++23 standard (in their support of modules). For now, because they're not fully compliant, the instructions below need to deal with the situation accordingly, such as the *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* constant in 4 below, which is a transitional constant only. It will be removed in a future release.

To use the module version of "FunctionTraits":

1. Ensure your project is set up to handle C++ modules if it's not already by default (again, since compiler support for modules is still evolving so isn't available out-of-the-box in some compilers). How to do this depends on the target compiler and your build environment (though C++20 or greater is always required), as each platform has its own unique way (such as the *-fmodules-ts* option in GCC - see GCC link just below). The details for each compiler are beyond the scope of this documentation but the following official (module) links can help get you started (though you'll likely need to do additional research if you're not already familiar with the process). Note that the "*CMake*" link below however does provide additional version details about GCC, Microsoft and Clang:
    1. [C++ modules (official specification)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1103r3.pdf)
    2. [GCC](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Modules.html) (not currently supported by "*FunctionTraits*" however due to this GCC [bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109679), now flagged as fixed but no release has been made available by GCC at this writing)
    3. [Microsoft](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170)
    4. [Clang](https://clang.llvm.org/docs/StandardCPlusPlusModules.html)
    5. [Intel](https://www.intel.com/content/www/us/en/developer/articles/technical/c20-features-supported-by-intel-cpp-compiler.html) (search page for [P1103R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1103r3.pdf) - no other Intel docs on modules can be found at this writing)
    6. [CMake](https://www.kitware.com/import-cmake-the-experiment-is-over/)
2. Add the primary module interface files "*FunctionTraits.cppm*" and "*CompilerVersions.cppm*" from this repository to your project, which builds the modules "*FunctionTraits*" and "*CompilerVersions*" respectively (corresponding to "*FunctionTraits.h*" and "*CompilerVersions.h*" described in the [Usage](#usage) section earlier - both ".h" files are still required however as each module simply defers to its ".h" file to implement the module). Ensure your build environment is set up to process these "*.cppm*" files as C++ modules if it doesn't handle it by default (based on the extension for instance). Consult the docs for your specific platform (such as changing the extension to "*.ixx*" on Microsoft platforms - more on this later). Note that you're free to change the "*.cppm*" extension to whatever you require, assuming "*.cppm*" doesn't suffice (again, more on this later). You can then import each module wherever you need it (read up on C++ modules for details), either using an "*import*" statement as would normally be expected (normally just "*import FunctionTraits*" - this is how modules are normally imported in C++), or by continuing to #include the header itself, normally just *#include "FunctionTraits.h"* (again, as described in the [Usage](#usage) section earlier). In the latter case (when you *#include "FunctionTraits.h"*), this will not only import "*FunctionTraits*" for you as described in 3 below, but also has the benefit of making all public macros in "*FunctionTraits.h*" available as well (should you require any of them), the reason you would choose to *#include "FunctionTraits.h*" in the module version instead of "*import FunctionTraits*" directly (more on this shortly).
3. #define the constant *STDEXT\_USE\_MODULES* when you build your project (add this to your project's build settings). Doing so changes the behavior of both "*FunctionTraits.h*" and "*CompilerVersions.h*" (again, "*FunctionTraits.h*" automatically #includes "*CompilerVersions.h*" - see [Usage](#usage) section), so that instead of declaring all C++ declarations as each header normally would (when *STDEXT\_USE\_MODULES* isn't #defined), each header simply imports the module instead (e.g., *#include "FunctionTraits.h"* simply issues an "*import FunctionTraits*" statement). All other C++ declarations in the file are then preprocessed out except for (public) macros, since they're not exported by C++ modules (so when required, the files that #define them must still be #included in the usual C++ way). Therefore, by #including "*FunctionTraits.h*", you're effectively just creating an "*import FunctionTraits*" statement (as well as "*import CompilerVersions*"), but also #defining all public macros in the header as well (including those in "*CompilerVersions.h*" - more on these macros shortly). All other declarations in the file are preprocessed out as noted (they're not needed because the "*import FunctionTraits*" statement itself makes them available). Note that if *STDEXT\_USE\_MODULES* isn't #defined however (though you normally should #define it), then each header is #included in the usual C++ way (no "*import*" statement will exist and all declarations in the header are declared normally), which effectively defeats the purpose of using modules (unless you manually code your own "*import FunctionTraits*" statement which can safely coexist with *#include "FunctionTraits.h"* if you use both but there's no reason to normally). Note that if you don't use any of the macros in "*FunctionTraits.h*" or "*CompilerVersions.h*" in your code however (again, more on these macros shortly), then you can simply apply your own "*import*" statement as usual, which is normally the natural way to do it (and #including the header instead is even arguably misleading since it will appear to the casual reader of your code that it's just a traditional header when it's actually applying an "*import*" statement instead, as just described). #including either ".h" file however to pick up the "*import*" statement instead of directly applying "*import*" yourself has the benefit of #defining all macros as well should you ever need any of them (though you're still free to directly import the module yourself at your discretion - redundant "*import*" statements are harmless if you directly code your own "*import FunctionTraits*" statement _**and**_ *#include "FunctionTraits.h"* as well, since the latter also applies its own "*import FunctionTraits*" statement as described). Note that macros aren't documented in this README file however since the focus of this documentation is on "*FunctionTraits*" itself (the subject of this GitHub repository). Both "*FunctionTraits.h*" and "*CompilerVersions.h*" contain various support macros however, such as the *TRAITS\_FUNCTION\_C* macro in "*FunctionTraits.h*" (see this in the [Helper templates](#helpertemplates) section earlier), and the "*Compiler Identification Macros*" and "*C++ Version Macros*" in "*CompilerVersions.h*" (a complete set of macros allowing you to determine which compiler and version of C++ is running - see these in "*CompilerVersions.h*" for details). "*FunctionTraits*" itself utilizes these macros for its own internal needs as required but end-users may wish to use them as well (for their own purposes). Lastly, note that as described in the [Usage](#usage) section earlier, #including "*FunctionTraits.h*" automatically picks up everything in "*CompilerVersions.h*" as well since the latter is a dependency, and this behavior is also extended to the module version (though if you directly "*import FunctionTraits*" instead of #include "*FunctionTraits.h*", it automatically imports module "*CompilerVersions*" as well but none of the macros in "*CompilersVersions.h*" will be available, again, since macros aren't exported by C++ modules - if you require any of the macros in "*CompilersVersions.h*" then you must *#include "FunctionTraits.h"* instead, or alternatively just *#include "CompilersVersions.h"* directly).
4. If targeting C++23 or later (the following constant is ignored otherwise), *and* the C++23 [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) feature is supported by your compiler (read on), optionally #define the constant *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* when you build your project (add this to your project's build settings). This constant is transitional only as a temporary substitute for the C++23 feature macro [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) (used to indicate that "*import std*" and "*import std.compat*" are supported). If *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* is #defined (in C++23 or later), then the "*FunctionTraits*" library will use an [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) statement to pick up all its internal dependencies from the "*std*" library instead of #including each individual "*std*" header it depends on. This is normally recommended in C++23 or later since it's much faster to rely on [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) than to #include each required header from the "*std*" library (the days of doing so will likely become a thing of the past). Note that if you #define *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* then it's assumed that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) is in fact supported on your platform or cryptic compilers errors will likely occur (if targeting C++23 or later - again, the constant is ignored otherwise). Also note that as described earlier, if targeting Microsoft platforms then your own project must also currently rely on it everywhere since you can't (currently) mix headers from the "*std*" library and "*import std*" (until Microsoft corrects this). In any case, please see [Footnotes](#footnotes) for module versioning information for each supported compiler (which includes version support information for "*import std*" and "*import std.compat*"). Note that *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* will be removed in a future release however once all supported compilers fully support the [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) feature macro (which will then be used instead). For now the latter macro either isn't #defined on all supported platforms at this writing (most don't support "*import std*" and "*import std.compat*" yet), or if it is #defined such as in recent versions of MSVC, "*FunctionTraits*" doesn't rely on it yet (for the reasons described but see the comments preceding the check for *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* in "*FunctionTraits.h*" for complete details). Instead, if you wish for "*FunctionTraits*" to rely on [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) then you must grant explicit permission by #defining *STDEXT\_IMPORT\_STD\_EXPERIMENTAL*. The C++ feature macro [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) itself is completely ignored by the "*FunctionTraits*" library in this release.

Finally, note that you're free to rename each "*.cppm*" extension to whatever you require on your particular platform. On Microsoft platforms for instance you can (and likely will) change it to "*.ixx*" (the default module extension for Microsoft), though you can optionally maintain the "*.cppm*" extension on Microsoft as well if you wish (or change it to anything else). If you do so however then in your Microsoft Visual Studio project you'll need to set each "*.cppm*" file's "*Compile As*" property to "*Compile as C++ Module Code (/interface)*" (you'll find this in the file's properties under "*Configuration Properties -> C/C++ -> Advanced*" - see the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) command line switch in Microsoft's documentation for details). "*CMake*" developers may wish to consult the following Microsoft [issue](https://gitlab.kitware.com/cmake/cmake/-/issues/25643) however (as described in the latter link, Visual Studio solutions built with CMake don't add the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) property as they normally should). In any case, note that each "*.cppm*" file in the "*FunctionTraits*" library is platform neutral so the extension isn't relevant, so long as your compiler is instructed to identify it as a module (if it doesn't recognize it by default). The C++ standard itself doesn't address the situation so while the extension "*.cppm*" has become widely adopted, there's no universal convention (and other extensions do exist).

<a name="WhyChooseThisLibrary"></a>
## Why choose this library
In a nutshell, because it's extremely easy to use, with syntax that's consistently very clean (when relying on [Technique 2 of 2](#technique2of2) as most normally will), it handles all mainstream function types you wish to pass as template args (see [here](#templateargf)), it has a very small footprint (once you ignore the many comments in "*FunctionTraits.h*"), and it may be the most complete function traits library available on the web at this writing (based on attempts to find an equivalent library with calling convention support in particular). The library offers almost every mainstream trait supported by current versions of C++ (additional traits may be added once reflection is available in C++26), and should normally meet the requirements of most users looking for a function traits library (cleanly and reliably).

Note that "*FunctionTraits*" is also significantly smaller than the Boost version ("*boost::callable\_traits*"), which consists of a bloated (excessive) number of files and roughly twice the amount of code at this writing (largely due to a needlessly complex design, no disrespect intended). "*FunctionTraits*" still provides the same features for all intents and purposes as well as some additional features such as templates for retrieving user-friendly type names (including the names of function arguments), iterating all argument types, detecting the existence of member functions, etc. It also supports all (mainstream) calling conventions as emphasized, and can therefore detect functions regardless of calling convention, which isn't officially available in "*boost::callable\_traits*" (and even if activated there though it's not officially documented or supported, it has several other caveats as described earlier in this document).

Also note that the Boost implementation has several shortcomings that could be improved for a better experience. For instance, its "*args*" template (or "*args\_t_*" helper template), the template that developers typically use the most in practice (to access a function's argument types and count), is less user-friendly than it should be, requiring additional overhead to access the individual argument types and/or argument count (since "*args*" itself just returns the collection of argument types - no Boost-specific helper templates exist to drill into it leaving most users having to make additional calls to [std::tuple\_element](https://en.cppreference.com/w/cpp/utility/tuple_element) and/or [std::tuple\_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size), or rely on their own wrappers to consolidate the calls). Additional helper templates to simplify the process would have been beneficial (though the code does seem to have at least one internal template the author may have been experimenting with towards this goal but the situation is murky). Moreover, note that "*args*" also has a few strange quirks, such as including the function's member class type itself when handling non-static member functions (always at index zero in "*args*"), even though the class type has nothing to do with a function's arguments (so it's presence in "*args*" is strange). It also means the function's first argument type is at index 1 in "*args*" for non-static member functions but index zero for free functions, which forces developers to have to subtract 1 from [std::tuple\_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size) to retrieve the number of arguments for non-static member functions, but not for free functions. This needlessly complicates things for any code that may be handling both non-static member functions and free functions (when retrieving the number of arguments or for any other purpose). The behavior also overlaps with the purpose of template "*qualified_class_of*", whose main purpose is to return a non-static member function's class type (the same type that template "*args*" stores at index zero), and even template "*class\_of*", rendering all these templates somewhat redundant and a bit confusing (and "*args*" also confusingly handles member *data* pointer types as well, which really has nothing to do with its intended purpose of handling function arguments). Note that by contrast, "*FunctionTraits*" provides clean and easy-to-use helper templates out-of-the-box (no need to explicitly call [std::tuple\_element](https://en.cppreference.com/w/cpp/utility/tuple_element) or [std::tuple\_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size)), and no such oddities or redundant behavior (or behavior unrelated to the task of handling functions only).

Lastly, note that "*boost::callable\_traits*" does support the experimental "*transaction\_safe*" keyword, but "*FunctionTraits*" doesn't by design. Since this keyword isn't in the official C++ standard (most have never likely heard of it), and it's questionable if it ever will be (it was first floated in 2015), I've deferred its inclusion until it's actually implemented, if ever. Very few users will be impacted by its absence and including it in "*FunctionTraits*" can likely be done in less than a day based on my review of the situation.

The upshot is that "*FunctionTraits*" is generally more complete than all other similar libraries (and easier to use than Boost at times), primarily due to both its (mainstream) calling convention support (so it can detect any function regardless of calling convention), as well as some additional features not available in any other similar library (some of which may not even be viable in those libraries without calling convention support). Very little could be added at this stage that would benefit most users and would usually require improvements to the C++ standard itself to accommodate (again, such features may be attainable once reflection becomes available in C++26).

<a name="DesignConsiderations"></a>
## "FunctionTraits" design considerations (for those who care - not required reading)
Note that "*FunctionTraits*" is not a canonical C++ "traits" class that would likely be considered for inclusion in the C++ standard itself. While it could be with some fairly easy-to-implement changes and (rarely-used) additions (read on), it wasn't designed for this. It was designed for practical, real-world use instead. For instance, it supports most mainstream calling conventions as emphasized earlier which isn't currently addressed in the C++ standard itself. Also note that template arg "*F*" of "*FunctionTraits*" and its helper templates isn't confined to pure C++ function types. As described earlier in this document, you can also pass pointers to functions, references to functions, references to pointers to functions, and functors (in all cases referring to their actual C++ types, not any actual instances). Note that references to non-static member function pointers aren't supported however because they're not supported by C++ itself.

While I'm not a member of the C++ committee, it seems very likely that a "canonical" implementation for inclusion in the standard itself likely wouldn't (or might not) address certain features, like calling conventions since they don't currently exist in the C++ standard as described earlier (unless it's ever added at some point). It's also unlikely to support all the variations of pointers and reference types (and/or possibly functors) that "*FunctionTraits*" handles. In the real world however function types include calling conventions so they need to be detected by a function traits library (not just the default calling convention), and programmers are also often working with raw function types or pointers to functions or functors (or references to functions or references to pointers to functions though these are typically encountered less frequently). For non-static member functions in particular, pointers to member functions are the norm. In fact, for non-static member functions, almost nobody in the real world ever works with their actual (non-pointer) function type. I've been programming in C++ for decades and don't recall a single time I've ever worked with the actual type of one. You work with pointers to non-static member functions instead when you need to, such as **void (YourClass::*)()**. Even for free functions (which for our purposes also includes static member functions), since the name of a free function decays into a pointer to the function in most expressions, you often find yourself working with a pointer to such functions, not the actual function type (though you do work with the actual function type sometimes, unlike for non-static member functions which is extremely rare in the real world). Supporting pointers and references to functions and even references to pointers to functions therefore just makes things easier even if some consider it unclean (I don't). A practical traits struct should just make it very easy to use without having to manually remove things using "*std::remove\_pointer*" and/or "*std::remove\_reference*" first (which would be required if such a traits struct was designed to handle pure C++ function types only, not pointers and references to function types or even references to pointers). It's just easier and more flexible to use it this way (syntactically cleaner). Some would argue it's not a clean approach but I disagree. It may not be consistent with how things are normally done in the standard itself (again, I suspect it might handle raw function types only), but it better deals with the real-world needs of most developers IMHO (so it's "clean" because its syntax is designed to cleanly support that).

<a name="Footnotes"></a>
### Footnotes
[^1]: **_GCC minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: GCC V10.2 or later
    2. Module (*\*.cppm*) version  of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Not currently compiling at this writing due to a GCC [bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109679) (flagged as fixed in the latter link but not yet released at this writing).

     Note that GCC compatible compilers are also supported based on the presence of the #defined constant \_\_GNUC\_\_
[^2]: **_Microsoft Visual C++ minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: Microsoft Visual C++ V19.16 or later for [Read traits](#readtraits) (normally installed with Visual Studio 2017 V15.9 or later), or V19.20 or later for [Write traits](#writetraits) (normally installed with Visual Studio 2019 or later). Note that [Write traits](#writetraits) are unavailable in Visual Studio 2017 releases of VC++ due to compiler bugs in those versions.
    2. Module (*\*.cppm*) version of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Microsoft Visual C++ V19.31 or later (normally installed with Visual Studio 2022 V17.1 or later - see [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170#enable-modules-in-the-microsoft-c-compiler)). Note that if using CMake then it has its own requirements (toolset 14.34 or later provided with Visual Studio 2022 V17.4 or later - see [here](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#compiler-support)).

    Note that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in C++23 is supported by Microsoft Visual C++ V19.35 or later (normally installed with Visual Studio 2022 V17.5 or later - see [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170#enable-modules-in-the-microsoft-c-compiler)), so the transitional *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* macro described in [Module support in C++20 or later](#moduleusage) is ignored in earlier versions
[^3]: **_Clang minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: Clang V11.0 or later
    2. Module (*\*.cppm*) version of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Clang V16.0 or later

     Note that [Microsoft Visual C++ compatibility mode](https://clang.llvm.org/docs/MSVCCompatibility.html) is also supported. Please note that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in C++23 is not supported by Clang yet (at this writing), so the *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* macro described in [Module support in C++20 or later](#moduleusage) shouldn't be #defined until it is (until then a compiler error will occur if targeting C++23 or later).

[^4]: **_Intel oneAPI DPC++/C++ minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: Intel oneAPI DPC++/C++ V2021.4.0 or later
    2. Module (*\*.cppm*) version of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Listed as partially supported by Intel [here](https://www.intel.com/content/www/us/en/developer/articles/technical/c20-features-supported-by-intel-cpp-compiler.html) (search for "*Modules: Merging Modules*"), but not yet working for unknown reasons (informally confirmed by Intel [here](https://community.intel.com/t5/Intel-oneAPI-Data-Parallel-C/Moving-existing-code-with-modules-to-Intel-c/m-p/1550610/thread-id/3448)). Note that even when it is finally working, if using CMake then note that it doesn't natively support module dependency scanning for Intel at this writing (see CMake [Compiler Support](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#compiler-support) for modules).

     Note that [Microsoft Visual C++ compatibility mode](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2024-0/microsoft-compatibility.html) is also supported. Please note that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in C++23 is not supported by Intel yet (at this writing), so the *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* macro described in [Module support in C++20 or later](#moduleusage) shouldn't be #defined until it is (until then a compiler error will occur if targeting C++23 or later).

[^5]: **_GCC and Clang compiler bugs affecting the member function detection templates:_**

    Note that when using the library's member function detection templates (see [here](#determiningifamemberfunctionexists)), both Clang and GCC have compiler bugs at this writing that can potentially occur when targeting non-public functions:

    1. *GCC*: Compilation in Clang may fail if you attempt to detect a function that exists and is declared "*private*". The issue exists due to the following [GCC bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=116639).
    2. *Clang*: Two known issues exist:
        1. Compilation in GCC may fail if you attempt to detect a function that exists and is declared "*protected*" (but strangely works if *private*). The issue exists due to the following [Clang bug](https://github.com/llvm/llvm-project/issues/103895).
        2. The library's member function detection templates might incorrectly return true if you attempt to detect a *private*, *overloaded* base class function via a derived class (and the function exists). The issue occurs due to the following [Clang bug](https://github.com/llvm/llvm-project/issues/107629).

    Note that the extent of these bugs remain unknown so they might potentially manifest themselves in other similar non-public function scenarios until corrected by GCC and Clang. Users of the member function detection templates should be aware of the situation.

[^6]: **_Microsoft design flaw affecting the member function detection templates:_**

    Due to a (flawed) design issue with how Microsoft handles the default calling convention when compiling for x86 only (the issue doesn't exist when compiling for other architectures, most notably x64, or when using non-Microsoft compilers), then the templates created by both the [DECLARE\_HAS\_FUNCTION](#declare_has_function) and [DECLARE\_CLASS\_HAS\_NON\_OVERLOADED\_FUNCTION](#declare_class_has_non_overloaded_function) macros might erroneously return the wrong (true or false) value if ***both*** of the following hold for template arg "*F*" of these templates:

    1. Template arg "*F"* is a "*raw*" (native) C++ function type (i.e., it satisfies [std::is\_function](https://en.cppreference.com/w/cpp/types/is_function) - see [C++ function types](#cppfunctiontypes)). If a *non-static* member function pointer instead then no issue occurs.
    2. Template arg "*F"* includes an *explicitly* declared calling convention that's set to the compiler default (via the /Gd (\_\_cdecl), /Gr (\_\_fastcall), /Gv (\_\_vectorcall) or /Gz (\_\_stdcall) compiler options available in Visual Studio under your project's properties - see "*Configuration Properties -> C/C++ -> Calling Convention*" - defaults to /Gd (\_\_cdecl) as described [here](https://learn.microsoft.com/en-us/cpp/build/reference/gd-gr-gv-gz-calling-convention?view=msvc-170)). If the calling convention is *implicitly* included however (i.e., none is explicitly set in template arg "*F*", which is usually the case in practice), then no issue occurs.

    The details are beyond the scope of this documentation but the upshot is that if template arg "*F*" is a "*raw*" (native) C++ function type (opposed to a *non-static* member function pointer type which has no such issue) *AND* you explicitly set the calling convention in template arg "*F*" (and specifically to the default calling convention in effect via the [/Gd, /Gr, /Gv or /Gz](https://learn.microsoft.com/en-us/cpp/build/reference/gd-gr-gv-gz-calling-convention?view=msvc-170) Microsoft compiler options), then these specific templates might erroneously return true when false should be returned or vice versa (though the condition under which it occurs will usually be rare in practice). If you wish to completely avoid the situation, then either always pass a *non-static* member function pointer type for "*F*" instead of a "*raw*" native function type (see [C++ function types](#cppfunctiontypes) for details on "*raw*" function types), or don't explicitly include the *default* calling convention in template arg "*F*" (if you wish to pass a "*raw*" C++ function type for "*F*" instead of a non-static member function pointer). In either case the issue is always avoided.

[^7]: **_"TypeName\_v" null termination:_**
    1. Note that the string returned by [TypeName\_v](#typename_v) is guaranteed to be null-terminated only if the internal constant *TYPENAME\_V\_DONT\_MINIMIZE\_REQD\_SPACE* isn't #defined. It's not #defined by default so null-termination is guaranteed by default. See this constant in "*FunctionTraits.h*" for full details (not documented in this README file since few users will ever need to #define it).

[^8]: **_Removing write traits if not required (optional):_**
    1. Note that most users don't use [Write traits](#writetraits) (most use [Read traits](#readtraits) only), so those who wish to remove them can optionally do so by #defining the preprocessor constant *REMOVE\_FUNCTION\_WRITE\_TRAITS*. Doing so will preprocess out all [Write traits](#writetraits) so they no longer exist, and therefore eliminate the overhead of compiling them (though the overhead is normally negligible). Please note however that the member function detection templates (see [here](#determiningifamemberfunctionexists)) sometimes depend on write traits to carry out their work so *REMOVE\_FUNCTION\_WRITE\_TRAITS* should not be #defined if you rely on them. Compiler errors may occur otherwise (since the required write traits will have been preprocessed out). Lastly, please note that [Write traits](#writetraits) are always unavailable in Microsoft Visual Studio 2017 releases of VC++ regardless of this constant due to compiler bugs in those versions (also noted in footnote 2 above).
