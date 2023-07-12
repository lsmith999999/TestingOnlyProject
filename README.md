<div id="ReadTraits"></div>

### Read traits
  
<span id="ArgCount_v"></span><details><summary><b>ArgCount_v</b></summary>
>`template <TRAITS_FUNCTION_C F>`<br/>`constexpr std::size_t ArgCount_v;`<br/><br/>
>"std::size_t" variable storing the number of arguments in "F" *not* including variadic arguments if any. Will be zero if function "F" has no (non-variadic) args. Note that this count is formally called "arity" but this variable is given a more user-friendly name.<br /><br /><ins>IMPORTANT</ins>:<br />Please note that if you wish to check if a function's argument list is completely empty, then inspecting this helper template for zero (0) is not sufficient, since it may return zero but still contain variadic args. To check for a completely empty argument list, call [IsEmptyArgList_v](#IsEmptyArgList_v) instead.</details>

<span id="ArgType_t"></span><details><summary><b>ArgType_t</b></summary>
>`template <TRAITS_FUNCTION_C F,`<br/>`          std::size_t I>`<br/>`using ArgType_t;`<br/><br/>
>Type alias for the type of the "Ith" arg in function "F", where "I" is in the range 0 to the number of (non-variadic) arguments in "F" minus 1 (see "ArgCount_v" variable just above). Pass "I" as the (zero-based) 2nd template arg (see earlier examples). Note that if "I" is greater than or equal to the number of args in "F" (again, see "ArgCount_v" just above), then a compiler error will occur (so if "F" has no non-variadic args whatsoever, a compiler error will always occur, even if passing zero).</details>

<span id="ArgTypeName_v"></span><details><summary><b>ArgTypeName_v</b></summary>
>`template <TRAITS_FUNCTION_C F,`<br/>`          std::size_t I>`<br/>`constexpr tstring_view ArgTypeName_v;`<br/><br/>
>Same as "ArgType_t" just above but returns this as a (WYSIWYG) string (of type "tstring_view" - see [TypeName_v](#TypeName_v) for details). A float would therefore be (literally) returned as "float" for instance (quotes not included).</details>

<span id="ArgTypes_t"></span><details><summary><b>ArgTypes_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using ArgTypes_t;`<br/><br/>
>Type alias for a "std::tuple" representing all non-variadic argument types in "F". Rarely required in practice however since you'll usually rely on "ArgType_t" or "ArgTypeName_v" to retrieve the type of a specific argument (see these above). If you require the "std::tuple" that stores all (non-variadic) argument types, then it's typically (usually) because you want to iterate all of them (say, to process the type of every argument in a loop). If you require this, then you can use the "ForEachArg()" helper function (template) further below. See this for details. </details>

<span id="CallingConvention_v"></span><details><summary><b>CallingConvention_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr CallingConvention CallingConvention_v;`<br/><br/>
> Calling convention of "F" returned as a "CallingConvention" enumerator (declared in "TypeTraits.h"). Calling conventions include "Cdecl", "Stdcall", etc. (note that functions with calling conventions not seen in this enumerator are not supported, but all mainstream calling conventions are). Also please note that compilers will sometimes change the calling convention declared on your functions to the "Cdecl" calling convention depending on the compiler options in effect at the time (in particular when compiling for 64 bits opposed to 32 bits, though the "Vectorcall" calling convention *is* supported on 64 bits but not on GCC since it doesn't this particular calling convention at all). In this case the calling convention on your function is ignored and "CallingConvention_v" will correctly return the "Cdecl" calling convention (if that's what the compiler actually used).</details>

<span id="CallingConventionName_v"></span><details><summary><b>CallingConventionName_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr tstring_view CallingConventionName_v;`<br/><br/>
> Same as "CallingConvention_v" just above but returns this as a (WYSIWYG) string (of type "tstring_view" - see [TypeName_v](#TypeName_v) for details).</details>

<span id="ForEachArg"></span><details><summary><b>ForEachArg</b></summary>
> `template <TRAITS_FUNCTION_C F,>`<br/>`          FOR_EACH_TUPLE_FUNCTOR_C ForEachTupleFunctorT>`<br/>`constexpr bool ForEachArg(ForEachTupleFunctorT &&);`                      <br/><br/>
> Not a traits template (unlike all others in this table), but a helper function template you can use to iterate all arguments for function "F" if required (though rare in practice since you'll usually rely on "ArgType_t" or "ArgTypeName_v" to retrieve the type of a specific argument - see these above). See [Looping through all function arguments](#LoopingThroughAllFunctionArguments) earlier for an example, as well as the declaration of "ForEachArg()" in "TypeTraits.h" for full details (or for a complete program that also uses it, see the [demo](https://godbolt.org/z/fx8MWGv99) program, also available in the repository itself).</details>

<span id="FunctionType_t"></span><details><summary><b>FunctionType_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using FunctionType_t;`<br/><br/>
> Type alias identical to "F" itself unless "F" is a functor (i.e., "IsFunctor_v" returns true), in which case it's a type alias for the "operator()" member of the functor (to retrieve the functor type itself in this case, see [MemberFunctionClass_t](#MemberFunctionClass_t))</details>

<span id="FunctionTypeName_v"></span><details><summary><b>FunctionTypeName_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool FunctionTypeName_v;`<br/><br/>
> Same as "FunctionType_t" just above but returns this as a (WYSIWYG) string (of type "tstring_view" - see [TypeName_v](#TypeName_v) for details).</details>

<span id="IsEmptyArgList_v"></span><details><summary><b>IsEmptyArgList_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsEmptyArgList_v;`<br/><br/>
> "bool" variable set to "true" if the function represented by "F" has an empty arg list (it has no args whatsoever including variadic args), or "false" otherwise. If true then note that "ArgCount_v" is guaranteed to return zero (0), and "IsVariadic_v" is guaranteed to return false.<br /><br /><ins>IMPORTANT</ins>:<br />Note that you should rely on this helper to determine if a function's argument list is completely empty opposed to checking the "ArgCount_v" helper for zero (0), since the latter returns zero only if "F" has no non-variadic args. If it has variadic args but no others, i.e., its argument list is "(...)", then the argument list isn't empty even though "ArgCount_v" returns zero (since it still has variadic args). Caution advised.</details>

<span id="IsFreeFunction_v"></span><details><summary><b>IsFreeFunction_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsFreeFunction_v;`<br/><br/>
> "bool" variable set to "true" if "F" is a free function (including static member functions), or "false" otherwise.</details>

<span id="IsFunctor_v"></span><details><summary><b>IsFunctor_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsFunctor_v;`<br/><br/>
> "bool" variable set to "true" if "F" is a functor (the functor's class/struct was passed for "F") or "false" otherwise. Note that when true, [IsMemberFunction_v](#IsMemberFunction_v) is also guaranteed to be true.</details>

<span id="IsMemberFunction_v"></span><details><summary><b>IsMemberFunction_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsMemberFunction_v;`<br/><br/>
> "bool" variable set to "true" if "F" is a non-static member function (including when "F" is a functor), or "false" otherwise (if you need to specifically check for functors only then see "IsFunctor_v" just above). Note that you may need to invoke this before accessing the following helper templates. Since the following are applicable to non-static member functions only, if you don't know whether "F" is a non-static member function ahead of time (or a functor), then you should normally call "IsMemberFunction_v" to determine this first. If it's "false" then "F" is a free function (which includes static member functions), so a call to any of the following will result in default values being returned that aren't applicable to free functions (so you shouldn't normally invoke them unless you're ok with the default values they return for free functions):<br /><br />- IsMemberFunctionConst_v<br />- IsMemberFunctionVolatile_v<br />- MemberFunctionClass_t<br />- MemberFunctionClassName_v<br />- MemberFunctionRefQualifier_v<br />- MemberFunctionRefQualifierName_v</details>

<span id="IsMemberFunctionConst_v"></span><details><summary><b>IsMemberFunctionConst_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsMemberFunctionConst_v;`<br/><br/>
> "bool" variable applicable only if "F" is a non-static member function (or a functor). Set to "true" if the function has the "const" cv-qualifier (it's declared with the "const" keyword) or "false" otherwise. Always "false" for free functions including static member functions (not applicable to either). You may therefore wish to invoke [IsMemberFunction_v](#IsMemberFunction_v) to detect if "F" is in fact a non-static member function (or functor) before using this trait.</details>

<span id="IsMemberFunctionVolatile_v"></span><details><summary><b>IsMemberFunctionVolatile_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsMemberFunctionVolatile_v;`<br/><br/>
> "bool" variable applicable only if "F" is a non-static member function  (or a functor). Set to "true" if the function has the "volatile" cv-qualifier (its declared with the "volatile" keyword) or "false" otherwise. Always "false" for free functions including static member functions (not applicable to either). You may therefore wish to invoke [IsMemberFunction_v](#IsMemberFunction_v) to detect if "F" is in fact a non-static member function (or functor) before using this trait.</details>

<span id="IsNoexcept_v"></span><details><summary><b>IsNoexcept_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsNoexcept_v;`<br/><br/>
> "bool" variable set to "true" if the function is declared as "noexcept" or "false" otherwise (always false if the "noexcept" specifier is absent in the function, otherwise, if present then it evaluates to "true" if no bool expression is present in the "noexcept" specifier (the expression has been omitted), or the result of the bool expression otherwise - WYSIWYG).</details>

<span id="IsVariadic_v"></span><details><summary><b>IsVariadic_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr bool IsVariadic_v;`<br/><br/>
> "bool" variable set to true if "F" is a variadic function (last arg of "F" is "...") or false otherwise.</details>

<span id="MemberFunctionClass_t"></span><details><summary><b>MemberFunctionClass_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionClass_t;`<br/><br/>
> If "F" is a non-static member function (or a functor), a type alias for the type of the class (or struct) that declares that function (same as "F" itself if "F" is a functor). Always "void" otherwise (for free functions including static member functions). You may therefore wish to invoke [IsMemberFunction_v](#IsMemberFunction_v) to detect if "F" is in fact a non-static member function (or functor) before applying this trait.</details>

<span id="MemberFunctionClassName_v"></span><details><summary><b>MemberFunctionClassName_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr tstring_view MemberFunctionClassName_v;`<br/><br/>
> Same as "MemberFunctionClass_t" just above but returns this as a (WYSIWYG) string (of type "tstring_view" - see [TypeName_v](#TypeName_v) for details).</details>

<span id="MemberFunctionRefQualifier_v"></span><details><summary><b>MemberFunctionRefQualifier_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr RefQualifier MemberFunctionRefQualifier_v;`<br/><br/>
> "RefQualifier" variable, a proprietary enumerator in "TypeTraits.h" applicable only if "F" is a non-static member function (or a functor). Set to "RefQualifier::None" if the function isn't declared with any reference qualifiers (usually the case for non-static member functions in practice, and always the case for free functions since it's not applicable), "RefQualifier::LValue" if the function is declared with the "&" reference qualifier, or "RefQualifier::RValue" if the function is declared with the "&&" reference qualifier. Note that you may wish to invoke [IsMemberFunction_v](#IsMemberFunction_v) to detect if "F" is in fact a non-static member function (or functor) before applying this trait.</details>

<span id="MemberFunctionRefQualifierName_v"></span><details><summary><b>MemberFunctionRefQualifierName_v</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          bool UseAmpersands = true>`<br/>`constexpr tstring_view MemberFunctionRefQualifierName_v;`<br/><br/>
> Same as "MemberFunctionRefQualifier_v" just above but returns this as a (WYSIWYG) string (of type "tstring_view" - see [TypeName_v](#TypeName_v) for details). Note that this template also takes an extra template arg besides function "F", a "bool" called "UseAmpersands", indicating whether the returned string should be returned as "&" or "&&" (if the function is declared with an "&" or "&&" reference qualifier respectively), or "LValue" or "RValue" otherwise. Defaults to "true" if not specified (returns "&" or "&&" by default). Not applicable however if no reference qualifiers are present ("None" is always returned).</details>

<span id="ReturnType_t"></span><details><summary><b>ReturnType_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using ReturnType_t;`<br/><br/>
> Type alias for the return type of function "F"</details>

<span id="ReturnTypeName_v"></span><details><summary><b>ReturnTypeName_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr tstring_view ReturnTypeName_v;`<br/><br/>
> Same as "ReturnType_t" just above but returns this as a (WYSIWYG) string (of type "tstring_view" - see [TypeName_v](#TypeName_v) for details). A float would therefore be (literally) returned as "float" for instance (quotes not included).</details>

<div id="TypeName_v"></div>
<details><summary><b>TypeName_v</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`constexpr tstring_view TypeName_v;`<br/><br/>
> Not a template associated with "FunctionTraits" per se, but a helper template you can use to return the user-friendly name of any C++ type as a "tstring_view" (more on this shortly). Just pass the type you're interested in as the template's only template arg. Note however that all helper aliases above such as "ArgType_t" have a corresponding helper "Name" template ("ArgTypeName_v" in the latter case) that simply rely on "TypeName_v" to return the type's user-friendly name (by simply passing the alias itself to "TypeName_v"). You therefore don't have to call "TypeName_v" directly in most cases since a helper variable template already exists that does this for you (again, one for every alias template above, where the name of the variable template returning the type's name is the same as the name of the alias template itself but with the "_t" suffix in the alias' name replaced with "Name_v", e.g., "ArgType_t" and "ArgTypeName_v"). The only time you may need to call "TypeName_v" directly when using "FunctionTraits" is when you use "ForEachArg()" as seen in the [Looping through all function arguments](#LoopingThroughAllFunctionArguments) section above. See the sample code in that section for an example (specifically the call to "TypeName_v" in the "displayArgType" lambda of the example). Note that "TypeName_v" can be passed any C++ type however, not just types associated with "FunctionTraits". You can therefore use it for your own purposes whenever you need the user-friendly name of a C++ type as a compile-time string. Note that "TypeName_v" returns a "tstring_view" (in the "StdExt" namespace) which always resolves to "std::string_view" on non-Microsoft platforms, and on Microsoft platforms, to "std::wstring_view" when compiling for Unicode (usually the case - strings are normally stored in UTF-16 in modern-day Windows), or "std::string_view" otherwise (when compiling for ANSI but this is very rare these days).</details>

<div id="WriteTraits"></div><br/>

### Write traits

<span id="AddVariadicArgs_t"></span><details><summary><b>AddVariadicArgs_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using AddVariadicArgs_t;`<br/><br/>
> Type alias for "F" after adding "..." to the end of its argument list if not already present. Note that the calling convention is also changed to the "Cdecl" calling convention for the given compiler. This is the only supported calling convention for variadic functions in this release but most platforms require this calling convention for variadic functions. It ensures that the calling function (opposed to the called function) pops the stack of arguments after the function is called, which is required by variadic functions. Other calling conventions that also do this are possible though none are currently supported in this release (since none of the currently supported compilers support this - such calling conventions are rare in practice).</details>

<span id="AddNoexcept_t"></span><details><summary><b>AddNoexcept_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using AddNoexcept_t;`<br/><br/>
>Type alias for "F" after adding "noexcept" to "F" if not already present</details>

<span id="MemberFunctionAddConst_t"></span><details><summary><b>MemberFunctionAddConst_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionAddConst_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after adding the "const" cv-qualifier to the function if not already present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since "const" applies to non-static member functions only).</details>

<span id="MemberFunctionAddCV_t"></span><details><summary><b>MemberFunctionAddCV_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionAddCV_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after adding both the "const" AND "volatile" cv-qualifiers to the function if not already present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since "const" and "volatile" apply to non-static member functions only).</details>

<span id="MemberFunctionAddLValueReference_t"></span><details><summary><b>MemberFunctionAddLValueReference_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionAddLValueReference_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after adding the "&" reference-qualifier to the function if not already present (replacing the "&&" reference-qualifier if present). If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since reference-qualifiers apply to non-static member functions only).</details>

<span id="MemberFunctionAddRValueReference_t"></span><details><summary><b>MemberFunctionAddRValueReference_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionAddRValueReference_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after adding the "&&" reference-qualifier to the function (replacing the "&" reference-qualifier if present). If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since reference-qualifiers apply to non-static member functions only).</details>

<span id="MemberFunctionAddVolatile_t"></span><details><summary><b>MemberFunctionAddVolatile_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionAddVolatile_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after adding the "volatile" cv-qualifier to the function if not already present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since "volatile" applies to non-static member functions only).</details>

<span id="MemberFunctionRemoveConst_t"></span><details><summary><b>MemberFunctionRemoveConst_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionRemoveConst_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after removing the "const" cv-qualifier from the function if present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since "const" applies to non-static member functions only so will never be present otherwise).</details>

<span id="MemberFunctionRemoveCV_t"></span><details><summary><b>MemberFunctionRemoveCV_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionRemoveCV_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after removing both the "const" AND "volatile" cv-qualifiers from the function if present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since "const" and "volatile" apply to non-static member functions only so will never be present otherwise).</details>

<span id="MemberFunctionRemoveReference_t"></span><details><summary><b>MemberFunctionRemoveReference_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionRemoveReference_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after removing the "&" or "&&" reference-qualifier from the function if present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since reference-qualifiers to non-static member functions only so will never be present otherwise).</details>

<span id="MemberFunctionRemoveVolatile_t"></span><details><summary><b>MemberFunctionRemoveVolatile_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using MemberFunctionRemoveVolatile_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after removing the "volatile" cv-qualifier from the function if present. If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since "volatile" applies to non-static member functions only so will never be present otherwise).</details>

<span id="MemberFunctionReplaceClass_t"></span><details><summary><b>MemberFunctionReplaceClass_t</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          typename NewClassT>`<br/>`using MemberFunctionReplaceClass_t;`<br/><br/>
>If "F" is a non-static member function, yields a type alias for "F" after replacing the class this function belongs to with "NewClassT". If "F" is a free function including static member functions, yields "F" itself (effectively does nothing since a "class" applies to non-static member functions only so will never be present otherwise - note that due to limitations in C++ itself, replacing the class for static member functions is not supported).</details>

<span id="RemoveVariadicArgs_t"></span><details><summary><b>RemoveVariadicArgs_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using RemoveVariadicArgs_t;`<br/><br/>
>If "F" is a variadic function (its last arg is "..."), yields a type alias for "F" after removing the "..." from the argument list. All non-variadic arguments (if any) remain intact (only the "..." is removed).</details>

<span id="RemoveNoexcept_t"></span><details><summary><b>RemoveNoexcept_t</b></summary>
> `template <TRAITS_FUNCTION_C F>`<br/>`using RemoveNoexcept_t;`<br/><br/>
>Type alias for "F" after removing "noexcept" from "F" if present</details>

<span id="ReplaceArgs_t"></span><details><summary><b>ReplaceArgs_t</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          typename... NewArgsT>`<br/>`using ReplaceArgs_t;`<br/><br/>
>Type alias for "F" after replacing all its existing non-variadic arguments with the args given by "NewArgsT" (a parameter pack of the types that become the new argument list). If none are passed then an empty argument list results instead, though if variadic args are present in "F" then they still remain intact (the "..." remains - read on). The resulting alias is identical to "F" itself except that the non-variadic arguments in "F" are completely replaced with "NewArgsT". Note that if "F" is a variadic function (its last parameter is "..."), then it remains a variadiac function after the call (the "..." remains in place). If you wish to explicitly add or remove the "..." as well then pass the resulting type to "AddVariadicArgs_t" or "RemoveVariadicArgs_t" respectively (either before or after the call to "ReplaceArgs_t"). Note that if you wish to remove specific arguments instead of all of them, then call "ReplaceNthArg_t" instead. Lastly, you can alternatively use "ReplaceArgsTuple_t" instead of "ReplaceArgs_t" if you have a "std::tuple" of types you wish to use for the argument list instead of a parameter pack. "ReplaceArgsTuple_t" is identical to "ReplaceArgs_t" otherwise (it ultimately defers to it)</details>

<span id="ReplaceArgsTuple_t"></span><details><summary><b>ReplaceArgsTuple_t</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          TUPLE_C NewArgsTupleT>`<br/>`using ReplaceArgsTuple_t;`<br/><br/>
>Identical to "ReplaceArgs_t" just above except the argument list is passed as a "std::tuple" instead of a parameter pack (via the 2nd template arg). The types in the "std::tuple" are therefore used for the resulting argument list. "ReplaceArgsTuple_t" is otherwise identical to "ReplaceArgs_t" (it ultimately defers to it)</details>

<span id="ReplaceCallingConvention_t"></span><details><summary><b>ReplaceCallingConvention_t</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          CallingConvention NewCallingConventionT>`<br/>`using ReplaceCallingConvention_t;`<br/><br/>
>Type alias for "F" after replacing its calling convention with the platform-specific calling convention corresponding to "NewCallingConventionT" (a "CallingConvention" enumerator declared in "TypeTraits.h"). For instance, if you pass  "CallingConvention::FastCall" then the calling convention on "F" is replaced with "\_\_attribute\_\_((cdecl))" on GCC and Clang, but "\_\_cdecl" on Microsoft platforms. Note however that the calling convention for variadic functions (those whose last arg is "...") can't be changed in this release. Variadic functions require that the calling function pop the stack to clean up passed arguments and only the "Cdecl" calling convention supports that in this release (on all supported compilers at this writing). Attempts to change it are therefore ignored. Note that you also can't change the calling convention of free functions to "CallingConvention::Thiscall" (including for static member functions since they're considered "free" functions). Attempts to do so are ignored since the latter calling convention applies to non-static member functions only. Lastly, please note that compilers will sometimes change the calling convention declared on your functions to the "Cdecl" calling convention depending on the compiler options in effect at the time (in particular when compiling for 64 bits opposed to 32 bits, though the "Vectorcall" calling convention *is* supported on 64 bits on compilers that support this calling convention). Therefore, if you specify a calling convention that the compiler changes to "Cdecl" based on the compiler options currently in effect, then "ReplaceCallingConvention_t" will also ignore your calling convention and apply "Cdecl" instead (since that's what the compiler actually uses)</details>

<span id="ReplaceNthArg_t"></span><details><summary><b>ReplaceNthArg_t</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          std::size_t N,`<br/>`          typename NewArgT>`<br/>`using ReplaceNthArg_t;`<br/><br/>
>Type alias for "F" after replacing its (zero-based) "Nth" argument with "NewArgT". Pass "N" via the 2nd template arg (i.e., the zero-based index of the arg you're targeting), and the type you wish to replace it with via the 3rd template arg ("NewArgT"). The resulting alias is therefore identical to "F" except its "Nth" argument is replaced by "NewArgT" (so passing, say, zero-based "2" for "N" and "int" for "NewArgT" would replace the 3rd function argument with an "int"). Note that "N" must be less than the number of arguments in the function or a "static_assert" will occur (new argument types can't be added using this trait, only existing argument types replaced). If you need to replace multiple arguments then recursively call "ReplaceNthArg_t" again, passing the result as the "F" template arg of "ReplaceNthArg_t" as many times as you need to (each time specifying a new "N" and "NewArgT"). If you wish to replace all arguments at once then call "ReplaceArgs_t" or "ReplaceArgsTuple_t" instead. Lastly, note that if "F" has variadic arguments (it ends with "..."), then these remain intact. If you need to remove them then call "RemoveVariadicArgs_t" before or after the call to "ReplaceNthArg_t"</details>

<span id="ReplaceReturnType_t"></span><details><summary><b>ReplaceReturnType_t</b></summary>
> `template <TRAITS_FUNCTION_C F,`<br/>`          typename NewReturnTypeT>`<br/>`using ReplaceReturnType_t;`<br/><br/>
>Type alias for "F" after replacing its return type with "NewReturnTypeT</details>


