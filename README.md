
`iOS`动态链接器`dyld`中有一个神秘的变量`__dso_handle`:



```


|  | // dyld/dyldMain.cpp |
| --- | --- |
|  | static const MachOAnalyzer* getDyldMH() |
|  | { |
|  | #if __LP64__ |
|  | // 声明 __dso_handle |
|  | extern const MachOAnalyzer __dso_handle; |
|  | return &__dso_handle; |
|  | #else |
|  | ... |
|  | #endif // __LP64__ |
|  | } |


```

这个函数内部声明了一个变量`__dso_handle`，其类型是`struct MachOAnalyzer`。


查看`struct MachOAnalyzer`的定义，它继承自`struct mach_header`:


![image](https://img2024.cnblogs.com/blog/489427/202411/489427-20241119030000335-1894092699.png)


`struct mach_header`正是`XNU`内核里面，定义的`Mach-O`文件头:



```


|  | // EXTENERL_HEADERS/mach-o/loader.h |
| --- | --- |
|  | struct mach_header { |
|  | uint32_t	magic;		/* mach magic number identifier */ |
|  | cpu_type_t	cputype;	/* cpu specifier */ |
|  | cpu_subtype_t	cpusubtype;	/* machine specifier */ |
|  | uint32_t	filetype;	/* type of file */ |
|  | uint32_t	ncmds;		/* number of load commands */ |
|  | uint32_t	sizeofcmds;	/* the size of all the load commands */ |
|  | uint32_t	flags;		/* flags */ |
|  | }; |


```

从上面函数`getDyldMH`的名字来看，它返回`dyld`这个`Mach-O`文件的文件头，而这确实也符合变量`__dso_handle`的类型定义。


但是奇怪的事情发生了，搜遍整个`dyld`源码库，都无法找到变量`__dso_handle`的定义。所有能搜到的地方，都只是对这个变量`__dso_handle`的声明。


众所周知，动态连接器`dyld`本身是静态链接的。


也就是说，动态连接器`dyld`本身是不依赖任何其他动态库的。


因此，这个变量`__dso_handle`不可能定义在其他动态库。


既然这样，动态链接器`dyld`本身是如何静态链接通过的呢？


答案只可能是静态链接器`ld`在链接过程中做了手脚。


查看静态链接器`ld`的源码，也就是`llvm`的源码，可以找到如下代码:



```


|  | // lld/MachO/SyntheticSections.cpp |
| --- | --- |
|  | void macho::createSyntheticSymbols() { |
|  | // addHeaderSymbol 的 lamba 表达式 |
|  | auto addHeaderSymbol = [](const char *name) { |
|  | symtab->addSynthetic(name, in.header->isec, /*value=*/0, |
|  | /*isPrivateExtern=*/true, /*includeInSymtab=*/false, |
|  | /*referencedDynamically=*/false); |
|  | }; |
|  |  |
|  | ... |
|  |  |
|  | // The Itanium C++ ABI requires dylibs to pass a pointer to __cxa_atexit |
|  | // which does e.g. cleanup of static global variables. The ABI document |
|  | // says that the pointer can point to any address in one of the dylib's |
|  | // segments, but in practice ld64 seems to set it to point to the header, |
|  | // so that's what's implemented here. |
|  | addHeaderSymbol("___dso_handle"); |
|  | } |


```

上面代码定义了一个`addHeaderSymbol`的`lamda`表达式，然后使用它添加了一个符号，这个符号正是`__dso_handle`。


调用`addHeaderSymbol`上方的注释使用`chatGPT`翻译过来如下:



> Itanium C\+\+ ABI 要求动态库传递一个指向 \_\_cxa\_atexit 的指针，该函数负责例如静态全局变量的清理。ABI 文档指出，指针可以指向动态库的某个段中的任意地址，但实际上，ld64（苹果的链接器）似乎将其设置为指向头部，所以这里实现了这种做法。


注释中提到的`Itanium C++ ABI`最初是为英特尔和惠普联合开发的`Itanium`处理器架构设计的。


但其影响已经超过了最初设计的架构范围，并被广泛用于其他架构，比如`x86`和 `x86-64`上的多种编译器，包括`GCC`和`Clang`。


而且，注释中还提到，`__dso_handle`在苹果的实现里，是指向了`Mach-O`的头部。


至此，谜底解开\~。


 本博客参考[FlowerCloud机场](https://yunbeijia.com)。转载请注明出处！
