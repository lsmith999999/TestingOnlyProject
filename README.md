# FunctionTraits
## C++ function traits library (single header-only) for retrieving info about any function (arg types, arg count, return type, etc.). Clean and easy-to-use, the most "complete" implementation on the web.

See [here](https://godbolt.org/z/KjGvef1x3) for a complete working example (demo that displays all available traits for various sample functions - for those who want to get right to the code). Also see usage examples further below.

"FunctionTraits" is a lightweight C++ traits struct (template) that allows you to quickly and easily determine the traits of any function at compile-time, such as argument types, number of arguments, return type, etc. (for C++17 and later). It's a natural extension to the C++ standard itself, whose [<type_traits>](https://en.cppreference.com/w/cpp/header/type_traits) header offers almost no support for handling function traits, save for [std::is_function](https://en.cppreference.com/w/cpp/types/is_function) and [std::is_member_function_pointer](https://en.cppreference.com/w/cpp/types/is_member_function_pointer) (and one or two other borderline cases). It's also a "complete" implementation in the sense that it handles (detects) all mainstream function syntax unlike any other implementation you'll normally find at this writing, which usually fail to address one or more issues (including the Boost implementation - read on). Many (actually most) of these implementations are just hacks or solutions quickly developed on-the-fly to answer someone's question on [stackoverflow.com](https://stackoverflow.com/) for instance. Only a small handful are more complete and far fewer are reasonably complete, though still missing at least one or two features, in particular calling convention support (again, read on).

Some have even written articles on the subject but still don't address various issues. Some of these issues are obvious (or should be), such as failing to detect functions declared as "*noexcept*" (part of a function's type since C++17), or "*const*" and/or "*volatile*" non-static member functions (usually "*const*" in practice), or variadic functions (those whose arguments end with "..."), or function pointers with cv-qualifiers (the pointers themselves are "*const*" and/or "*volatile*"), among other things (often resulting in a lot of cryptic compiler errors).

Some are less obvious, like failing to detect non-static [Member functions with ref-qualifiers](https://en.cppreference.com/w/cpp/language/member_functions#Member_functions_with_ref-qualifier) (rarely used but still required for completeness), or much more prominently (and important usually), functions with calling conventions other than the default (usually "*cdecl*"), which isn't addressed in the C++ standard itself (and a bit more tricky to implement than it first appears as far as creating a functions traits class is concerned). Without (mainstream) calling convention support however, all functions in the entire Microsoft Windows API would fail to be detected for instance, since it's almost entirely based on the "*stdcall*" calling convention. Only the one from Boost ("*boost::callable_traits*") made a half-hearted attempt to handle calling conventions but it's not officially active (more on this later), it targets the Microsoft C++ compiler only (and in a limited capacity only - again, more on this later), and the feature is even discouraged by the author's own internal comments (referring to it to as "*likely broken*" and "*too much work to maintain*"). For all intents and purposes it's therefore not supported and no other implementation I'm aware of addresses calling conventions at all, even in the (very) few other (mostly) "complete" solutions for handling function traits that are out there (that I've ever come across). Again, the result is usually a lot of cryptic compiler errors or possibly incorrect behavior (including in the Boost version since its limited support for calling conventions is officially inactive).

"*FunctionTraits*" does address all these issues however. It provides traits info about every (mainstream) aspect of any function within the limitations of what C++ currently supports (or easily supports), and allows you to change every trait as well, where applicable (including function parameter types, which will be rarely used by most but no other implementation I've seen at this writing supports it, including the one from Boost).

<a name="CppFunctionTypes"></a>
#### C++ function types
Note that function types in C++ are always of the following general form, where items in square brackets are optional, This is what [std::is_function](https://en.cppreference.com/w/cpp/types/is_function) in the C++ standard itself looks for to determine if a type qualifies as a function (though the function's "*CallingConvention*" seen below isn't addressed in the C++ standard itself but is part of its type anyway in most implementation). The following notation is somewhat imprecise but sufficient for most purposes (the precise legalese is beyond the scope of this documentation):

    ReturnType [<CallingConvention>] (Args [, ...]) [const] [volatile] [&|&&] [noexcept[(true|false)]]
    
Based on the above format, you can therefore obtain and/or change (where applicable) every component seen above by simply passing your function's type to the templates in this library (and other useful features also exist, like the ability to [determine if a member function exists](#determiningifamemberfunctionexists)):

1. `ReturnType` - Function's return type and/or its name as a WYSIWYG string (std::basic_string_view)
2. `CallingConvention` - The function's calling convention ("*cdecl*", "*stdcall*", etc.). As briefly described previously, all other implementations I'm aware of can detect functions with the default calling convention only (usually "*cdecl*" or possibly "*thiscall*" for non-static member functions depending on the platform). Only "*callable_traits*" from Boost was partially designed to handle calling conventions but for the Microsoft C++ compiler only, though again, it's officially unavailable (the documentation makes this clear and doesn't even describe how to turn it on), its support is limited even if it is turned on (by intrepid developers who wish to inspect the code - it won't handle certain syntax, it won't compile for 64 bit builds, etc.), and its use is discouraged by the author's own internal comments (see [Why choose this library](#whychoosethislibrary)). Note that calling conventions aren't addressed in the C++ standard itself and are platform specific. As a result, there are numerous calling conventions in the wild that are specific to one platform only and it's therefore inherently difficult for any function traits library to support them all. For now "*FunctionTraits*" currently supports the most mainstream calling conventions which will usually meet the needs of most users ("*cdecl*", "*stdcall*", "*fastcall*", "*vectorcall*", though GCC itself doesn't support "*vectorcall*", "*thiscall*", and on Clang and Intel platforms only, "*regcall*").
3. `Args` - The type of any (non-variadic) function argument (based on its zero-based index in the parameter list), and/or its name as a WYSIWYG string (std::basic_string_view). The number of (non-variadic) arguments in the function is also available (formally called "*arity*"). You can also obtain the [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) containing all arguments and even iterate them using the [ForEachArg](#foreacharg) function template (though rarely required by most in practice - targeting specific function arguments by their index is far more common). See [Looping through all function arguments](#loopingthroughallfunctionarguments) for details.
4. `...` - Whether a function has variadic args (last arg is "...")
5. `const` - Whether a non-static member function is declared with the "*const*" cv-qualifier
6. `volatile` - Whether a non-static member function is declared with the "*volatile*" cv-qualifier
7. `&|&&` - Whether a non-static member function is declared with the "*&*" or "*&&*" reference qualifiers (mutually exclusive)
8. `noexcept` - Whether a function is declared with "*noexcept*"
9. `class/struct type` - While not part of the native C++ type format seen above, the class/struct associated with a non-static member function (including functors), and/or its name as a WYSIWYG string (std::basic_string_view)
10. `Function's classification` - Whether the function is a free function (which includes static member functions), a non-static member function, or a functor

The above covers almost all traits most programmers will ever be interested in. Other possible traits such as whether a function is a "*static*" member function, whether a non-static member function is "*virtual*" (and/or the "*override*" keyword is in effect), etc., are not available in this release, normally due to limitations in the language itself (either impossible or difficult/unwieldy to implement in current releases of C++). They may be available in a future version however if C++ better supports it.

<a name="Usage"></a>
## Usage (C++17 and later: GCC[^1], Microsoft[^2], Clang[^3] and Intel[^4] compilers only)
To use "*FunctionTraits*", simply add both "*TypeTraits.h*" and "*CompilerVersions.h*" to your code and then *#include "TypeTraits.h"* wherever you require it (an experimental module version is also now available - see [Module support in C++20 or later](#moduleusage)). All code is declared in namespace "*StdExt*". Note that you need not explicitly *#include "CompilerVersions.h"* unless you wish to use it independently of "*TypeTraits.h*", since "*TypeTraits.h*" itself #includes it as a dependency ("*CompilerVersions.h*" simply declares various #defined constants used to identify the version of C++ you're using, and a few other compiler-related declarations - you're free to use these in your own code as well if you wish). The struct (template) "*FunctionTraits*" is then immediately available for use (see [Technique 1 of 2](#technique1of2) below), though you'll normally rely on its [Helper templates](#helpertemplates) instead (see [Technique 2 of 2](#technique2of2) below). Note that both files above have no platform-specific dependencies except when targeting Microsoft, where the native Microsoft header *<tchar.h>* is expected to be in the usual #include search path (and it normally will be on Microsoft platforms). Otherwise they rely on the C++ standard headers only which are therefore (also) expected to be in the usual search path on your platform.

<a name="TemplateArgF"></a>
#### Template arg "F"
Note that template arg "*F*" is the first (and usually only) template arg of "*FunctionTraits*" and all its [Helper templates](#helpertemplates), and refers to the function's type which can be any of the following:

1. Free functions which also includes static member functions, in either case referring to the function's actual C++ type (which are always considered to be free functions - non-static member functions are always handled by 4 or 5 below).
2. Pointers and references to free functions
3. References to pointers to free functions
4. Pointers to non-static member functions
5. References to pointers to non-static member functions (note that C++ doesn't support references to non-static member functions, only pointers)
6. Non-overloaded functors (i.e., functors or references to functors with a single "*operator()*" member only, otherwise which overload to target becomes ambiguous - if present then you'll need to target the specific overload you're interested in using 4 or 5 above instead). Includes lambdas as well (just syntactic sugar for creating functors on-the-fly). Simply apply "*decltype*" to your lambda to retrieve its compiler-generated class type (officially known as its "*closure type*"), which you can then pass to "*FunctionTraits*" or any of its [Helper templates](#helpertemplates). Please note however that generic lambdas are _not_ supported using this technique for technical reasons beyond the scope of this documentation (their class type can't be passed for "*F*"). They can still be accessed via 4 and 5 above however using some unconventional syntax described in "*TypeTraits.h*" itself. Search the latter file for "_Example 3 (generic lambda)_" (quotes not included), which you'll find in the comments preceding the "*FunctionTraits*" specialization for functors (the comments for this example also provides additional technical details on this issue).

Note that the code also incorporates [concepts](https://en.cppreference.com/w/cpp/language/constraints) when targeting C++20 or later or "*static_assert*" for C++17 (again, earlier versions aren't supported). In either case this will trap invalid function types with cleaner error messages at compile-time (if anything other than the above is passed for "*F*").

Once you've #included "*TypeTraits.h*", there are two ways to use the "*FunctionTraits*" class. You can either use "*FunctionTraits*" directly like so (but with verbose syntax that [Technique 2 Of 2](#technique2of2) below eliminates - most will normally always rely on it so feel free to jump there now if you wish):

<a name="Technique1Of2"></a>
## Technique 1 of 2 - Using "FunctionTraits" directly (not usually recommended)

```C++
// Only file you need to explicitly #include (see "Usage" section just above)
#include "TypeTraits.h"

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

Alternatively, instead of using "*FunctionTraits*" directly ([Technique 1 of 2](#technique1of2) above), you can rely on the second technique just below instead, which is normally much cleaner (and you should normally use it). As seen in the first technique above, relying on "*FunctionTraits*" directly can result in verbose syntax. For instance, due to the syntax of C++ itself, accessing the type of a given function arg using the "*Args*" member (template) of "*FunctionTraits*" is ugly because you have to apply both the "*typename*" and "*template*" keywords, as well as access the "*Type*" member (alias) of "*Args*" itself (see "*SomeFuncArg3Type_t*" alias in the [Technique 1 of 2](#technique1of2) example). A helper template to eliminate all the messy syntax therefore exists not only for the latter example ([ArgType_t](#argtype_t)), but for every member of "*FunctionTraits*". Therefore, instead of relying on "*FunctionTraits*" directly as seen in the above examples, you can rely on the [Helper templates](#helpertemplates) instead. They're easier and cleaner, making the job of extracting or modifying a function's traits a breeze.

Note that two sets of helper templates exist, one where each template takes a "*FunctionTraits*" template arg, and a corresponding set taking a function type template arg instead (just pass your function's type "*F*" as described [here](#templateargf)). The latter (second) set of helper templates simply create a "*FunctionTraits*" for you from the function type you pass (by passing "*F*" itself as the template arg to "*FunctionTraits*"), and then defer to the corresponding "*FunctionTraits*" helper template in the first set. The "*FunctionTraits*" helper templates (the first set) are rarely used directly in practice however so they're not documented in this README file. Only the [Helper templates](#helpertemplates) taking a function template arg "*F*" (the second set) are therefore documented below. The two sets are functionally identical however other than their respective template args ("*FunctionTraits*" and "*F*"), and template names (the first set is normally named the same as the second but just adds the prefix "*FunctionTraits*" to the name), so the documentation below effectively applies to both sets. Most users will exclusively rely on the set taking template arg "*F*" however (so all remaining documentation below specifically targets it), though you can always quickly inspect the code itself if you ever want to use the "*FunctionTraits*" helper templates directly (since the second set taking template arg "*F*" just defers to them as described). Note that the "*FunctionTraits*" helper templates are just thin wrappers around the members of struct "*FunctionTraits*" itself, which inherits all its members from its base classes.

Here's the same example seen in [Technique 1 of 2](#technique1of2) above but using these easier [Helper templates](#helpertemplates) instead (the second set taking template arg "*F*"):

```C++
// Only file you need to explicitly #include (see "Usage" section earlier)
#include "TypeTraits.h"

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

<a name="DeterminingIfAMemberFunctionExists"></a>
## Determining if a member function exists

Determining if a member function with a specific signature exists in an arbitrary class is a very common requirement for many users. If you need to determine whether a function, say, ```int Whatever(std::string_view, float) const noexcept``` exists in an arbitrary class, then you'll rely on one of six possible macros in the library to do so. While relying on macros may seem a dirty technique to experienced developers, and six of them to choose from no less, the language itself doesn't leave any choice and the situation isn't as bad as it first seems.

First, each macro mererly declares a C++ (struct) template targeting the function you're interested in by name (a "*std::bool_constant*" derivative that resolves to "*std::true_type*" if the function exists or "*std::false_type*" otherwise - a *bool* helper variable is also declared that just defers to it), and C++ provides no other way to do it besides using macros at this writing (which do nothing more than provide the necessary boilerplate code to create the necessary templates). You'll therefore be relying on the declared templates themselves to do the actual work, not on the macros, which just declare the templates as noted. Again, no other way to do it currently exists. Until C++ supports reflection, which will be available in C++26 (and which will hopefully provide improved techniques), a function name can't be passed as a compile-time argument in C++ so it needs to be hardcoded into the templates used to do the work. The six macros simply declare slightly different flavors of these templates targeting the particular function you're interested in, such as "*Whatever*" above. You'll normally just apply one of these macros for your particular needs however (whichever flavor you require).

Second, the six macros are really comprised of just two complementay sets of three macros each, where the first set targets non-static member functions and the second set targets *static* member functions. The two sets are effectively identical otherwise. There are therefore really just three macros to choose from each serving slightly different needs, so you just need to choose the one you require (from the three) and then select either the static or non-static version (based on the function you're targeting). The usage of the static and non-static macros is the same otherwise. These macros are briefly described in the table just below with examples in the last column (note that the table is somewhat lengthy so you'll need to rely on the horizontal scrollbar beneath it). For now the table provides a brief introduction only but most users can start using these macros and the templates they create after reading the examples in this table. Consult the macros themselves for complete details (see links in the first column of the table).

### Member function detection macros

<table>
  <thead>
    <tr>
      <th>Macro</th>
      <th>Templates created by macro (implementation details omitted or obscured)</th>
      <!--Trailing spaces required in following to prevent
          column from being too narrow prior to user clicking
          our example <details> element to open them (creating
          an ergonomically bad experience). Dirty technique but
          couldn't find any work-around for GitHub ".md" files.
          Using arbitrary number of spaces here (workable for
          our needs - pinning down an accurate number a pain). -->
      <th style="text-align:left;">Example&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td valign="top"><b><a href="#declare_class_has_function">DECLARE_CLASS_HAS_FUNCTION(NAME)</a></b><br><br>Use when you need to check for an <i>exact</i> type match of the <i>non-static</i> member function given by the <i>NAME</i> (macro) arg. Just pass the target class as the 1st arg to the templates created by this macro (see next column), and the function type itself as the 2nd arg, such as:<br><br><pre>int (std::string_view, float) const noexcept</pre>The function you're targeting (*NAME*) must have this <i>exact</i> function type for a match to occur. See <i>Example</i> column for details.</td>
      <td valign="top"><pre><b>template &lt;IS_CLASS_C T, IS_FUNCTION_C U&gt;<br>struct ClassHasFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, IS_FUNCTION_C U&gt;<br>inline constexpr bool ClassHasFunction_##NAME##_v;</b></pre>First template above inherits from "<i>std::true_type</i>" or "<i>std::false_type</i>" accordingly (if the <i>non-static</i> member function given by the macro's NAME arg exists in the class/struct passed as the template's 1st arg, and exactly matches the function type given by the 2nd arg). Second template is just a bool helper variable you'll usually rely on instead (that defers to the first template). See <i>Example</i> column for details.</td>
      <td><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;TypeTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>///////////////////////////////////////////////////////////////<br>// Class you wish to check for <i>non-static</i> function &quot;Whatever&quot;<br>// with the exact signature seen below<br>///////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    int Whatever(std::string_view, float) const noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasFunction_Whatever_v</i>&quot; called below<br>// which is set to true if the <i>non-static</i> function &quot;<i>Whatever</i>&quot;<br>// exists in the class you pass it (via the 1st template arg),<br>// and exactly matches the function type you pass it (via the <br>// 2nd template arg). False returned otherwise.<br>/////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a <i>non-static</i> function called &quot;<i>Whatever</i>&quot; with the<br>    // exact type being passed via the 2nd template arg,<br>    // &quot;<i>int (std::string_view, float) const noexcept</i>&quot; in this<br>    // case.<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasFunction_Whatever = ClassHasFunction_Whatever_v&lt;Test, int (std::string_view, float) const noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr>
      <td valign="top"><b><a href="#declare_class_has_static_function">DECLARE_CLASS_HAS_STATIC_FUNCTION(NAME)</a></b><br><br>Identical to macro just above but targets a <i>static</i> function instead of non-static. See above macro for details.</td>
      <td valign="top"><pre><b>template &lt;IS_CLASS_C T, IS_FREE_FUNCTION_C U&gt;<br>struct ClassHasStaticFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, IS_FREE_FUNCTION_C U&gt;<br>inline constexpr bool ClassHasStaticFunction_##NAME##_v;</b></pre>Functionally equivalent to templates on the row just above but targets a <i>static</i> function instead of non-static. See templates on row just above for details.</td>
      <td><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;TypeTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>///////////////////////////////////////////////////////////<br>// Class you wish to check for <i>static</i> function &quot;<i>Whatever</i>&quot;<br>// with the exact signature seen below<br>///////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    static int Whatever(std::string_view, float) noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasStaticFunction_Whatever_v</i>&quot; called<br>// below which is set to true if the <i>static</i> function &quot;<i>Whatever</i>&quot;<br>// exists in the class you pass it (via the 1st template arg),<br>// and exactly matches the function type you pass it (via the<br>// 2nd template arg). False returned otherwise.<br>/////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_STATIC_FUNCTION(Whatever)<br><br>int main()<br>{<br>    //////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template<br>    // arg) has a <i>static</i> function called &quot;<i>Whatever</i>&quot; with <br>    // the exact type being passed via the 2nd template <br>    // arg, &quot;<i>int (std::string_view, float) noexcept</i>&quot; in<br>    // this case.<br>    //////////////////////////////////////////////////////<br>    constexpr bool classHasStaticFunction_Whatever = ClassHasStaticFunction_Whatever_v&lt;Test, int (std::string_view, float) noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr>
      <td valign="top"><b><a href="#declare_class_has_non_overloaded_function">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)</a></b><br><br>Functionally equivalent to macro <i>DECLARE_CLASS_HAS_FUNCTION</i> above (see first row), but used when you need to target a <i>non-overloaded</i> function instead. That is, the templates created by the latter macro will check for a function's existence whether it's overloaded or not, while the templates created by the following macro will only detect functions that are not overloaded. The templates are functionally identical otherwise. See <i>Example</i> column for details.</td>
      <td valign="top"><pre><b>template &lt;IS_CLASS_C T, IS_FUNCTION_C U&gt;<br>struct ClassHasNonOverloadedFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, IS_FUNCTION_C U&gt;<br>inline constexpr bool ClassHasNonOverloadedFunction_##NAME##_v;</b></pre>Functionally equivalent to the templates created by the <i>DECLARE_CLASS_HAS_FUNCTION</i> macro on the first row above but unlike those templates, the templates just above will only inherit from "<i>std::true_type</i>" if the target function is not overloaded (unlike the templates created by the latter macro which detect the target function even if overloaded). See <i>Example</i> column for details.</td>
      <td><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;TypeTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>///////////////////////////////////////////////////////////<br>// Class you wish to check for non-overloaded, non-static<br>// function &quot;<i>Whatever</i>&quot; with the exact signature seen below<br>//////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    int Whatever(std::string_view, float) const noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedFunction_Whatever_v</i>&quot;<br>// called below which is set to true if the non-static function<br>// &quot;<i>Whatever</i>&quot; exists in the class you pass it (via the 1st<br>// template arg), is not overloaded (unlike the example on the<br>// first row above which detects the target function even if<br>// overloaded), and exactly matches the function type you pass<br>// it (via the 2nd template arg). False returned otherwise.<br>/////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ////////////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg) has<br>    // a non-overloaded, non-static function called &quot;Whatever&quot;<br>    // with the exact type being passed via the 2nd template arg,<br>    // &quot;<i>int (std::string_view, float) const noexcept</i>&quot; in this case.<br>    ////////////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedFunction_Whatever = ClassHasNonOverloadedFunction_Whatever_v&lt;Test, int (std::string_view, float) const noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr>
      <td valign="top"><b><a href="#declare_class_has_non_overloaded_static_function">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(NAME)</a></b><br><br>Identical to macro just above but targets a <i>static</i> function instead of non-static. See above macro for details.</td>
      <td valign="top"><pre><b>template &lt;IS_CLASS_C T, IS_FREE_FUNCTION_C U&gt;<br>struct ClassHasNonOverloadedStaticFunction_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, IS_FREE_FUNCTION_C U&gt;<br>inline constexpr bool ClassHasNonOverloadedStaticFunction_##NAME##_v;</b></pre>Functionally equivalent to templates on the row just above but targets a <i>static</i> function instead of non-static. See templates on row just above for details.</td>
      <td><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;TypeTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>///////////////////////////////////////////////////////<br>// Class you wish to check for non-overloaded, <i>static</i><br>// function &quot;<i>Whatever</i>&quot; with the exact signature seen<br>// below<br>///////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    static int Whatever(std::string_view, float) noexcept;<br>};<br><br>///////////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedStaticFunction_Whatever_v</i>&quot;<br>// called below which is set to true if the static function &quot;<i>Whatever</i>&quot;<br>// exists in the class you pass it (via the 1st template arg), is not<br>// overloaded (unlike the example on the second row above which<br>// detects the target function even if overloaded), and exactly<br>// matches the function type you pass it (via the 2nd template arg).<br>// False returned otherwise.<br>///////////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(Whatever)<br><br>int main()<br>{<br>    ////////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a non-overloaded, <i>static</i> function called &quot;<i>Whatever</i>&quot;<br>    // with the exact type being passed via the 2nd template <br>    // arg, &quot;<i>int (std::string_view, float) noexcept<i>&quot; in this<br>    // case.<br>    ////////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedStaticFunction_Whatever = ClassHasNonOverloadedStaticFunction_Whatever_v&lt;Test, int (std::string_view, float) noexcept&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr>
      <td valign="top"><b><a href="#declare_class_has_non_overloaded_member_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(NAME)</a></b><br><br>Use instead of <i>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</i> (see third row above) when the latter macro doesn't suffice because you need to do a more granular check for a non-overloaded function (say, if you want to ignore the function's return type, its <i>noexcept</i> specification, check a specific arg type only, etc.). You have complete control over what to check for opposed to the latter macro which always looks for an exact match of the function type you pass (so instead of passing a function type, you'll effectively pass the exact function traits you wish to check). See the <i>Example</i> column for details.</td>
      <td valign="top"><pre><b>template &lt;IS_CLASS_C T, template &lt;TRAITS_FUNCTION_C&gt; class HasFunctionTraitsT&gt;<br>struct ClassHasNonOverloadedFunctionTraits_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, template &lt;TRAITS_FUNCTION_C&gt; class HasFunctionTraitsT&gt;<br>inline constexpr bool ClassHasNonOverloadedFunctionTraits_##NAME##_v;</b></pre>Similar to the templates created by the DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION macro on row 3 above except the 2nd template arg is no longer the function's type you wish to match, but a customized struct template you&#39;ll write instead. Your template is passed the function's type as its only template arg, and it always inherits from <i>"std::bool_constant"</i> which you'll specialize on true or false to indicate if the function matches your requirements (by testing the function's traits using the templates in this library). This gives you precise control over what to check. See <i>Example</i> column for details.</td>
      <td><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;TypeTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>////////////////////////////////////////////////////////////<br>// Class you wish to check for non-overloaded, <i>non-static</i><br>// function &quot;<i>Whatever</i>&quot; based on the particular traits seen<br>// below (class &quot;<i>HasWhateverTraits</i>&quot;)<br>////////////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    int Whatever(std::string_view, float) const noexcept;<br>};<br><br>//////////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot;<br>// called below which is set to true if the non-static function<br>// &quot;<i>Whatever</i>&quot; exists in the class you pass it (via the 1st template<br>// arg), is not overloaded, and has the traits passed via the 2nd<br>// template arg (&quot;<i>HasWhateverTraits</i>&quot; declared just below in this<br>// example). Variable set to false otherwise.<br>///////////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(Whatever)<br><br>////////////////////////////////////////////////////////////////////<br>// The following struct (template) creates the condition used to<br>// determine if the non-static, non-overloaded function &quot;<i>Whatever</i>&quot;<br>// exists in the call just below (by testing the specific function<br>// traits you're interested in). In this particular example however<br>// the following struct effectively performs the equivalent of the<br>// example on the third row above but does so by checking the<br>// individual traits instead. That is, every trait comprising a<br>// <a href = "#CppFunctionTypes">C++ function type</a> is tested for in the following struct so its<br>// equivalent to the example on the third row (but the individual<br>// traits are tested for here instead of the function's actual C++<br>// type). The struct is therefore passed as the 2nd template arg to<br>// &quot;<i>ClassHasNonOverloadedFunctionTraits_Whatever_v</i>&quot; below, unlike<br>// the example on the third row where the function's actual C++<br>// type is passed instead. The example on the third row would<br>// therefore normally be more suitable in this particular case<br>// since its interface is more natural and less verbose. However,<br>// if you need to customize your check then the example on the<br>// third row can't be used since it always checks the entire<br>// function type for a match (i.e., every component of the function<br>// described under <a href = "#CppFunctionTypes">C++ function types</a>). You'll therefore rely on<br>// the following technique instead and just modify it for your<br>// particular needs (say, if you don't want to check the return<br>// type for instance - you can just remove the call to<br>// &quot;<i>HasReturnType_v</i>&quot; seen in the struct below). The following<br>// technique therefore gives you complete granular control over<br>// what to check compared to the technique used on the third row.<br>// For complete details see<br>// <a href = "#declare_class_has_non_overloaded_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</a><br>////////////////////////////////////////////////////////////////////<br>template &lt;TRAITS_FUNCTION_C F&gt;<br>struct HasWhateverTraits : std::bool_constant&lt;HasReturnType_v&lt;F, int&gt; && // <b><i>Must have "int" return type</i></b><br>                                              CallingConvention_v&lt;F&gt == DefaultCallingConvention_v&lt;false&gt; && // <b><i>Must have the default calling convention</i></b><br>                                              HasArgTypes_v&lt;F, std::string_view, float&gt; && // <b><i>Must have these 2 args (only): "std::string_view" and "float"</i></b><br>                                              !IsVariadic_v&lt;F&gt; && // <b><i>Must not be variadic (can't end with "...")</i></b><br>                                              IsMemberFunctionConst_v&lt;F&gt; && // <b><i>Must be "const"</i></b><br>                                              !IsMemberFunctionVolatile_v&lt;F&gt; && // <b><i>Must not be "volatile"</i></b><br>                                              MemberFunctionRefQualifier_v&lt;F&gt; == RefQualifier::None && // <b><i>Must not have any ref-qualifiers ("&" or "&&" though rare in practice)</i></b><br>                                              IsNoexcept_v&lt;F&gt; // <b><i>Must be "noexcept"</i></b><br>                                             ><br>{<br>};<br><br>int main()<br>{<br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a non-overloaded, non-static function called<br>    // &quot;<i>Whatever</i>&quot; matching the condition just above (passed<br>    // as the 2nd template arg in the following call):<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedFunctionTraits_Whatever = ClassHasNonOverloadedFunctionTraits_Whatever_v&lt;Test, HasWhateverTraits&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
    <tr>
      <td valign="top"><b><a href="#declare_class_has_non_overloaded_static_function_traits">DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(NAME)</a></b><br><br>Identical to macro just above but targets a <i>static</i> function instead of non-static. See above macro for details.</td>
      <td valign="top"><pre><b>template &lt;IS_CLASS_C T, template &lt;TRAITS_FUNCTION_C&gt; class HasFunctionTraitsT&gt;<br>struct ClassHasNonOverloadedStaticFunctionTraits_##NAME : std::bool_constant&lt&gt;<br>{<br>};<br><br>template &lt;IS_CLASS_C T, template &lt;TRAITS_FUNCTION_C&gt; class HasFunctionTraitsT&gt;<br>inline constexpr bool ClassHasNonOverloadedStaticFunctionTraits_##NAME##_v;</b></pre>Functionally equivalent to templates on the row just above but targets a <i>static</i> function instead of non-static. See templates on row just above for details.</td>
      <td><details><summary>Click here to open/close example 👀</summary><pre>// Standard C/C++ headers<br>#include &lt;string_view&gt;<br><br>// Only file in this repository you need to explicitly #include<br>#include &quot;TypeTraits.h&quot;<br><br>// Everything in the library declared in this namespace<br>using namespace StdExt;<br><br>///////////////////////////////////////////////////////<br>// Class you wish to check for non-overloaded, <i>static</i><br>// function &quot;<i>Whatever</i>&quot; based on the particular traits<br>// seen below (class &quot;<i>HasWhateverTraits</i>&quot;)<br>///////////////////////////////////////////////////////<br>class Test<br>{<br>public:<br>    static int Whatever(std::string_view, float) noexcept;<br>};<br><br>/////////////////////////////////////////////////////////////////////////////<br>// Declares template &quot;<i>ClassHasNonOverloadedStaticFunctionTraits_Whatever_v</i>&quot;<br>// called below which is set to true if the <i>static</i> function &quot;<i>Whatever</i>&quot;<br>// exists in the class you pass it (via the 1st template arg), is not<br>// overloaded, and has the traits passed via the 2nd template arg<br>// (&quot;<i>HasWhateverTraits</i>&quot; declared just below in this example). Variable set<br>// to false otherwise.<br>/////////////////////////////////////////////////////////////////////////////<br>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(Whatever)<br><br>///////////////////////////////////////////////////////////////////<br>// See the same struct declaration in the example on the row just<br>// above. The same long comments there apply here as well but the<br>// following targets a static function instead. The check for<br>// those traits that apply to non-static functions only (as seen<br>// in the &quot;<i>HasWhateverTraits</i>&quot; struct for the latter example) have<br>// therefore been removed from the following &quot;<i>HasWhateverTraits</i>&quot;<br>// declaration (namely &quot;<i>const</i>&quot;, &quot;<i>volatile</i>&quot; and the reference<br>// qualifiers &quot;<i>&amp;</i>&quot; and &quot;<i>&amp;&amp;</i>&quot;).<br>///////////////////////////////////////////////////////////////////<br>template &lt;TRAITS_FUNCTION_C F&gt;<br>struct HasWhateverTraits : std::bool_constant&lt;HasReturnType_v&lt;F, int&gt; && // <b><i>Must have "int" return type</i></b><br>                                              CallingConvention_v&lt;F&gt == DefaultCallingConvention_v&lt;false&gt; && // <b><i>Must have the default calling convention</i></b><br>                                              HasArgTypes_v&lt;F, std::string_view, float&gt; && // <b><i>Must have these 2 args (only): "std::string_view" and "float"</i></b><br>                                              !IsVariadic_v&lt;F&gt; && // <b><i>Must not be variadic (can't end with "...")</i></b><br>                                              IsNoexcept_v&lt;F&gt; // <b><i>Must be "noexcept"</i></b><br>                                             ><br>{<br>};<br><br>int main()<br>{<br>    ///////////////////////////////////////////////////////////<br>    // Returns true since class &quot;<i>Test</i>&quot; (the 1st template arg)<br>    // has a non-overloaded, static function called &quot;Whatever&quot;<br>    // matching the condition just above (passed as the 2nd<br>    // template arg in the following call):<br>    ///////////////////////////////////////////////////////////<br>    constexpr bool classHasNonOverloadedStaticFunctionTraits_Whatever_v = ClassHasNonOverloadedStaticFunctionTraits_Whatever_v&lt;Test, HasWhateverTraits&gt;;<br><br>    return 0;<br>}</pre></details></td>
    </tr>
  </tbody>
</table>

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
#include "TypeTraits.h"

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
                                    // "StdExt" when you #include "TypeTraits.h"), always
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

<a name="HelperTemplates"></a>
## Helper templates (complete, alphabetical list)

The following provides a complete (alphabetical) list of all helper templates. Two separate sections exist, the first for [Read traits](#readtraits), allowing you to read any part up a function's type, and the second for [Write traits](#writetraits), allowing you to update any part of a function's type. Note that the first template arg of every template is the function you're targeting, whose name is always "*F*" (see [here](#templateargf) for details). IMPORTANT: When "*F*" is a functor, note that all traits implicitly refer (apply) to the non-static member function "*F::operator()*" unless noted otherwise (so if "*F*" is the type of a lambda for instance then the traits target "*operator()*" of the compiler-generated class for that lambda). It should therefore be understood that whenever "*F*" is a functor and it's cited in the description of each template below, the description is normally referring to member function "*F::operator()*", not class "*F*" itself.

Please note that for all traits, a *TRAITS_FUNCTION_C* [concept](https://en.cppreference.com/w/cpp/language/constraints) (declared in "*TypeTraits.h*") will kick in for illegal values of "*F*" in C++20 or later ("*F*" is declared a *TRAITS_FUNCTION_C* in all templates below), or a "*static_assert*" in C++17 otherwise (again, earlier versions aren't supported). Note that *TRAITS_FUNCTION_C* is just a #defined macro that resolves to the [concept](https://en.cppreference.com/w/cpp/language/constraints) "*StdExt::TraitsFunction_c*" in C++20 or later (ensuring "*F*" is legal), or simply the "*typename*" keyword in C++17 (so in the latter case a "*static_assert*" will be applied for illegal types of "*F*" instead, as noted). For most of the traits, "*F*" is the only template arg (again, see [here](#templateargf)). Only a small handful of templates take additional (template) args which are described on a case-by-case basis below. Lastly, note that the "=" sign is omitted after the template name in each template declaration below. The "=" sign is removed since the implementation isn't normally relevant for most users. You can inspect the actual declaration in "*TypeTraits.h*" if you wish to see it.

Finally, please note that each template below simply wraps the corresponding member of the "*FunctionTraits*" struct itself
(or indirectly targets it). As previously described, you can access "*FunctionTraits*" directly (see this class and its various base classes in "*TypeTraits.h*" - you can access all public members), but the helper templates below normally make it unnecessary . You can also access any of its own helper templates _not_ documented in this README file (each takes a "*FunctionTraits*" template arg as described in [Technique 2 of 2](#technique2of2)). The following helper templates however taking a function template arg "*F*" instead are normally much easier (again, as described in [Technique 2 of 2](#technique2of2)). The syntax for accessing "*FunctionTraits*" directly (or using any of the helper templates taking a "*FunctionTraits*" template arg) is therefore not shown in this document (just the earlier examples in [Technique 1 of 2](#technique1of2) only). You can simply review the public members of "*FunctionTraits*" inherited from its base classes in "*TypeTraits.h*" in the unlikely event you ever need to work with them directly (or alternatively review the helper templates in "*TypeTraits.h*" taking a "*FunctionTraits*" template arg). However, every public member of "*FunctionTraits*" inherited from its base classes has a helper template taking a function template arg "*F*" as seen in each template declaration below, including some additional helper templates not available when accessing "*FunctionTraits*" directly. These helper templates should therefore normally be relied on since they're easier and cleaner, so the following documentation is normally all you ever need to consult.

<a name="ReadTraits"></a>
### _Read traits_

<a name="ArgCount_v"></a><details><summary>ArgCount_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr std::size_t ArgCount_v;
```

***"*std::size_t*" variable storing the number of arguments in "*F*" *not* including variadic arguments if any. Will be zero if function "*F*" has no (non-variadic) args. Note that this count is formally called "*arity*" but this variable is given a more user-friendly name.<br /><br /><ins>IMPORTANT</ins>:<br />Please note that if you wish to check if a function's argument list is completely empty, then inspecting this helper template for zero (0) is not sufficient, since it may return zero but still contain variadic args. To check for a completely empty argument list, call [IsEmptyArgList_v](#isemptyarglist_v) instead.***</details></blockquote>

<a name="ArgType_t"></a><details><summary>ArgType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t I
          bool AssertIfInvalid = true>
using ArgType_t;
```

Type alias for the type of the "*I*th" arg in function "*F*", where "*I*" is in the range 0 to the number of (non-variadic) arguments in "*F*" minus 1 (see [ArgCount_v](#argcount_v) just above). Pass "*I*" as the (zero-based) 2nd template arg. Note that if "*I*" is greater than or equal to the number of args in "*F*" (again, see [ArgCount_v](#argcount_v) just above), then a "*static_assert*" will trigger if "*AssertIfInvalid*" is true (the default), or "*void*" will be returned otherwise (note that "*void*" function arg types are invalid in C++ so if returned then it's guaranteed that "*I*" isn't valid). Note that if "*F*" has no non-variadic args whatsoever then all values of "*I*" are invalid even if passing zero (so it's always guaranteed that either a "*static_assert*" will trigger or "*void*" will result, as controlled by "*AssertIfInvalid*"). Note that passing false for the "*AssertIfInvalid*" arg is often useful in [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae) situations, where you don't want a "*static_assert*" to trigger when "*I*" is illegal but for template substituation to simply fail. For instance, when using the [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS](#declare_class_has_non_overloaded_function_traits) macro (see macro for details), if the "*HasTraitsT*" arg you pass to the templates created by this macro needs to call [IsArgTypeSame_v](#isargtypesame_v), which ultimate relies on "*ArgType_t*", you'll normally want to pass false for the "*AssertIfInvalid*" arg of [IsArgTypeSame_v](#isargtypesame_v) (which just forwards it along to "*ArgType_t*"). Otherwise the call to "*HasTraitsT*" might trigger a "*static_assert*" instead of just returning false (again, see [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS](#declare_class_has_non_overloaded_function_traits) for details).</details></blockquote>

<a name="ArgTypeName_v"></a><details><summary>ArgTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t I,
          bool AssertIfInvalid = true>
inline constexpr tstring_view ArgTypeName_v;
```
Same as [ArgType_t](#argtype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring_view*" - see [TypeName_v](#typename_v) for details). A *float* would therefore be (literally) returned as "*float*" for instance (quotes not included). Note that if "*I*" is greater than or equal to the number of args in "*F*", then a "*static_assert*" will trigger if "*AssertIfInvalid*" is true (the default), or "*void*" will be returned otherwise. See this arg in [ArgType_t](#argtype_t) for further details.</details></blockquote>

<a name="ArgTypes_t"></a><details><summary>ArgTypes_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using ArgTypes_t;
```

Type alias for a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) representing all non-variadic argument types in "*F*". Rarely required in practice however since you'll usually rely on [ArgType_t](#argtype_t) or [ArgTypeName_v](#argtypename_v) to retrieve the type of a specific argument (see these above). If you require the [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) that stores all (non-variadic) argument types, then it's typically (usually) because you want to iterate all of them (say, to process the type of every argument in a loop). If you require this, then you can use the [ForEachArg](#foreacharg) helper function (template) further below. See this for details.</details></blockquote>

<a name="CallingConvention_v"></a><details><summary>CallingConvention_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr CallingConvention CallingConvention_v;
```

Calling convention of "*F*" returned as a "*CallingConvention*" enumerator (declared in "*TypeTraits.h*"). Calling convention enumerators at this writing include "*Cdecl*", "*Stdcall*", "*Fastcall*", "*Vectorcall*", "*Thiscall*" (applicable to non-static member functions only) and "*Regcall*". Note that in all cases only those calling conventions that are supported by a given compiler can be returned by "*CallingConvention_v*" (so for instance, "*Vectorcall*" will never be returned on GCC since it doesn't support this calling convention). Also please note that compilers will sometimes change the calling convention declared on your functions to the "*Cdecl*" calling convention depending on the compiler options in effect at the time (in particular when compiling for 64 bits opposed to 32 bits, though some calling conventions like "*Vectorcall*" *are* supported on 64 bits though again, GCC doesn't support "*Vectorcall*"). In this case the calling convention on your function is ignored and "*CallingConvention_v*" will correctly return the "*Cdecl*" calling convention (if that's what the compiler actually uses).</details></blockquote>

<a name="CallingConventionName_v"></a><details><summary>CallingConventionName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view CallingConventionName_v;
```

Same as [CallingConvention_v](#callingconvention_v) just above but returns this as a (WYSIWYG) string (of type "*tstring_view*" - see [TypeName_v](#typename_v) for details). Note that unlike [CallingConvention_v](#callingconvention_v) itself however, these strings are always returned in lowercase ("*cdecl*", "*stdcall*", "*fastcall*", "*vectorcall*", "*thiscall*" or "*regcall*"). </details></blockquote>

<a name="DefaultCallingConvention_v"></a><details><summary>DefaultCallingConvention_v</summary>

<blockquote>

```C++
template <bool IsFreeFuncT>
inline constexpr CallingConvention DefaultCallingConvention_v ;
```

Variable storing the default calling convention. Pass true for "*IsFreeFuncT*" if you require the default calling convention for a free function (which includes static member functions), or false for a non-static member function. Note that the default calling convention refers to the calling convention applied to a function by the compiler when not explicitly specified by the user (developer). Returned as a "*CallingConvention*" enumerator (declared in "*TypeTraits.h*"). Calling convention enumerators at this writing include "*Cdecl*", "*Stdcall*", "*Fastcall*", "*Vectorcall*", "*Thiscall*" (applicable to non-static member functions only) and "*Regcall*". Note that in all cases only those calling conventions that are supported by a given compiler can be returned by "*DefaultCallingConvention_v*" (so for instance, "*Vectorcall*" will never be returned on GCC since it doesn't support this calling convention).</details></blockquote>

<a name="FunctionType_t"></a><details><summary>FunctionType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using FunctionType_t;
```

Type alias identical to "*F*" itself unless "*F*" is a functor (i.e., [IsFunctor_v](#isfunctor_v) returns true), in which case it's a type alias for the "*operator()*" member of the functor (to retrieve the functor type itself in this case see [MemberFunctionClass_t](#memberfunctionclass_t)).</details></blockquote>

<a name="FunctionTypeName_v"></a><details><summary>FunctionTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool FunctionTypeName_v;
```

Same as [FunctionType_t](#functiontype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring_view*" - see [TypeName_v](#typename_v) for details).</details></blockquote>

<a name="IsArgTypeSame_v"></a><details><summary>IsArgTypeSame_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t I,
          typename T,
          bool AssertIfInvalid = true>
inline constexpr bool IsArgTypeSame_v;
```
"*bool*" variable set to "*true*" if the (zero-based) "*Ith*" arg of function "*F*" is the same as the given type "*T*" or false otherwise. This template is just a thin wrapper around [std::is_same_v](https://en.cppreference.com/w/cpp/types/is_same), where the latter template is passed the type of the "*Ith*" arg in function "*F*" and type "*T*". It therefore simply compares the type of the "*Ith*" arg in function "*F*" with type "*T*". This is a common requirement for many users (possibly in conjunction with [IsReturnTypeSame_v](isreturntypesame_v)), so this template provides a convenient wrapper. Note that if "*I*" is greater than or equal to the number of args in "*F*" then a "*static_assert*" will trigger if "*AssertIfInvalid*" is true (the default), or "*void*" will be compared to template arg "*T*" otherwise. In the latter case false is therefore guaranteed to be returned so long as you don't pass "*void*" itself for template arg "*T*". This is usually the case unless you're specifically checking for this situation (an illegal value for "*I*"), since function arg types can't be "*void*" in C++ (so you won't normally pass "*void*" for template arg "*T*" - it will always result in a false return unless "*I*" itself is illegal). See the "*AssertIfInvalid*" arg in [ArgType_t](#argtype_t) for further details on this parameter.</details></blockquote>

<a name="IsEmptyArgList_v"></a><details><summary>IsEmptyArgList_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsEmptyArgList_v;
```
"*bool*" variable set to "*true*" if the function represented by "*F*" has an empty arg list (it has no args whatsoever including variadic args), or "*false*" otherwise. If "*true*" then note that [ArgCount_v](#argcount_v) is guaranteed to return zero (0), and [IsVariadic_v](#isvariadic_v) is guaranteed to return false.<br /><br /><ins>IMPORTANT</ins>:<br />Note that you should rely on this helper to determine if a function's argument list is completely empty opposed to checking the [ArgCount_v](#argcount_v) helper for zero (0), since the latter returns zero only if "*F*" has no non-variadic args. If it has variadic args but no others, i.e., its argument list is "(...)", then the argument list isn't empty even though [ArgCount_v](#argcount_v) returns zero (since it still has variadic args). Caution advised.</details></blockquote>

<a name="IsFreeFunction_v"></a><details><summary>IsFreeFunction_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsFreeFunction_v;
```
"*bool*" variable set to "*true*" if "*F*" is a free function (including static member functions), or "*false*" otherwise.</details></blockquote>

<a name="IsFunctor_v"></a><details><summary>IsFunctor_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsFunctor_v;
```

"*bool*" variable set to "*true*" if "*F*" is a functor (the functor's class/struct was passed for "*F*") or "*false*" otherwise. Note that when true, [IsMemberFunction_v](#ismemberfunction_v) is also guaranteed to be true.</details></blockquote>

<a name="IsMemberFunction_v"></a><details><summary>IsMemberFunction_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsMemberFunction_v;
```

"*bool*" variable set to "*true*" if "*F*" is a non-static member function (including when "*F*" is a functor), or "*false*" otherwise (if you need to specifically check for functors only then see [IsFunctor_v](#isfunctor_v)). Note that you may need to invoke this before accessing the following helper templates. Since the following are applicable to non-static member functions only, if you don't know whether "*F*" is a non-static member function ahead of time (or a functor), then you should normally call "*IsMemberFunction_v*" to determine this first. If it's "*false*" then "*F*" is a free function (which includes static member functions), so a call to any of the following will result in default values being returned that aren't applicable to free functions (so you shouldn't normally invoke them unless you're ok with the default values they return for free functions):<br /><br />- [IsMemberFunctionConst_v](#ismemberfunctionconst_v)<br />- [IsMemberFunctionVolatile_v](#ismemberfunctionvolatile_v)<br />- [MemberFunctionClass_t](#memberfunctionclass_t)<br />- [MemberFunctionClassName_v](#memberfunctionclassname_v)<br />- [MemberFunctionRefQualifier_v](#memberfunctionrefqualifier_v)<br />- [MemberFunctionRefQualifierName_v](#memberfunctionrefqualifiername_v)</details></blockquote>

<a name="IsMemberFunctionConst_v"></a><details><summary>IsMemberFunctionConst_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsMemberFunctionConst_v;
```

"*bool*" variable applicable only if "*F*" is a non-static member function (or a functor). Set to "*true*" if the function has the "*const*" cv-qualifier (it's declared with the "*const*" keyword) or "*false*" otherwise. Always "*false*" for free functions including static member functions (not applicable to either). You may therefore wish to invoke [IsMemberFunction_v](#ismemberfunction_v) to detect if "*F*" is in fact a non-static member function (or functor) before using this trait.</details></blockquote>

<a name="IsMemberFunctionVolatile_v"></a><details><summary>IsMemberFunctionVolatile_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsMemberFunctionVolatile_v;
```

"*bool*" variable applicable only if "*F*" is a non-static member function (or a functor). Set to "*true*" if the function has the "*volatile*" cv-qualifier (its declared with the "*volatile*" keyword) or "*false*" otherwise. Always "*false*" for free functions including static member functions (not applicable to either). You may therefore wish to invoke [IsMemberFunction_v](#ismemberfunction_v) to detect if "*F*" is in fact a non-static member function (or functor) before using this trait.</details></blockquote>

<a name="IsNoexcept_v"></a><details><summary>IsNoexcept_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsNoexcept_v;
```

"*bool*" variable set to "*true*" if "*F*" is declared as "*noexcept*" or "*false*" otherwise (always false if the "*noexcept*" specifier is absent in the function, otherwise if present then it evaluates to "*true*" if no bool expression is present in the "*noexcept*" specifier (the expression has been omitted), or the result of the bool expression otherwise - WYSIWYG).</details></blockquote>

<a name="IsReturnTypeSame_v"></a><details><summary>IsReturnTypeSame_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename T>
inline constexpr bool IsReturnTypeSame_v;
```

"*bool*" variable set to "*true*" if the return type of function "*F*" is the same as the given type "*T*" or false otherwise. This template is just a thin wrapper around [std::is_same_v](https://en.cppreference.com/w/cpp/types/is_same), where the latter template is passed the return type of function "*F*" and type "*T*". It therefore simply compares the return type of function "*F*" with type "*T*". This is sometimes required when testing function signatures for instance (possibly in conjunction with [IsArgTypeSame_v](isargtypesame_v)), so this template provides a convenient wrapper.</details></blockquote>

<a name="IsVariadic_v"></a><details><summary>IsVariadic_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsVariadic_v;
```

"*bool*" variable set to "*true*" if "*F*" is a variadic function (last arg of "*F*" is "...") or "*false*" otherwise.</details></blockquote>

<a name="IsVoidReturnType_v"></a><details><summary>IsVoidReturnType_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr bool IsVoidReturnType_v;
```

"*bool*" variable set to "*true*" if the return type of "*F*" is "*void*" (ignoring cv-qualifiers if any), or "*false*" otherwise. Note that this variable is identical to passing [ReturnType_t](#returntype_t) to [std::is_void](https://en.cppreference.com/w/cpp/types/is_void). The variable simply provides a convenient wrapper.</details></blockquote>

<a name="MemberFunctionClass_t"></a><details><summary>MemberFunctionClass_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionClass_t;
```

If "*F*" is a non-static member function (or a functor), yields a type alias for the type of the class (or struct) that declares that function (same as "*F*" itself if "*F*" is a functor). Always "*void*" otherwise (for free functions including static member functions). You may therefore wish to invoke [IsMemberFunction_v](#ismemberfunction_v) to detect if "*F*" is in fact a non-static member function (or functor) before applying this trait.</details></blockquote>

<a name="MemberFunctionClassName_v"></a><details><summary>MemberFunctionClassName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view MemberFunctionClassName_v;
```

Same as [MemberFunctionClass_t](#memberfunctionclass_t) just above but returns this as a (WYSIWYG) string (of type "*tstring_view*" - see [TypeName_v](#typename_v) for details).</details></blockquote>

<a name="MemberFunctionRefQualifier_v"></a><details><summary>MemberFunctionRefQualifier_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr RefQualifier MemberFunctionRefQualifier_v;
```

"*RefQualifier*" variable, a proprietary enumerator in "*TypeTraits.h*" applicable only if "*F*" is a non-static member function (or a functor). Set to "*RefQualifier::None*" if the function isn't declared with any reference qualifiers (usually the case for non-static member functions in practice, and always the case for free functions since it's not applicable), "*RefQualifier::LValue*" if the function is declared with the "*&*" reference qualifier, or "*RefQualifier::RValue*" if the function is declared with the "*&&*" reference qualifier. Note that you may wish to invoke [IsMemberFunction_v](#ismemberfunction_v) to detect if "*F*" is in fact a non-static member function (or functor) before applying this trait.</details></blockquote>

<a name="MemberFunctionRefQualifierName_v"></a><details><summary>MemberFunctionRefQualifierName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          bool UseAmpersands = true>
inline constexpr tstring_view MemberFunctionRefQualifierName_v;
```

Same as [MemberFunctionRefQualifier_v](#memberfunctionrefqualifier_v) just above but returns this as a (WYSIWYG) string (of type "*tstring_view*" - see [TypeName_v](#typename_v) for details). Note that this template also takes an extra template arg besides function "*F*", a "*bool*" called "*UseAmpersands*", indicating whether the returned string should be returned as "*&*" or "*&&*" (if the function is declared with an "*&*" or "*&&*" reference qualifier respectively), or "*LValue*" or "*RValue*" otherwise. Defaults to "*true*" if not specified (so returns "*&*" or "*&&*" by default). Not applicable however if no reference qualifiers are present ("*None*" is always returned).</details></blockquote>

<a name="ReturnType_t"></a><details><summary>ReturnType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using ReturnType_t;
```

Type alias for the return type of function "*F*".</details></blockquote>

<a name="ReturnTypeName_v"></a><details><summary>ReturnTypeName_v</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
inline constexpr tstring_view ReturnTypeName_v;
```

Same as [ReturnType_t](#returntype_t) just above but returns this as a (WYSIWYG) string (of type "*tstring_view*" - see [TypeName_v](#typename_v) for details). A *float* would therefore be (literally) returned as "*float*" for instance (quotes not included).</details></blockquote>

<a name="TypeName_v"></a><details><summary>TypeName_v</summary>

<blockquote>

```C++
template <typename T>
inline constexpr tstring_view TypeName_v;
```

Not a template associated with "*FunctionTraits*" per se, but a helper template you can use to return the user-friendly name of any C++ type as a "*tstring_view*" (more on this shortly). The user-friendly name is normally WYSIWYG, such as "int", "float", etc., though the actual name of some types can vary slightly depending on the target platform (but the names usually match or closely match the actual C++ type name). Just pass the type you're interested in as the template's only template arg. Note however that all helper aliases above such as [ArgType_t](#argtype_t) have a corresponding helper "*Name*" template ([ArgTypeName_v](#argtypename_v) in the latter case) that simply rely on "*TypeName_v*" to return the type's user-friendly name (by simply passing the alias itself to "*TypeName_v*"). You therefore don't have to call "*TypeName_v*" directly for any of the type aliases in this library since a helper variable template already exists that does this for you (again, one for every alias template above, where the name of the variable template returning the type's name is the same as the name of the alias template itself but with the "*_t*" suffix in the alias' name replaced with "*Name_v*", e.g., [ArgType_t](#argtype_t) and [ArgTypeName_v](#argtypename_v)). The only time you might (typically) need to call "*TypeName_v*" directly when using "*FunctionTraits*" is when you use [ForEachArg](#foreacharg) as seen in the [Looping through all function arguments](#loopingthroughallfunctionarguments) section above. See the sample code in that section for an example (specifically the call to "*TypeName_v*" in the "*displayArgType*" lambda of the example).<br/><br/>Note that "*TypeName_v*" can be passed any C++ type however, not just types associated with "*FunctionTraits*". You can therefore use it for your own purposes whenever you need the user-friendly name of a C++ type as a compile-time string. Note that "*TypeName_v*" returns a "*tstring_view*" (in the "*StdExt*" namespace) which always resolves to "*std::string_view*" on non-Microsoft platforms, and on Microsoft platforms, to "*std::wstring_view*" when compiling for Unicode (usually the case - strings are normally stored in UTF-16 in modern-day Windows), or "*std::string_view*" otherwise (when compiling for ANSI but this is very rare these days). Note that the returned "*tstring_view*" is always guaranteed to be null-terminated[^5], so you can pass its "*data()*" member to a function that expects this for instance. Its "*size()*" member does not include the null-terminator however, as would normally be expected.</details></blockquote>

---
<a name="WriteTraits"></a>
### _Write traits_[^6]

<a name="AddVariadicArgs_t"></a><details><summary>AddVariadicArgs_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using AddVariadicArgs_t;
```

Type alias for "*F*" after adding "..." to the end of its argument list if not already present. Note that the calling convention is also changed to the "*Cdecl*" calling convention for the given compiler. This is the only supported calling convention for variadic functions in this release, but most platforms require this calling convention for variadic functions. It ensures that the calling function (opposed to the called function) pops the stack of arguments after the function is called, which is required by variadic functions. Other calling conventions that also do this are possible though none are currently supported in this release (since none of the currently supported compilers support this - such calling conventions are rare in practice).</details></blockquote>

<a name="AddNoexcept_t"></a><details><summary>AddNoexcept_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using AddNoexcept_t;
```

Type alias for "*F*" after adding "*noexcept*" to "*F*" if not already present.</details></blockquote>

<a name="MemberFunctionAddConst_t"></a><details><summary>MemberFunctionAddConst_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionAddConst_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after adding the "*const*" cv-qualifier to the function if not already present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since "*const*" applies to non-static member functions only).</details></blockquote>

<a name="MemberFunctionAddCV_t"></a><details><summary>MemberFunctionAddCV_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionAddCV_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after adding both the "*const*" AND "*volatile*" cv-qualifiers to the function if not already present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since "*const*" and "*volatile*" apply to non-static member functions only).</details></blockquote>

<a name="MemberFunctionAddLValueReference_t"></a><details><summary>MemberFunctionAddLValueReference_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionAddLValueReference_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after adding the "*&*" reference-qualifier to the function if not already present (replacing the "*&&*" reference-qualifier if present). If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since reference-qualifiers apply to non-static member functions only).</details></blockquote>

<a name="MemberFunctionAddRValueReference_t"></a><details><summary>MemberFunctionAddRValueReference_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionAddRValueReference_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after adding the "*&&*" reference-qualifier to the function if not already present (replacing the "*&*" reference-qualifier if present). If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since reference-qualifiers apply to non-static member functions only).</details></blockquote>

<a name="MemberFunctionAddVolatile_t"></a><details><summary>MemberFunctionAddVolatile_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionAddVolatile_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after adding the "*volatile*" cv-qualifier to the function if not already present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since "*volatile*" applies to non-static member functions only).</details></blockquote>

<a name="MemberFunctionRemoveConst_t"></a><details><summary>MemberFunctionRemoveConst_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionRemoveConst_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after removing the "*const*" cv-qualifier from the function if present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since "*const*" applies to non-static member functions only so will never be present otherwise).</details></blockquote>

<a name="MemberFunctionRemoveCV_t"></a><details><summary>MemberFunctionRemoveCV_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionRemoveCV_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after removing both the "*const*" AND "*volatile*" cv-qualifiers from the function if present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since "*const*" and "*volatile*" apply to non-static member functions only so will never be present otherwise).</details></blockquote>

<a name="MemberFunctionRemoveReference_t"></a><details><summary>MemberFunctionRemoveReference_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionRemoveReference_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after removing the "*&*" or "*&&*" reference-qualifier from the function if present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since reference-qualifiers to non-static member functions only so will never be present otherwise).</details></blockquote>

<a name="MemberFunctionRemoveVolatile_t"></a><details><summary>MemberFunctionRemoveVolatile_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using MemberFunctionRemoveVolatile_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after removing the "*volatile*" cv-qualifier from the function if present. If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since "*volatile*" applies to non-static member functions only so will never be present otherwise).</details></blockquote>

<a name="MemberFunctionReplaceClass_t"></a><details><summary>MemberFunctionReplaceClass_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          IS_CLASS_C NewClassT>
using MemberFunctionReplaceClass_t;
```

If "*F*" is a non-static member function, yields a type alias for "*F*" after replacing the class this function belongs to with "*NewClassT*". If "*F*" is a free function including static member functions, yields "*F*" itself (effectively does nothing since a "class" applies to non-static member functions only so will never be present otherwise - note that due to limitations in C++ itself, replacing the class for static member functions is not supported).</details></blockquote>

<a name="RemoveNoexcept_t"></a><details><summary>RemoveNoexcept_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using RemoveNoexcept_t;
```

Type alias for "*F*" after removing "*noexcept*" from "*F*" if present.</details></blockquote>

<a name="RemoveVariadicArgs_t"></a><details><summary>RemoveVariadicArgs_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F>
using RemoveVariadicArgs_t;
```

If "*F*" is a variadic function (its last arg is "..."), yields a type alias for "*F*" after removing the "..." from the argument list. All non-variadic arguments (if any) remain intact (only the "..." is removed).</details></blockquote>

<a name="ReplaceArgs_t"></a><details><summary>ReplaceArgs_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename... NewArgsT>
using ReplaceArgs_t;
```

Type alias for "*F*" after replacing all its existing non-variadic arguments with the args given by "*NewArgsT*" (a parameter pack of the types that become the new argument list). If none are passed then an empty argument list results instead, though if variadic args are present in "*F*" then they still remain intact (the "..." remains - read on). The resulting alias is identical to "*F*" itself except that the non-variadic arguments in "*F*" are completely replaced with "*NewArgsT*". Note that if "*F*" is a variadic function (its last parameter is "..."), then it remains a variadiac function after the call (the "..." remains in place). If you wish to explicitly add or remove the "..." as well then pass the resulting type to [AddVariadicArgs_t](#addvariadicargs_t) or [RemoveVariadicArgs_t](#removevariadicargs_t) respectively (either before or after the call to "*ReplaceArgs_t*"). Note that if you wish to replace specific arguments instead of all of them, then call [ReplaceNthArg_t](#replacentharg_t) instead. Lastly, you can alternatively use [ReplaceArgsTuple_t](#replaceargstuple_t) instead of "*ReplaceArgs_t*" if you have a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) of types you wish to use for the argument list instead of a parameter pack. [ReplaceArgsTuple_t](#replaceargstuple_t) is identical to "*ReplaceArgs_t*" otherwise (it ultimately defers to it).</details></blockquote>

<a name="ReplaceArgsTuple_t"></a><details><summary>ReplaceArgsTuple_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          TUPLE_C NewArgsTupleT>
using ReplaceArgsTuple_t;
```

Identical to [ReplaceArgs_t](#replaceargs_t) just above except the argument list is passed as a [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) instead of a parameter pack (via the 2nd template arg). The types in the [std::tuple](https://en.cppreference.com/w/cpp/utility/tuple) are therefore used for the resulting argument list. "*ReplaceArgsTuple_t*" is otherwise identical to [ReplaceArgs_t](#replaceargs_t) (it ultimately defers to it).</details></blockquote>

<a name="ReplaceCallingConvention_t"></a><details><summary>ReplaceCallingConvention_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          CallingConvention NewCallingConventionT>
using ReplaceCallingConvention_t;
```

Type alias for "*F*" after replacing its calling convention with the platform-specific calling convention corresponding to "*NewCallingConventionT*" (a "*CallingConvention*" enumerator declared in "*TypeTraits.h*"). For instance, if you pass  "*CallingConvention::FastCall*" then the calling convention for "*F*" is replaced with "*_\_\_attribute\_\_((fastcall))_*" on GCC and Clang, but "*_\_\_fastcall_*" on Microsoft platforms. Note however that the calling convention for variadic functions (those whose last arg is "...") can't be changed in this release. Variadic functions require that the calling function pop the stack to clean up passed arguments and only the "*Cdecl*" calling convention supports that in this release (on all supported compilers at this writing). Attempts to change it are therefore ignored. Note that you also can't change the calling convention to an unsupported calling convention. For instance, passing "*CallingConvention::Thiscall*" for a free function (which includes static member functions) is ignored since the latter calling convention applies to non-static member functions only. Similarly, passing a calling convention that's not supported by a given compiler is also ignored, such as passing "*CallingConvention::VectorCall*" on GCC (since it doesn't support this calling convention).

Lastly, please note that compilers will sometimes change the calling convention declared on your functions to the "*Cdecl*" calling convention depending on the compiler options in effect at the time (in particular when compiling for 64 bits opposed to 32 bits, though some calling conventions such as "*Vectorcall*" *are* supported on 64 bits, assuming the compiler supports that particular calling convention). Therefore, if you specify a calling convention that the compiler changes to "*Cdecl*" based on the compiler options currently in effect, then "*ReplaceCallingConvention_t*" will also ignore your calling convention and apply "*Cdecl*" instead (since that's what the compiler actually uses).</details></blockquote>

<a name="ReplaceNthArg_t"></a><details><summary>ReplaceNthArg_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          std::size_t N,
          typename NewArgT>
using ReplaceNthArg_t;
```

Type alias for "*F*" after replacing its (zero-based) "*N*th" argument with "*NewArgT*". Pass "*N*" via the 2nd template arg (i.e., the zero-based index of the arg you're targeting), and the type you wish to replace it with via the 3rd template arg ("*NewArgT*"). The resulting alias is therefore identical to "*F*" except its "*N*th" argument is replaced by "*NewArgT*" (so passing, say, zero-based "*2*" for "*N*" and "*int*" for "*NewArgT*" would replace the 3rd function argument with an "*int*"). Note that "*N*" must be less than the number of arguments in the function or a "*static_assert*" will occur (new argument types can't be added using this trait, only existing argument types replaced). If you need to replace multiple arguments then recursively call "*ReplaceNthArg_t*" again, passing the result as the "*F*" template arg of "*ReplaceNthArg_t*" as many times as you need to (each time specifying a new "*N*" and "*NewArgT*"). If you wish to replace all arguments at once then call [ReplaceArgs_t](#replaceargs_t) or [ReplaceArgsTuple_t](#replaceargstuple_t) instead. Lastly, note that if "*F*" has variadic arguments (it ends with "..."), then these remain intact. If you need to remove them then call [RemoveVariadicArgs_t](#removevariadicargs_t) before or after the call to "*ReplaceNthArg_t*".</details></blockquote>

<a name="ReplaceReturnType_t"></a><details><summary>ReplaceReturnType_t</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,
          typename NewReturnTypeT>
using ReplaceReturnType_t;
```

Type alias for "*F*" after replacing its return type with "*NewReturnTypeT*".</details></blockquote>

---
<a name="AdditonalHelperTemplates"></a>
### Additional Helper Templates

<details><summary>Function Detection Macros</summary>
<a name="DECLARE_CLASS_HAS_FUNCTION"></a><blockquote><details><summary>DECLARE_CLASS_HAS_FUNCTION</summary>

```C++
#define DECLARE_CLASS_HAS_FUNCTION(NAME)
```

This macro and its cousins below declare templates allowing you to determine if an arbitrary class/struct has a member function with a given name and function type (signature). For a comparative summary of these macros and the templates they create, please see [here](#determiningifamemberfunctionexists). The *DECLARE_CLASS_HAS_FUNCTION* macro specifically targets non-static member functions only, without regard to whether they're overloaded or not. It allows you do to this by creating templates taking the class/struct type you're targeting as the 1st template arg, and the non-static function type (signature) itself as the 2nd template arg, such as ```int (std::string_view, float") const noexcept```. The template will then determine if the passed class/struct (your 1st arg) has a non-static member function that exactly matches the latter type (your 2nd arg). Note that the function name itself, "*Whatever*" in the latter case, is passed to the macro itself and embedded in the template's name. The templates created by the macro then always target functions with that specific name. The *DECLARE_CLASS_HAS_FUNCTION* macro itself is used to declare helper template "*ClassHasFunction_\#\#NAME*" which is used to determine if any class or struct has a non-static member function with the name given by *NAME* (as passed to this macro). The macro simply declares this template and its "*\_v*" (*bool*) helper template you'll normally use instead (read on). Just pass the macro the name of the non-static member function you're interested in via the macro's NAME arg like so (passing function name "*Whatever*" in this example). You'll declare this at namespace scope:

```C++
DECLARE_CLASS_HAS_FUNCTION(Whatever)
```

This declares the following two templates used to check for the function "*Whatever*", where the name of each template always starts with "*ClassHasFunction_*" followed by the *NAME* arg passed to the macro (representing the name of the non-static function you're targeting, "*Whatever*" in this case). Note that while not shown in the following declaration (since the implementations details aren't relevant), the "*ClassHasFunction_Whatever*" template always inherits from "*std::true_type*" or "*std::false_type*" as described below. The 2nd template is just a helper (bool) variable template with the usual "*_v*" suffix that defers to the 1st template in the usual C++ way:

```C++
    template<IS_CLASS_C T, IS_FUNCTION_C U>
    struct ClassHasFunction_Whatever; // Inherits from "std::true_type" or "std::false_type" as described below

    template<IS_CLASS_C T, IS_FUNCTION_C U>
    inline constexpr bool ClassHasFunction_Whatever_v = ClassHasFunction_Whatever<T, U>::value;
```    

These templates (you'll usually use the "*_v*" version just above) can then be used to determine if any class or struct has a non-static function called "*Whatever*" with the exact function type you're interested in. The "*ClassHasFunction_Whatever*" template above always inherits from "*std::true_type*" if so or "*std::false_type*" otherwise. Again, the second template is just the "*bool*" equivalent.

For instance, after calling this macro via *DECLARE_CLASS_HAS_FUNCTION(Whatever)* seen above, you can then check any class or struct to determine if it has a non-static function called "*Whatever*" with the given function type (signature) you're interested in, such as the following:

```C++
/////////////////////////////////////////////////
// Class you wish to check for a non-overloaded
// function "Whatever" with the signature below
/////////////////////////////////////////////////    
class Test
{
public:
    int Whatever(std::string_view, float) const noexcept;
};

////////////////////////////////////////////////////////////
// Returns true since class "Test" (the 1st template arg)
// has a non-static member function called "Whatever" with
// the exact function type being passed via the 2nd
// template arg (so we're passing the actual C++ function
// type matching "Whatever*" above in this case):
////////////////////////////////////////////////////////////
constexpr bool hasMemberFunction_Whatever = ClassHasFunction_Whatever_v<Test, int (std::string_view, float) const noexcept>;
```
The above variable "*hasMemberFunction_Whatever*" will be set to true because class "*Test*" (the 1st template arg passed in the above call) has a non-overloaded, non-static member function called "*Whatever*" (the *NAME* arg passed to this macro), whose type exactly matches the 2nd template arg that was passed. Note that the 2nd template arg is the native C++ function type you're interested in, so "*int (std::string_view, float) const noexcept*" in this case (exactly matching the type of function "*Test::Whatever*"). It's therefore not a function pointer but the actual C++ function type itself (so it must meet the criteria of [std::is_function](https://en.cppreference.com/w/cpp/types/is_function)). Note that "*true*" is returned only if the class or struct you pass for the 1st template arg has a non-static member function named "*Whatever*" with this exact function type. To this end, note that function types in C++ are always of the following general form, where items in square brackets are optional (and again, this is what [std::is_function](https://en.cppreference.com/w/cpp/types/is_function) in the C++ standard itself looks for to determine if a type qualifies as a function without getting into the precise legalese):

    ReturnType [<CallingConvention>] (Args [, ...]) [const] [volatile] [&|&&] [noexcept[(true|false)]]

Note however that the function's (platform-specific) "*CallingConvention*" seen above ("*cdecl*", "*stdcall*", "*thiscall*", etc.), isn't part of the C++ standard itself at this writing but it's considered part of a function's type regardless (by all C++ implementations supported by this library). The above templates therefore expect a function in this specific format for the 2nd template arg "*U*", and will only return true if the 1st template arg (class or struct type "*T*") has a member function called "*Whatever*" that precisely matches type "*U*". This means that every component of the function seen above is tested for in function "*Whatever*" and must match "*U*" exactly for these templates to return true (including the return type, calling convention, etc.). In fact, the function does its job by simply calling [std::is_same_v](https://en.cppreference.com/w/cpp/types/is_same), passing the type of "*T::Whatever*" as its 1st template arg and type "*U*" as the 2nd template arg. The comparison therefore looks for an exact match so caution advised. In the above example for instance, if the return type of "*Test::Whatever*" is changed from "*int*" to, say, "*char*", then the call to "*ClassHasFunction_Whatever_v*" will no longer return true. It will now return false unless you also change "*int*" to "*char*" in its 2nd template arg (so that the return types match again). Again, even the calling conventions of both functions must match or false will be returned.
</details></blockquote>

<a name="DECLARE_CLASS_HAS_STATIC_FUNCTION"></a><blockquote><details><summary>DECLARE_CLASS_HAS_STATIC_FUNCTION</summary>

```C++
#define DECLARE_CLASS_HAS_STATIC_FUNCTION(NAME)
```

This macro is functionally equivalent to macro [DECLARE_CLASS_HAS_FUNCTION](#declare_class_has_function) but targets *static* member functions instead of non-static. See the latter macro for complete details. The following templates therefore won't detect functions unless they're *static*. The macro creates the following two templates for this:

```C++

template <IS_CLASS_C T, IS_FREE_FUNCTION u>
struct ClassHasStaticFunction_Whatever; // Inherits from "std::true_type" or "std::false_type" (see DECLARE_CLASS_HAS_FUNCTION for details)

template<IS_CLASS_C T, IS_FREE_FUNCTION U>
inline constexpr bool ClassHasStaticFunction_Whatever_v = ClassHasedStaticFunction_Whatever<T, U>::value;

```

These templates are just the *static* counterparts of the non-static templates created by [DECLARE_CLASS_HAS_FUNCTION](#declare_class_has_function). Again see the latter macro for complete details.
</details></blockquote>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION"></a><blockquote><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION</summary>

```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(NAME)
```

This macro is very similar to macro [DECLARE_CLASS_HAS_FUNCTION](#declare_class_has_function) except unlike the latter macro, the templates created by *DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION* will only return true if the function you specify is not overloaded. If it's overloaded then false is always returned instead. The templates created by both macros are functionally equivalent otherwise. See [DECLARE_CLASS_HAS_FUNCTION](#declare_class_has_function) for details.

Similar to [DECLARE_CLASS_HAS_FUNCTION](#declare_class_has_function),  *DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION* creates two templates for determining if a non-overloaded member function exists, which are declared as follows:

```C++
DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(Whatever)
```

This declares the following two templates used to check for the non-static function "*Whatever*", where the name of each template always starts with "*ClassHasNonOverloadedFunction_*" followed by the *NAME* arg passed to the macro (representing the name of the member function you're targeting, "*Whatever*" in this case). These are analogous to the two templates created by [DECLARE_CLASS_HAS_FUNCTION](#declare_class_has_function) but which only return true if the function "*Whatever*" exists and is not overloaded (unlike the latter macro whose templates will return true whether "*Whatever*" is overloaded or not):

```C++
template<IS_CLASS_C T, IS_FUNCTION_C U>
struct ClassHasNonOverloadedFunction_Whatever; // Inherits from "std::true_type" or "std::false_type" as described below

template<IS_CLASS_C T, IS_FUNCTION_C U>
inline constexpr bool ClassHasNonOverloadedFunction_Whatever_v = ClassHasNonOverloadedFunction_Whatever<T, U>::value;
```
These templates (you'll usually use the "*_v*" version just above) can then be used to determine if any class or struct has a non-static, non-overloaded member function called "*Whatever*" with the exact function type you're interested in. The "*ClassHasNonOverloadedFunction_Whatever*" template above always inherits from "*std::true_type*" if so or "*std::false_type*" otherwise. Again, the second template is just the "*bool*" equivalent.

For instance, after calling this macro via ```DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION(Whatever)``` seen above, you can then check any class or struct to determine if it has a non-static, non-overloded member function called "*Whatever*" with the given function type (signature) you're interested in, such as the following:

```C++
/////////////////////////////////////////////////
// Class you wish to check for a non-overloaded
// function "Whatever" with the signature below
/////////////////////////////////////////////////    
class Test
{
public:
    int Whatever(std::string_view, float) const noexcept;
};

//////////////////////////////////////////////////////
// Returns true since class "Test" (the 1st template
// arg) has a non-static, non-overloaded member
// function called "Whatever" with the exact function
// type being passed via the 2nd template arg (so
// we're passing the actual C++ function type
// matching "Whatever" above in this case):
//////////////////////////////////////////////////////
constexpr bool classHasNonOverloadedFunction_Whatever = ClassHasNonOverloadedFunction_Whatever_v<Test, int (std::string_view, float) const noexcept>;
```

The above variable "*classHasNonOverloadedFunction_Whatever*" will be set to true because class "*Test*" (the 1st template arg passed in the above call) has a non-overloaded, non-static member function called "*Whatever*" (the *NAME* arg passed to this macro), whose type exactly matches the 2nd template arg that was passed. If you add another "*Whatever*" function however then false will result instead since the function is now overloaded.
</details></blockquote>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION"></a><blockquote><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION</summary>
```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION(NAME)
```
This macro is functionally equivalent to macro [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function) but targets *static* member functions instead of non-static. See the latter macro for complete details. The following templates therefore won't detect functions unless they're *static* as well as non-overloaded (since both macros detect non-overloaded functions only). The macro creates the following two templates for this:

```C++
template <IS_CLASS_C T, IS_FREE_FUNCTION u>
struct ClassHasNonOverloadedStaticFunction_Whatever; // Inherits from "std::true_type" or "std::false_type" (see DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION for details)

template<IS_CLASS_C T, IS_FREE_FUNCTION U>
inline constexpr bool ClassHasNonOverloadedStaticFunction_Whatever_v = ClassHasNonOverloadedStaticFunction_Whatever<T, U>::value;
```

These templates are just the *static* counterparts of the non-static templates created by [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function). Again see the latter macro for details.
</details></blockquote>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS"></a><blockquote><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS</summary>

```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(NAME)
```

This macro and its cousin [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function) just above allow you to determine if a class or struct has a non-overloaded, non-static member function with a given name and function type (signature). Additional details are discussed under [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function) and you should consult this before proceeding further (a comparative summary of these two macros and the templates they create can also be found [here](#determiningifamemberfunctionexists).

As described under [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function), the latter macro creates templates used to perform an *exact* match of the function you're checking for with the function signature you pass (say, ```int (std::string_view, float) const noexcept```. The following macro however, DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS, creates templates allowing you to perform a customized match instead (so instead of passing a function type to compare with, as required by the [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function) templates), you can effectively pass any criteria required, typically to ignore certain components of the function like its return type, its *noexcept* specification, etc.. You can even target a particular parameter only if you don't need to check them all. You have complete control ove the matching process. Therefore, choose [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function) if you wish to check if a non-overloaded, non-static function precisely matches the exact (entire) signature you pass, or *DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS* if you need more fine-grained control.

The DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS macro is called the same way as [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function). You simply pass the name of the non-static member function you're interested in via the macro's NAME arg ("*Whatever*" in the following example). You'll declare this at namespace scope:

```C++
DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS(Whatever)
```

This declares the following two templates, where the name of each template always starts with "*HasNonOverloadedMemberFunctionTraits_*" followed by the *NAME* arg passed to the macro (representing the name of the member function you're targeting, "*Whatever*" in this case). Note that the 1st template inherits from "*std::true_type*" or "*std::false_type*" as described below (note that the template arg for this is ommitted in the declaration below since the implementation details aren't relevant). The 2nd template is just a (bool) helper variable template with the usual "_v" suffix that defers to the 1st template in the usual C++ way:

```C++
template <IS_CLASS_C T, template <TRAITS_FUNCTION_C> class HasFunctionTraitsT>
struct HasNonOverloadedMemberFunctionTraits_Whatever; // Inherits from "std::true_type" or "std::false_type" as described below

template<IS_CLASS_C T, template <TRAITS_FUNCTION_C> class HasFunctionTraitsT>
inline constexpr bool HasNonOverloadedMemberFunctionTraits_Whatever_v = HasNonOverloadedMemberFunctionTraits_Whatever<T, HasFunctionTraitsT>::value;
```

These templates (you'll usually use the "_v" version just above) can then be used to determine if any class or struct pass via the 1st template arg has a non-overloaded member function called "*Whatever*" whose traits match those passed via the 2nd template arg. The struct itself (the first template) always inherits from "*std::true_type*" if so or "*std::false_type*" otherwise. The second template is just the "*bool*" equivalent. Note tht the 2nd arg of both templates is a template (class/struct) in its own right that inherits from "*std::true_type*" if the function matches your criteria or "*std::false_type*" otherwise. For instance, after calling this macro as seen above (passing "*Whatever*" in this case), you can then do the following.

Given any class or struct like so:

```C++
class Test
{
public:
    int Whatever(std::string_view, float) const noexcept;
};
```

You can determine if it has a non-overloaded, non-static member function called "*Whatever*" with the specific traits you're interested in. Just pass the class or struct via the 1st template arg ("*Test*" in this case), and a template used to test for the traits you're interested in via the 2nd arg (described below):

```C++
constexpr bool hasNonOverloadedMemberFunction_Whatever = HasNonOverloadedMemberFunctionTraits_Whatever_v<Test, HasReturnAndArgTypes>;
```

In this case the "*HasReturnAndArgTypes*" arg (template) is declared as follows but all such templates will be modeled the same way (in this example we simply test function "*F*" for an "*int*" return type and an argument list matching "*(std::string_view, float)*" - you can test for anything you wish however, as described below):

```C++
template <TRAITS_FUNCTION_C F>
struct HasReturnAndArgTypes : std::bool_constant<HasReturnType_v<F, int> && // Must have "int" return type
                                                 HasArgTypes_v<F, std::string_view, float> && // Must have "(std::string_view, float)" arg list
                                                 !IsVariadic_v<F>; // Must not be variadic (can't end with "...")
                                                >
{
};
```

Note that the above template (struct) has two requirements (constraints) only:

1. It must take a TRAITS_FUNCTION_C template arg as seen (declared in "*TypeTraits.h*"), which constrains the arg ("*F*") to any function type supported by this library (see [here](#templateargf) for details). Note that TRAITS_FUNCTION_C only applies in C++20 or later however (when concepts were introduced), so it simply resolves to the C++ "*typename*" keyword in C++17 (note that no earlier versions of C++ are supported by this library)
2. The struct must be a "*std::bool_constant*" derivative, meaning it must (directly or indirectly) inherit from "*std::true_type*" or "*std::false_type*", indicating if the traits you're interested in exist or not (details below).
   
As seen in the above example, you can normally just inherit from "*std::bool_constant*", passing it whatever traits you're interested in from this library. In the above example, we're only interested in the function's return type and argument (parameter) list (types), and excluding variadic functions, hence we passed [HasReturnType_v](#hasreturntype_v)<F, int>, [HasArgTypes_v](#hasargtypes_v)<F, std::string_view, float> and [IsVariadic_v](#isvariadic)<F>. Therefore, when "*HasReturnAndArgs*" is passed as the 2nd arg to "*HasNonOverloadedMemberFunctionTraits_Whatever_v*", the latter template will return true if its 1st template arg is a class or struct with a non-overloaded member function called "*Whatever*" (i.e., the *NAME* arg passed to this macro), but only if the return type of "*Whatever*" is declared an "*int*" and the function takes 2 args of type ""*std::string_view*" and *float*" (and exactly those types only - conversions to those types don't factor in because the "*HasReturnAndArgs*" struct is relying on [HasReturnType_v](#hasreturntype_v)<F, int> and [HasArgTypes_v](#hasargtypes_v)<F, xstd::string_view, float> which both do an exact comparison - if you wish to loosen the check to support conversions as well, or anything else then you must modify the "*HasReturnAndArgs*" struct do so, and likely change its name as well to something more suitable for what it's checking).

The upshot is that the 2nd template arg of "*HasNonOverloadedMemberFunctionTraits_Whatever_v*" can be customized to check for a non-overloaded function called "*Whatever*" whose signature matches any criteria you wish, as specified by the template you pass as the 2nd arg. To this end, as described under [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function), and repeated here for your convenience, function types in C++ are always of the following form, where items in square brackets are optional (and this is what [std::is_function](https://en.cppreference.com/w/cpp/types/is_function) in the C++ standard itself looks for to determine if a type is a function):

    ReturnType [<CallingConvention>] (Args [, ...]) [const] [volatile] [&|&&] [noexcept[(true|false)]]
    
You're therefore free to check for any combination of these traits by writing a template analogous to "*HasReturnAndArgs*". For instance, if "*Test::Whatever()*" were declared with every possible trait above such as the following (noting that the default calling convention is always implicitly declared if not explicitly declared - we've explicitly declared STDEXT_CC_STDCALL below to override the default however, which is typically/usually STDEXT_CC_THISCALL for member functions):

```C++
class Test
{
public:
    int STDEXT_CC_STDCALL Whatever(float, std::string_view, ...) const volatie & noexcept;
};
```    

... then a template to check for every trait in "*Whatever*" above would look like the following (but if checking *every* trait then you can just rely on [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION](#declare_class_has_non_overloaded_function) instead - that's its purpose and it's both easier and cleaner):

```C++
struct HasAllTraits : std::bool_constant<HasReturnType_v<F, int> && // **Must have "int" return type**
                                         CallingConvention_v<F> == CallingConvention::Stdcall && // **Must have the "Stdcall" calling convention (though usually you'll rely on [DefaultCallingConvention_v](#defaultcallingconvention_v) here instead)**
                                         HasArgTypes_v<F, std::string_view, float> && // **Must have 2 args "std::string_view" and "float"**
                                         IsVariadic_v<F> && // **Must be variadic (must end with "...")**
                                         IsMemberFunctionConst_v<F> && // **Must be "const"**
                                         IsMemberFunctionVolatile_v<F> && // **Must be "volatile"**
                                         MemberFunctionRefQualifier_v<F> == RefQualifier::LValue && // **Must have any ref-qualifier "&" (though rare in practice)**
                                         IsNoexcept_v<F>
                                        >
{
};
```

Note that in practice, you can customize your own version of the above by simply copying the above and removing any of the conditions that don't matter to you (and renaming the struct to something appropriate for whatever it's checking). For instance, removing the call to "*HasCallingConvention_v"* means the calling convention will no longer be checked so it can be anything (i.e., the calling convention of "*Whatever*" isn't relevant to you). The same applies to the other conditions as well. Remove those you don't care about and keep the others (and again, give your struct a meaningful name). Note that for the call to [HasArgTypes_v](#hasargtypes_v), you can even replace this with calls to [IsArgTypeSame_v](#isargtypesame_v) if you wish to drill down into individual arguments (if you don't need to check all of them, otherwise just rely on [HasArgTypes_v](#hasargtypes_v) as seen above).
</details></blockquote>

<a name="DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS"></a><blockquote><details><summary>DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS</summary>
```C++
#define DECLARE_CLASS_HAS_NON_OVERLOADED_STATIC_FUNCTION_TRAITS(NAME)
```

This macro is functionally equivalent to macro [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS](#declare_class_has_non_overloaded_function_traits) but targets *static* member functions instead of non-static. See the latter macro for complete details. The following templates therefore won't detect functions unless they're *static* as well as non-overloaded (since both macros detect non-overloaded functions only). The macro creates the following two templates for this:

```C++
template <IS_CLASS_C T, template <TRAITS_FUNCTION_C> class HasFunctionTraitsT>
struct ClassHasNonOverloadedStaticFunctionTraits_Whatever; // Inherits from "std::true_type" or "std::false_type" (see DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS for details)

template<IS_CLASS_C T, template <TRAITS_FUNCTION_C> class HasFunctionTraitsT>
inline constexpr bool ClassHasNonOverloadedStaticFunctionTraits_Whatever_v = ClassHasNonOverloadedStaticFunctionTraits_Whatever<T, U>::value;
```

These templates are just the *static* counterparts of the non-static templates created by [DECLARE_CLASS_HAS_NON_OVERLOADED_FUNCTION_TRAITS](#declare_class_has_non_overloaded_function_traits). Again see the latter macro for details.

</details>
</blockquote>
</details>

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
#include "TypeTraits.h"

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
      2) Classification: Non-static member (class/struct="Test::SomeClass")
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

Note that for free functions (including static member functions), which don't support "*const*", "*volatile*" or ref-qualifiers, items 6, 7 and 8 above will therefore be removed. Instead, item 9 above, "*noexcept*", will simply be renumbered. The "*Classification*" above will also indicate "*Free or static member*".      
      
Lastly, note that if "*F*" is a functor type (including a lambda type), then the format will be identical to the above except that the "*Classification*" will now indicate "*Functor*" instead of "*Non-static member*", and the type itself in item 1 above will reflect the function type of "*F::operator()*"

</details></blockquote>

<a name="ForEachArg"></a><details><summary>ForEachArg</summary>

<blockquote>

```C++
template <TRAITS_FUNCTION_C F,>
          FOR_EACH_TUPLE_FUNCTOR_C ForEachTupleFunctorT>
inline constexpr bool ForEachArg(ForEachTupleFunctorT &&);
```

Not a traits template (unlike most others), but a helper function template you can use to iterate all arguments for function "*F*" if required (though rare in practice since you'll usually rely on [ArgType_t](#argtype_t) or [ArgTypeName_v](#argtypename_v) to retrieve the type of a specific argument - see these above). See [Looping through all function arguments](#loopingthroughallfunctionarguments) earlier for an example, as well as the declaration of "*ForEachArg()*" in "*TypeTraits.h*" for full details (or for a complete program that also uses it, see the [demo](https://godbolt.org/z/KjGvef1x3) program, also available in the repository itself).</details></blockquote>

<a name="ModuleUsage"></a>
## Module support in C++20 or later (experimental)

Note that you can also optionally use the module version of "*FunctionTraits*" if you're targeting C++20 or later (modules aren't available in C++ before that). Module support is still experimental however since C++ modules are still relatively new at this writing and not all compilers support them yet (or fully support them). GCC for instance has a compiler [bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109679) at this writing that until fixed, results in a compiler error (note that this bug has now been flagged as fixed but no release has been made available by GCC at this writing). The Intel compiler may also fail to compile depending on the platform (informally confirmed by Intel [here](https://community.intel.com/t5/Intel-oneAPI-Data-Parallel-C/Moving-existing-code-with-modules-to-Intel-c/m-p/1550610/thread-id/3448)), even though its documentation states that modules are partially supported at this writing. Only Microsoft and Clang therefore support the module version of "*FunctionTraits*" for now though GCC and Intel may be fixed before too long (and this documentation will be updated as soon as they are).

In addition to supporting modules in C++20, the "*std*" library is also available as a module in C++23 (see the official document for this feature [here](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf)). This occurs in the form of "*import std*" and/or "*import std.compat*" statements (see latter link). Support for this is also very new and not available on most compilers yet (nor supported by CMake at this writing - see CMake [Limitations](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#limitations)). Microsoft recently started supporting it however as of Visual Studio 2022 V17.5 (see [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170#enable-modules-in-the-microsoft-c-compiler)), though this version doesn't yet support mixing #include statements from the "*std*" library and the "*import std*" statement in C++23. Until available you must therefore always exclusively rely on one or the other only (i.e., for now you can't mix #includes from the "*std*" library and the "*import std*" statement in C++23 - Microsoft is on record about this). Also note that not all supported compilers may #define the C++23 feature macro [__cpp_lib_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) yet, indicating that "*import std*" and "*import std.compat*" are available. Where they are available however they're still considered experimental so "*FunctionTraits*" won't rely on them without your explicit permission first (via the transitional *STDEXT_IMPORT_STD_EXPERIMENTAL* macro described shortly).

As a result of all these issues, module support for "*FunctionTraits*" is therefore considered experimental until all compilers fully support both C++ modules (a C++20 feature), and (though less critically), the importing of the standard library in C++23 (again, via "*import std*" and "*import std.compat*"). The instructions below therefore reflect the incomplete state of module support in all compilers at this writing, and are therefore (unavoidably) longer they should be (for now). Don't be discouraged however, as they're much simpler than they first appear.

Note that the following instructions will be condensed and simplified in a future release however, once all compilers are fully compliant with the C++20 and C++23 standard (in their support of modules). For now, because they're not fully compliant, the instructions below need to deal with the situation accordingly, such as the *STDEXT_IMPORT_STD_EXPERIMENTAL* constant in 4 below, which is a transitional constant only. It will be removed in a future release.

To use the module version of "TypeTraits":

1. Ensure your project is set up to handle C++ modules if it's not already by default (again, since compiler support for modules is still evolving so isn't available out-of-the-box in some compilers). How to do this depends on the target compiler and your build environment (though C++20 or greater is always required), as each platform has its own unique way (such as the *-fmodules-ts* option in GCC - see GCC link just below). The details for each compiler are beyond the scope of this documentation but the following official (module) links can help get you started (though you'll likely need to do additional research if you're not already familiar with the process). Note that the "*CMake*" link below however does provide additional version details about GCC, Microsoft and Clang:
    1. [C++ modules (official specification)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1103r3.pdf)
    2. [GCC](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Modules.html) (not currently supported by "*FunctionTraits*" however due to this GCC [bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109679), now flagged as fixed but no release has been made available by GCC at this writing)
    3. [Microsoft](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170)
    4. [Clang](https://clang.llvm.org/docs/StandardCPlusPlusModules.html)
    5. [Intel](https://www.intel.com/content/www/us/en/developer/articles/technical/c20-features-supported-by-intel-cpp-compiler.html) (search page for [P1103R3](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1103r3.pdf) - no other Intel docs on modules can be found at this writing)
    6. [CMake](https://www.kitware.com/import-cmake-the-experiment-is-over/)
2. Add the primary module interface files "*TypeTraits.cppm*" and "*CompilerVersions.cppm*" from this repository to your project, which builds the modules "*TypeTraits*" and "*CompilerVersions*" respectively (corresponding to "*TypeTraits.h*" and "*CompilerVersions.h*" described in the [Usage](#usage) section earlier - both ".h" files are still required however as each module simply defers to its ".h" file to implement the module). Ensure your build environment is set up to process these "*.cppm*" files as C++ modules if it doesn't handle it by default (based on the extension for instance). Consult the docs for your specific platform (such as changing the extension to "*.ixx*" on Microsoft platforms - more on this later). Note that you're free to change the "*.cppm*" extension to whatever you require, assuming "*.cppm*" doesn't suffice (again, more on this later). You can then import each module wherever you need it (read up on C++ modules for details), either using an "*import*" statement as would normally be expected (normally just "*import TypeTraits*" - this is how modules are normally imported in C++), or by continuing to #include the header itself, normally just *#include "TypeTraits.h"* (again, as described in the [Usage](#usage) section earlier). In the latter case (when you *#include "TypeTraits.h"*), this will not only import "*TypeTraits*" for you as described in 3 below, but also has the benefit of making all public macros in "*TypeTraits.h*" available as well (should you require any of them), the reason you would choose to *#include "TypeTraits.h*" in the module version instead of "*import TypeTraits*" directly (more on this shortly).
3. #define the constant *STDEXT_USE_MODULES* when you build your project (add this to your project's build settings). Doing so changes the behavior of both "*TypeTraits.h*" and "*CompilerVersions.h*" (again, "*TypeTraits.h*" automatically #includes "*CompilerVersions.h*" - see [Usage](#usage) section), so that instead of declaring all C++ declarations as each header normally would (when *STDEXT_USE_MODULES* isn't #defined), each header simply imports the module instead (e.g., *#include "TypeTraits.h"* simply issues an "*import TypeTraits*" statement). All other C++ declarations in the file are then preprocessed out except for (public) macros, since they're not exported by C++ modules (so when required, the files that #define them must still be #included in the usual C++ way). Therefore, by #including "*TypeTraits.h*", you're effectively just creating an "*import TypeTraits*" statement (as well as "*import CompilerVersions*"), but also #defining all public macros in the header as well (including those in "*CompilerVersions.h*" - more on these macros shortly). All other declarations in the file are preprocessed out as noted (they're not needed because the "*import TypeTraits*" statement itself makes them available). Note that if *STDEXT_USE_MODULES* isn't #defined however (though you normally should #define it), then each header is #included in the usual C++ way (no "*import*" statement will exist and all declarations in the header are declared normally), which effectively defeats the purpose of using modules (unless you manually code your own "*import TypeTraits*" statement which can safely coexist with *#include "TypeTraits.h"* if you use both but there's no reason to normally). Note that if you don't use any of the macros in "*TypeTraits.h*" or "*CompilerVersions.h*" in your code however (again, more on these macros shortly), then you can simply apply your own "*import*" statement as usual, which is normally the natural way to do it (and #including the header instead is even arguably misleading since it will appear to the casual reader of your code that it's just a traditional header when it's actually applying an "*import*" statement instead, as just described). #including either ".h" file however to pick up the "*import*" statement instead of directly applying "*import*" yourself has the benefit of #defining all macros as well should you ever need any of them (though you're still free to directly import the module yourself at your discretion - redundant "*import*" statements are harmless if you directly code your own "*import TypeTraits*" statement _**and**_ *#include "TypeTraits.h"* as well, since the latter also applies its own "*import TypeTraits*" statement as described). Note that macros aren't documented in this README file however since the focus of this documentation is on "*FunctionTraits*" itself (the subject of this GitHub repository). Both "*TypeTraits.h*" and "*CompilerVersions.h*" contain various support macros however, such as the *TRAITS_FUNCTION_C* macro in "*TypeTraits.h*" (see this in the [Helper templates](#helpertemplates) section earlier), and the "*Compiler Identification Macros*" and "*C++ Version Macros*" in "*CompilerVersions.h*" (a complete set of macros allowing you to determine which compiler and version of C++ is running - see these in "*CompilerVersions.h*" for details). "*FunctionTraits*" itself utilizes these macros for its own internal needs as required but end-users may wish to use them as well (for their own purposes). Lastly, note that as described in the [Usage](#usage) section earlier, #including "*TypeTraits.h*" automatically picks up everything in "*CompilerVersions.h*" as well since the latter is a dependency, and this behavior is also extended to the module version (though if you directly "*import TypeTraits*" instead of #include "*TypeTraits.h*", it automatically imports module "*CompilerVersions*" as well but none of the macros in "*CompilersVersions.h*" will be available, again, since macros aren't exported by C++ modules - if you require any of the macros in "*CompilersVersions.h*" then you must *#include "TypeTraits.h"* instead, or alternatively just *#include "CompilersVersions.h"* directly).
4. If targeting C++23 or later (the following constant is ignored otherwise), *and* the C++23 [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) feature is supported by your compiler (read on), optionally #define the constant *STDEXT_IMPORT_STD_EXPERIMENTAL* when you build your project (add this to your project's build settings). This constant is transitional only as a temporary substitute for the C++23 feature macro [__cpp_lib_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) (used to indicate that "*import std*" and "*import std.compat*" are supported). If *STDEXT_IMPORT_STD_EXPERIMENTAL* is #defined (in C++23 or later), then the "*FunctionTraits*" library will use an [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) statement to pick up all its internal dependencies from the "*std*" library instead of #including each individual "*std*" header it depends on. This is normally recommended in C++23 or later since it's much faster to rely on [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) than to #include each required header from the "*std*" library (the days of doing so will likely become a thing of the past). Note that if you #define *STDEXT_IMPORT_STD_EXPERIMENTAL* then it's assumed that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) is in fact supported on your platform or cryptic compilers errors will likely occur (if targeting C++23 or later - again, the constant is ignored otherwise). Also note that as described earlier, if targeting Microsoft platforms then your own project must also currently rely on it everywhere since you can't (currently) mix headers from the "*std*" library and "*import std*" (until Microsoft corrects this). In any case, please see [Footnotes](#footnotes) for module versioning information for each supported compiler (which includes version support information for "*import std*" and "*import std.compat*"). Note that *STDEXT_IMPORT_STD_EXPERIMENTAL* will be removed in a future release however once all supported compilers fully support the [__cpp_lib_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) feature macro (which will then be used instead). For now the latter macro either isn't #defined on all supported platforms at this writing (most don't support "*import std*" and "*import std.compat*" yet), or if it is #defined such as in recent versions of MSVC, "*FunctionTraits*" doesn't rely on it yet (for the reasons described but see the comments preceding the check for *STDEXT_IMPORT_STD_EXPERIMENTAL* in "*TypeTraits.h*" for complete details). Instead, if you wish for "*FunctionTraits*" to rely on [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) then you must grant explicit permission by #defining *STDEXT_IMPORT_STD_EXPERIMENTAL*. The C++ feature macro [__cpp_lib_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) itself is completely ignored by the "*FunctionTraits*" library in this release.

Finally, note that you're free to rename each "*.cppm*" extension to whatever you require on your particular platform. On Microsoft platforms for instance you can (and likely will) change it to "*.ixx*" (the default module extension for Microsoft), though you can optionally maintain the "*.cppm*" extension on Microsoft as well if you wish (or change it to anything else). If you do so however then in your Microsoft Visual Studio project you'll need to set each "*.cppm*" file's "*Compile As*" property to "*Compile as C++ Module Code (/interface)*" (you'll find this in the file's properties under "*Configuration Properties -> C/C++ -> Advanced*" - see the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) command line switch in Microsoft's documentation for details). "*CMake*" developers may wish to consult the following Microsoft [issue](https://gitlab.kitware.com/cmake/cmake/-/issues/25643) however (as described in the latter link, Visual Studio solutions built with CMake don't add the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) property as they normally should). In any case, note that each "*.cppm*" file in the "*FunctionTraits*" library is platform neutral so the extension isn't relevant, so long as your compiler is instructed to identify it as a module (if it doesn't recognize it by default). The C++ standard itself doesn't address the situation so while the extension "*.cppm*" has become widely adopted, there's no universal convention (and other extensions do exist).

<a name="WhyChooseThisLibrary"></a>
## Why choose this library
In a nutshell, because it's extremely easy to use, with syntax that's consistently very clean (when relying on [Technique 2 of 2](#technique2of2) as most normally will), it handles all mainstream function types you wish to pass as template args (see [here](#templateargf)), it has a very small footprint (once you ignore the many comments in "*TypeTraits.h*"), and it may be the most complete function traits library available on the web at this writing (based on attempts to find an equivalent library with calling convention support in particular).

Note that "*FunctionTraits*" is also significantly smaller than the Boost version ("*boost::callable_traits*"), which consists of a bloated (excessive) number of files and roughly twice the amount of code at this writing (largely due to a needlessly complex design, no disrespect intended). "*FunctionTraits*" still provides the same features for all intents and purposes however (and a few extra including templates to retrieve argument type names, iterate all argument types, etc.), as well as support for (mainstream) calling conventions as emphasized, which isn't officially available in "*boost::callable_traits*" (and even if activated there, it has several other caveats as described earlier in this document).

Also note that the Boost "*_args_*" template (or its "*_args\_t_*" helper template), the template that developers typically use the most in practice (to access a function's argument types and count), is less user-friendly than it should be, requiring additional overhead to access the individual argument types and/or argument count (since "*_args_*" itself just returns the collection of argument types - no Boost-specific helper templates exist to drill into it leaving most users having to make additional calls to [std::tuple_element](https://en.cppreference.com/w/cpp/utility/tuple_element) and/or [std::tuple_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size), or rely on their own wrappers to consolidate the calls). Additional helper templates to simplify the process would have been beneficial (though the code does seem to have at least one internal template the author may have been experimenting with towards this goal but the situation is murky). Moreover, note that "*_args_*" also has a few strange quirks, such as including the function's member class type itself when handling non-static member functions (always at index zero in "*_args_*"), even though the class type has nothing to do with a function's arguments. It also means the function's first argument type is at index 1 in "*_args_*" for non-static member functions but index zero for free functions, which forces developers to have to subtract 1 from [std::tuple_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size) to retrieve the number of arguments for non-static member functions, but not for free functions. This needlessly complicates things for any code that may be handling both non-static member functions and free functions (when retrieving the number of arguments or for any other purpose). The behavior also overlaps with the purpose of template "*_qualified_class_of_*", whose main purpose is to return a non-static member function's class type (the same type that template "*_args_*" stores at index zero), and even template "*_class\_of_*", rendering all these templates somewhat redundant and a bit confusing (and "*_args_*" also confusingly handles member *data* pointer types as well, which really has nothing to do with its intended purpose of handling function arguments). Note that by contrast, "*FunctionTraits*" provides clean and easy-to-use helper templates out-of-the-box (no need to explicitly call [std::tuple_element](https://en.cppreference.com/w/cpp/utility/tuple_element) or [std::tuple_size](https://en.cppreference.com/w/cpp/utility/tuple/tuple_size)), and no such oddities or redundant behavior (or behavior unrelated to the task of handling functions only).

Lastly, note that "*boost::callable_traits*" does support the experimental "*transaction_safe*" keyword, but "*FunctionTraits*" doesn't by design. Since this keyword isn't in the official C++ standard (most have never likely heard of it), and it's questionable if it ever will be (it was first floated in 2015), I've deferred its inclusion until it's actually implemented, if ever. Very few users will be impacted by its absence and including it in "*FunctionTraits*" can likely be done in less than a day based on my review of the situation.

The upshot is that "*FunctionTraits*" is generally more complete than all other similar libraries I'm aware of (and easier to use than Boost at times), in particular due to its (mainstream) calling convention support (noting that very few other libraries even exist that can be considered "complete"). Very little could be added at this stage that would benefit most users and would usually require improvements to the C++ standard itself to accommodate. However, even wider support for calling conventions may be added later, to target less frequently used calling conventions that are platform specific (if there's a demand for it). For now its mainstream calling convention support will usually meet the needs of most users.

<a name="DesignConsiderations"></a>
## "FunctionTraits" design considerations (for those who care - not required reading)
Note that "*FunctionTraits*" is not a canonical C++ "traits" class that would likely be considered for inclusion in the C++ standard itself. While it could be with some fairly easy-to-implement changes and (rarely-used) additions (read on), it wasn't designed for this. It was designed for practical, real-world use instead. For instance, it supports most mainstream calling conventions as emphasized earlier which isn't currently addressed in the C++ standard itself. Also note that template arg "*F*" of "*FunctionTraits*" and its helper templates isn't confined to pure C++ function types. As described earlier in this document, you can also pass pointers to functions, references to functions, references to pointers to functions, and functors (in all cases referring to their actual C++ types, not any actual instances). Note that references to non-static member function pointers aren't supported however because they're not supported by C++ itself.

While I'm not a member of the C++ committee, it seems very likely that a "canonical" implementation for inclusion in the standard itself likely wouldn't (or might not) address certain features, like calling conventions since they don't currently exist in the C++ standard as described earlier (unless it's ever added at some point). It's also unlikely to support all the variations of pointers and reference types (and/or possibly functors) that "*FunctionTraits*" handles. In the real world however function types include calling conventions so they need to be detected by a function traits library (not just the default calling convention), and programmers are also often working with raw function types or pointers to functions or functors (or references to functions or references to pointers to functions though these are typically encountered less frequently). For non-static member functions in particular, pointers to member functions are the norm. In fact, for non-static member functions, almost nobody in the real world ever works with their actual (non-pointer) function type. I've been programming in C++ for decades and don't recall a single time I've ever worked with the actual type of one. You work with pointers to non-static member functions instead when you need to, such as **void (YourClass::*)()**. Even for free functions (which for our purposes also includes static member functions), since the name of a free function decays into a pointer to the function in most expressions, you often find yourself working with a pointer to such functions, not the actual function type (though you do work with the actual function type sometimes, unlike for non-static member functions which is extremely rare in the real world). Supporting pointers and references to functions and even references to pointers to functions therefore just makes things easier even if some consider it unclean (I don't). A practical traits struct should just make it very easy to use without having to manually remove things using "*std::remove_pointer*" and/or "*std::remove_reference*" first (which would be required if such a traits struct was designed to handle pure C++ function types only, not pointers and references to function types or even references to pointers). It's just easier and more flexible to use it this way (syntactically cleaner). Some would argue it's not a clean approach but I disagree. It may not be consistent with how things are normally done in the standard itself (again, I suspect it might handle raw function types only), but it better deals with the real-world needs of most developers IMHO (so it's "clean" because its syntax is designed to cleanly support that).

Lastly, note that "*FunctionTraits*" doesn't support the so-called [abominable function types](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0172r0.html). These are simply non-static member function types (not pointers to such types but the actual raw C++ type) with cv ("*const*" and/or "*volatile*") and/or reference qualifiers (*&* and *&&*). For full details, see the comments preceding "*Private::IsFreeFunction*" in the "*TypeTraits.h*" header. Abominable functions are arguably a flawed feature of the C++ type system, and a wider discussion of them isn't warranted here. The bottom line is that most programmers will never encounter them throughout their entire career, and for practical reasons (not discussed here), their lack of support by "*FunctionTraits*" makes its design cleaner and easier to use. "*FunctionTraits*" will therefore reject such functions via a [concept](https://en.cppreference.com/w/cpp/language/constraints) in C++20 or later (possibly depending on the context), or a "*static_assert*" otherwise (always in C++17). Instead, non-static member functions are always handled using mainstream pointer-to-member function syntax. Raw C++ function types are always treated as free functions by "*FunctionTraits*" (which includes static member functions for our purposes), so if you pass a raw C++ function type (i.e., an actual function type and not a pointer or reference to such a type), no cv or ref qualifiers are allowed on that type (i.e., abominable functions aren't supported), since free functions don't have such qualifiers. They're only legal in C++ for non-static member functions. Passing an abominable function is therefore considered illegal by "*FunctionTraits*" even though it's a valid C++ function type (but they're "abominable" nevertheless so we don't support them because the design is cleaner and easier to use when we don't - in the real world almost nobody will care).

<a name="Footnotes"></a>
### Footnotes
[^1]: **_GCC minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: GCC V10.2 or later
    2. Module (*\*.cppm*) version  of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Not currently compiling at this writing due to a GCC [bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=109679) (flagged as fixed in the latter link but not yet released at this writing).

     Note that GCC compatible compilers are also supported based on the presence of the #defined constant \_\_GNUC\_\_
[^2]: **_Microsoft Visual C++ minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: Microsoft Visual C++ V19.16 or later for [Read traits](#readtraits) (normally installed with Visual Studio 2017 V15.9 or later), or V19.20 or later for [Write traits](#writetraits) (normally installed with Visual Studio 2019 or later). Note that [Write traits](#writetraits) are unavailable in Visual Studio 2017 releases of VC++ due to compiler bugs in those versions.
    2. Module (*\*.cppm*) version of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Microsoft Visual C++ V19.31 or later (normally installed with Visual Studio 2022 V17.1 or later - see [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170#enable-modules-in-the-microsoft-c-compiler)). Note that if using CMake then it has its own requirements (toolset 14.34 or later provided with Visual Studio 2022 V17.4 or later - see [here](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#compiler-support)).

    Note that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in C++23 is supported by Microsoft Visual C++ V19.35 or later (normally installed with Visual Studio 2022 V17.5 or later - see [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170#enable-modules-in-the-microsoft-c-compiler)), so the transitional *STDEXT_IMPORT_STD_EXPERIMENTAL* macro described in [Module support in C++20 or later](#moduleusage) is ignored in earlier versions
[^3]: **_Clang minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: Clang V11.0 or later
    2. Module (*\*.cppm*) version of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Clang V16.0 or later

     Note that [Microsoft Visual C++ compatibility mode](https://clang.llvm.org/docs/MSVCCompatibility.html) is also supported. Please note that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in C++23 is not supported by Clang yet (at this writing), so the *STDEXT_IMPORT_STD_EXPERIMENTAL* macro described in [Module support in C++20 or later](#moduleusage) shouldn't be #defined until it is (until then a compiler error will occur if targeting C++23 or later).

[^4]: **_Intel oneAPI DPC++/C++ minimum required version:_**
    1. Non-module (*\*.h*) version of FunctionTraits: Intel oneAPI DPC++/C++ V2021.4.0 or later
    2. Module (*\*.cppm*) version of FunctionTraits (see [Module support in C++20 or later](#moduleusage)): Listed as partially supported by Intel [here](https://www.intel.com/content/www/us/en/developer/articles/technical/c20-features-supported-by-intel-cpp-compiler.html) (search for "*Modules: Merging Modules*"), but not yet working for unknown reasons (informally confirmed by Intel [here](https://community.intel.com/t5/Intel-oneAPI-Data-Parallel-C/Moving-existing-code-with-modules-to-Intel-c/m-p/1550610/thread-id/3448)). Note that even when it is finally working, if using CMake then note that it doesn't natively support module dependency scanning for Intel at this writing (see CMake [Compiler Support](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#compiler-support) for modules).

     Note that [Microsoft Visual C++ compatibility mode](https://www.intel.com/content/www/us/en/docs/dpcpp-cpp-compiler/developer-guide-reference/2024-0/microsoft-compatibility.html) is also supported. Please note that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in C++23 is not supported by Intel yet (at this writing), so the *STDEXT_IMPORT_STD_EXPERIMENTAL* macro described in [Module support in C++20 or later](#moduleusage) shouldn't be #defined until it is (until then a compiler error will occur if targeting C++23 or later).

[^5]: **_"TypeName_v" null termination:_**
    1. Note that the string returned by [TypeName_v](#typename_v) is guaranteed to be null-terminated only if the internal constant *TYPENAME_V_DONT_MINIMIZE_REQD_SPACE* isn't #defined. It's not #defined by default so null-termination is guaranteed by default. See this constant in "*TypeTraits.h*" for full details (not documented in this README file since few users will ever need to #define it).

[^6]: **_Removing write traits if not required (optional):_**
    1. Note that most users don't use [Write traits](#writetraits) (most use [Read traits](#readtraits) only), so those who wish to remove them can optionally do so by #defining the preprocessor constant *REMOVE_FUNCTION_WRITE_TRAITS*. Doing so will preprocess out all [Write traits](#writetraits) so they no longer exist, and therefore eliminate the overhead of compiling them (though the overhead is normally negligible). Note however that [Write traits](#writetraits) are always unavailable in Microsoft Visual Studio 2017 releases of VC++ due to compiler bugs in those versions (also noted in footnote 2 above).
