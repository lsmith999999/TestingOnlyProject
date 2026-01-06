# Module support in C++20 or later (experimental)

> [!NOTE]
> *This file describes how to use the module version of "*FunctionTraits*" and is not required reading unless you either wish to use the modules version (most don't at this writing), and/or you're using "*import std*" in your own code and wish for "*FunctionTraits*" to use it as well (instead of #including the "*std*" headers - if so then simply #define the macro *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* which is described in detail later on). The documentation that follows is necessarily long however due to the nature of C++ modules (a deep subject), and in particular because it provides detailed information about the C++ module compilation process in general (independent of the "*FunctionTraits*" library, which saves you the trouble of researching it yourself). To this end, it assumes you're already familiar with the basics of C++ modules (it doesn't describe what they are or how to create them), but does demonstrate how to compile modules in extensive detail (with the ultimate focus of using the module version of "*FunctionTraits*"). If you're already familiar with how to compile modules however then you can simply read the following preamble only, ignoring the lengthy compiler-specific [FunctionTraits Module Examples](#functiontraitsmoduleexamples) that immediately follow it. The following introduction is sufficient to use the module version of "*FunctionTraits*" for those already familiar with how to compile C++ modules in general (but this introduction is still required reading).*

Note that you can optionally use the module version of "*FunctionTraits*" if you're targeting C++20 or later (modules aren't available in C++ before that). Module support is still experimental however since no compiler nor "*CMake*" fully supports them yet (now years after modules were introduced but the situation has improved in recent times). The level of support also differs depending on the compiler. For instance, in current versions of GCC and Clang, an "*import*" statement must not normally precede any *#include* statement or compilation will (or might) fail with a lot of cryptic errors (in particular *#include* statements from the "*std*" headers themselves). For GCC it's unlikely the situation will be fixed anytime soon (based on feedback from GCC itself). For Clang the situation remains unknown but see [here](https://clang.llvm.org/docs/StandardCPlusPlusModules.html#including-headers-after-import-is-not-well-supported) at this writing ("*Including headers after import is not well-supported*", though Clang will presumably remove this if the situation is ever corrected - see Clang issue [61465](https://github.com/llvm/llvm-project/issues/61465)). The Intel oneAPI compiler also isn't supported in "*CMake*" since "*CMake*" itself doesn't support it yet at this writing (likely because Intel probably doesn't support it yet). All existing compilers however (and "*CMake*") currently have caveats in their documentation that modules still have certain limitations and aren't fully supported yet (sometimes citing the term "*experimental*"). Testing shows that the technology seems to be reasonably stable however, within the limitations of each compiler's currently documented support (the module version of "*FunctionTraits*" itself seems to work without any significant issues for instance). The situation remains tenuous though (a GCC issue prevented the module version of "*FunctionTraits*" from initially working for instance but has now been fixed by GCC).

In addition to supporting modules in C++20, the "*std*" library is also available as a module in C++23 (see the official document for this feature [here](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf)). This occurs in the form of "*import std*" and/or "*import std.compat*" statements (usually the former for most), though support for this is also relatively new at this writing and official documentation for it still quite limited. Also note that not all supported compilers may #define the C++23 feature macro [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) yet, indicating that "*import std*" and "*import std.compat*" are available. Where they are available however compilers generally consider them experimental (to some extens) so "*FunctionTraits*" won't rely on them without your explicit permission first (via the transitional *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* macro described shortly).

As a result of all these issues and others, module support for "*FunctionTraits*" is therefore considered experimental until all compilers fully support both C++ modules (a C++20 feature), and (though less critically), the importing of the standard library in C++23 (again, via "*import std*" and "*import std.compat*"). The instructions that follow therefore reflect the incomplete state of module support in all compilers at this writing, and are therefore (unavoidably) longer than they should be (for now). Don't be discouraged however, as they're simpler than they first appear (much of the documentation exists only to provide sometimes hard-to-find information about the C++ module build process in general, though with an emphasis on "*FunctionTraits*").fs

Note that the following instructions will be condensed and simplified in a future release however, once all compilers are fully compliant with the C++20 and C++23 standard (in their support of modules). For now, because they're not fully compliant, the instructions below need to deal with the situation accordingly, such as the [STDEXT\_IMPORT\_STD\_EXPERIMENTAL](#stdext\_import_std_experimental) macro described further below, which is a transitional macro only. It will be removed in a future release.


To use the module version of "*FunctionTraits*" in general (for all compilers), you need to do the following (note that compiler-specific details follow later):

1. Add the primary module interface files "*FunctionTraits.cppm*" and "*CompilerVersions.cppm*" from this repository to your project, which builds the modules "*FunctionTraits*" and "*CompilerVersions*" respectively (corresponding to "*FunctionTraits.h*" and "*CompilerVersions.h*" described in the [Usage](#usage) section earlier - both "*.h*" files are still required however as each module simply defers to its "*.h*" file to implement the module). Note that you're free to change the "*.cppm*" extension to whatever you require but only if your particular compiler supports it. Microsoft developers may wish to change theirs to "*.ixx*" for instance, the default extension in MSVC, but it's not mandatory (more on this in the [Microsoft](#msvcmodulecompilationexample) section later). Clang developers however should normally keep the "*.cppm*" extension, or any of the other supported module extensions (see Clang [File name requirements](https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#file-name-requirements)). Regardless of the module's name, you can then import each module wherever you need it (read up on C++ modules for details). You can do this either using an "*import*" statement as would normally be expected (normally just "*import FunctionTraits*" - this is how modules are normally imported in C++), or by continuing to *#include* the header itself, normally just *#include "FunctionTraits.h"* (again, as described in the [Usage](#usage) section earlier). In the latter case, the use of *#include "FunctionTraits.h"* is provided as a convenience only. Doing so not only imports "*FunctionTraits*" for you as described in item 2 below (the module version of *#include "FunctionTraits.h"* implements an "*import FunctionTraits*" statement instead of #including the actual code as it would normally do), but it also has the benefit of making all public macros in "*FunctionTraits.h*" available as well (should you require any of them). This is the only reason you might choose to *#include "FunctionTraits.h*" in the module version instead of "*import FunctionTraits*" directly (more on this shortly).
2. #define the macro *STDEXT\_USE\_MODULES* when you build your project (add this to your project's build settings in whatever way it requires). Doing so changes the behavior of both "*FunctionTraits.h*" and "*CompilerVersions.h*" (again, "*FunctionTraits.h*" automatically #includes "*CompilerVersions.h*" - see [Usage](#usage) section), so that instead of declaring all C++ declarations as each header normally would (when *STDEXT\_USE\_MODULES* isn't #defined), each header simply imports the module instead (e.g., *#include "FunctionTraits.h"* simply issues an "*import FunctionTraits*" statement though this can be disabled if required - see footnote[^1]). All other C++ declarations in the file are then preprocessed out except for (public) macros, since macros aren't exported by C++ modules (so when required, header files that #define macros must still be #included in the usual C++ way - implementors of such headers must therefore avoid [ODR](https://en.cppreference.com/w/cpp/language/definition.html) violations and "*FunctionTraits*" does). Therefore, by #including "*FunctionTraits.h*" in the module version (i.e., when  *STDEXT\_USE\_MODULES* is #defined), you're effectively just creating an "*import FunctionTraits*" statement (which also imports "*CompilerVersions*"), but also #defining all public macros in the header as well (including those in "*CompilerVersions.h*" - more on these macros shortly). All other declarations in the file are preprocessed out as noted (they're not needed because the "*import FunctionTraits*" statement itself makes them available). Note that if *STDEXT\_USE\_MODULES* isn't #defined however, then an error to this effect will be displayed when the modules themselves are compiled. This is by design since it defeats the purpose of using modules if you attempt to compile the "*FunctionTraits*" modules in your program but *STDEXT\_USE\_MODULES* isn't #defined (since this macro notifies the "*.h*" files that they should defer to the modules as just described). Note that if you don't use any of the macros in "*FunctionTraits.h*" or "*CompilerVersions.h*" in your code however (again, more on these macros shortly), then you can simply apply your own "*import*" statement as usual, which is normally the natural way to do it (and #including the header instead is even arguably misleading since it will appear to the casual reader of your code that it's just a traditional header when it's actually applying an "*import*" statement instead, as just described). #including either of the library's "*.h*" files however ("*FunctionTraits.h*" only usually) to pick up the "*import*" statement instead of directly applying "*import*" yourself has the benefit of #defining all macros as well should you ever need any of them (though you're still free to directly import the module yourself at your discretion - redundant "*import*" statements are harmless if you directly code your own "*import FunctionTraits*" statement ***and*** *#include "FunctionTraits.h"* as well, since the latter also applies its own "*import FunctionTraits*" statement as described). Note that macros aren't documented in this README file however (with the exception of the member function detection macros described [here](README.md#determiningifamemberfunctionexists)), since the focus of this documentation is on "*FunctionTraits*" itself (the subject of this GitHub repository). Both "*FunctionTraits.h*" and "*CompilerVersions.h*" contain various support macros however, such as the *TRAITS\_FUNCTION\_C* macro in "*FunctionTraits.h*" (see this in the [Helper templates](README.MD#helpertemplates) section of the library's main "*README.md*" file), and the "*Compiler Identification Macros*" and "*C++ Version Macros*" in "*CompilerVersions.h*" (a complete set of macros allowing you to determine which compiler and version of C++ is running - see these in "*CompilerVersions.h*" for details). Other useful macros also exist. "*FunctionTraits*" itself utilizes these macros for its own internal needs as required but end-users may wish to use them as well (for their own purposes). Lastly, note that as described in the [Usage](README.md#usage) section of the library's main README.md file, #including "*FunctionTraits.h*" automatically picks up everything in "*CompilerVersions.h*" as well since the latter is a dependency, and this behavior is also extended to the module version (though if you directly "*import FunctionTraits*" instead of #include "*FunctionTraits.h*", it automatically imports module "*CompilerVersions*" as well but none of the macros in "*CompilersVersions.h*" will be available, again, since macros aren't exported by C++ modules - if you require any of the macros in "*CompilersVersions.h*" then you must *#include "FunctionTraits.h"* instead, or alternatively just *#include "CompilersVersions.h"* directly).
<a name="STDEXT_IMPORT_STD_EXPERIMENTAL*"></a>
3. If targeting C++23 or later (the following macro is ignored otherwise), you can optionally #define the macro *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* when you build your project (add this to your project's build settings in whatever way it requires). This macro is transitional only as a temporary substitute for the C++23 feature macro [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) (used to indicate that "*import std*" and "*import std.compat*" are supported). If *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* is #defined (in C++23 or later), then the "*FunctionTraits*" library will use an [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) statement to pick up all of its internal dependencies from the "*std*" library instead of #including each individual "*std*" header it depends on. This is normally recommended in C++23 or later since it's much faster to rely on [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) than to #include each required header from the "*std*" library (the days of doing so will likely become a thing of the past). Note that if you #define *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* then it's assumed that [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) is in fact supported by your app or many cryptic compilers errors will likely occur (if targeting C++23 or later - again, the macro is ignored otherwise). The use of [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) in an application is still relatively new and support for it still somewhat shaky. GCC and Clang for instance still require that #include statements from the "*std*" library precede calls to "*import*", so if your app violates this (if your porting an older app to use modules in particular so it still relies on "*std*" headers in some cases), then you will very likely get many compiler errors (often citing redefinition errors but others as well). If targeting Microsoft platforms however then your project must currently rely on "*import std*" everywhere since you can't (currently) mix headers from the "*std*" library and "*import std*" (until Microsoft corrects this). Given the experimental nature of C++ modules at this writing, mixing "*import std*" and headers from the "*std*" library should be done with caution for other compilers as well. In any case, please see the [Footnotes](README.md#footnotes) (1 through 4) for module versioning information for each supported compiler (which includes version support information for "*import std*" and "*import std.compat*"). Note that *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* will be removed in a future release however once all supported compilers fully support the [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) feature macro (which will then be used instead). For now the latter macro either isn't #defined on all supported platforms at this writing (most don't support "*import std*" and "*import std.compat*" yet), or if it is #defined such as in recent versions of MSVC, "*FunctionTraits*" doesn't rely on it yet (for the reasons described but see the comments preceding the check for *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* in "*FunctionTraits.h*" for complete details). Instead, if you wish for "*FunctionTraits*" to rely on [import std](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2465r3.pdf) then you must grant explicit permission by #defining *STDEXT\_IMPORT\_STD\_EXPERIMENTAL*. The C++ feature macro [\_\_cpp\_lib\_modules](https://en.cppreference.com/w/cpp/feature_test#cpp_lib_modules) itself is completely ignored by the "*FunctionTraits*" library in this release.

<a name="FunctionTraitsModuleExamples"></a>
### *FunctionTraits* Module Examples

The following module examples will help get you started, though experienced module developers need not consult these examples if they don't wish to. The section above is sufficient to get started. The compiler-specific examples follow further below, but each utilizes the following source file, "*Test.cpp*", a client of "*FunctionTraits*". The compiler-specific examples then show how to build this file and the "*FunctionTraits*" module it relies on (for each supported compiler).

Given this file, a client of "*FunctionTraits*":

<a name="TestCppModule"></a>
```C++

// Test.cpp

/////////////////////////////////////////////////////
// When compiling for the module version, you can
// optionally continue to use #include here instead
// of changing it to "import FunctionTraits". In the
// module version, "FunctionTraits.h" changes its
// behavior to call "import FunctionTraits" for you
// instead of #including the header's code as it
// does in the non-module version. That code is now
// picked up from the call to "import FunctionTraits"
// but the header also continues to #include all
// public macros as well. You're free to change the
// following call to "import FunctionTraits" if you
// wish, but if you require any macros such as the
// library's function detection macros (or any of
// the macros #defined in "CompilerVersions.h", such
// as the "C++ Version Macros" though these macros
// aren't officially documented at the library's
// main site - you can still use them however), then
// you'll have to #include "FunctionTraits.h" (since
// modules don't export macros in C++ so the header
// itself is still required to pick them up - see
// footnote 11 if you wish to #include "FunctionTraits.h"
// to pick up its macros but wish to suppress its
// call to "import FunctionTraits" if you have reason
// to do so).
/////////////////////////////////////////////////////
#include "FunctionTraits.h"

int main()
{
    using namespace StdExt;

    double SomeFunc(int, float);

    using F = decltype(SomeFunc);

    constexpr auto SomeFuncArg3TypeName = ArgTypeName_v<F, 2>;
}
```

You can compile the above file as seen in the examples that follow further below. Just choose your target compiler but if you're not already familiar with how C++ import statements like "*import FunctionTraits*" actually go about looking for the specified module ("*FunctionTraits*" in this case), then you should continue reading this section first (i.e., if you're not already familiar with "*BMI*" files, AKA "*CMI*" files, which are explained below). Otherwise you can immediately proceed to the following link(s) you require below.

1. [CMake](#cmakemodulecompilationexample) (most non-MSVC developers will rely on this)
2. [GCC](#gccmodulecompilationexample) (shell example but most GCC developers will rely on "*CMake*" just above)
3. [Microsoft](#msvcmodulecompilationexample) (Visual Studio instructions *and* pure MSVC example - most MSVC developers will rely on Visual Studio)
4. [Clang](#clangmodulecompilationexample) (shell example but most Clang developers will rely on "*CMake*" in 1 above)

<a name="ModuleDependencies"></a>
Note that all compiler examples above except for [CMake](#cmakemodulecompilationexample) (since it's not a compiler) demonstrate how to compile modules at the command line itself (MSVC additionally describes how to do it in Visual Studio). However, the command line examples aren't realistic since most developers will normally (usually) rely on a professional build system instead, usually (normally) "*CMake*" for non-MSVC and Visual Studio for MSVC (though the "*CMake*" example can be used for MSVC as well, though usually just to create its Visual Studio solution). The command line examples however are shown merely to provide you with insight into what "*CMake*" and Visual Studio do behind the scenes (automating what the command line examples do and more). A professional build system is normally required however due to the complexity of managing most projects. This is obvious to most developers but a professional build system is even more critical for module-based projects opposed to traditional non-module based ones. Module-based projects require dependency checks that non-module based projects don't, because when you call something like "*import FunctionTraits*" in a client source file like [Test.cpp](#testcppmodule), the compiler needs to locate the code for module "*FunctionTraits*" in order to satisfy the call to "*import FunctionTraits*". Therefore, with the introduction of modules in C++, for the first time, C++ files like [Test.cpp](#testcppmodule) now have dependencies on other C++ files like "*FunctionTraits.cppm*", the module file for module "*FunctionTraits*". This is radically different than traditional header dependencies such as "*FunctionTraits.h*", where the latter file is simply located and incorporated into the current file at the point of  call (where the #include statement occurs). Module dependencies however are a different breed entirely, and need to be processed on-the-fly as they're encountered (i.e., when calls to "*import*" are encountered), in order to satisfy client calls to the exported code in the module files for those "*import*" statements (the module files are not simply #included into the client files like traditional headers). To this end, compilers therefore need to scan all file dependencies to determine which source files (modules) must be "*compiled*" first, such as "*FunctionTraits.cppm*", before client files like [Test.cpp](#testcppmodule) can import them. Note that that the term "*compiled*" here doesn't mean "*compiled*" into an object file at this stage, but compiled into a format (file) so that its exported declarations can be made available when the module appears in an "*import*" statement). Prior to modules, this wasn't required but modules now make the entire build process much more complicated than a traditional, non-module based app (because of module dependency issues).

For this reason, when a module is compiled, two types of output files now result (in all compilers supported by "*FunctionTraits*"). Before modules, only an object file would be compiled from a given C++ source file but with modules, not only is an object file still required, but a "*BMI*" ("*Binary Module Interface*") file is now required as well (AKA [Built Module Interface](https://clang.llvm.org/docs/StandardCPlusPlusModules.html#built-module-interface), or sometimes called "*CMI*" ("*Compiled Module Interface*") depending on the source - search for "*CMI*" [here](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Modules.html) for instance). A "*BMI*" file can be thought of as similar to an object file, but whereas an object file is required at link time (i.e., you pass an object file to your linker), a "*BMI*" file is required by the compiler itself at "*import*" time (i.e., when an "*import*" statement is encountered). "*BMI*" files are just compiled versions of C++ modules such as "*FunctionTraits.cppm*" (the main module for this library), containing the necessary information to satisfy client calls to "*import*" (such as "*import FunctionTraits*"). Each particular compiler handles "*BMI*" files in its own unique way however since the C++ standard doesn't address it. "*BMI*" files are compiler-specific implementation details but the basic process is the same for all mainstream compilers. When a call to import a module is encountered, such as "*import FunctionTraits*", the compiler needs to find the "*BMI*" file for module "*FunctionTraits*". That file must already exist when the compiler sees "*import FunctionTraits*" (lazy, on-the-fly creation of "*BMI*" files aren't supported by any compiler), but how a given compiler creates the "*BMI*" file and locates it is compiler-specific (as noted). Each of the examples that follow further below describes the process accordingly for [GCC](#gccmodulecomilationexample), [Clang](#clangmodulecompilationexample) and [MSVC](#msvcmodulecomilationexample). Note that [CMake](cmakemodulecompilationexample) and Visual Studio simply wrap the compiler-specific details for you. When manually compiling yourself at the command line however, you control the process yourself, normally through various compiler switches or in some cases some things may be hardwired into a given compiler. Regardless of how it's done however, each compiler creates a "*BMI*" file to satisfy client calls to "*import*". The "*BMI*" file must also be created before compiling any source file that tries to "*import*" the corresponding module, since the call to "*import*" itself needs to locate the "*BMI*" file.

To this end, note that the term "*BMI*" is just one of several unofficial generic terms used to refer to such a file in a given compiler's implementation (i.e., the file used to satisfy calls to "*import*"). "*CMI*" and other terms also exist as previously noted (and even the definition of the "*B*" in "*BMI*" may mean "*Binary*" or "*Built*" depending on the source). Note that each compiler also applies its own unique name in the file extensions it uses for "*BMI*" files. GCC uses the extension "*.gcm*" ("*Gcc Compile Module*"), Clang uses "*.pcm*" ("*PreCompiled Module*"), and MSVC uses "*.ifc*" (strangely, Microsoft doesn't seem to document what it officially stands for in any mainstream documentation at this writing but some have reported it stands for "*Interface File Compiled*" or "*Interface File for C++*"). These are all just "*BMI*" files for the given compiler however, using the nomenclature that each one has chosen.

The process of compiling a module-based app therefore necessarily involves the creation of "*BMI*" files regardles of the compiler-specific "*BMI*" extensions each compiler uses. Conceptually however all the compilers rely on the same basic technique to handle modules, i.e., each creates a "*BMI*" file to handle calls to "*import*" so when manually compiling such an app (opposed to using an automated system, again, "*CMake*" and Visual Studio usually, which create the "*BMI*" files for you), you only need to wade through the details of how each one does it (creates the "*BMI*" files as well as locate them to satisfy calls to "*import*"). Various compiler switches exist to do this as noted, each informing your compiler where to create its "*BMI*" files when compiling modules, and where to look for them when processing calls to "*import*" in module clients (and in some cases there may be multiple ways to do either - also note that modules which "*import*" other modules are also module clients in their own right). There may also be defaults if you don't pass any explicit switch to override the defaults. The behavior and requirements of the entire process are completely unique for each given compiler.

Lastly, note that the examples below show some of the mainstream ways of handling modules at the command line, but they are not exhaustive since a complete explanation is beyond the scope of this documentation (consult each compiler for details). The examples are simply meant to help you acquire a basic understanding. Again, as previously noted, compiling from the command line is not a realisitic option for most users. Among other things, doing so requires you ensure all "*BMI*" files are built first or client calls to "*import*" won't find them. Maintaining the correct build order is normally not something most developers are manually going to do themselves (in addition to the usual tracking to ensure C++ source files are only compiled when out-of-date with respect to their object files - this is completely independent of modules themselves however).

The need for a professional build system is therefore normally required to automate the process of dealing with module dependencies in addition to the other benefits such systems provide. For further details on module dependencies in particular, research document P1689R5[^2] (*Format for describing dependencies of source files*). Again, most will usually rely on "*CMake*" or "*Visual Studio*", so the command line examples below merely demonstrate what these tools do behind the scenes. See the [CMake](#cmakemodulecompilationexample) and [MSVC](#msvcmodulecompilationexample) examples for actual (real-world) scenarios that most *will* rely on (both of them providing minimal examples to get you started).

<a name="CMakeModuleCompilationExample"></a>
### CMake module compilation (official CMake documentation [here](https://www.kitware.com/import-cmake-the-experiment-is-over/))
Most non-MSVC developers (those targeting GCC, Clang and Intel oneAPI) will normally rely on "CMake" for their builds (directly or indirectly). The examples that follow for those compilers later on merely show the low-level details for compiling at the terminal itself, which provides insight into how modules are handled by those compilers (and what "CMake" ultimately does for you and more). The "*CMakeList.txt*" sample file that follows a bit further below handles all compilers supported by "*FunctionTraits*" except for Intel oneAPI since it's not yet supported by CMake at this writing (so it can't target Intel oneAPI yet). For details on this "CMakeList.txt" file in general (how to create such a file to target modules), see [import CMake; the Experiment is Over!](https://www.kitware.com/import-cmake-the-experiment-is-over/) (from the developers of "CMake" itself). For details on how to support [import std](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) in "*CMakeLists.txt*" (again, from the developers of "CMake" itself), see [import std in CMake 3.30](https://www.kitware.com/import-std-in-cmake-3-30/) The following provides an optional "CMake" variable (arg) to activate [import std in CMake 3.30](https://www.kitware.com/import-std-in-cmake-3-30/) (read on).

Note that the following file has no bells and whistles beyond the basics to demonstrate how to build the module version of *FunctionTraits* including support for [import std](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) if requested (read on). Some things like [CMAKE\_CXX\_STANDARD](https://cmake.org/cmake/help/latest/variable/CMAKE_CXX_STANDARD.html) are therefore hardcoded to keep the example simple (to a minimum). The example "*CMakeLists.txt*" file simply performs the minimum calls to target whatever compiler you pass to "*CMake*", usually via its [CMAKE\_CXX\_COMPILER](https://cmake.org/cmake/help/latest/variable/CMAKE_LANG_COMPILER.html#variable:CMAKE_%3CLANG%3E_COMPILER) variable (switch). See the example that immediately follows the following "*CMakeLists.txt*" file.

If you wish to support [import std](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) in your code then pass (define) the custom *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* variable when you call "*CMake*". If you do and you're targeting Clang, note that you **must** also pass [CMAKE\_CXX\_FLAGS](https://cmake.org/cmake/help/latest/variable/CMAKE_LANG_FLAGS.html)="[-stdlib](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-stdlib)*=libc++*" since the [stdlibc++](https://gcc.gnu.org/onlinedocs/libstdc++/) library from GCC isn't currently supported at this writing (in "*CMake*" builds where "*import std*" is required). The "CMake" documentation on [import std](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) currently indicates it is supported but testing shows it fails at this writing. That's because "CMake" ultimately relies on a file from Clang called "*libc++.modules.json*" which GCC doesn't provide yet (when targeting [stdlibc++](https://gcc.gnu.org/onlinedocs/libstdc++/) but presumably it will at some point). That file stores the required information for "CMake" to locate the "*std.cppm*" file described in the *Clang* example earlier, and can be obtained in recent versions of *Clang* via its [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path) option. When targeting [stdlibc++](https://gcc.gnu.org/onlinedocs/libstdc++/) however, "CMake" fails to locate the file and returns an error with the failure reason set to "*libstdc++.modules.json resource does not exist*". However, you could theoretically circumvent the issue if you manually add the "*std.cppm*" file provided by Clang to your "*CMakeLists.txt*" file and process it yourself. This was demonstrated in the Clang example earlier but it's now your responsibility to incorporate that logic into the "*CMakeLists.txt*" file (but not demonstrated here since most should rely on the native "CMake" techniques to handle this - it's unclear however when GCC will start supporting [stdlibc++](https://gcc.gnu.org/onlinedocs/libstdc++/) for "*CMake*" builds requiring [import std](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) in spite of the "CMake" documentation showing it's already supported - again, testing shows it's not because GCC doesn't provide "*libc++.modules.json*" yet as described, but Clang's "*std.cppm*" file still exists as a fallback if you absolutely require it at the expense of having to processing it manually in "CMake" for now).

Lastly, note that the following "*CMakeLists.txt*" remains potentially brittle due to the experimental nature of C++ modules in general (across all compilers and "CMake" itself). Modules still remain in a state of flux at this writing, and while testing shows this "*CMakeLists.txt*" works for current versions of GCC, Clang and MSVC (again, modules in Intel oneAPI are not yet supported by "CMake"), the situation is tenuous. Your own mileage may vary depending on your version of these compilers and particular environment. If failure occurs however, it may be a matter of tweaking the situation to correct it, such as updating the UUID passed to [CMAKE\_EXPERIMENTAL\_CXX\_IMPORT_STD](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) seen in the "*CMakeLists.txt*" file below (which is subject to change by "CMake" itself). Actual errors running the file might also be corrected with minimal fuss by those already familiar with the esoteric details of using modules in "CMake". Info on the subject is often scattered, poorly documented and hard to come by. This "*CMakeLists.txt*" file does its best to follow the current "CMake" guidelines for modules including [import std](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html#import-std-support) when requested, though it does work around one or two known issues (such as when passing the [Visual Studio 17 2022](https://cmake.org/cmake/help/latest/generator/Visual%20Studio%2017%202022.html) generator to "*CMake*" and *STDEXT\_IMPORT\_STD\_EXPERIMENTAL*, since that generator doesn't respect the documented "CMake" guidelines at this writing itsel - without the workaround it fails).

✅ *Note, don't stress at the length of this file, it's only around 80 lines once the comments are removed!*

<a name="CMakeListsTxtExample"></a>
```cmake
######################################################
# STDEXT_IMPORT_STD_EXPERIMENTAL not passed (defined)
# by user? If not then support for "import std" isn't
# required. See:
# https://www.kitware.com/import-std-in-cmake-3-30/
######################################################
if(NOT DEFINED STDEXT_IMPORT_STD_EXPERIMENTAL)
    ###############################################
    # CMake 3.28 required to support modules. See
    # the following:
    #     https://www.kitware.com/import-cmake-the-experiment-is-over/ (search for 3.28)
    #     https://cmake.org/cmake/help/latest/command/target_sources.html#command:target_sources (search for CXX_MODULES)
    ###############################################
    cmake_minimum_required(VERSION 3.28)

    ##########################################
    # Set the version of C++ for the project
    # (C++20 required to support modules)
    ##########################################
    set(CMAKE_CXX_STANDARD 20)
else() # Support for "import std" required ...
    ###########################################################
    # CMAKE_CXX_EXTENSIONS must be ON to support "import std"
    # at this writing. Otherwise CMake currently fails with
    # the following errors (followed by a series of compiler
    # errors):
    #
    #     error: GNU extensions was enabled in precompiled file 'CMakeFiles/__cmake_cxx23.dir/std.pcm' but is currently disabled
    #     error: module file CMakeFiles/__cmake_cxx23.dir/std.pcm cannot be loaded due to a configuration mismatch with the current compilation [-Wmodule-file-config-mismatch]
    #
    # The situation should be reviewed again in a future
    # release (module support by CMake still officially
    # experimental)
    ###########################################################
    if(DEFINED CMAKE_CXX_EXTENSIONS AND NOT CMAKE_CXX_EXTENSIONS)
        # Abort processing!
        message(FATAL_ERROR "❌ STDEXT_IMPORT_STD_EXPERIMENTAL was passed but CMAKE_CXX_EXTENSIONS is OFF and must be ON to support \"import std\". Please pass -DCMAKE_CXX_EXTENSIONS=ON or undefine it.")
    endif()

    message(STATUS "⚠️ STDEXT_IMPORT_STD_EXPERIMENTAL variable passed to support "
                   "\"import std\". Note that \"import std\" is still officially "
                   "experimental by CMake and most compilers, and can potentially "
                   "result in build and/or runtime errors (build errors in particular "
                   "can be a common occurrence in GCC for instance). A subsequent "
                   "warning about this may also be displayed by CMake itself at "
                   "this writing (further below), unless you've passed the -Wno-dev "
                   "option to suppress it.")

    ###############################################
    # CMake 3.30 required to support "import std".
    # See the following:
    #     https://www.kitware.com/import-std-in-cmake-3-30/
    #     https://cmake.org/cmake/help/latest/prop_tgt/CXX_MODULE_STD.html
    ###############################################
    cmake_minimum_required(VERSION 3.30)

    ###########################################
    # Set the version of C++ for the project
    # (C++23 required to support "import std")
    ###########################################
    set(CMAKE_CXX_STANDARD 23)

    set(CMAKE_EXPERIMENTAL_CXX_IMPORT_STD
        #####################################################
        # This specific UUID changes as experimental support
        # evolves. See `Help/dev/experimental.rst` in the
        # CMake source corresponding to your CMake build for
        # the exact value to use. For instance, search for
        # CMAKE_EXPERIMENTAL_CXX_IMPORT_STD  here:
        # https://gitlab.kitware.com/cmake/cmake/-/blob/master/Help/dev/experimental.rst
        #####################################################
        "d0edc3af-4c50-42ea-a356-e2862fe7a444")

    message (STATUS "⚠️ This is just a harmless notification and may be ignored unless "
             "a subsequent error occurs from CMake itself (not displayed by all generators "
             "however - Ninja normally will). Setting CMAKE_EXPERIMENTAL_CXX_IMPORT_STD to "
             "\"${CMAKE_EXPERIMENTAL_CXX_IMPORT_STD}\". This may be changed by CMake in a "
             "future version however (since its support for modules is still experimental), "
             "requiring it be changed in this file as well. For details, search for "
             "CMAKE_EXPERIMENTAL_CXX_IMPORT_STD here: "
             "https://gitlab.kitware.com/cmake/cmake/-/blob/master/Help/dev/experimental.rst")

    #######################################################
    # Testing shows that Visual Studio generators (only
    # "Visual Studio 17 2022" or later at this writing are
    # relevant for modules) don't follow the CMake rules
    # for modules so the following alternate logic is req'd
    # (which does work, at least for now). The "else()"
    # clause below handles the normal case documented by
    # CMake at this writing (for all other generators).
    # The situation remains brittle however until all
    # compilers and CMake itself become # more stable
    # (in their handling of modules which remains
    # experimental until then).
    #
    # Note that according to CMake's docs on generators,
    # here are the possibilities for Visual Studio at this
    # writing (click "Visual Studio Generators" seen at the
    # top of the link - again, for modules, only Visual
    # Studio 17 2022 or later is relevant):
    #
    #    Visual Studio Generators
    #    https://cmake.org/cmake/help/latest/manual/cmake-generators.7.html#visual-studio-generators
    #
    #    Visual Studio 6
    #    Visual Studio 7
    #    Visual Studio 7 .NET 2003
    #    Visual Studio 8 2005
    #    Visual Studio 9 2008
    #    Visual Studio 10 2010
    #    Visual Studio 11 2012
    #    Visual Studio 12 2013
    #    Visual Studio 14 2015
    #    Visual Studio 15 2017
    #    Visual Studio 16 2019
    #    Visual Studio 17 2022
    #    Visual Studio 18 2026
    #
    # The following does a simple check only for the above
    # generators by simply checking for those starting with
    # "Visual Studio" (case-sensitive and single space
    # separating the words exactly as seen). Presumably
    # (hopefully) CMake will never deviate from this for
    # the existing strings above anyway (but likely future
    # ones as well so long as MSFT doesn't change things -
    # unlikely). This code could go further though,
    # extracting the MSVC version number and VS version
    # from the CMAKE_GENERATOR variable as well, such as
    # the following:
    #
    #     if(CMAKE_GENERATOR MATCHES "^Visual Studio ([0-9]+) ([0-9]+)")
    #         set(VS_GENERATOR_MAJOR "${CMAKE_MATCH_1}")
    #         set(VS_GENERATOR_YEAR  "${CMAKE_MATCH_2}")
    #         set(VS_GENERATOR_IS_DETECTED TRUE)
    #     else()
    #         set(VS_GENERATOR_IS_DETECTED FALSE)
    #     endif()
    #
    # It can be made even more precise though such as
    # verifying the year is 4 digits for instance but why
    # wouldn't it be (at least for the 2005 and later
    # strings above but almost nobody will be using any
    # versions that old anymore of course, let alone the
    # different formats for the even older versions seen
    # above):
    #
    # For our simple needs here though (it's just a sample
    # file), the following will do for now ...
    #######################################################
    if(CMAKE_GENERATOR MATCHES "^Visual Studio")
        #############################################
        # This can be done before or after project()
        # on MSVC. It works regardless of
        # CMAKE_EXPERIMENTAL_CXX_IMPORT_STD (called
        # above). See here as well:
        #
        #  https://gitlab.kitware.com/cmake/cmake/-/issues/25965#note_1560489
        #############################################
        set(CMAKE_CXX_SCAN_FOR_MODULES ON)

        ###################################################
        # Causes this error in the "Visual Studio 17 2022"
        # generator at this writing (and presumably future
        # versions though the situation could potentially
        # change):
        #
        #    CMake Error in CMakeLists.txt:
        #      The "CXX_MODULE_STD" property on the target "Test" requires that the
        #      "__CMAKE::CXX23" target exist, but it was not provided by the toolchain.
        #      Reason:
        #
        #         Unsupported generator: Visual Studio 17 2022
        ###################################################
        #set(CMAKE_CXX_MODULE_STD ON)
    else()
        set(CMAKE_CXX_MODULE_STD ON)
    endif()
endif()

##############################################
# Don't allow CMake to "decay" to an earlier
# version of C++ prior to CMAKE_CXX_STANDARD
##############################################
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

project(Test CXX)

##########################################
# Define a target name (variable) for
# passing to calls below (where required)
##########################################
set(EXEC_NAME Test)

#############################################
# IMPORTANT: In the module version, the
# following must precede the call to
# "target_sources" just below. Clang fails
# to build otherwise (GCC and MSVC build ok
# - go figure). Don't reverse the calls IOW!
#############################################
add_executable(${EXEC_NAME})

##############################################
# #define STDEXT_USE_MODULES which is always
# required by the module version of
# "FunctionTraits"
##############################################
target_compile_definitions(${EXEC_NAME}
                           PRIVATE STDEXT_USE_MODULES=)

# Caller passes this to support "import std ...
if(DEFINED STDEXT_IMPORT_STD_EXPERIMENTAL)
    ################################################
    # #define STDEXT_IMPORT_STD_EXPERIMENTAL which
    # is always required by the version of
    # "FunctionTraits" that supports "import std"
    # (the macro's value isn't relevant, only that
    # it's #defined but we'll pass the caller's
    # value in anyway, though it's usually empty).
    # See here:
    #    https://github.com/HexadigmSystems/FunctionTraits/#moduleusage
    target_compile_definitions(${EXEC_NAME}
                               PRIVATE STDEXT_IMPORT_STD_EXPERIMENTAL=${STDEXT_IMPORT_STD_EXPERIMENTAL})
endif()

#######################################
# Add this project's source files not
# including modules (we'll add those
# separately just below)
#######################################
target_sources(${EXEC_NAME}
               PRIVATE
                   # .cpp files
                   Test.cpp # FunctionTraits module client

                   # .h files (FunctionTraits headers)
                   CompilerVersions.h
                   FunctionTraits.h
              )

# Add "FunctionTraits" modules (.cppm files)
target_sources(${EXEC_NAME}
               PRIVATE
                   ########################################################
                   # Add all modules to the project. See the "Hello world"
                   # example here (search for the "target_sources" line
                   # in the example - we do the same):
                   #
                   #     https://www.kitware.com/import-cmake-the-experiment-is-over/
                   ########################################################
                   FILE_SET CXX_MODULES FILES
                       CompilerVersions.cppm
                       FunctionTraits.cppm
              )
```

To run this "CMakeList.txt" file (with only minimum "CMake" flags shown), just uncomment the call to "CMake" that your interested in below (for your particular compiler) and run it. The code always assumes you're targeting the same compiler on each run or you'll have to empty out the "_build_" folder seen below as well (if targeting another compiler). The code is intended as a very basic example only to keep things simple. Real world code will obviously be more elaborate.

```bash
#!/bin/bash
# Compile files (output to the "build" subdirectory)
mkdir -p "build"
cd build

#######################################################
# Delete cache to ensure clean configuration on each
# run (but if targeting another compiler then all
# other files still remain from the previous compiler -
# you'll want to clean these out first but this is
# just a simple example only).
#######################################################
rm -f CMakeCache.txt

# GCC
#cmake .. -DCMAKE_CXX_COMPILER=g++ -G Ninja -DCMAKE_CXX_STANDARD=20

# Clang
#cmake .. -DCMAKE_CXX_COMPILER=clang++ -G Ninja -DCMAKE_CXX_STANDARD=20

# Intel (not yet supported however until CMake itself supports Intel)
#cmake .. -DCMAKE_CXX_COMPILER=icpx -G Ninja -DCMAKE_CXX_STANDARD=20

# Visual Studio 2022
#cmake .. -DCMAKE_CXX_COMPILER="cl.exe" -G "Visual Studio 17 2022" -DCMAKE_CXX_STANDARD=20

# All compilers (after running the above command)
cmake --build .
```

<a name="GCCModuleCompilationExample"></a>
### GCC module compilation (official GCC documentation [here](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Modules.html))
Most GCC developers will likely rely on [CMake](#cmakemodulecompilationexample). The following example demonstrates how to compile modules in the terminal for learning purposes only usually (using pure *g++*). A professional build tool like [CMake](#cmakemodulecompilationexample) is normally required however to handle module dependency issues among others. The example that follows manually builds everything in the correct dependency order for instance but manually handling this isn't realisitic for most real-world apps (which would be tedious and error-prone). See the earlier discussion for details (see [here](#moduledependencies)).

In GCC, when an import statement such as "*import FunctionTraits*" is encountered, GCC will look for a file called "*gcm.cache/FunctionTraits.gcm*" to resolve it (recall that "*.gcm*" files are the BMI files created by GCC when you compile a module - see the earlier discussion in [FunctionTraits Module Examples](#functiontraitsmoduleexamples) for details). This is the default behavior but still evolving in GCC at this writing. See links in the Bash file comments just below for details ([Module Mapper](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Module-Mapper.html) in particular). When the default behavior is in effect (and it's unclear if GCC officially allows this to be changed at this writing - see [here](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=121963)), the directory "*gcm.cache*" in the latter file ("*gcm.cache/FunctionTraits.gcm*") is automatically created by GCC during module compilation, so no special processing to create the directory itself is required. Nor do you need to explicitly create the "*.gcm*" file itself ("*FunctionTraits.gcm*" in this example). GCC will automatically create "*gcm.cache/FunctionTraits.gcm*" as a side-effect of compiling "*FunctionTraits.cppm*". You therefore simply need to compile each "*.cppm*" file no different than you would a normal (non-module) "*.cpp*" file (but always targeting C++20 or later since modules didn't exist previously). GCC only requires that you pass the flag [-fmodules](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#index-fmodules)[^3], and the "*FunctionTraits*" library itself also requires that you *#define STDEXT\_USE\_MODULES* (and *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* if you wish for "*FunctionTraits*" to rely on "*import std*" instead of #including the "*std*" headers it relies on - see both macros earlier for details).

Here's the minimum code (Bash script) to build the module version of [Test.cpp](#testcppmodule). Copy this to the same folder as [Test.cpp](#testcppmodule), along with the library's usual header files "*CompilerVersions.h*" and *FunctionTraits.h*", but additionally their corresponding module files "*CompilerVersions.cppm*" and *FunctionTraits.cppm*" (the modules are implemented using the header files). Then simply run the script from the same directory which creates the app called "*build/Test*" that you can then run (again, see [Test.cpp](#testcppmodule)).

```bash
#!/bin/bash

set -e # Exit if any command fails

########################################
# Trivial helper function to display
# any passed command and then run it
# (unsophisticated but meets our needs)
########################################
run() {
    echo "$1"
    eval "$1"
    echo
}

########################################################
# Compile all files which we output to the "build"
# subdirectory created just below (modules are compiled
# to object files no different than non-module files).
# Note however that each call to compile a module's
# ".cppm" file also creates the module's ".gcm" file
# automatically in a subdirectory GCC creates called
# "gcm.cache". This is the default in GCC at this
# writing but see the following for details (GCC's
# support for modules is still officially evolving at
# this writing):
#
#   Module Mapper
#   https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Module-Mapper.html
#
#   Compiled Module Interface
#   https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Compiled-Module-Interface.html
#
#   Bug 121963 - Make it easier to change the location of gcm.cache dir and document it
#   https://gcc.gnu.org/bugzilla/show_bug.cgi?id=121963
#
# Lastly, note that we compile all files in module
# dependency order so that when "import" statements are
# encountered in a given file, the compiler can locate
# its ".gcm" file since it's already been previously
# built. Build systems like "CMake" handle the order
# for you (see footnote 12 at the end of this file),
# but we're manually compiling the files ourselves here
# so we need to ensure they're compiled in the correct
# module dependency order (though we could and normally
# should rely on footnote 12 in a real-world app but it
# would obviously make this example much more
# complicated).
########################################################
mkdir -p "build"
run 'g++ -std=c++20 -fmodules -DSTDEXT_USE_MODULES -c "CompilerVersions.cppm" -o "build/CompilerVersions.o"'
run 'g++ -std=c++20 -fmodules -DSTDEXT_USE_MODULES -c "FunctionTraits.cppm" -o "build/FunctionTraits.o"'
run 'g++ -std=c++20 -fmodules -DSTDEXT_USE_MODULES -c "Test.cpp" -o "build/Test.o"'

# Link
run 'g++ -std=c++20 "build/CompilerVersions.o" "build/FunctionTraits.o" "build/Test.o" -o "build/Test"'

echo 'Done! (successful - created "build/Test")'
```

<a name="GccModuleCompilationSupportingImportStdExample"></a>
#### GCC module compilation supporting "*import std*"

The above code demonstrates how to support "*import FunctionTraits*" in a GCC program (using pure *g++*). If you wish to support "*import std*" as well, then several changes are required to the code just above. These are as follows (an updated example incorporating the following steps immediately follows):

1. Add the following call to compile the module version of the "*std*" library into its own "*.pcm*" file similar to "*CompilerVersions.pcm*" and "*FunctionTraits.pcm*" and above:
   ```bash
   run 'g++ -std=c++23 -fmodules -fmodule-only -c -fsearch-include-path "bits/std.cc"'
   ```

   The module version of the "*std*" library is no different than any other module in this respect (though it doesn't have to be explicitly linked in like other modules since it occurs implicitly via the vendor's libraries at link time[^4]). *GCC* ships the "*std*" module in a file called "*bits/std.cc*" relative to the *#include* path as specified via the [fsearch-include-path](https://gcc.gnu.org/onlinedocs/cpp/Invocation.html#index-fsearch-include-path) option seen in the call above (the latter link shows a very similar example in fact, though we also pass [fmodule-only](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#index-fmodule-only) to suppress creation of the object file which isn't required for the module version of the "*std*" library - it doesn't need to be explicitly linked since any required code from the "*std*" library is implicitly pulled in at link time[^4] - the [-c](https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html#index-c) option is still required however to prevent linking, though a bit odd since [fmodule-only](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#index-fmodule-only) is being passed so one might think that [-c](https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html#index-c) wouldn't be required, similar to Microsoft's [/ifcOnly](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#summary) option which doesn't require [/c](https://learn.microsoft.com/en-us/cpp/build/reference/c-compile-without-linking?view=msvc-170)). Note that "*bits/std.cc*" is hardcoded for now but it's unclear if GCC will be providing a more dynamic way of discovering this in a future release (their own example at [fsearch-include-path](https://gcc.gnu.org/onlinedocs/cpp/Invocation.html#index-fsearch-include-path) currently hardcodes it at this writing but whether it's brittle in future releases remains unknown for now).
2. Change "*[-std](https://gcc.gnu.org/onlinedocs/gcc/C-Dialect-Options.html#index-std-1)=c++20*" to "*[-std](https://gcc.gnu.org/onlinedocs/gcc/C-Dialect-Options.html#index-std-1)=c++23*" (or later). "*import std*" only became available in C++23.
3. Add the "*FunctionTraits*" macro *STDEXT_IMPORT_STD_EXPERIMENTAL* to all "*g++"* calls (via the [-D](https://gcc.gnu.org/onlinedocs/gcc/Preprocessor-Options.html#index-D-1) option), except the final call to link (since macros aren't required during linking). This macro notifies "*FunctionTraits*" that it should rely on "*import std*" instead of the usual "*#include*" calls to the "*std*" headers it normally relies on (in a non-module build). This macro isn't required for "*bits/std.cc*" itself (the macro affects "*FunctionTraits*" files only).

The following code implements the changes just described:

```bash
#!/bin/bash

set -e # Exit if any command fails

########################################
# Trivial helper function to display
# any passed command and then run it
# (unsophisticated but meets our needs)
########################################
run() {
    echo "$1"
    eval "$1"
    echo
}

#########################################################
# Compile all files which we output to the "build"
# subdirectory created just below (modules are compiled
# to object files no different than non-module files).
# For the "std" library itself however (1st compiled
# file below), we don't build its object file since
# it's not explicitly required by GCC at link time (see
# bullet 1 just above this example). Note however
# that each call to compile a module's ".cppm" file
# ("std.cc" for the "std" library) also creates the
# module's ".gcm" file automatically in a subdirectory
# GCC creates called "gcm.cache" (including "std.gcm"
# for the "std" library). This is the default in GCC at
# this writing but see the following for details (GCC's
# support for modules is still officially evolving at
# this writing):
#
#   Module Mapper
#   https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Module-Mapper.html
#
#   Compiled Module Interface
#   https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Compiled-Module-Interface.html
#
#   Bug 121963 - Make it easier to change the location of gcm.cache dir and document it
#   https://gcc.gnu.org/bugzilla/show_bug.cgi?id=121963
#
# Lastly, note that we compile all files in module
# dependency order so that when "import" statements are
# encountered in a given file, the compiler can locate
# its ".gcm" file since it's already been previously
# built. Build systems like "CMake" handle the order
# for you (see footnote 12 at the end of this file),
# but we're manually compiling the files ourselves here
# so we need to ensure they're compiled in the correct
# module dependency order (though we could and normally
# should rely on footnote 12 in a real-world app but it
# would obviously make this example much more
# complicated).
########################################################
mkdir -p "build"
run 'g++ -std=c++23 -fmodules -fmodule-only -c -fsearch-include-path "bits/std.cc"'
run 'g++ -std=c++23 -fmodules -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -c "CompilerVersions.cppm" -o "build/CompilerVersions.o"'
run 'g++ -std=c++23 -fmodules -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -c "FunctionTraits.cppm" -o "build/FunctionTraits.o"'
run 'g++ -std=c++23 -fmodules -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -c "Test.cpp" -o "build/Test.o"'

###########################################################
# Link (note that the "std" library object file not req'd
# here - we didn't build it above in fact, only its ".gcm"
# file - all required code from the "std" library will be
# implicily linked in in the following call using the
# vendor's libraries containing that code).
###########################################################
run 'g++ -std=c++23 "build/CompilerVersions.o" "build/FunctionTraits.o" "build/Test.o" -o "build/Test"'

echo 'Done! (successful - created "build/Test")'
```

<a name="MSVCModuleCompilationExample"></a>
### MSVC module compilation (official MSVC documentation [here](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170))
Like all other compilers, few MSVC developers will compile modules at the command prompt but see [MSVC module compilation at the command prompt](#msvcmodulecompilationcommandpromptexample) for an introduction. Most will do it in Visual Studio or less commonly some other IDE (and use of [CMake](#cmakemodulecompilationexample) is also supported). For details on modules in MSVC in general, see [Overview of modules in C++](https://learn.microsoft.com/en-us/cpp/cpp/modules-cpp?view=msvc-170) (and its associated links seen under "*Modules*" in the left-hand pane of the latter link). Note that Microsoft also started supporting "*import std*" as of Visual Studio 2022 V17.5 (search for "*17.5*" at the latter link), though only when using MSVC ("*cl.exe*") itself in that version. V17.6 is required to support it within Visual Studio itself (as cited here: [Build ISO C++23 Standard Library Modules](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#build-iso-c23-standard-library-modules)). Details on supporting "*import std*" are described further below.

To use the module version of "*FunctionTraits*" in Visual Studio (V16.8 or later - see [Footnote 2](README.md#footnotes)), you need to do the following (note that the following instructions are inherently long because they provide detailed information on how to organize modules in Visual Studio in general, including addressing certain Visual Studio limitations at this writing, as well as how to support "*import std*" - adding the modules for the "*FunctionTraits*" library itself is not complicated):

1. Determine where to place the modules within your solution. At this writing however, Microsoft provides almost no guidance on how to do this. The only reference to the situation at this writing (after extensive searching) can be found in a short blurb [here](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-named-modules-cpp?view=msvc-170) (search for it):

    > "*Your code can consume modules in the same project, or any referenced projects, automatically by using project-to-project references to static library projects*"

    Unfortunately there's more to the story, including some broken behavior in Visual Studio at this writing that *may* need to be circumvented (details to follow).

    Note that there are fundamentally 5 places where modules can be placed in a Visual Studio solution. The following addresses each of them. Note that [options ii](#vs_static_library_project) (*Static Library Project*) and [option *iii*](#vs_dll_project) (*Dynamic-Link Library Project*) are the fastest to compile, but potentially not always safe. See the caveat described in [option *ii*](#vs_static_library_project) (*Static Library Project*) below for details (which applies to [option *iii*](#vs_dll_project) (*Dynamic-Link Library Project*) as well).

    <a name="VS_Module_Locations"></a>
    1. ***Within each client project of your solution***

        This technique addresses the first half of Microsoft's recommendation referred to in the quote cited above (again, see [here](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-named-modules-cpp?view=msvc-170)):

        > "*Your code can consume modules in the same project*"

        This technique applies to modules that are only used within a specific project only (never by any other project). For shareable modules like "*FunctionTraits*" however, that can be used by multiple client projects (opposed to a module required by a single project only), this obviously isn't an ideal approach, since it will result in duplicate copies of the module if you add it to more than one client project. You therefore shouldn't normally add a shareable module to a specific client project unless you have one client project in your solution only, and even then it's arguably cleaner to locate shareable modules in a project-independent location (i.e., a shareable location), which the remaining items below address.
    <a name="VS_Static_Library_Project"></a>
    2. ***Static Library Project***

        This technique addresses the other half of the suggestion by Microsoft referred to in the quote cited further above (again, see [here](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-named-modules-cpp?view=msvc-170)):

        > "*Your code can consume modules ... by using project-to-project references to static library projects*"

        By adding a "*Static Library Project*" to your solution and then copying all your modules there, in this case "*CompilerVersions.cppm*" and "*FunctionTraits.cppm*" (discussed in bullet 2 further below), you can then add a [project-to-project reference](https://learn.microsoft.com/en-us/visualstudio/ide/managing-references-in-a-project?view=visualstudio#project-to-project-references) to the static library from any client project. All modules in the static library then become automatically available to that client (i.e., Visual Studio itself facilitates this automatically so no further work is required by developers other than to import the modules they require from that static library). The following steps describe the process:

        1. Add a new "*Static library*" project to your solution as follows:
            <ol type="1">
            <li>Right click the solution node in Solution Explorer and select "<i>Add -> New Project</i>" (or alternatively use <i><Ctrl+Shift+N></i> or "<i>File -> New -> Project</i>")</li>
            <li>Search for "<i>Static Library</i>" (or even just "<i>static</i>") in the "<i>Search for templates (Alt+S)</i>" box at the top of the "<i>Create a new project</i>" dialog. Double click the latter project or click "<i>Next</i>" and name the project "<i>SharedModules</i>" or whatever name you wish</li>
            </ol>

           The resulting static library project will then be created to house all your modules (added in [step 2](#msvcmodulecompilationitem) further below)

        2. For a given client project in Visual Studio that needs access to the modules you'll be adding to the above static library project, right-click the "*References*" node under the client project's node in Solution Explorer
        3. Select "*Add Reference...*" from the context menu
        4. Expand the "*Projects*" item seen in the left pane of the "*Add Reference*" dialog (if not already expanded), then check the checkbox for the "*SharedModules*" project created in step *a* above (or whatever you named it), as seen in the centre window of the dialog
        5. Press *OK* to exit
        6. Repeat steps *b* through *e* above for all other client projects

        After doing the above, you can then import any module from the static library project into any client project. You don't need to do anything else, such as setting *[`Configuration Properties -> Linker -> General -> Additional Library Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/linker-property-pages?view=msvc-170#additional-library-directories)* and *[`Configuration Properties -> Linker -> Input -> Additional Dependencies`](https://learn.microsoft.com/en-us/cpp/build/reference/linker-property-pages?view=msvc-170#additional-dependencies)*. The static library will be linked in automatically due to the [project-to-project reference](https://learn.microsoft.com/en-us/visualstudio/ide/managing-references-in-a-project?view=visualstudio#project-to-project-references) that was established in steps *b* through *e* above. In fact, all modules in the static library project even show up in Intellisense when you type "*import*" followed by a space in any client code that references the static library (via steps *b* through *e* above). This allows you to select the module you're interested in from the list that Intellisense displays (and Intellisense also kicks in other places). Please note however that the static library's directory is *not* added to the *#include* search directories for your client projects so if you need to locate any headers in the static library from your clients, then you still have to manually add the static library's directory to the *#include* search path of each client via the client project's *[`Configuration Properties -> C++ -> General -> Additional Include Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-include-directories)* property. You'll therefore need to do this if you wish to "*#include "FunctionTraits.h*" (details in step 3 later on).

        Note that housing modules in a static library project as described above comes with an important caveat however. Due to the nature of how C++ modules work, "*.ifc*" files (and more generally BMI files regardless of compiler), need to be built with the same compatible compiler options as the clients that use them. Otherwise, compiler, linker or even (hard-to-find) runtime errors are possible. This can occur if a module is compiled with a particular compiler option for instance, but the client project isn't (or vice versa). Depending on the particular compiler option, the module may not be compatible for importing into the client unless that particular compiler option is consistent in both the module and the client. This usually isn't an issue when the module is part of the same project as the client itself, since both are normally *usually*) compiled with the same compiler options. If compiled separately however, which usually occurs when the module lives in one project and the client in another, meaning both projects have their own project settings (though shared [*.props*](https://learn.microsoft.com/en-us/cpp/build/create-reusable-property-configurations?view=msvc-170) files can be used to minimize any differences), some settings may render the module incompatible for importing unless its the same as (or consistent with) the client's. Which settings may make them incompatible isn't published by any of the mainstream compilers however including MSVC. In practice most settings won't have any impact so they can differ in the module's project and the client's without any issue, but some potentially can impact their compatibility, leading to potential compiler or linker errors as previously noted (or more rarely, potentially hard-to-find runtime issues can potentially occur). Most developers won't be aware of which compiler options can lead to such issues however, again, since such information isn't published by any mainstream compiler vendor.

        Housing modules in static library projects are therefore susceptible to this issue, since the "*.ifc*" files are built when the static library project itself is built. Client projects that reference the library project then simply pick up these "*.ifc*" files from the static library when they "*import*" any of it modules. As result, client projects don't actually build the "*.ifc*" file themselves. The static library project builds it. This creates the issue described above. Unless all client projects use the same compatible compiler options as the static library project (specifically, any options that may impact the client's compatibility with the "*.ifc*" file, which aren't publicized by Microsoft), then using the "*.ifc*" files produced by the static library project is potentially brittle. Changing a (possibly esoteric) property in the static library project for instance and/or any of its clients can potentially render the "*.ifc*" files incompatible for use by the affected client(s). Testing shows that some incompatibilities might trigger MSVC warning [C5050]( https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/c5050?view=msvc-170intsance)[^7] which can help bring the situation to your attention, but there's no guarantee. Microsoft doesn't address the issue in any documentation that can be (easily) found at this writing so the situation is potentially unstable. Applying [*.props*](https://learn.microsoft.com/en-us/cpp/build/create-reusable-property-configurations?view=msvc-170) files to the static library project and all its clients may be one way to deal with the situation as previously noted, ensuring their options are consistent (if you're able and willing to support that), but there's no guidance from Microsoft at this writing so the situation remains unclear (and potentially error-prone). In practice however there may not be any issues within a given solution since many solutions often rely on options that are perfectly harmonious to the situation (so the static library project housing the modules and all its client projects may be compatible with each other but the situation still remains potentially brittle).

        The safer alternative to guarantee stability however is to simply not rely on a static library project for housing C++ modules (or a DLL project as described in option [*iii*](#vs_dll_project) below), but to simply put the modules in a common location that every client can independently compile with its own compiler options instead. This eliminates the problem altogether (option [*iv*](#vs_shared_items_project) below, "*Shared Items Project*", and option [*v*](#vs_standalone_directory) below, "*Standalone Directory*" can be used for this, usually the former). In this case each client generates the "*.ifc*" files itself, rather than relying on the static library project to generate the "*.ifc*" files. This is always safe (assuming all files within the project are compiled with same options which is usually the case), but comes at the expense of being slower to compile since every client must now generate its own "*.ifc*" files from the modules opposed to the static library approach, which generates the "*.ifc*" files just once and all clients can then consume them (but at the risk of producing an "*.ifc*" file that's incomptible with one or more client projects as described). Having each client generate its own "*.ifc*" files instead is therefore (normally) inherently safer when dealing with a generic library like "*FunctionTraits*", since generic (reusable) libraries normally aren't (necessarily) intended to share the same "*.ifc*" compatible compiler options as the clients that use them (unless you're able to ensure it). It's therefore arguably better not to house the "*FunctionTraits*" modules in a static library project as described above, unless you're comfortable that there will never be any incompatible compiler options with its clients (that may produce an incompatible "*.ifc*" file). See option [*iv*](#vs_shared_items_project) below ("*Shared Items Project*") and option [*v*](#vs_standalone_directory) below ("*Standalone Directory*") if you wish to pursue this (but usually option [*iv*](#vs_shared_items_project)).
    <a name="VS_DLL_Project"></a>
    3. ***Dynamic-Link Library Project***

        This technique is effectively identical to the [Static Library Project](#vs_static_library_project) in option *ii* above except all modules will be stored in a DLL instead of a static library (so you'll need to ship your DLL with your project as usual, no different than a non-module application). However, housing modules in a DLL carries some extra baggage compared to housing them in a static library because Microsoft strangely doesn't support them quite as well yet (and a Visual Studio bug at this writing also adds some additional work to overcome the situation - more on this shortly). Until Microsoft adds the same level of support as static libraries (for housing modules), some manual intervention is required to get things fully working (again, more on this shortly).

        As a sidenote (before proceeding), note that for a "*header-only*" style module like "*FunctionTraits*", which doesn't generate any storage (because "*header-only*" files consist entirely of templates, using statements, constexpr declarations, etc., i.e., declarations with no implementation file to link to so they're self-contained[^16]), testing shows that client projects don't need to link to the DLL's import library. It only becomes necessary if you add other modules to the library that *aren't* "*header-only*" (i.e., they depend on implementation files that do need to be linked in by module clients - unresovled external errors will occur at link time otherwise). Nevertheless, even if all modules in the DLL are "*header only*", it's cleaner and safer to link to the DLL's import library regardless, though for a DLL that consists entirely of "*header only*" modules the import library is just an empty stub for all intents and purposes.

        Note that adding your modules to a DLL essentially follows the same process as adding them to a static library, as described in [option *ii*](#vs_static_library_project) above, except you'll create a new "*Dynamic-Link Library (DLL)*" project instead of a "*Static Library*" project. The following steps therefore just repeat what's already documented in [option *ii*](#vs_static_library_project) above but creates a "*Dynamic-Link Library (DLL)*" project instead. However, as briefly mentioned above, some additional work is required since the Visual Studio tooling for referencing modules in a DLL isn't 100% complete by Microsoft yet (unlike for a static library). Moreover, a Visual Studio bug exists that also needs to be circumvented. This adds to the workload of housing modules in a DLL project at this writing.

        The following steps describe the process:

        <a name="MSVC_DLL_Step_A"></a>
        1. Add a new "*Dynamic-Link Library (DLL)*" project to your solution as follows:

            <ol type="1">
            <li>Right click the solution node in Solution Explorer and select "*Add -> New Project*" (or alternatively use *<Ctrl+Shift+N>* or "*File -> New -> Project*")</li>
            <li>Search for "*Dynamic-Link Library (DLL)*" (or even just "*DLL*") in the "*Search for templates (Alt+S)*" box at the top of the "*Create a new project*" dialog. Double click the latter project or click "*Next*" and name the project "*SharedModules*" or whatever name you wish</li>
            </ol>

           The resulting DLL project will then be created to house all your modules (added in [step 2](#msvcmodulecompilationitem) further below).
        <a name="MSVC_DLL_Step_B"></a>
        2. At this stage you'll add your modules to the DLL project just created. This is described in [step 2](#msvcmodulecompilationitem) further below. For DLL projects however, a Visual Studio bug exists at this writing that results in the wrong name being assigned to your modules' "*.ifc*" files when each module is compiled. This is fully described later on (just after [step h](#msvc_dll_step_h) further below). To circumvent the bug, you have two options. The first and generally recommended option is to simply override (correct) the buggy name of the "*.ifc*" file Visual Studio creates which you'll do here (again, the generally recommended option). If you ***don't*** choose that option then you can immediately continue with [step c](#msvc_dll_step_c) just below. In that case the option for circumventing the bug is described in [step g](#msvc_dll_step_g) below (consult [sub-step 2](#MSVC_DLL_STEP_G_SUB_STEP_2) there). If you ***do*** choose to correct the buggy name of the "*.ifc*" file however, then proceed as follows. As you add each module file to your DLL in the step you're now reading (again, details follow in [step 2](#msvcmodulecompilationitem) further below), you'll need to explicitly set the file's "*.ifc*" name via the module file's *`Configuration Properties -> C/C++ -> Output Files -> Module Output File Name`* property (i.e., [/ifcOutput](https://learn.microsoft.com/en-us/cpp/build/reference/ifc-output?view=msvc-170)). See [To set this compiler option in the Visual Studio development environment](https://learn.microsoft.com/en-us/cpp/build/reference/ifc-output?view=msvc-170#to-set-this-compiler-option-in-the-visual-studio-development-environment) for details in general. The base name of the "*.ifc*" file you specify here must match the name of your module as seen in your module's "*export module*" statement. This follows the default naming convention normally expected by MSVC so it circumvents the Visual Studio bug. For the "*FunctionTraits*" module itself, the name originates from "*export module FunctionTraits*" so you'll normally set the latter property to "*$(IntDir)FunctionTraits.ifc*" (see "*$(IntDir)*" under [List of common macros](https://learn.microsoft.com/en-us/cpp/build/reference/common-macros-for-build-commands-and-properties?view=msvc-170#list-of-common-macros)). Note that the property is usually set to "*$(IntDir)*" by default so "*$(IntDir)FunctionTraits.ifc*" overrides it as seen (correcting the buggy name Visual Studio would otherwise apply - note that a trailing backslash after the "*$(IntDir)*" isn't included since the latter macro already contains one). See [/ifcOutput](https://learn.microsoft.com/en-us/cpp/build/reference/ifc-output?view=msvc-170) for details on this property in general and details on the bug itself follow further below (just after [step h](#msvc_dll_step_h)). Note that by correcting the "*.ifc*" file name using this property, you can then simply set the *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)*  (i.e., [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)) property of each client project to resolve all "*import*" calls in the client. This is described in [step g](#msvc_dll_step_g) below.
        <a name="MSVC_DLL_Step_C"></a>
        3. For a given client project in Visual Studio that needs access to the modules you'll be adding to the DLL project created in [step a](#msvc_dll_step_a) above, right-click the "*References*" node under the client project's node in Solution Explorer
        4. Select "*Add Reference...*" from the context menu
        5. Expand the "*Projects*" item seen in the left pane of the "*Add Reference*" dialog (if not already expanded), then check the checkbox for the "*SharedModules*" project created in [step a](#msvc_dll_step_a) above (or whatever you named it), as seen in the centre window of the dialog
        6. Press *OK* to exit
        <a name="MSVC_DLL_Step_G"></a>
        7. You now need to circumvent the "*.ifc*" naming bug briefly described in [step b](#msvc_dll_step_b) above (and fully described just after [step h](#msvc_dll_step_h) further below). How you fix it depends on how you handled things in [step b](#msvc_dll_step_b) above. Only one of the following applies (they are mutually exclusive):
            <ol type="1">
                <li id="MSVC_DLL_STEP_G_SUB_STEP_1">If you corrected the buggy name of each "<i>.ifc</i>" file in <a href="#msvc_dll_step_b">step b</a> above, then simply set the <i><a href="https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories"><code>Configuration Properties -> C/C++ -> General -> Additional BMI Directories</code></a></i> (i.e., <a href="https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir">/ifcSearchDir</a>) property for your client project to the directory containing the DLL's "<i>.ifc</i>" files, such as "<i>..\SharedModules\Debug\\</i>" (or wherever the "<i>.ifc</i>" files live). Having to hardcode the latter directory name this way is ugly but there aren't any good alternatives, and even replacing portions of the latter directory like "<i>Debug</i>" with a Visual Studio macro like "<i>$(Configuration)</i>" (see this under <a href="https://learn.microsoft.com/en-us/cpp/build/reference/common-macros-for-build-commands-and-properties?view=msvc-170#list-of-common-macros">List of common macros</a>) is brittle because "<i>$(Configuration)</i>" may not be the same in the DLL project as it is in the client project you're now targeting. In any case, setting this property is a one-time only operation. Once the property is set, each call to "<i>import</i>" in your client project will locate the module you're importing via the latter property (since calls to "<i>import</i>" implicitly look for the name of the module you're importing by appending "<i>.ifc</i>" to the imported module name and that file will now be found in the directory you just set - again, full details follow <a href="#msvc_dll_step_h">step h</a> further below).</li>
                <li id="MSVC_DLL_STEP_G_SUB_STEP_2">If you <i>didn't</i> correct the buggy name of each "<i>.ifc</i>" file in <a href="#msvc_dll_step_b">step b</a> above, then for each module you import in your client app, you need to explicitly add the fully qualified name of the module's "<i>.ifc</i>" file via the client project's <i><a href="https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-module-dependencies"><code>Configuration Properties -> C/C++ -> General -> Additional Module Dependencies</code></i> property (i.e., <a href=https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170">/reference</a> switch in MSVC). Within Visual Studio itself this property is a semicolon delimited list of <a href="https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170">/reference</a> calls so you can add the "<i>FunctionTraits</i>" module for instance by appending <i><code>FunctionTraits=..\SharedModules\Debug\FunctionTraits.cppm.ifc`</code></i> to the existing property (or wherever that file lives in your DLL project). Note that the name of the "<i>.ifc</i>" file in this case is "<i>FunctionTraits.cppm.ifc</i>" as seen, the buggy name created by Visual Studio (more on this after <a href="#msvc_dll_step_h">step h</a> below), but if you changed the module's extension to "<i>.ixx</i>" as described in <a href=""#msvcmodulecompilationitem">step 2</a> then it should obviously be called "<i>FunctionTraits.ixx.ifc</i>" instead (i.e., the "<i>.cppm</i>" extension is simply changed to "<i>.ixx</i>"). Note that each module you add to this property should be separated by a semicolon, though the semicolon is not supported by the <a href="https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170">/reference</a> switch itself. Visual Studio supports the semicolon in its own environment that is but MSVC ("<i>cl.exe</i>") doesn't (so Visual Studio takes care of breaking this semicolon delimited property into its individual <a href="https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170">/reference</a> switches when it invokes "<i>cl.exe</i>").</li>
            </ol>

           <a name="MSVC_DLL_STEP_H"></a>
        8. Repeat steps *b* through *g* above for all other client projects

        Note that [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above correct a Visual Studio bug at this writing. This bug exists even if you house your modules in a [Static Library Project](#vs_static_library_project) (option *ii* above), but Visual Studio automatically circumvents the bug for you in clients that reference static library projects. In clients that reference DLL projects however, the Visual Studio tooling isn't quite as complete as it is in clients that reference static library projects (for unknown reasons), so this must be explicitly circumvented. Again, [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above handle this but the following addresses the situation in detail (though the explanation isn't required reading so long as you've corrected the problem in [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above - you can therefore bypass the remaining paragraphs just below if you wish, and proceed immediately to the [Shared Items Project](VS_Shared_Items_Project) option that immediately follows).

        The first thing to note is that for unknown reasons, at this writing Visual Studio doesn't automatically apply the MSVC [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) option nor [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) option to locate any imported modules when it builds your client app (to locate modules in referenced DLL projects). These are the two compiler options MSVC makes available for locating an "*.ifc*" file when an "*import*" statement is encountered. See [MSVC module compilation at the command prompt](#msvcmodulecompilationcommandpromptexample) later in this README file for details on these options (and the latter Microsoft links themselves). Visual Studio does apply [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) automatically to locate modules in referenced *static library projects* ([option *ii*](#vs_static_library_project) above), so it's unclear why it's not automatically applied to locate modules in referenced DLL projects as well (which would eliminate the need to deal with the situation). The tooling for handling modules simply isn't as complete for referenced DLL projects at this writing as it is for referenced static library projects, so until Microsoft corrects the situation (presumably they will but the situation is unclear), you need to manually handle this yourself (for any module you import from a DLL - [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above take care of this). However, due to the unconventional (effectively buggy) behaviour of how Visual Studio names the "*.ifc*" files it creates, the need to manually add the [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) compiler option (or alternatively [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)) takes more effort to do than it should. To understand this (and what [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) do to correct it), you first need to know how Visual Studio incorrectly names the "*.ifc*" files it creates due to a Microsoft bug described [here](https://developercommunity.visualstudio.com/t/testixx-module-interface-creates-testi/10068448) (though Microsoft responds at the latter link by suggesting the issue should be classified as a "*suggestion ticket*" but it can only be reasonably described as a bug).

         Recall from earlier in this document that compiling a module for any compiler builds a BMI file in addition to its object file (though a given compiler might optionaly support a switch to do it in two separate steps), and for MSVC, "*.ifc*" is the BMI file's extension. When Visual Studio builds a module such as "*FunctionTraits.cppm*", by default it names the resulting "*.ifc*" file "*FunctionTraits.cppm.ifc*". That is, it simply appends the "*.ifc*" extension to the module's file name, leaving the existing "*.cppm*" extension intact (or whatever extension it has, such as "*.ixx*", the MSVC default extension for modules, but details to follow in [step 2](#msvcmodulecompilationitem) further below). This name deviates from the norm however (again, see [here](https://developercommunity.visualstudio.com/t/testixx-module-interface-creates-testi/10068448)), since it conflicts with the default behavior of MSVC itself ("*cl.exe*"), which correctly names the file after the module's actual name instead (i.e., after the "*export module*" name in the module, not the name of the module file itself). MSVC ("*cl.exe*") will therefore name the file "*FunctionTraits.ifc*" instead (when directly using "*cl.exe*" to compile as described in [MSVC module compilation at the command prompt](#msvcmodulecompilationcommandpromptexample)), where the name "*FunctionTraits*" in the latter "*.ifc*" file name originates from the "*export module FunctionTraits*" statement in the module file (again, not from the module file name itself). This is further confirmed by Microsoft [here](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/) (referring to the cited example at this link):

        > *"The compiler will derive a name for the resulting IFC file based on the module interface name. The resulting IFC in this case is derived from the module name MyModule transformed into MyModule.ifc. Note that the name of the input file has no bearing on the exported module interface name, they are completely orthogonal to each other so if this file were named foobar.ixx the generated IFC name would still be MyModule.ifc."*

        Visual Studio doesn't name the "*.ifc*" this way however, as described above, it simply appends the extension "*.ifc*" to the file name "*FunctionTrats.cppm*" instead (or "*FunctionTraits.ixx*" if you renamed the file's extension as described in [step 2](#msvcmodulecompilationitem) later on). This also violates Microsoft's own recommended best practices for how to name a module. See [Module naming](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-named-modules-cpp?view=msvc-170#module-naming):

        > *"The name of the file that contains the module primary interface is generally the name of the module. For example, given the module name BasicPlane.Figures, the name of the file containing the primary interface would be named BasicPlane.Figures.ixx."*

        The above link then goes on to describe how "*.ifc*" files are normally named after the module name itself (though it describes this in the context of a module partition).

        The upshot is that MSVC ("*cl.exe*") simply adds an "*.ifc*" extension to the "*export module*" name when creating the "*.ifc*" file (the default and recommended behavior as confirmed in the quotes above), but Visual Studio changes this behavior to ignore the name of the module in the file's "*export module*" statement, simply adding the "*.ifc*" extension to the name of the module file instead (as described). Again, this behavior deviates from the norm and is therefore buggy for most intents and purposes. However, in most real-world cases developers normally name their module's file after the "*export module*" name anyway, by simply tacking on an "*.ixx*" or "*.cppm*" extension to the "*export module*" name (though again, "*.ixx*" is the Microsoft default). This is the most intuitive way and also recommended by Microsoft. In this case the default name of the "*.ifc*" file that Visual Studio creates will be the name (including extension) of the module's file with an extra "*.ifc*" extension as described (e.g., "*FunctionTraits.cppm.ifc*"). Again, the name "*FunctionTraits*" in the latter "*.ifc*" file name originates from the "*FunctionTraits.cppm*" file itself, not its "*export module*" statement. However, because the name "*FunctionTraits*" in the file name "*FunctionTraits.cppm*" was explicitly named after the "*export module FunctionTraits*" statement in the file (again, since most developers name the module file after the "*export module*" name), the name "*FunctionTraits*" in the resulting "*FunctionTraits.cppm.ifc*" file matches the module's actual name in the "*export module*" statement anyway. Therefore, the base name of a module's "*.ifc*" file ("*FunctionTraits*" in this case) will still match the "*export module*" name in most cases, so long as the developers name the module file after the "*export module*" statement in the file (and most do).

        As a result, even though Visual Studio incorrectly creates the "*.ifc*" file name based on the module file name instead of the "*export module*" name within the file, the "*.ifc*" file usually winds up with the "*export module*" name anyway (in the base name portion of the "*.ifc*" file name). The Visual Studio bug of failing to name the "*.ifc*" file after the "*export module*" statement would therefore be harmless in most cases (since it will still usually match the "*export module*" name anyway), except that Visual Studio also fails to remove the module's extension as well ("*.cppm*" or "*.ixx*"). The resulting "*.ifc"* file name is therefore still problematic, i.e., "*FunctionTraits.cppm.ifc*" instead of "*FunctionTraits.ifc*" (it should be the latter but it's not). By contrast, the default behavior when compiling with MSVC ("*cl.exe*") directly (unlike when compiling with Visual Studio), always results in the correct name, "*FunctionTraits.ifc*", where "*FunctionTraits*" in the latter name *does* originate from the actual "*export module FunctionTraits*" statement in the module file, not from the module file name "*FunctionTraits.cppm*" itself. The "*.cppm*" extension in the module's file is also correctly dropped. Most developers however will be compiling with Visual Studio so the "*.ifc*" file naming issue (bug) exists, not "*cl.exe*" directly (where the "*.ifc*" file naming issue doesn't exist).

        With the above (incorrect) naming of the "*.ifc*" file by Visual Studio now explained (it will become relevant in a moment), the issue of housing modules in a DLL project is now this. When an "*import*" statement is encountered in a client, the compiler will search for the specified module using either the MSVC option [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) or [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) (though Visual Studio actually uses the former). For clients importing a module in a static library project ([option *ii*](#vs_static_library_project) above), Visual Studio automatically adds the call to [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) in order to find the the "*.ifc*" file for the imported module when the client is compiled. The fact that Visual Studio incorrectly names the "*.ifc*" file as previously described, which occurs in static library projects as well (not just DLL projects), therefore isn't relevant in clients that reference static library projects. The "*.ifc*" file for an imported module is found regardless, in spite of the buggy "*.ifc*" naming issue, without the developer having to do anything (because Visual Studio itself automatically adds a [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) to the imported module and does so correctly, circumventing the issue with the incorrect "*.ifc*" file name). Merely setting a reference in a client project to a [Static Library Project](#vs_static_library_project) therefore takes care of everything. Visual Studio does all the work without any intervention by the developer whatsoever (in spite of the buggy "*.ifc*" file name).

        In client references to a *DLL* project however, Visual Studio inexplicably doesn't add a [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) automatically to the client project as it does for modules in a referenced [Static Library Project](#vs_static_library_project). Developers therefore have to manually take care of this on their own, which [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above take care of. To do so (i.e., what [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above implement), developers can either set the *[`Configuration Properties -> C/C++ -> General -> Additional Module Dependencies`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-module-dependencies)* property in the client project (which is just the [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) switch in MSVC), or the *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)* property instead (which is just the [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) switch in MSVC). The latter property ([/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)) is actually easier however, since you only need to set it once, to the directory where the DLL project creates its "*.ifc*" files, and the client project will then find *all* "*.ifc*" files there for any "*import*" statement in the client project (that target modules in the DLL project). If you choose the other option above however (again, *[`Configuration Properties -> C/C++ -> General -> Additional Module Dependencies`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-module-dependencies)* which is just the [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) switch in MSVC), then you need to set it to handle every module the client imports from the DLL project (so you need to update the latter property whenever a new "*import*" statement is added in your client projects, and update the switch as well if you rename a module). The *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)* property (again, [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)) is therefore easier as described, but because of the buggy name Visual Studio assigns to modules as described in the previous paragraphs (in particular, its failure to drop the module's "*.cppm*" or "*.ixx*" extension from the "*.ifc*" file name), a cumbersome workaround needs to be applied to overcome it (read on).

        The workaround is cumbersome for the following reason. If you set the *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)* property (i.e., [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)) of the client project to the directory of your DLL's "*.ifc*" files, then a client call such as "*import FunctionTraits*" will then search the latter directory for a file called "*FunctionTraits.ifc*". The name "*FunctionTraits*" in the latter file name originates from "*import FunctionTraits*" itself and the "*.ifc*" extension is simply tacked on by the compiler. However, because of the buggy default name that Visual Studio assigned to the "*.ifc*" file when the DLL project was compiled, the file name is actually "*FunctionTraits.cppm.ifc*" (or "*FunctionTraits.ixx.ifc*" if you renamed the "*FunctionTraits.cppm*" file with a "*.ixx*" extension - again, see [step 2](#msvcmodulecompilationitem) further below). The call to "*import FunctionTraits*" therefore won't find a file called "*FunctionTraits.ifc*" because Visual Studio incorrectly named the file "*FunctionTraits.cppm.ifc*" instead (i.e., it left the "*.cppm*" extension in place). The following error will therefore occur when compiling the client at this writing:

        >*Error C2230: could not find module 'FunctionTraits'*

        To circumvent this issue, you can either rely on the *[`Configuration Properties -> C/C++ -> General -> Additional Module Dependencies`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-module-dependencies)* (i.e., [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170)) property, which means (painfully) having to update it for every imported module in your client projects (to point to the buggy "*.ifc*" file name in the referenced DLL project), or rely on the (normally simpler) *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)* property instead (i.e., [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)), but in the latter case you'll need to explicitly override (correct) the buggy Visual Studio default naming behavior of the "*.ifc*" files it creates (ensuring that instead of the file being named "*FunctionTraits.cppm.ifc*" by Visual Studio, it will be named "*FunctionTraits.ifc*" instead, i.e., the correct default name MSVC ("*cl.exe*") creates, and what the compiler looks for when it encounters "*import FunctionTraits*"). This is the more natural fix (correcting the buggy Visual Studio "*.ifc*" file name), but it has to be done for every module in your DLL project so it's also a painful alternative (though arguably less so than updating the *[`Configuration Properties -> C/C++ -> General -> Additional Module Dependencies`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-module-dependencies)* property (i.e., [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170)) for every imported module in every client project). If you wish to do the latter, i.e., correct the buggy Visual Studio "*.ifc*" file name (generally the recommend fix), then [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) above takes care of this. [Step b](#msvc_dll_step_b) sets the *[`Configuration Properties -> C/C++ -> Output Files -> Module Output File Name`](https://learn.microsoft.com/en-us/cpp/build/reference/ifc-output?view=msvc-170#to-set-this-compiler-option-in-the-visual-studio-development-environment)* property (i.e., [/ifcOutput](https://learn.microsoft.com/en-us/cpp/build/reference/ifc-output?view=msvc-170)) for each module in the DLL project itself (correcting the buggy Visual Studio "*.ifc*" file name), and [sub-step 1](#MSVC_DLL_STEP_G_SUB_STEP_1) of [step g](#msvc_dll_step_g) then sets (one-time only) the *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)* property (i.e., [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir)) of the given client project to the DLL's "*.ifc*" file directory (which now contains the correctly named "*.ifc*" files). It normally defaults to "*$(IntDir)*" as seen when you go into this setting (see this Visual Studio macro under [List of common macros](https://learn.microsoft.com/en-us/cpp/build/reference/common-macros-for-build-commands-and-properties?view=msvc-170#list-of-common-macros)), so you'll normally need to change it to "*$(IntDir)FunctionTraits.ifc*" instead (where a trailing '\\' after the "*$(IntDir)*" in the latter propery is omitted as seen since it's already present in the "*$(IntDir)*" macro itself normally). Note that "*FunctionTraits*" in the latter property is the actual name of the module in its "*export module*" statement. You need to hardcode this name in the property as seen to match the "*export module FunctionTraits*" name in the module file (since there is no Visual Studio macro such as "*$(ModuleName)*" you can rely on). If you ever change the module's name in "*export module FunctionTraits*" however (though the library itself never will), then you'll have to update the property to match it or the "*C2230*" error previously shown will surface again (so keep this in mind for your own modules).

        To summarize, when housing a module in a DLL project, calls to "*import ModuleName*" in client projects that reference the DLL project need to ensure that "*ModuleName.ifc*" will be found. Visual Studio strangely doesn't handle this automatically for referenced modules in DLL projects, only for referenced modules in [Static Library Projects](#vs_static_library_project). The compiler searches for a file called "*ModuleName.ifc"* in this case which won't normally be found due to the broken (buggy) naming convention that Visual Studio applies when it creates "*.ifc*" files. The instructions in the paragraph just above (what [step b](#msvc_dll_step_b) and [step g](#msvc_dll_step_g) do) describe how to correct the problem for modules in a DLL project (again, the problem doesn't exist for modules in a referenced *static* library because Visual Studio automatically adds a [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) statement for any module you import from a static library - for DLL projects it doesn't but hopefully this will be rectified by Microsoft in a future release).
    <a name="VS_Shared_Items_Project"></a>
    4. ***Shared Items Project***

        This technique is effectively the same to set up as the static or DLL project techniques described above but uses a "*Shared Items Project*" instead of a static or DLL library project. By adding a "*Shared Items Project*" to your solution and then copying all your modules there, in this case "*CompilerVersions.cppm*" and "*FunctionTraits.cppm*" (discussed in bullet 2 further below), you can then add a [Shared project reference](https://learn.microsoft.com/en-us/visualstudio/ide/managing-references-in-a-project?view=visualstudio#shared-project-reference) to your "*Shared Items Project*. This ensures all modules in the "*Shared Items Project*" are now compiled for each client, as described below:

        1. Add a new "*Shared Items Project*" to your solution as follows:

            1. Right click the solution node in Solution Explorer
            2. Select "*Add -> New Project*"
            3. Search for "*Shared Items Project*" - double click the latter project or click "*Next*" and name the project "*SharedModules*" or whatever name you wish
        2. Right-click the "*References*" node of any client project in Solution Explorer
        3. Select "*Add Reference...*"
        4. Check the shared library's project (checkbox) created in step a above, as seen under the "*Shared Projects*" section of the "*Add Reference*" dialog

        Now when you build your client project, it will also build all modules added to your "*Shared Items Project*" above ("*FunctionTraits*" and "*CompilerVersions*" only for now but you can add any other required modules to 2 below in the future). After doing the above, you can then import any module from the "*Shared Items Project*" project into the client project. In fact, all modules in the "*Shared Items Project*" project even show up in Intellisense when you type "*import*" followed by a space in any client code, allowing you to select the module you're interested in from the list that Visual Studio displays. Please note however that the "*Shared Items Project*" directory is *not* added to the *#include* search directories for your client project, so if any are required from the "*Shared Items Project*" project then you still have to manually add them to *[`Configuration Properties -> General -> Additional Include Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-include-directories)* (in this case you'll therefore need to add it if you wish to "*#include "FunctionTraits.h*").

        Note that using this technique, any modules you add to "*Shared Items Project*" are effectively treated as part of each client project (the purpose of a "*Shared Items Project*"), so the modules will be built using the same compiler options as each client project itself. The "*.ifc*" files generated during compilation will therefore be compiled with the same project settings as each client project. This eliminates the issue with incompatible compiler options when using a static or DLL library to store your modules as discussed in items 2 and 3 above.
    <a name="VS_Standalone_Directory"></a>
    5. ***Standalone Directory (non-project)***

        TODO
<a name="MSVCModuleCompilationItem"></a>
2. Add the modules "*CompilerVersions.cppm*" and "*FunctionTraits.cppm*" from this repository to the your Visual Studio solution in the location you created in [step 1](#vs_module_locations) above (usually the [Static Library Project](#vs_static_library_project), [DLL library project](#vs_dll_project) or [Shared Items Project](#vs_shared_items_project)). However, MSVC developers may want to rename these files with a "*.ixx*" extension first, since that's the native extension Visual Studio normally expects for C++ modules. Replacing the "*.cppm*" extension with "*.ixx*" *before* adding them to your Visual Studio solution is optional however, since Visual Studio also recognizes the "*.cppm*" extension. This is documented by Microsoft under *[`Configuration Properties -> C/C++ -> Advanced -> Compile As`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#compile-as)* (see the *`Compile as C++ Module Code (/interface)`* setting in its [Choices](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#choices-21) section), but note that MSVC itself ("*cl.exe*") *doesn't* natively support "*.cppm*" files (unlike Visual Studio itself). Its [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) option as seen in the latter *`Compile as C++ Module Code (/interface)`* string itself makes this clear (see latter link). It explictly indicates the option is required for any modules that don't have an "*.ixx*" extension and "*.cppm*" files are no exception. It even shows an example at this writing by applying the option to a "*.cppm*" file and testing confirms the option is in fact required for "*.cppm*" files (i.e., an error occurs if you omit the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) option for "*.cppm*" files when calling "*cl.exe*" directly, and moreover if you omit [/TP](https://learn.microsoft.com/en-us/cpp/build/reference/tc-tp-tc-tp-specify-source-file-type?view=msvc-170) as well - "*cl.exe*" requires *both* of the latter options for modules that don't have an "*.ixx*" extension, as noted in the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) documentation - search for "*/TP*" at the latter link as well as [here](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1)). Visual Studio itself however seemingly treats both the *`Compile as C++ Code (/TP)`* *and* *`Compile as C++ Module Code (/interface)`* options the same (again, see these under [Choices](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#choices-21)), since testing shows that *either* of them result in *both* the [/TP](https://learn.microsoft.com/en-us/cpp/build/reference/tc-tp-tc-tp-specify-source-file-type?view=msvc-170) *and* [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) options being passed to "*cl.exe*" behind the scenes (inspection of the Visual Studio build log confirms this). This is confusing and misleading behavior however, since their corresponding settings under [Choices](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#choices-21) don't precisely match the documented behavior of these settings in "*cl.exe*" itself, but neverthless, the Visual Studio docs for its *`Compile as C++ Module Code (/interface)`* setting officically indicates it supports both "*.ixx*" _and_ "*.cppm*" files (again, see this under [Choices](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#choices-21)). You therefore don't need to change the *[`Configuration Properties -> C/C++ -> Advanced -> Compile As`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#compile-as)* option at all. The *`Default`* setting seen under [Choices](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#choices-21) simply works for "*.cppm*" files in Visual Studio in addition to the native "*.ixx*" extension. For any other extension that you may wish to use however (besides "*.ixx*" and "*.cppm*"), the *`Default`* setting *must* be changed to *`Compile as C++ Module Code (/interface)`* (though again, testing shows that even *`Compile as C++ Code (/TP)`* works but it's likely safer to rely on *`Compile as C++ Module Code (/interface)`*). If you do change the extension to something else however (though it would be unusual), then note that those using "*CMake*" may wish to consult the following Microsoft [issue](https://gitlab.kitware.com/cmake/cmake/-/issues/25643). As described in the latter link, Visual Studio solutions built with "*CMake*" don't set the latter property as they normally should (though this could be corrected in a future release).
3. Add "*CompilerVersions.h*" and "*FunctionTraits.h*" from this repository to your Visual Studio project in the same location as the "*.cppm*" files described in [step 1](#vs_module_locations) above. These are the library's two header files but also used in the module version. The modules in [step 2](#msvcmodulecompilationitem) above are implemented using these "*.h*" files but you can also continue to use the "*.h*" files themselves in the module version if you wish, instead of importing the modules directly (using "*import FunctionTraits*"). As described earlier in this document, "*FunctionTraits.h*" simply calls "*import FunctionTraits*" for you in the module version of the library (which also imports module "*CompilerVersions*"), but with the added benefit of picking up all public macros in both "*.h*" files as well (since macros aren't exported by C++ modules so the headers are still required to #define them). See footnote [^1] if you wish to suppress "*import FunctionTraits*" when you *#include "FunctionTraits.h"*. Only macros will be #defined when you do and it's then your responsibility to explicitly call "*import FunctionTraits*". "*FunctionTraits.h"* then only needs to be #included if you require any of its macros, such as when you need to detect if a member function exists using the [DECLARE\_HAS\_FUNCTION](#declare_has_function) family of macros. Otherwise, if you don't require any of its macros, then your call to "*import FunctionTraits*" is all you require. Again, see footnote [^1] for complete details.
4. Ensure *STDEXT\_USE\_MODULES* (a "*FunctionTraits*" macro) is #defined in your project settings as described earlier in this document (normally to an empty value since its value isn't relevant, it merely needs to be #defined). Just add this to the project's properties (macros) for the project where you located your modules under *[`Configuration Properties -> C/C++ -> Preprocessor -> Preprocessor Definitions`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#preprocessor-definitions)* (which controls the [/D](https://learn.microsoft.com/en-us/cpp/build/reference/d-preprocessor-definitions?view=msvc-170) MSVC option, though semicolons aren't allowed in the MSVC option unlike in the Visual Studio properties page itself - search for "*semicolons*" [here](https://learn.microsoft.com/en-us/cpp/build/reference/d-preprocessor-definitions?view=msvc-170) for details). A compiler error notifying you that *STDEXT\_USE\_MODULES* isn't #defined will be triggered otherwise (by "*FunctionTraits*" itself by design, since adding the modules in [step 2](#msvcmodulecompilationitem) above serves no purpose unless your project actually uses them, which *STDEXT\_USE\_MODULES* is designed to support).
5. Set the project property *[`Configuration Properties -> General -> C++ Language Standard`](https://learn.microsoft.com/en-us/cpp/build/reference/general-property-page-project?view=msvc-170#c-language-standard)* (or alternatively[^5] *[`Configuration Properties -> C/C++ -> Language -> C++ Language Standard`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#cpplang)*) to *`/std:c++20`* or later for all affected projects (since modules only became available in C++20). This property controls the MSVC option [/std](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170). Note that if you wish to support "*import std*" then C++23 or later is required (see step 6 just below).
6. If you wish to also use "*import std*" in your app then also do the following (but see ***IMPORTANT*** note below):

    1. Set the following two properties under `Configuration Properties -> C/C++ -> Language` for all affected projects in Visual Studio:

        1. *[`C++ Language Standard`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#cpplang)* property (or alternatively[^5] *[`Configuration Properties General -> C++ Language Standard`](https://learn.microsoft.com/en-us/cpp/build/reference/general-property-page-project?view=msvc-170#c-language-standard)*) to either *`Preview - ISO C++23 Standard (/std:c++23preview)`* or *`Preview - Features from the Latest C++ Working Draft (/std:c++latest)`* (until Microsoft adds an actual *non-preview* option for C++23 which doesn't yet exist at this writing but will presumably be added in the not-too-distant future - "*import std*" only became available in C++23). Note that this property controls the MSVC option *[/std](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170)*
        2. *[`Build ISO C++23 Standard Library Modules`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#build-iso-c23-standard-library-modules)* property to *`Yes`* (note that this doesn't correspond with any "*cl.exe*" compiler option however, it merely adds an MSBuild property called *\<BuildStlModules\>* to your project's "*.vcxproj*" file). Also please note that just above this property in Visual Studio you'll find an *[`Enable Experimental C++ Standard Library Modules`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#enable-experimental-c-standard-library-modules)* property (in current versions of Visual Studio), but this property isn't relevant here (it's not associated with "*import std*"), and is even now obsolete in the latest versions of Visual Studio (note that the latter property controls the "*cl.exe*" option [/experimental:module](https://learn.microsoft.com/en-us/cpp/build/reference/experimental-module?view=msvc-170))
    2. Add the macro *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* to your project's existing macros under *[`Configuration Properties -> C/C++ -> Preprocessor -> Preprocessor Definitions`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#preprocessor-definitions)*, similar to *STDEXT\_USE\_MODULES* in step 3 above (both macros are required). "*FunctionTraits.h*" will then call "*import std*" internally where required instead of #including the "*std*" headers as it normally does (when *STDEXT\_IMPORT\_STD\_EXPERIMENTAL* isn't #defined). See this macro discussed earlier in this document.

    ***IMPORTANT️***: Note that at this writing Visual Studio doesn't yet support mixing *#include* statements from the "*std*" library and the "*import std*" statement. Until available you *must* therefore always exclusively rely on one or the other only (i.e., for now you can't mix, say, *#include \<vector\>* and the "*import std*" statement in the same file - Microsoft is on record about this - see [Standard library named module considerations](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module?view=msvc-170#standard-library-named-module-considerations)). If you add support for "*import std*" as described above then you must therefore use it exclusively in any given C++ file (i.e., a C++ file with "*import std*" can't also directly or indirectly *#include* a header from the "*std*" library until Microsoft corrects the situation).

You can now "*import FunctionTraits*" as required or rely on "*#include FunctionTraits.h*" instead, as described earlier (again, the latter simply calls "*import FunctionTraits*" for you in the module version but also picks up all macros from "*FunctionTraits.h*" itself).

<a name="MSVCModuleCompilationCommandPromptExample"></a>
#### MSVC module compilation at the command prompt

Most MSVC developers will likely rely on Visual Studio as described just above. The following example demonstrates how to compile modules at the command prompt for learning purposes only usually (using pure "*cl.exe*" and "*link.exe*"). A professional build tool like Visual Studio (or even [CMake](#cmakemodulecompilationexample)) is normally required however to handle module dependency issues among others. The example that follows manually builds everything in the correct dependency order for instance but manually handling this isn't realisitic for most real-world apps (which would be tedious and error-prone). See the earlier discussion for details (see [here](ModuleDependencies)).

In MSVC, when an import statement such as "*import FunctionTraits*" is encountered in a client file like [Test.cpp](#testcppmodule), MSVC will look for an "*.ifc*" file to resolve it. Recall that "*.ifc*" files are the BMI files created by MSVC when you compile a module (see the earlier discussion in [FunctionTraits Module Examples](#functiontraitsmoduleexamples) for details). The location of this file is specified via one of two possible switches which you pass on the "*cl.exe*" command line when you compile a file with an "*import*" statement (see [Using C++ Modules in MSVC from the Command Line Part 1: Primary Module Interfaces](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/)). The most prominent switch that most will likely rely on is [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) which just contains the directory to search for "*.ifc*" files, similar to MSVC's [/I](https://learn.microsoft.com/en-us/cpp/build/reference/i-additional-include-directories?view=msvc-170) switch for locating header files (but in this case applicable to "*import*" statements instead of "*#include*" - note that semicolons are *not* allowed as directory separators however - multiple [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) args must be passed instead, one for each directory to search, even though Visual Studio itself allows multiple directories separated by semicolons in its corresponding GUI setting under *[`Configuration Properties -> C/C++ -> General -> Additional BMI Directories`](https://learn.microsoft.com/en-us/cpp/build/reference/c-cpp-prop-page?view=msvc-170#additional-bmi-directories)* - they're converted to multiple [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) args behind the scenes however). The other switch is [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) (search for multiple occurrences of "*/reference*" [here](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/)), but the following example will use [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) instead. It's more likely to be used by developers in practice though interestingly, Visual Studio currently uses [/reference](https://learn.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170) behind the scenes when you build in that environment (but that's just an implementation detail).

If you have a subdirectory called, say, "*ifc.cache*" for instance (so named only to match the naming convention that GCC uses, except their's is called "*gcm.cache*" - you can store your "*.ifc*" files anywhere you wish however, such as directories "*Debug*" or "*Release*" or whatever project configuration name you're targeting), and "*ifc.cache*" exists as a subdirectory of the [Test.cpp](#testcppmodule) directory, then by passing *`/ifcSearchDir "ifc.cache\"`* when compiling [Test.cpp](#testcppmodule) (assuming you're compiling from the same directory where [Test.cpp](#testcppmodule) is located - adjust [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) otherwise), MSVC will look for "*FunctionTraits.ifc*" in directory "*ifc.cache*" to resolve any [Test.cpp](#testcppmodule) call to "*import FunctionTraits*". Note that the trailing backslash in *`"ifc.cache\"`* passed in the latter call to *`/ifcSearchDir "ifc.cache\"`* is by design (not mandatory but makes it more clear it's a directory). The quotes surrounding *`"ifc.cache\"`* are also optional in this case but provided for clarity (but required if the directory you pass has any spaces).

The above applies to clients actually calling "*import FunctionTraits*". "*FunctionTraits.ifc*" has to be built first though (from its module "*FunctionTraits.cppm*"), and unless you change the extension from "*.cppm*" to "*.ixx*" (the MSVC default - see [item 2](#msvcmodulecompilationitem2) in the Visual Studio discussion previously), you'll need to pass the switches [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) and [/TP](https://learn.microsoft.com/en-us/cpp/build/reference/tc-tp-tc-tp-specify-source-file-type?view=msvc-170) when you compile "*FunctionTraits.cppm*". Both switches are required as indicated in the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) docs itself, as well as [here](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/) (search for "*/TP*" at both links). The example that follows does this (since it retains the "*.cppm*" extension), but if you change the extension to "*.ixx*" then you can remove the [/interface](https://learn.microsoft.com/en-us/cpp/build/reference/interface?view=msvc-170) and [/TP](https://learn.microsoft.com/en-us/cpp/build/reference/tc-tp-tc-tp-specify-source-file-type?view=msvc-170) options.

Note that when you compile a module, the file's "*.ifc*" file is created in the directory specified via the "*/ifcOutput*" option (search for this [here](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#summary)), so in this case we pass the "*ifc.cache*" subdirectory, the same directory we passed via the [/ifcSearchDir](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#ifc-search-dir) option when compiling [Test.cpp](#testcppmodule) previously (the location where it expects to find "*FunctionTraits.ifc*"). Because we're passing a subdirectory name here instead of an actual file name (we explicitly append a trailing backslash to "*ifc.cache\\*" when passing it to "*/ifcOutput*" in the example below to ensure this), MSVC will create the file "*FunctionTraits.ifc*" in that directory so it has the same base name of the interface itself as would be expected (i.e., the name of the module based on its "*export module*" declaration, not the name of the file itself which isn't relevant whatsoever - interestingly however, when compiling in Visual Studio it creates "*FunctionTraits.cppm.ifc*" instead at this writing, i.e., it maintains the "*.cppm*" extension or "*.ixx*" if you renamed it). Note that you can also pass an explicit file name for the "*.ifc*" file to "*/ifcOutput*" if you wish (such as "*/ifcOutput .\\ifc.cache\\FunctionTraits.ifc*"), instead of passing just the target directory itself as we're doing in this example (where the name "*FunctionTraits.ifc*" is then implicitly generated for you). Most will pass the directory only though to "*/ifcOutput*" as this example does.

In addition to "*FunctionTraits.ifc*" however, the module's object file ("*FunctionTraits.obj*") is also created as usual, no different than any other compiled C++ file. The location of the object file is specified via the [/Fo](https://learn.microsoft.com/en-us/cpp/build/reference/fo-object-file-name?view=msvc-170) option. We simply pass "*build\\*" in the following example for simplicity (in a more real world example it would likely be under a "*Debug*" or "*Release*" directory instead, the project configuration names used in most Visual Studio builds). Note that the trailing backslash in "*build\\*" is recommended to ensure it's properly interpreted as a directory (in the very unlikely case that "*build*" exists as a file).

Lastly, note that it's assumed you're running the following "*.cmd*" file from the same directory as [Test.cpp](#testcppmodule) itself. The "*.cmd*" file also assumes that "*cl.exe*" and "*link.exe*" are in the path which you should normally make available by opening an "*x86 Native Tools Command Prompt*" (from the Windows Start menu, type "*x86 Native*" and the prompt should appear in the list of apps). Compiler errors will occur if you use a version of MSVC that doesn't support modules (i.e., versions prior to V16.8 - see [Footnote 2](README.md#footnotes)).

```cmd
@echo off

REM ///////////////////////////////////////////////////////
REM // Compile all files which we output to the "build"
REM // subdirectory created just below (modules are
REM // compiled to object files no different than
REM // non-module files). Note however that by passing
REM // /interface which is only required here because
REM // we've opted not to change the ".cppm" extensions
REM // below to ".ixx" (the default for MSVC modules though
REM // most MSVC developers likely will), each call to
REM // compile a module's ".cppm" file also creates the
REM // module's ".ifc" file in a subdirectory called
REM // "ifc.cache" (also created just below and so named
REM // only to be consistent with GCC's "gcm.cache" folder
REM // that's the hardwired default in GCC - in MSVC
REM // however you can output the ".ifc" files anywhere
REM // you want via /ifcOutput). See here for a summary of
REM // all MSVC module options:
REM //
REM //    Summary of C++ modules options
REM //    https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#summary
REM //
REM // Lastly, note that we compile all files in module
REM // dependency order so that when "import" statements
REM // are encountered in a given file, the compiler can
REM // locate its ".ifc" file since it's already been
REM // previously built. Build systems like Visual Studio
REM // and "CMake" handle the order for you (see footnote
REM // 12 at the end of this file), but we're manually
REM // compiling the files ourselves here so we need to
REM // ensure they're compiled in the correct module
REM // dependency order (though we could and normally
REM // should rely on footnote 12 in a real-world app but
REM // it would obviously make this example much more
REM // complicated).
REM ///////////////////////////////////////////////////////
if not exist ifc.cache md ifc.cache
if not exist build md build
call :Run cl.exe /nologo /std:c++20 /EHsc /D STDEXT_USE_MODULES /ifcSearchDir "ifc.cache\\" /interface /TP /c "CompilerVersions.cppm" /ifcOutput "ifc.cache\\" /Fo:"build\\" || goto Error
call :Run cl.exe /nologo /std:c++20 /EHsc /D STDEXT_USE_MODULES /ifcSearchDir "ifc.cache\\" /interface /TP /c "FunctionTraits.cppm" /ifcOutput "ifc.cache\\" /Fo:"build\\" || goto Error
call :Run cl.exe /nologo /std:c++20 /EHsc /D STDEXT_USE_MODULES /ifcSearchDir "ifc.cache\\" /c "Test.cpp" /Fo:"build\\" || goto Error

REM ///////////////////////////////////////////////////////
REM // Link
REM ///////////////////////////////////////////////////////
call :Run link.exe /nologo "build\\CompilerVersions.obj" "build\\FunctionTraits.obj" "build\\Test.obj" /OUT:"build\\Test.exe" || goto Error
echo Done! (successful - created "build\Test.exe")
goto Done

REM ########################################
REM # Trivial helper subroutine to display
REM # any passed command and then run it
REM # (unsophisticated but meets our needs)
REM ########################################
:Run
echo %*
%* || exit /B 1
echo.
exit /B 0

REM ##########################################
REM # Error handler
REM ##########################################
:Error
echo Failed!
exit /B 1

REM ##########################################
REM # Done (all successful)
REM ##########################################
:Done
```

<a name="MSVCModuleCompilationCommandPromptImportStdExample"></a>
#### MSVC module compilation at the command prompt supporting "*import std*"

The above code demonstrates how to support "*import FunctionTraits*" in an MSVC program at the command prompt (using pure "*cl.exe*" and "*link.exe*"). If you wish to support "*import std*" as well, then several changes are required to the above code (an updated example follows). These are as follows (for further details see [Tutorial: Import the C++ standard library using modules from the command line](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module?view=msvc-170)):

1. Add the following call to compile the module version of the "*std*" library into its own "*.ifc*" file similar to "*CompilerVersions.ifc*" and "*FunctionTraits.ifc*" in the above example:
   ```cmd
   call :Run cl.exe /nologo /std:c++23preview /EHsc /ifcOnly "%VCToolsInstallDir%\\modules\\std.ixx" /ifcOutput "ifc.cache\\" || goto Error
   ```

   The module version of the "*std*" library is no different than any other module in this respect (though the object file itself normally doesn't have to be created - read on). *MSVC* ships the "*std*" module in a file called "*modules\std.ixx*", where "*modules*" is at the same hierarchal level as the "*include*" directory containing the "*std*" headers themselves.  When you open an "*x86 Native Tools Command Prompt*" as described earlier, the parent directory of both "*include*" and "*modules*" is stored in the *%VCToolsInstallDir%* environment variable so "*std.ixx*" is located under *%VCToolsInstallDir%\modules* as seen in the call above. Note that we also pass [/ifcOnly](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#summary) above to create the "*.ifc*" file only, and therefore suppreses creation of the object file which isn't required for the module version of the "*std*" library. It doesn't need to be explicitly linked since any required code from the "*std*" library is implicitly pulled in at link time[^4] (note that the [/c](https://learn.microsoft.com/en-us/cpp/build/reference/c-compile-without-linking?view=msvc-170) option can also be omitted since [/ifcOnly](https://devblogs.microsoft.com/cppblog/using-cpp-modules-in-msvc-from-the-command-line-part-1/#summary) is being passed).
2. Change *[/std:c++20](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170)* to *[/std:c++23preview](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170)* (or later). "*import std*" only became available in C++23 but *[c++23preview](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170)* or even *[c++latest](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170)* is still required by Microsoft at this writing (until the "*c++23*" keyword is eventually supported).
3. Not including "*std.ixx*" itself (read on), add the "*FunctionTraits*" macro *STDEXT_IMPORT_STD_EXPERIMENTAL* to all "*cl.exe"* calls (via the [/D](https://learn.microsoft.com/en-us/cpp/build/reference/d-preprocessor-definitions?view=msvc-170) option). This macro notifies "*FunctionTraits*" that it should rely on "*import std*" instead of the usual "*#include*" calls to the "*std*" headers it normally relies on (in a non-module build). This macro isn't required for "*std.ixx*" itself however (the macro affects "*FunctionTraits*" files only).

The following code implements the changes just described:

```cmd
@echo off

REM ///////////////////////////////////////////////////////
REM // Compile all files similar to the previous example.
REM // Everything there applies here as well except we've
REM // now added support for "import std" as described
REM // above.
REM ///////////////////////////////////////////////////////
if not exist ifc.cache md ifc.cache
if not exist build md build
call :Run cl.exe /nologo /std:c++23preview /EHsc /ifcOnly "%VCToolsInstallDir%\\modules\\std.ixx" /ifcOutput "ifc.cache\\" || goto Error
call :Run cl.exe /nologo /std:c++23preview /EHsc /D STDEXT_USE_MODULES /D STDEXT_IMPORT_STD_EXPERIMENTAL /ifcSearchDir "ifc.cache\\" /interface /TP /c "CompilerVersions.cppm" /ifcOutput "ifc.cache\\" /Fo:"build\\" || goto Error
call :Run cl.exe /nologo /std:c++23preview /EHsc /D STDEXT_USE_MODULES /D STDEXT_IMPORT_STD_EXPERIMENTAL /ifcSearchDir "ifc.cache\\" /interface /TP /c "FunctionTraits.cppm" /ifcOutput "ifc.cache\\" /Fo:"build\\" || goto Error
call :Run cl.exe /nologo /std:c++23preview /EHsc /D STDEXT_USE_MODULES /D STDEXT_IMPORT_STD_EXPERIMENTAL /ifcSearchDir "ifc.cache\\" /c "Test.cpp" /Fo:"build\\" || goto Error

REM ////////////////////////////////////////////////////////////
REM // Link (note that the "std" library object file not req'd
REM // here - we didn't build it above in fact, only its ".ifc"
REM // file - all required code from the "std" library will be
REM // implicily linked in in the following call using the MSVC
REM // libraries containing that code).
REM ////////////////////////////////////////////////////////////
call :Run link.exe /nologo "build\\CompilerVersions.obj" "build\\FunctionTraits.obj" "build\\Test.obj" /OUT:"build\\Test.exe" || goto Error
echo Done! (successful - created "build\Test.exe")
goto Done

REM ########################################
REM # Trivial helper subroutine to display
REM # any passed command and then run it
REM # (unsophisticated but meets our needs)
REM ########################################
:Run
echo %*
%* || exit /B 1
echo.
exit /B 0

REM ##########################################
REM # Error handler
REM ##########################################
:Error
echo Failed!
exit /B 1

REM ##########################################
REM # Done (all successful)
REM ##########################################
:Done
```

<a name="ClangModuleCompilationExample"></a>
### Clang module compilation (official Clang documentation [here](https://clang.llvm.org/docs/StandardCPlusPlusModules.html))
Most Clang developers will likely rely on [CMake](#cmakemodulecompilationexample). The following example demonstrates how to compile modules in the terminal for learning purposes only usually (using pure *clang++*). A professional build tool like [CMake](#cmakemodulecompilationexample) is normally required however to handle module dependency issues among others. The example that follows manually builds everything in the correct dependency order for instance but manually handling this isn't realisitic for most real-world apps (which would be tedious and error-prone). See the earlier discussion for details (see [here](#moduledependencies)).

In Clang, when an import statement such as "*import FunctionTraits*" is encountered in a client file like [Test.cpp](#testcppmodule), Clang will look for a "*.pcm*" file to resolve it. Recall that "*.pcm*" files are the BMI files created by Clang when you compile a module (see the earlier discussion in [FunctionTraits Module Examples](#functiontraitsmoduleexamples) for details). The location of this file is specified via one of two possible switches which you pass on the "*clang++*" command line when you compile a file with an "*import*" statement (see [Specifying dependent BMIs](https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#specifying-dependent-bmis) in the Clang docs). The most prominent switch that most will likely rely on is "*-fprebuilt-module-path=*" which just contains the path to search for "*.pcm*" files, similar to Clang's [-I](https://clang.llvm.org/docs/ClangCommandLineReference.html#include-path-management) switch for locating header files (but in this case applicable to "*import*" statements instead of "*#include*").

If you have a subdirectory called, say, "*pcm.cache*" for instance (so named only to match the naming convention that GCC uses, except their's is called "*gcm.cache*" - you can store your "*.pcm*" files anywhere you wish however), and "*pcm.cache*" exists as a subdirectory of the [Test.cpp](#testcppmodule) directory (again, where "*pcm.cache*" is so named simply to parallel GCC's own "*gcm.cache*" directory), then by passing "*-fprebuilt-module-path=pcm.cache*" when compiling [Test.cpp](#testcppmodule) (assuming you're compiling from the same directory where [Test.cpp](#testcppmodule) is located - adjust "*-fprebuilt-module-path*" otherwise), Clang will look for "*pcm.cache/FunctionTraits.pcm*" to resolve any [Test.cpp](#testcppmodule) call to "*import FunctionTraits*".

The above applies to clients actually calling "*import FunctionTraits*". "*FunctionTraits.pcm*" has to be built first though (from its module "*FunctionTraits.cppm*"), and unlike GCC, which automatically creates "*gcm.cache/FunctionTraits.gcm*" for you when you compile "*FunctionTraits.cppm*" (the default behavior when you compile a module - its "*.gcm*" file is automatically created in the "*gcm.cache*" directory), Clang requires you to explicitly pass an appropriate switch to build your "*.pcm*" files instead. Two possible switches exist for this, "*--precompile*" and "*-fmodule-output*" (see [How to produce a BMI](https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#how-to-produce-a-bmi) in the Clang docs). When you use the "*--precompile*" switch, Clang compiles your "*.cppm*" file into a "*.pcm*" file (in the directory specified via the [-o](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-o-file) switch), but doesn't compile the "*.cppm*" file into an object file yet. A separate compile for that is still required (read on), unlike when using the "*-fmodule-output*" switch which creates both the "*.pcm*" file *and* the object file in a single call (again, see [How to produce a BMI](https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#how-to-produce-a-bmi)). When you use the "*--precompile*" switch however, note that you should no longer build the "*.cppm*" file directly itself (into an object file). It still needs to be compiled because all "*--precompile*" does is create the "*.pcm*" file (for resolving client calls to "*import*"), but instead of compiling the "*.cppm*" file directly, you should compile the "*.pcm*" file instead (into an object file). While you could still compile the "*.cppm*" file instead, if you do and it's been changed since creating the "*.pcm*" file, then the resulting object file might not match the "*.pcm*" file that was previously created (using the "*--precompile*" switch). Since the "*.pcm*" file is used to resolve calls to "*import*" in client code, such as "*import FunctionTraits*", any mismatch between the "*.pcm*" file and the object file will result in possible compiler errors (or more seriously but probably unlikely, a runtime error but the situation is murky - it isn't currently mentioned in the Clang documentation). The "*.pcm*" file created by Clang is therefore designed to be directly compiled and it should be instead of the original "*.cppm*" file it was created from (to create its object file). Doing so therefore guarantees that the "*.pcm*" file Clang looks for to resolve "*import*" statements correctly matches the module's object file that's later passed to the linker. They'll match because the object file itself is created from the same "*.pcm*" file. When using the "*-fmodule-output*" switch instead of "*--precompile*" however, there is no such issue because both the "*.pcm*" and "*.o*" (object) file are created at the same time (so they'll always match - note that the [GCC](#gccmodulecompilationexample) and [MSVC](#msvcmodulecompilationexample) examples do it that way and unlike Clang, neither supports compiling their respective BMI files into object files anyway). The "*-fmodule-output*" switch is therefore less error-prone and simpler to use than "*--precompile*" but the latter switch has the potential to compile faster. Again, see [How to produce a BMI](https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#how-to-produce-a-bmi) for details (you're free of course to rely on "*-fmodule-output*" instead of "*--precompile*" if you wish - consult the latter link for details).

Here's the minimum code to build the module version of [Test.cpp](#testcppmodule) (also see [Quick Start](https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#quick-start) in the Clang docs). As described above, this example relies on the "*--precompile*" switch to create the "*.pcm*" files first (placing them in a subdirectory called "*pcm.cache*"), and then subsequently compiles these "*.pcm*" files into object files as described above (instead of compiling the original "*.cppm*" files they originated from as previously described). [Test.cpp](#testcppmodule) itself is also compiled but normally (i.e., not as a module, just a module client so no "*.pcm*" is required for it). Note that the following code relies on the GCC 15.2 tool chain passed via the *[--gcc-toolchain](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-gcc-toolchain)=/opt/gcc-15.2* option (the latest version of GCC at this writing), rather than the default version of GCC on the host machine (affecting most distros of Linux for instance - environments where the default version of GCC is sufficiently new to handle modules, or where Clang is exclusively used such as on macOS, can remove this option). It also defaults to the [-stdlib](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-stdlib)*=stdlibc++* option (consult [here](https://libcxx.llvm.org/UserDocumentation.html#using-libc-when-it-is-not-the-system-default)), since [-stdlib](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-stdlib) isn't explicitly passed so Clang defaults to the [The GNU C++ Library](https://gcc.gnu.org/onlinedocs/libstdc++/), AKA GCC standard library on most Linux systems (which this example assumes). To use the [libc++](https://libcxx.llvm.org/) standard library from Clang, a separate install is normally required on Linux (but it's usually the default on other OSs like macOS, FreeBSD, etc.), and you'll then normally pass the [-stdlib](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-stdlib)*=libc++* switch to target it (again, unless it's already the default on your OS). Note that this example also assumes clang++ is installed (and in the PATH), and that GCC 15.2 was also installed under "*/opt/gcc-15.2*" (overriding the default version of GCC on the host version of Linux in this example - the [--gcc-toolchain](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-gcc-toolchain) switch ensures this). Your own environment will likely differ of course, so you'll have to adjust these particular switches accordingly in the code below.

```bash
#!/bin/bash

set -e # Exit if any command fails

########################################
# Trivial helper function to display
# any passed command and then run it
# (unsophisticated but meets our needs)
########################################
run() {
    echo "$1"
    eval "$1"
    echo
}

##########################################################
# Create ".pcm" files which we output to the "pcm.cache"
# subdirectory created just below (so named only to be
# consistent with GCC's "gcm.cache" folder that's the
# hardwired default in GCC - in Clang however you can
# output the ".pcm" files anywhere you want). Note that
# we rely on Clang's --precompile flag below but we
# could rely on the -fmodule-output option instead
# (which is actually simpler). See here for details:
#
#   How to produce a BMI
#   https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#how-to-produce-a-bmi
#
#   Quick Start (Clang)
#   https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#quick-start
#########################################################
mkdir -p "pcm.cache"
run 'clang++ -std=c++20 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -fprebuilt-module-path="pcm.cache" --precompile "CompilerVersions.cppm" -o "pcm.cache/CompilerVersions.pcm"'
run 'clang++ -std=c++20 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -fprebuilt-module-path="pcm.cache" --precompile "FunctionTraits.cppm" -o "pcm.cache/FunctionTraits.pcm"'

########################################################
# Compile all files which we output to the "build"
# subdirectory created just below (modules are
# compiled to object files no different than
# non-module files). We rely on the GCC 15.2 toolchain
# as seen in the calls below (any version of GCC >=
# 15.2 will do here though Clang and libc++ can also
# be used instead), and the "-fprebuilt-module-path"
# here is used to tell Clang where to look for ".pcm"
# files to resolve calls to "import" in these files
# (in this case the "pcm.cache" subdirectory where
# we created these files just above). See the Clang
# docs for details:
#
#    Specifying dependent BMI
#    https:/releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#specifying-dependent-bmis
#
# Lastly, note that we compile all files in module
# dependency order so that when "import" statements
# are encountered in a given file, the compiler can
# locate its ".pcm" file since it's already been
# previously built. Build systems like "CMake" handle
# the order for you (see footnote 12 at the end of
# this file), but we're manually compiling the files
# ourselves here so we need to ensure they're compiled
# in the correct module dependency order (though we
# could also rely on footnote 12 ourselves for that but
# it would obviously make this example much more
# complicated).
########################################################
mkdir -p "build"
run 'clang++ -std=c++20 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -fprebuilt-module-path="pcm.cache" -c "pcm.cache/CompilerVersions.pcm" -o "build/CompilerVersions.o"'
run 'clang++ -std=c++20 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -fprebuilt-module-path="pcm.cache" -c "pcm.cache/FunctionTraits.pcm" -o "build/FunctionTraits.o"'
run 'clang++ -std=c++20 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -fprebuilt-module-path="pcm.cache" -c "Test.cpp" -o "build/Test.o"'

# Link
run 'clang++ -std=c++20 --gcc-toolchain=/opt/gcc-15.2 "build/CompilerVersions.o" "build/FunctionTraits.o" "build/Test.o" -o "build/Test"'

echo 'Done! (successful - created "build/Test")'
```

<a name="ClangModuleCompilationSupportingImportStd"></a>
#### Clang module compilation supporting "*import std*"

The above code demonstrates how to support "*import FunctionTraits*" in a Clang program (using pure *clang++*). If you wish to support "*import std*" as well, then several changes are required to the code just above. These are as follows (an updated example incorporating the following steps immediately follows):

1. Add the following call to compile the module version of the "*std*" library into its own "*.pcm*" file similar to "*CompilerVersions.pcm*" and "*FunctionTraits.pcm*" and above (note the following targets "*libc++*" since "*libstdc++*" isn't yet supported by Clang at this writing - details to follow):

   ```bash
   run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -stdlib=libc++ --precompile -Wno-reserved-module-identifier "/opt/clang-21.1/share/libc++/v1/std.cppm" -o "pcm.cache/std.pcm"'
   ```

   The module version of the "*std*" library is no different than any other module in this respect (though it doesn't have to be explicitly linked in like other modules since it occurs implicitly via the vendor's libraries at link time[^4]). Assuming your version of *Clang* was built with [LIBCXX_INSTALL_MODULES=ON](https://libcxx.llvm.org/VendorDocumentation.html#cmdoption-arg-LIBCXX_INSTALL_MODULES-BOOL) (normally the default but see footnote [^6]), *Clang* ships the "*std*" module in a file called "*std.cppm*" though you (ideally) shouldn't hardcode this name as this example does (to simplify things). Clang provides a compiler option called [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path) to locate the "*std*" module instead. Unfortunately this doesn't return the path of "*std.cppm*" itself, but the path to a file called "*libc++.modules.json*" instead (in current releases of "*libc++*"). Moreover, when targeting "*libstdc++*" instead of "*libc++*", as many developers will do, it returns the path to a file called "*libstdc++.modules.json*" instead. In either case, whether it returns "*libc++.modules.json*" when targeting "*libc++*" or "*libstdc++.modules.json*" when targeting "*libstdc++*", you must then read this file using a JSON parser to locate the path to the "*std*" module itself, either "*std.cppm*" in "*libc++*" or "*std.cc*" in "*libstdc++*" (in the latter case, note that this is the same file seen in the [GCC module compilation supporting "*import std*](#gccmodulecompilationsupportingimportstdexample) example earlier). However, at this writing, "*libstdc++*" isn't yet supported by Clang (for use with "*import std*"), so "*libc++*" *must* be used instead, at least for now (presumably this will be corrected at a later date - in current releases Clang can compile "*std.cc*" but the resulting "*std.pcm*" file fails to import correctly). Moreover, even when targeting "*libc++*", current versions of Clang are also buggy and *may* return "*\<NOT PRESENT\>*" when [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path) is called, requiring a work-around to locate "*libc++.modules.json*" - the work-around is discussed momentarily).

   "*libc++.modules.json*" (and when "*libstdc++*" is eventually supported, "*libstdc++.modules.json*"), is a fairly small file you can easily inspect for yourself to determine how to locate "*std.cppm*" (or "*std.cc*" when "*libstdc++*" is eventually supported). Its "*source-path*" entry (see file) shows its location relative to the "*libc++.modules.json*" file itself. If you have, say, Clang V21.1 installed in "*opt/clang-21.1/*" for instance and the "*source-path*" entry in "*libc++.modules.json*" shows *"../../share/libc++/v1/std.cppm*", then "*std.cppm*" will be found under "*clang-21.1/share/libc++/v1/std.cppm*". The upshot is that to compile "*std.cppm*" (or when "*libstdc++*" is eventually supported, "*std.cc*"), the module to satisfy calls to "*import std*", you need to locate it first as just described, which is fairly simple to do with a JSON parser like Python. This is usually preinstalled on Linux for instance (for versions of Python >= 2.6), or you can use any other JSON parser like [jq](https://jqlang.org/) which you'll have to install. Writing the function to locate "*std.cppm*" itself is beyond the scope of this documentation however. It's fairly easy though and all AI engines are quite capable of writing the code for you with decent accuracy (though always verify it first of course). Note that normally you'll need to simply add this function to the above Bash file and have it first invoke "*clang++ [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path)*" to return the full path of "*libc++.modules.json*" (or "*libstdc++.modules.json*" when "*libstdc++*" is eventually supported). Then simply parse this file with Python or [jq](https://jqlang.org/) (or any JSON parser) in order to obtain the full path of "*std.cppm*" itself (or "*std.cc*" when "*libstdc++*" is eventually supported). However, as noted previously, "*clang++ [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path)*" is often (officially) buggy at this writing (see [here](https://github.com/llvm/llvm-project/issues/73089#issuecomment-2196357282) which leads to [here](https://github.com/llvm/llvm-project/issues/97025)) so [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path) *may* erroneously return "*\<NOT PRESENT\>*" in current releases (though testing shows it seems to be stable in the version of Clang being used in this example, V21.1, but the situation remains unclear at this writing). Your function will therefore have to test for this situation ("*\<NOT PRESENT\>*" returned) and other possible failure scenarios, usually because "*libc++.modules.json*" isn't intalled (normally because you haven't installed the module version of *Clang* itself, meaning versions where the following are passed to the "*CMake*" install app for Clang):
    1. "*-D[LIBCXX_INSTALL_MODULES](https://libcxx.llvm.org/VendorDocumentation.html#cmdoption-arg-LIBCXX_INSTALL_MODULES-BOOL)=ON*"
    2. "*-DLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind*" (in order to install "*libc++*" itself, at least until "*libstdc++*" is eventually supported). These are normally the default libraries that are installed. Search for (multiple occurences) of *"LLVM_ENABLE_RUNTIMES*" (quotes not included) [here](https://llvm.org/docs/CMake.html), and see [The default build](https://libcxx.llvm.org/VendorDocumentation.html#the-default-build) for further details.

    If  "*\<NOT PRESENT\>*" is returned by [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path), you can always fallback to calling "*clang++ --[print-file-name](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-file-name)=libc++.modules.json*" instead if you wish (see [Can someone give an authoritative answer as to what mechanism a build system vendor should use](https://github.com/llvm/llvm-project/issues/73089#issuecomment-2196389804) - note that the poster "*ChuanqiXu9*" is an official modules contributor for LLVM - search for his name [here](https://clang.llvm.org/docs/Maintainers.html)). This is effectively the same as calling "*clang++ [--print-library-module-manifest-path](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-library-module-manifest-path)*", normally the de facto way (again, see *ChuanqiXu9's* post in the link above), so relying on "*clang++ --[print-file-name](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-print-file-name)=libc++.modules.json*" instead may not be potentially reliable (though it very likely is when targeting "*libc++*"). You'll therefore have to check if that fails as well. If it does it will normally just return the same (pathless) "*libc++.modules.json*" arg you passed it so you should test for this.

   To avoid the necessity of having to write such a function in the above code however (since the focus of this document is on the "*FunctionTraits*" library, not the intricacies of locating "*std.cppm*" to support "*import std*"), the following code simply hardcodes the path to "*std.cppm*" instead. In real-world code however a function should be written to retrieve it using the official Clang method described above. Therefore, to support "*import std*" in the example above (using the hardcoded path as described), simply add the above line as the first line in the section that precompiles the "*FunctionTraits*" modules themselves (and simply change the path of "*std.cppm*" to match your own version of this file, or write the function just described to do it). Note that it must be the first line since it must precede any modules that have (or might have) an "*import std*" statement (since C++ modules must always be precompiled to their BMI files before compiling any file that imports them). Hence we call it before building the "*CompilerVersions*" and "*FunctionTraits*" modules (since they both depend on "*import std*").
2. Change "*[-std](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-std)=c++20*" to "*[-std](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-std)=c++23*" (or later). "*import std*" only became available in C++23.
3. Add "*[-stdlib](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-stdlib)=libc++*" to each file so that it uses [libc++](https://libcxx.llvm.org/) (which you installed as described in 1 above - as described there, the use of "*stdlib*" isn't yet supported by Clang when "*import std*" is required). [libc++](https://libcxx.llvm.org/) may already be the default anyway on your particular system (such as macOS), but it's arguably better to pass "*[-stdlib](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-stdlib)=libc++*" anyway just to make things clear.
4. You can continue to use "*[--gcc-toolchain](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-gcc-toolchain)=/opt/gcc-15.2*" (or whatever GCC toolchain you're using in your own code assuming you are - normally the case on Linux for instance), but if you want to completely rely on the Clang toolchain (on Linux or any platform where GCC is the native C++ compiler - Clang is already the default on platforms like macOS), you'll need to research the additional clang++ options required to do so (not a typical setup for most users and beyond the scope of this documentation).
5. Add the "*FunctionTraits*" macro *STDEXT_IMPORT_STD_EXPERIMENTAL* to all "*clang++"* calls (via the [-D](https://clang.llvm.org/docs/ClangCommandLineReference.html#cmdoption-clang-define-macro) option), except the final call to link (since macros aren't required during linking). This macro notifies "*FunctionTraits*" that it should rely on "*import std*" instead of the usual "*#include*" calls to the "*std*" headers it normally relies on (in a non-module build). This macro isn't required for "*std.cppm*" itself (the macro affects "*FunctionTraits*" files only).

The following code implements the changes just described:

```bash
#!/bin/bash

set -e # Exit if any command fails

########################################
# Trivial helper function to display
# any passed command and then run it
# (unsophisticated but meets our needs)
########################################
run() {
    echo "$1"
    eval "$1"
    echo
}

#########################################################
# Create ".pcm" files which we output to the "pcm.cache"
# subdirectory created just below (so named only to be
# consistent with GCC's "gcm.cache" folder that's the
# hardwired default in GCC - in Clang however you can
# output the ".pcm" files anywhere you want). Note that
# we rely on Clang's --precompile flag below but we
# could rely on the -fmodule-output option instead
# (which is actually simpler). See here for details:
#
#   How to produce a BMI
#   https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#how-to-produce-a-bmi
#
#   Quick Start (Clang)
#   https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#quick-start
#
# Note that 1st call below targets the "std" library
# itself, and we're using the hardcoded path to
# "std.cppm" installed with the module version of libc++
# (as described earlier, Clang's libc++ is req'd since
# GCC's libstdc++ isn't yet supported by Clang when
# targeting a module application in Clang). In a
# production release however you should always resolve
# "std.cppm" properly as described earlier. Also note
# that "-Wno-reserved-module-identifier" is passed to
# eliminate the following warning at this writing:
#
#    "std" is a reserved name for a module"
#
# This occurs in "std.cppm" itself (on its own call to
# "export module std") so we suppress the warning. See:
# https://clang.llvm.org/docs/StandardCPlusPlusModules.html#module-name-requirements
##########################################################
mkdir -p "pcm.cache"
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -stdlib=libc++ --precompile -Wno-reserved-module-identifier "/opt/clang-21.1/share/libc++/v1/std.cppm" -o "pcm.cache/std.pcm"'
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -stdlib=libc++ -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -fprebuilt-module-path="pcm.cache" --precompile "CompilerVersions.cppm" -o "pcm.cache/CompilerVersions.pcm"'
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -stdlib=libc++ -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -fprebuilt-module-path="pcm.cache" --precompile "FunctionTraits.cppm" -o "pcm.cache/FunctionTraits.pcm"'

##########################################################
# Compile files (output to the "build" subdirectory that
# we create here). We pass "-fprebuilt-module-path" here
# to tell Clang where to look for ".pcm" files to resolve
# calls to "import" (in the "pcm.cache" subdirectory we
# created these files in just above). See the Clang docs
# for details:
#
#   Specifying dependent BMI
#   https://releases.llvm.org/20.1.0/tools/clang/docs/StandardCPlusPlusModules.html#specifying-dependent-bmis
##########################################################
mkdir -p "build"
# Note that -stdlib=libc++ not required when compiling ".pcm" files (results in "argument unused" warnings) ...
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -fprebuilt-module-path="pcm.cache" -c "pcm.cache/CompilerVersions.pcm" -o "build/CompilerVersions.o"'
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -fprebuilt-module-path="pcm.cache" -c "pcm.cache/FunctionTraits.pcm" -o "build/FunctionTraits.o"'
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -stdlib=libc++ -DSTDEXT_USE_MODULES -DSTDEXT_IMPORT_STD_EXPERIMENTAL -fprebuilt-module-path="pcm.cache" -c "Test.cpp" -o "build/Test.o"'

##########################################################
# Link (note that a "std" library object file not req'd
# here - we didn't build one above in fact, only its
# ".pcm" file - any required code for the "std" library
# will be implicily linked in in the following call using
# the vendor's libraries containing that code).
##########################################################
run 'clang++ -std=c++23 --gcc-toolchain=/opt/gcc-15.2 -stdlib=libc++ "build/CompilerVersions.o" "build/FunctionTraits.o" "build/Test.o" -o "build/Test"'

echo 'Done! (successful - created "build/Test")'
```

<a name="Footnotes"></a>
## Footnotes

[^1]: ***Disabling "import FunctionTraits" statement in module version of "FunctionTraits.h"***

    As described in [Module support in C++20 or later](#moduleusage), when the macro STDEXT\_USE\_MODULES is #defined to activate the module version of "FunctionTraits", the behavior of both "FunctionTraits.h" and "CompilerVersions.h" changes so that #including these files preprocesses out all code declarations and replaces them with an import statement instead. The statement _#include "FunctionTraits.h"_ for instance effectively becomes identical to "*import FunctionTraits*" instead, but with the benefit of #defining all macros in "FunctionTraits.h" as well (since macros aren't exported by C++ modules). If you don't require any macros from "FunctionTraits.h" however (or "CompilerVersions.h" since "FunctionTraits.h" always #includes it), than you can simply issue you own "*import FunctionTraits*" statement instead. This is more natural than relying on "*#include FunctionTraits.h*" which will appear misleading to casual readers not familiar with the situation (that it actually turns into an "*import FunctionTraits*" statement in the module version of the library).

    If you don't wish for "FunctionTraits.h" to include an "*import FunctionTraits*" statement in the module version however, but still wish to _#include "FunctionTraits.h"_ to pick up any of its macros, then simply #define either the macro FUNCTION\_TRAITS\_MACROS\_ONLY or STDEXT\_MACROS\_ONLY. The former affects "FunctionTraits.h" only and is usually intended to be #defined just before any _#include "FunctionTraits.h"_ statement and #undefined right after. The latter macro however, STDEXT\_MACROS\_ONLY, should be #defined in your project settings itself since it affects all .h files in the library, though there are only two of them ("FunctionTraits.h" and "CompilerVersions.h"). Relying on STDEXT\_MACROS\_ONLY therefore means you don't need to rely on the  FUNCTION\_TRAITS\_MACROS\_ONLY macro whatsoever. Both macros simply inform "FunctionTraits.h" that it shouldn't include an "*import FunctionTraits*" statement (the purpose of #defining these macros), so it will simply #define all its public macros instead (all code in the header is preprocessed out by the presence of the STDEXT\_MACROS\_ONLY macro itself). It's then your responsibility however to issue your own "*import FunctionTraits*" statement where required to pick up the code from the "FunctionTraits" module (since it was preprocessed out of "FunctionTraits.h" itself). The only difference between using FUNCTION\_TRAITS\_MACROS\_ONLY and STDEXT\_MACROS\_ONLY is that the former affects "FunctionTraits.h" only (so it's usually intended to be #defined just before you #include it and #undefined right after), while the latter macro affects both "FunctionTraits.h" _and_ "CompilerVersions.h" (both headers so it's usually intended to be #defined in your project settings). Note that for "CompilerVersions.h" (for those who may wish to #include it directly but it's usually not necessary since it's #included automatically in "FunctionTraits.h" itself), it has its own macro called COMPILER\_VERSIONS\_MACROS\_ONLY that's analogous to FUNCTION\_TRAITS\_MACROS\_ONLY. Therefore, if you wish to #include "CompilerVersions.h" directly for some reason (again, not necessary if you #include "FunctionTraits.h" itself which automatically #includes it), and you only wish to #define the macros in "CompilerVersions.h" and not issue an _import CompilerVersions_ statement, then similar to FUNCTION\_TRAITS\_MACROS\_ONLY, #define COMPILER\_VERSIONS\_MACROS\_ONLY just before any _#include "CompilerVersions.h"_ statement and the "_import CompilerVersions_" declaration in the file will no longer occur (and you should then #undef COMPILER\_VERSIONS\_MACROS\_ONLY right after _#include "CompilerVersions.h_"). The "_import CompilerVersions_" in "CompilerVersions.h" will then be suppressed so only its macros will then be #defined. Since all its code is already preprocessed out by the presence of the STDEXT_USE_MODULES macro, it's therefore your responsibility to issue your own "_import CompilerVersions_" statement if you require any of the exported items in the module (though "import FunctionTraits" will also work since it exports the "CompilerVersions" module as well). #defining STDEXT\_MACROS\_ONLY will also do however instead of relying on COMPILER\_VERSIONS\_MACROS\_ONLY (as previously described).

    Please note that all three macros described above, FUNCTION\_TRAITS\_MACROS\_ONLY, COMPILER\_VERSIONS\_MACROS\_ONLY and STDEXT\_MACROS\_ONLY, only apply to the module version of the "FunctionTraits" library, i.e., when the macro STDEXT\_MACROS\_ONLY has been #defined as described under [Module support in C++20 or later](#moduleusage). When STDEXT_USE_MODULES isn't #defined, i.e., when you're using the non-module version of the library, then the above three macros are completely ignored. #including either "FunctionTraits.h" or "CompilerVersions.h" (though again, "FunctionTraits.h" always #includes it for you), then behaves in the usual way, declaring everything in each file as would be expected (nothing is preprocessed out because you're using the "*.h*" version of the library, not the module version).

    Lastly, please note that you normally don't need to #define any of these macros, whose purpose is to disable the "*import FunctionTraits*" and/or "*import CompilerVersions*" statements in the module version of the library (in "FunctionTraits.h" and "CompilerVersions.h" as just desribed), since those statements are normally harmless (and actually beneficial since you don't have to issue them yourself, you can just rely on "FunctionTraits.h" and "CompilerVersions.h" in both the non-module and module version of the libray and they do the right thing). However, none of the supported "FunctionTraits" compilers at this writing fully supports modules yet (each has some limitations or missing features still), and GCC in particular has its own current limitation whereby *import* statements should normally follow any *#include* statements. Failing to adhere to this runs a risk of many cryptic compiler errors the full scope of when this can happen isn't well understood (as GCC's documentation is very sketchy on the subject). Therefore, if you #include "*FunctionTraits.h*" for instance and then #include a header from the standard library *afterwards* (subsequent to the #include "*FunctionTraits.h*" statement), you'll likely encounter many cryptic compiler errors because "FunctionTraits.h" issues an "*import FunctionTraits*" statement by default in the module version but you then #included a standard header afterward (i.e., you issued a "#include" statement after an "*import*" statement). GCC doesn't currently support this however (in most mainstream cases it would seem), so the use of the FUNCTION\_TRAITS\_MACROS\_ONLY or STDEXT\_USE\_MODULES macros therefore become necessary to suppress the call to "*import FunctionTraits*" in "FunctionTraits.h" (again, as described above). You can then just issue your own "*import FunctionTraits*" statement later on, after all .h files have been #included however, in order to prevent these GCC compiler errors. Having to do this annoying but necessary until GCC corrects the situation in its handling of C++ modules (likely not available until version 17.X of GCC at the earliest - see the "*Comment 20*" reply [here](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=114990#c20)). Note that MSVC and Clang don't have this issue however so the macros described in this footnote aren't normally required (though harmless if you wish to use them to suppress the "*import*" statements in "FunctionTraits.h" and/or "CompilerVersions.h" as described, and have them #define their macros only - it's then your responsibility to issue those "*import*" statements yourself, usually just "*import FunctionTraits*" since it always imports the "*CompilerVersions*" module as well).

[^2]: ***Document [P1689R5](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1689r5.html) (Format for describing dependencies of source files)***

    Note that each compiler provides a command line switch or separate tool to create the JSON file described by document [P1689R5](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1689r5.html), which can then used by build systems such as "*CMake*" to determine the correct build order of modules. See the following for each compiler:
    1. GCC: [fdeps-format](https://gcc.gnu.org/onlinedocs/gcc/C-Dialect-Options.html#index-fdeps-format)
    2. Clang: [clang-scan-deps](https://clang.llvm.org/docs/StandardCPlusPlusModules.html#discovering-dependencies)
    3. MSVC: [/scanDependencies](https://learn.microsoft.com/en-us/cpp/build/reference/scandependencies?view=msvc-170)

    The use of these switches/tools is beyond the scope this documentation however.

[^3]: ***fmodules vs fmodules-ts flag in GCC***

    TODO

[^4]: ***"std" library module's object file doesn't need to be explicitly created***

    TODO

[^5]: *** `Configuration Properties -> General -> C++ Language Standard` and `Configuration Properties -> C/C++ -> C++ Language Standard` ***

    Note that these two project properties in Visual Studio are the same. Changing one automatically changes the other.

[^6]: *** LIBCXX_INSTALL_MODULES may have been set to OFF ***

    versions installed using your system’s package manager usually set it to *OFF* instead - downloading the _full_ upstream version from the [LLVM Download Page](https://releases.llvm.org/download.html) or building from source is therefore required)

[^16]
"*inline*" functions are ultimately just a hint to the compiler so storage in the DLL is possible
