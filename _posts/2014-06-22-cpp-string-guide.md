---
layout: post
keywords: cpp,字符串
description: C++字符串完全指南 
title: C++字符串完全指南 
categories: [Cpp]
tags: [Cpp]
group: Cpp
---
{% include dmc/setup %}
C++字符串完全指南 - Win32字符编码
============================

前言
--------------------------------------------------
字符串的表现形式各异，象`TCHAR`，`std::string`，`BSTR`等等，有时还会见到怪怪的用`_tcs`起头的宏。这个指南的目的就是说明各种字符串类型及其用途，并说明如何在必要时进行类型的相互转换。

在指南的第一部分，介绍三种字符编码格式。理解编码的工作原理是致为重要的。即使你已经知道字符串是一个字符的数组这样的概念，也请阅读本文，它会让你明白各种字符串类之间的关系。

指南的第二部分，将阐述各个字符串类，什么时候使用哪种字符串类，及其相互转换。

字符串基础 - ASCII, DBCS, Unicode
--------------------------------------------------
所有的字符串类都起源于C语言的字符串，而C语言字符串则是字符的数组。首先了解一下字符类型。有三种编码方式和三种字符类型。

- 第一种编码方式是**单字节字符集**，称之为**SBCS**，它的所有字符都只有一个字节的长度。ASCII码就是SBCS。SBCS字符串由一个零字节结尾。

- 第二种编码方式是**多字节字符集**，称之为**MBCS**，它包含的字符中有单字节长的字符，也有多字节长的字符。Windows用到的MBCS只有二种字符类型，单字节字符和双字节字符。因此Windows中用得最多的字符是双字节字符集，即DBCS，通常用它来代替MBCS。
在DBCS编码中，用一些保留值来指明该字符属于双字节字符。
> 例如，Shift-JIS(通用日语)编码中，值0x81-0x9F 和 0xE0-0xFC 的意思是：“这是一个双字节字符，下一个字节是这个字符的一部分”。这样的值通常称为前导字节(lead byte)，总是大于0x7F。前导字节后面是跟随字节(trail byte)。DBCS的跟随字节可以是任何非零值。与SBCS一样，DBCS字符串也由一个零字节结尾。

- 第三种编码方式是**Unicode**。Unicode编码标准中的所有字符都是双字节长。有时也将Unicode称为**宽字符集(wide characters)**，因为它的字符比单字节字符更宽(使用更多内存)。注意，Unicode不是MBCS - 区别在于MBCS编码中的字符长度是不同的。Unicode字符串用二个零字节字符结尾(一个宽字符的零值编码)。

单字节字符集是拉丁字母，重音文字，用ASCII标准定义，用于DOS操作系统。双字节字符集用于东亚和中东语言。Unicode用于COM和Windows NT内部。
读者都很熟悉单字节字符集，它的数据类型是`char`。双字节字符集也使用`char`数据类型(双字节字符集中的许多古怪处之一)。Unicode字符集用`wchar_t`数据类型。Unicode字符串用L前缀起头，如：
{% highlight cpp script %}
wchar_t  wch = L'1';      // 2 个字节, 0x0031
wchar_t* wsz = L"Hello";  // 12 个字节, 6 个宽字符
{% endhighlight %}
字符串的存储
--------------------------------------------------
单字节字符串顺序存放各个字符，并用零字节表示字符串结尾。例如，字符串"Bob"的存储格式为：

Unicode编码中，L"Bob"的存储格式为：

用0x0000 (Unicode的零编码)结束字符串。

DBCS 看上去有点象SBCS。以后我们会看到在串处理和指针使用上是有微妙差别的。字符串"日本语" (nihongo) 的存储格式如下(用LB和TB分别表示前导字节和跟随字节)：

注意，"ni"的值不是WORD值0xFA93。值93和FA顺序组合编码为字符"ni"。(在高位优先CPU中，存放顺序正如上所述)。

字符串处理函数
--------------------------------------------------
C语言字符串处理函数，如`strcpy()`, `sprintf()`, `atol()`等只能用于单字节字符串。在标准库中有只用于Unicode字符串的函数，如`wcscpy()`, `swprintf()`, `_wtol()`。

微软在C运行库(CRT)中加入了对DBCS字符串的支持。对应于`strxxx()`函数，DBCS使用`_mbsxxx()`函数。在处理DBCS字符串(如日语，中文，或其它DBCS)时，就要用`_mbsxxx()`函数。这些函数也能用于处理SBCS字符串(因为DBCS字符串可能就只含有单字节字符)。

现在用一个示例来说明字符串处理函数的不同。
> 如有Unicode字符串L"Bob"：
x86 CPU的排列顺序是低位优先(little-endian)的，值0x0042的存储顺序为42 00。这时如用`strlen()`函数求字符串的长度就发生问题。函数找到第一个字节42，然后是00，意味着字符串结尾，于是返回1。反之，用`wcslen()`函数求"Bob"的长度更糟糕。`wcslen()`首先找到0x6F42，然后是0x0062，以后就在内存缓冲内不断地寻找00 00直至发生一般性保护错(GPF)。

`strxxx()`及其对应的`_mbsxxx()`究竟是如何运作的？二者之间的不同是非常重要的，直接影响到正确遍历DBCS字符串的方法。下面先介绍字符串遍历，然后再回来讨论`strxxx()`和 `_mbsxxx()`。

字符串处理函数
--------------------------------------------------
我们中的大多数人都是从SBCS成长过来的，都习惯于用指针的 ++ 和 -- 操作符来遍历字符串，有时也使用数组来处理字符串中的字符。这二种方法对于SBCS 和 Unicode 字符串的操作都是正确无误的，因为二者的字符都是等长的，编译器能够的正确返回我们寻求的字符位置。

但对于DBCS字符串就不能这样了。用指针访问DBCS字符串有二个原则，打破这二个原则就会造成错误。
**1. 不可使用 ++ 算子，除非每次都检查是否为前导字节。
2. 绝不可使用 -- 算子来向后遍历。**

先说明原则2，因为很容易找到一个非人为的示例。
假设，有一个配制文件，程序启动时要从安装路径读取该文件，如：C:\Program Files\MyCoolApp\config.bin。文件本身是正常的。
假设用以下代码来配制文件名：
{% highlight cpp script %}
bool GetConfigFileName ( char* pszName, size_t nBuffSize )
{
    char szConfigFilename[MAX_PATH];
    
    // 这里从注册表读取文件的安装路径，假设一切正常。
    // 如果路径末尾没有反斜线，就加上反斜线。
    // 首先，用指针指向结尾零：
    char* pLastChar = strchr ( szConfigFilename, '\0' );
    
    // 然后向后退一个字符：
    pLastChar--;  
    if ( *pLastChar != '\\' )
        strcat ( szConfigFilename, "\\" );
        
    // 加上文件名：
    strcat ( szConfigFilename, "config.bin" );
    
    // 如果字符串长度足够，返回文件名：
    if ( strlen ( szConfigFilename ) >= nBuffSize )
        return false;
    else
    {
        strcpy ( pszName, szConfigFilename );
        return true;
    }
}
{% endhighlight %}
这段代码的保护性是很强的，但用到DBCS字符串还是会出错。假如文件的安装路径用日语表达：C:\ヨウユソ，该字符串的内存表达为：

这时用上面的GetConfigFileName()函数来检查文件路径末尾是否含有反斜线就会出错，得到错误的文件名。

错在哪里？注意上面的二个十六进制值0x5C(蓝色)。前面的0x5C是字符"\"，后面则是字符值83 5C，代表字符"ソ"。可是函数把它误认为反斜线了。

正确的方法是用DBCS函数将指针指向恰当的字符位置，如下所示：
{% highlight cpp script %}
bool FixedGetConfigFileName ( char* pszName, size_t nBuffSize )
{
    char szConfigFilename[MAX_PATH];
    
    // 这里从注册表读取文件的安装路径，假设一切正常。
    // 如果路径末尾没有反斜线，就加上反斜线。
    // 首先，用指针指向结尾零：
    char* pLastChar = _mbschr ( szConfigFilename, '\0' );
    
    // 然后向后退一个双字节字符：
    pLastChar = CharPrev ( szConfigFilename, pLastChar );
    if ( *pLastChar != '\\' )
        _mbscat ( szConfigFilename, "\\" );
        
    // 加上文件名：
    _mbscat ( szConfigFilename, "config.bin" );
    
    // 如果字符串长度足够，返回文件名：
    if ( _mbslen ( szInstallDir ) >= nBuffSize )
        return false;
    else
    {
        _mbscpy ( pszName, szConfigFilename );
        return true;
    }
} 
{% endhighlight %}
这个改进的函数用`CharPrev()` API 函数将指针`pLastChar`向后移动一个字符。如果字符串末尾的字符是双字节字符，就向后移动2个字节。这时返回的结果是正确的，因为不会将字符误判为反斜线。

现在可以想像到第一原则了。例如，要遍历字符串寻找字符":"，如果不使用`CharNext()`函数而使用++算子，当跟随字节值恰好也是":"时就会出错。

与原则2相关的是数组下标的使用：

>　**2a. 绝不可在字符串数组中使用递减下标。**

出错原因与原则2相同。例如，设置指针`pLastChar`为：
{% highlight cpp script %}
char* pLastChar = &szConfigFilename [strlen(szConfigFilename) - 1];
{% endhighlight %}
结果与原则2的出错一样。下标减1就是指针向后移动一个字节，不符原则2。

###**再谈strxxx() 与_mbsxxx()**
现在可以清楚为什么要用 `_mbsxxx()` 函数了。`strxxx()` 函数不认识DBCS字符而 `_mbsxxx()`认识。如果调用`strrchr("C:\\", '\\')`函数可能会出错，但 `_mbsrchr()`认识双字节字符，所以能返回指向最后出现反斜线字符的指针位置。     

最后提一下`strxxx()` 和 `_mbsxxx()` 函数族中的字符串长度测量函数，它们都返回字符串的字节数。如果字符串含有3个双字节字符，`_mbslen()`将返回6。而Unicode的函数返回的是`wchar_ts`的数量，如`wcslen(L"Bob")` 返回3。

Win32 API中的MBCS 和 Unicode API的二个字符集
--------------------------------------------------
也许你没有注意到，Win32的API和消息中的字符串处理函数有二种，一种为MCBS字符串，另一种为Unicode字符串。例如，Win32中没有`SetWindowText()`这样的接口，而是用`SetWindowTextA()`和 `SetWindowTextW()`函数。后缀A (表示ANSI)指明是MBCS函数，后缀W(表示宽字符)指明是Unicode函数。

编写Windows程序时，可以选择用MBCS或Unicode API接口函数。用VC AppWizards向导时，如果不修改预处理器设置，缺省使用的是MBCS函数。但是在API接口中没有`SetWindowText()`函数，该如何调用呢？实际上，在`winuser.h`头文件中做了以下定义：
{% highlight cpp script %}
BOOL WINAPI SetWindowTextA ( HWND hWnd, LPCSTR lpString );
BOOL WINAPI SetWindowTextW ( HWND hWnd, LPCWSTR lpString );
#ifdef UNICODE
#define SetWindowText  SetWindowTextW
#else
#define SetWindowText  SetWindowTextA
#endif
{% endhighlight %}
编写MBCS应用时，不必定义UNICODE，预处理为：
{% highlight cpp script %}
#define SetWindowText  SetWindowTextA
{% endhighlight %}
然后将`SetWindowText()`处理为真正的API接口函数`SetWindowTextA()` (如果愿意的话，可以直接调用`SetWindowTextA()` 或`SetWindowTextW()`函数，不过很少有此需要)。

如果要将缺省应用接口改为Unicode，就到预处理设置的预处理标记中去掉 `_MBCS`标记，加入`UNICODE` 和 `_UNICODE` (二个标记都要加入，不同的头文件使用不同的标记)。不过，这时要处理普通字符串反而会遇到问题。如有代码：
{% highlight cpp script %}
HWND hwnd = GetSomeWindowHandle();
char szNewText[] = "we love Bob!";
SetWindowText ( hwnd, szNewText );
{% endhighlight %}
编译器将`SetWindowText`置换为`SetWindowTextW`后，代码变为：
{% highlight cpp script %}
HWND hwnd = GetSomeWindowHandle();
char szNewText[] = "we love Bob!";
SetWindowTextW ( hwnd, szNewText );
{% endhighlight %}
看出问题了吧，这里用一个Unicode字符串处理函数来处理单字节字符串。

**第一种解决办法是使用宏定义：**
{% highlight cpp script %}
HWND hwnd = GetSomeWindowHandle();
#ifdef UNICODE
  wchar_t szNewText[] = L"we love Bob!";
#else
  char szNewText[] = "we love Bob!";
#endif
SetWindowText ( hwnd, szNewText );
{% endhighlight %}
要对每一个字符串都做这样的宏定义显然是令人头痛的。所以用`TCHAR`来解决这个问题：
**TCHAR的救火角色**
`TCHAR` 是一种字符类型，适用于MBCS 和 Unicode二种编码。程序中也不必到处使用宏定义。
`TCHAR`的宏定义如下：
{% highlight cpp script %}
#ifdef UNICODE
  typedef wchar_t TCHAR;
#else
  typedef char TCHAR;
#endif
{% endhighlight %}
所以，`TCHAR`中在MBCS程序中是`char`类型，在Unicode中是 `wchar_t` 类型。

对于Unicode字符串，还有个 _T() 宏，用于解决 L 前缀：
{% highlight cpp script %}
#ifdef UNICODE
  #define _T(x) L##x
#else
  #define _T(x) x
#endif
{% endhighlight %}
`##` 是预处理算子，将二个变量粘贴在一起。不管什么时候都对字符串用 `_T` 宏处理，这样就可以在Unicode编码中给字符串加上L前缀，如：
{% highlight cpp script %}
TCHAR szNewText[] = _T("we love Bob!");
{% endhighlight %}
`SetWindowTextA/W` 函数族中还有其它隐藏的宏可以用来代替`strxxx()` 和 `_mbsxxx()` 字符串函数。例如，可以用 `_tcsrchr` 宏取代`strrchr()`，`_mbsrchr()`，或 `wcsrchr()`函数。`_tcsrchr` 根据编码标记为`_MBCS` 或 `UNICODE`，将右式函数做相应的扩展处理。宏定义方法类似于`SetWindowText`

不止`strxxx()`函数族中有`TCHAR`宏定义，其它一些函数中也有。例如，`_stprintf` (取代`sprintf()`和`swprintf()`)，和 `_tfopen` (取代`fopen()` 和 `_wfopen()`)。MSDN的全部宏定义在"`Generic-Text Routine Mappings`"栏目下。


**String 和 TCHAR 类型定义**
Win32 API 文件中列出的函数名都是通用名(如`SetWindowText`)，所有的字符串都按照`TCHAR`类型处理。(只有XP除外，XP只使用Unicode类型)。下面是MSDN给出的常用类型定义：
| 类型        | MBCS 编码中的意义   |  Unicode 编码中的意义  |
| :--------:   | :-----:  | :----:  |
| WCHAR | wchar_t | wchar_t|
| LPSTR | zero-terminated string of char (char*) | zero-terminated string of char (char*)|
| LPCSTR | constant zero-terminated string of char (constchar*) | constant zero-terminated string of char (constchar*)|
| LPWSTR | zero-terminated Unicode string (wchar_t*) | zero-terminated Unicode string (wchar_t*)|
| LPCWSTR | constant zero-terminated Unicode string (const wchar_t*) | constant zero-terminated Unicode string (const wchar_t*)|
| TCHAR | char | wchar_t|
| LPTSTR | zero-terminated string of TCHAR (TCHAR*) | zero-terminated string of TCHAR (TCHAR*)|
| LPCTSTR | constant zero-terminated string of TCHAR (const TCHAR*) | constant zero-terminated string of TCHAR (const TCHAR*)|


**何时使用TCHAR 和Unicode**
可能会有疑问：“为什么要用Unicode？我一直用的都是普通字符串。”

在三种情况下要用到Unicode：
> **1.程序只运行于Windows NT。**

> **2.处理的字符串长于MAX_PATH定义的字符数。**

> **3.程序用于Windows XP中的新接口，那里没有A/W版本之分。**

大部分Unicode API不可用于Windows 9x。所以如果程序要在Windows 9x上运行的话，要强制使用MBCS API (微软推出一个可运行于Windows 9x的新库，叫做Microsoft Layer for Unicode。但我没有试用过，无法说明它的好坏)。相反，NT内部全部使用Unicode编码，使用Unicode API可以加速程序运行。每当将字符串处理为MBCS API时，操作系统都会将字符串转换为Unicode并调用相应的Unicode API 函数。对于返回的字符串，操作系统要做同样的转换。尽管这些转换经过了高度优化，模块尽可能地压缩到最小，但毕竟会影响到程序的运行速度。

NT允许使用超长文件名(长于`MAX_PATH` 定义的260)，但只限于Unicode API使用。Unicode API的另外一个优点是程序能够自动处理输入的文字语言。用户可以混合输入英文，中文和日文作为文件名。不必使用其它代码来处理，都按照Unicode编码方式处理。

最后，作为Windows 9x的结局，微软似乎抛弃了MBCS API。例如，SetWindowTheme() 接口函数的二个参数只支持Unicode编码。使用Unicode编码省却了MBCS与Unicode之间的转换过程。

如果程序中还没有使用到Unicode编码，要坚持使用`TCHAR`和相应的宏。这样不但可以长期保持程序中DBCS编码的安全性，也利于将来扩展使用到Unicode编码。那时只要改变预处理中的设置即可！

各种字符串类（一）
============================
前言
--------------------------------------------------
C语言的字符串容易出错，难以管理，并且往往是黑客到处寻找的目标。于是，出现了许多字符串包装类。可惜，人们并不很清楚什么情况下该用哪个类，也不清楚如何将C语言字符串转换到包装类。

本文涉及到Win32 API，MFC，STL，WTL和Visual C++运行库中使用到的所有的字符串类型。说明各个类的用法，如何构造对象，如何进行类转换等等。Nish为本文提供了Visual C++ 7的managed string 类的用法。

阅读本文之前，应完全理解本指南第一部分中阐述的字符类型和编码。
字符串类的首要原则：
--------------------------------------------------
不要随便使用类型强制转换，除非转换的类型是明确由文档规定的。

之所以撰写字符串指南这二篇文章，是因为常有人问到如何将X类型的字符串转换到Z类型。提问者使用了强制类型转换`cast`，但不知道为什么不能转换成功。各种各样的字符串类型，特别是`BSTR`，在任何场合都不是三言二语可以讲清的。因此，我以为这些提问者是想让强制类型转换来处理一切。

除非明确规定了转换算子，不要将任何其它类型数据强制转换为`string`。一个字符串不能用强制类型转换到`string`类。例如：
{% highlight cpp script %}
void SomeFunc ( LPCWSTR widestr );
main()
{
  SomeFunc ( (LPCWSTR) "C:\\foo.txt" );  // 错！
}
{% endhighlight %}
这段代码100%错误。它可以通过编译，因为类型强制转换超越了编译器的类型检验。但是，能够通过编译，并不证明代码是正确的。

下面，我将指出什么时候用类型强制转换是合理的。
C语言字符串与类型定义：
--------------------------------------------------
如指南的第一部分所述，Windows API定义了`TCHAR`术语。它可用于MBCS或Unicode编码字符，取决于预处理设置为`_MBCS` 或 `_UNICODE`标记。关于`TCHAR`的详细说明请阅指南的第一部分。为便于叙述，下面给出字符类型定义：
| Type        | Meaning   | 
| :--------:   | :-----:  |
| WCHAR | Unicode character (**wchar_t**) |
| TCHAR | MBCS or Unicode character, depending on preprocessor settings |
| LPSTR | string of char (**char***) |
| LPCSTR | constant string of char (**const char***) |
| LPWSTR | string of WCHAR (**WCHAR***) |
| LPCWSTR | constant string of WCHAR (**const WCHAR***) |
| LPTSTR | string of TCHAR (**TCHAR***) |
| LPCTSTR | constant string of TCHAR (**const TCHAR***) |

另外还有一个字符类型`OLECHAR`。这是一种对象链接与嵌入的数据类型(比如嵌入Word文档)。这个类型通常定义为`wchar_t`。如果将预处理设置定义为`OLE2ANSI`，`OLECHAR`将被定义为`char`类型。现在已经不再定义`OLE2ANSI`(它只在MFC 3以前版本中使用)，所以我将`OLECHAR`作为`Unicode`字符处理。

下面是与`OLECHAR`相关的类型定义：
| Type        | Meaning   | 
| :--------:   | :-----:  |
| OLECHAR | Unicode character (**wchar_t**) |
| LPOLESTR | string of OLECHAR (**OLECHAR***) |
| LPCOLESTR | constant string of OLECHAR (**const OLECHAR***) |

还有以下二个宏让相同的代码能够适用于MBCS和Unicode编码：
| Type        | Meaning   | 
| :--------:   | :-----:  |
| _T(x) | Prepends L to the literal in Unicode builds. |
| OLESTR(x) | Prepends L to the literal to make it an LPCOLESTR. |

宏`_T`有几种形式，功能都相同。如： `TEXT`, `_TEXT`, `__TEXT`, 和 `__T`这四种宏的功能相同。

COM中的字符串 - BSTR 与 VARIANT
--------------------------------------------------
许多COM接口使用BSTR声明字符串。BSTR有一些缺陷，所以我在这里让它独立成章。

BSTR是Pascal类型字符串(字符串长度值显式地与数据存放在一起)和C类型字符串(字符串长度必须通过寻找到结尾零字符来计算)的混合型字符串。BSTR属于Unicode字符串，字符串中预置了字符串长度值，并且用一个零字符来结尾。下面是一个"Bob"的BSTR字符串：

注意，字符串长度值是一个DWORD类型值，给出字符串的字节长度，但不包括结尾零。在上例，"Bob"含有3个Unicode字符(不计结尾零)，6个字节长。因为明确给出了字符串长度，所以当BSTR数据在不同的处理器和计算机之间传送时，COM库能够知道应该传送的数据量。

附带说一下，BSTR可以包含任何数据块，不单是字符。它甚至可以包容内嵌零字符数据。这些不在本文讨论范围。

C++中的BSTR变量其实就是指向字符串首字符的指针。BSTR是这样定义的：
{% highlight cpp script %}
typedef OLECHAR* BSTR;
{% endhighlight %}

这个定义很糟糕，因为事实上BSTR与Unicode字符串不一样。有了这个类型定义，就越过了类型检查，可以混合使用`LPOLESTR`和`BSTR`。向一个需要`LPCOLESTR` (或 `LPCWSTR`)类型数据的函数传递BSTR数据是安全的，反之则不然。所以要清楚了解函数所需的字符串类型，并向函数传递正确类型的字符串。

要知道为什么向一个需要`BSTR`类型数据的函数传递`LPCWSTR`类型数据是不安全的，就别忘了`BSTR`必须在字符串开头的四个字节保留字符串长度值。但`LPCWSTR`字符串中没有这个值。当其它的处理过程(如`Word`)要寻找`BSTR`的长度值时就会找到一堆垃圾或堆栈中的其它数据或其它随机数据。这就导致方法失效，当长度值太大时将导致崩溃。

许多应用接口都使用`BSTR`，但都用到二个最重要的函数来构造和析构`BSTR`。就是`SysAllocString()`和`SysFreeString()`函数。`SysAllocString()`将Unicode字符串拷贝到`BSTR`，`SysFreeString()`释放`BSTR`。示例如下：
{% highlight cpp script %}
BSTR bstr = NULL;
bstr = SysAllocString ( L"Hi Bob!" );
if ( NULL == bstr )
// 内存溢出
// 这里使用bstr
SysFreeString ( bstr );
{% endhighlight %}
当然，各种BSTR包装类都会小心地管理内存。

自动接口中的另一个数据类型是`VARIANT`。它用于在无类型语言，诸如`JScript`，`VBScript`，以及`Visual Basic`，之间传递数据。`VARIANT`可以包容许多不用类型的数据，如`long`和`IDispatch*`。如果`VARIANT`包含一个字符串，这个字符串是`BSTR`类型。在下文的`VARIANT`包装类中我还会谈及更多的`VARIANT`。

各种字符串类- CRT类
============================
字符串包装类
--------------------------------------------------
我已经说明了字符串的各种类型，现在讨论包装类。对于每个包装类，我都会说明它的对象构造过程和如何转换成C类型字符串指针。应用接口的调用，或构造另一个不同类型的字符串类，大多都要用到C类型指针。本文不涉及类的其它操作，如排序和比较等。

再强调一下，在完全了解转换结果之前不要随意使用强制类型转换。

_bstr_t
--------------------------------------------------
`_bstr_t` 是`BSTR`的完全包装类。实际上，它隐含了`BSTR`。它提供多种构造函数，能够处理隐含的C类型字符串。但它本身却不提供`BSTR`的处理机制，所以不能作为`COM`方法的输出参数`[out]`。如果要用到`BSTR*` 类型数据，用`ATL`的`CComBSTR`类更为方便。

`_bstr_t` 数据可以传递给需要`BSTR`数据的函数，但必须满足以下三个条件：
> **1.`_bstr_t` 具有能够转换为`wchar_t*`类型数据的函数。**

> **2. 根据`BSTR`定义，使得`wchar_t*` 和`BSTR`对于编译器来说是相同的。**

> **3.`_bstr_t`内部保留的指向内存数据块的指针 `wchar_t*` 要遵循`BSTR`格式。**

满足这些条件，即使没有相应的`BSTR`转换文档，`_bstr_t` 也能正常工作。示例如下：
{% highlight cpp script %}
 // 构造
_bstr_t bs1 = "char string";        // 从LPCSTR构造 
_bstr_t bs2 = L"wide char string"; // 从LPCWSTR构造
_bstr_t bs3 = bs1;              // 拷贝另一个 _bstr_t
_variant_t v = "Bob";
_bstr_t bs4 = v;              // 从一个含有字符串的 _variant_t 构造

// 数据萃取
LPCSTR psz1 = bs1;              // 自动转换到MBCS字符串
LPCSTR psz2 = (LPCSTR) bs1;     // cast OK, 同上
LPCWSTR pwsz1 = bs1;            // 返回内部的Unicode字符串
LPCWSTR pwsz2 = (LPCWSTR) bs1;  // cast OK, 同上
BSTR    bstr = bs1.copy();      // 拷贝bs1, 返回BSTR

// ...
SysFreeString ( bstr );
{% endhighlight %}
注意，`_bstr_t` 也可以转换为`char*` 和 `wchar_t*`。这是个设计问题。虽然`char*` 和 `wchar_t*`不是常量指针，但不能用于修改字符串，因为可能会打破内部`BSTR`结构。

_variant_t 
--------------------------------------------------
`_variant_t` 是`VARIANT`的完全包装类。它提供多种构造函数和数据转换函数。本文仅讨论与字符串有关的操作。
{% highlight cpp script %}
// 构造
_variant_t v1 = "char string"; // 从LPCSTR 构造
_variant_t v2 = L"wide char string"; // 从LPCWSTR 构造
_bstr_t bs1 = "Bob";
_variant_t v3 = bs1; // 拷贝一个 _bstr_t 对象

// 数据萃取
_bstr_t bs2 = v1; // 从VARIANT中提取BSTR
_bstr_t bs3 = (_bstr_t) v1; // cast OK, 同上
{% endhighlight %}
注意，`_variant_t` 方法在转换失败时会抛出异常，所以要准备用catch 捕捉`_com_error`异常。

另外要注意 `_variant_t` 不能直接转换成`MBCS`字符串。要建立一个过渡的`_bstr_t` 变量，用其它提供转换Unicode到`MBCS`的类函数，或ATL转换宏来转换。

与`_bstr_t` 不同，`_variant_t` 数据可以作为参数直接传送给COM方法。`_variant_t` 继承了`VARIANT`类型，所以在需要使用`VARIANT`的地方使用`_variant_t` 是C++语言规则允许的。


STL和ATL类
============================
STL类
--------------------------------------------------
STL只有一个字符串类，即`basic_string`。`basic_string`管理一个零结尾的字符数组。字符类型由模板参数决定。通常，`basic_string`被处理为不透明对象。可以获得一个只读指针来访问缓冲区，但写操作都是由`basic_string`的成员函数进行的。

`basic_string`预定义了二个特例：`string`，含有`char`类型字符；`which`，含有`wchar_t`类型字符。没有内建的`TCHAR`特例，可用下面的代码实现：
{% highlight cpp script %}
// 特例化
typedef basic_string tstring; // TCHAR字符串
// 构造
string str = "char string"; // 从LPCSTR构造
wstring wstr = L"wide char string"; // 从LPCWSTR构造
tstring tstr = _T("TCHAR string"); // 从LPCTSTR构造
// 数据萃取
LPCSTR psz = str.c_str(); // 指向str缓冲区的只读指针
LPCWSTR pwsz = wstr.c_str(); // 指向wstr缓冲区的只读指针
LPCTSTR ptsz = tstr.c_str(); // 指向tstr缓冲区的只读指针
{% endhighlight %}
与`_bstr_t` 不同，`basic_string`不能在字符集之间进行转换。但是如果一个构造函数接受相应的字符类型，可以将由`c_str()`返回的指针传递给这个构造函数。例如：
{% highlight cpp script %}
// 从basic_string构造_bstr_t 
_bstr_t bs1 = str.c_str();  // 从LPCSTR构造 _bstr_t
_bstr_t bs2 = wstr.c_str(); // 从LPCWSTR构造 _bstr_t
{% endhighlight %}
ATL类
--------------------------------------------------
###**CComBSTR**
`CComBSTR` 是`ATL`的`BSTR`包装类。某些情况下比`_bstr_t` 更有用。最主要的是，`CComBSTR`允许操作隐含`BSTR`。就是说，传递一个`CComBSTR`对象给`COM`方法时，`CComBSTR`对象会自动管理`BSTR`内存。例如，要调用下面的接口函数：
{% highlight cpp script %}
// 简单接口
struct IStuff : public IUnknown
{
  // 略去COM程序...
  STDMETHOD(SetText)(BSTR bsText);
  STDMETHOD(GetText)(BSTR* pbsText);
};
{% endhighlight %}
`CComBSTR` 有一个`BSTR`操作方法，能将`BSTR`直接传递给`SetText()`。还有一个引用操作(`operator &`)方法，返回`BSTR*`，将`BSTR*`传递给需要它的有关函数。
{% highlight cpp script %}
CComBSTR bs1;
CComBSTR bs2 = "new text";
pStuff->GetText ( &bs1 );       // ok, 取得内部BSTR地址
pStuff->SetText ( bs2 );        // ok, 调用BSTR转换
pStuff->SetText ( (BSTR) bs2 ); // cast ok, 同上
{% endhighlight %}
###**CComBSTR**
`CComBSTR`有类似于 `_bstr_t` 的构造函数。但没有内建`MBCS`字符串的转换函数。可以调用`ATL`宏进行转换。
{% highlight cpp script %}
// 构造
CComBSTR bs1 = "char string"; // 从LPCSTR构造
CComBSTR bs2 = L"wide char string"; // 从LPCWSTR构造
CComBSTR bs3 = bs1; // 拷贝CComBSTR
CComBSTR bs4;
bs4.LoadString ( IDS_SOME_STR ); // 从字符串表加载

// 数据萃取
BSTR bstr1 = bs1; // 返回内部BSTR，但不可修改！
BSTR bstr2 = (BSTR) bs1; // cast ok, 同上
BSTR bstr3 = bs1.Copy(); // 拷贝bs1, 返回BSTR
BSTR bstr4;
bstr4 = bs1.Detach(); // bs1不再管理它的BSTR

// ...
SysFreeString ( bstr3 );
SysFreeString ( bstr4 );
{% endhighlight %}
上面的最后一个示例用到了`Detach()`方法。该方法调用后，`CComBSTR`对象就不再管理它的BSTR或其相应内存。所以`bstr4`就必须调用`SysFreeString()`。

最后讨论一下引用操作符(`operator &`)。它的超越使得有些`STL`集合(如`list`)不能直接使用`CComBSTR`。在集合上使用引用操作返回指向包容类的指针。但是在`CComBSTR`上使用引用操作，返回的是`BSTR*`，不是`CComBSTR*`。不过可以用`ATL`的`CAdapt`类来解决这个问题。例如，要建立一个`CComBSTR`的队列，可以声明为：
{% highlight cpp script %}
std::list< CAdapt> bstr_list;
{% endhighlight %}
`CAdapt` 提供集合所需的操作，是隐含于代码的。这时使用`bstr_list` 就象在操作一个`CComBSTR`队列。

###**CComVariant**
`CComVariant` 是`VARIANT`的包装类。但与 `_variant_t` 不同，它的`VARIANT`不是隐含的，可以直接操作类里的`VARIANT`成员。`CComVariant` 提供多种构造函数和多类型操作。这里只介绍与字符串有关的操作。
{% highlight cpp script %}
// 构造
CComVariant v1 = "char string";       // 从LPCSTR构造
CComVariant v2 = L"wide char string"; // 从LPCWSTR构造
CComBSTR bs1 = "BSTR bob";
CComVariant v3 = (BSTR) bs1;          // 从BSTR拷贝

// 数据萃取
CComBSTR bs2 = v1.bstrVal;            // 从VARIANT提取BSTR
{% endhighlight %}
跟`_variant_t` 不同，`CComVariant`没有不同`VARIANT`类型之间的转换操作。必须直接操作`VARIANT`成员，并确定该`VARIANT`的类型无误。调用`ChangeType()`方法可将`CComVariant`数据转换为`BSTR`。
{% highlight cpp script %}
CComVariant v4 = ... // 从某种类型初始化 v4
CComBSTR bs3;
if ( SUCCEEDED( v4.ChangeType ( VT_BSTR ) ))
    bs3 = v4.bstrVal;
{% endhighlight %}

ATL转换宏
--------------------------------------------------
`ATL`的字符串转换宏可以方便地转换不同编码的字符，用在函数中很有效。宏按照`[source type]2[new type]` 或 `[source type]2C[new type]`格式命名。后者转换为一个常量指针 (名字内含"C")。类型缩写如下：
> A：MBCS字符串，`char*` (A for ANSI) 

> W：Unicode字符串，`wchar_t*` (W for wide) 

> T ：TCHAR字符串，`TCHAR*` 

> OLE：OLECHAR字符串，`OLECHAR*` (实际等于W) 

> BSTR：BSTR (只用于目的类型)

例如，`W2A()`将`Unicode`字符串转换为`MBCS`字符串，`T2CW()`将`TCHAR`字符串转换为`Unicode`字符串常量。

要使用宏转换，程序中要包含`atlconv.h`头文件。可以在非`ATL`程序中使用宏转换，因为头文件不依赖其它的`ATL`，也不需要 `_Module`全局变量。如在函数中使用转换宏，在函数起始处先写`上USES_CONVERSION`宏。它表明某些局部变量由宏控制使用。

转换得到的结果字符串，只要不是`BSTR`，都存储在堆栈中。如果要在函数外使用这些字符串，就要将这些字符串拷贝到其它的字符串类。如果结果是`BSTR`，内存不会自动释放，因此必须将返回值分配给一个`BSTR`变量或`BSTR`的包装类，以避免内存泄露。

下面是若干宏转换示例：
{% highlight cpp script %}
// 带有字符串的函数：
void Foo ( LPCWSTR wstr );
void Bar ( BSTR bstr );
// 返回字符串的函数：
void Baz ( BSTR* pbstr );
#include 

main()
{
using std::string;
USES_CONVERSION;    // 声明局部变量由宏控制使用

// 示例1：送一个MBCS字符串到Foo()
LPCSTR psz1 = "Bob";
string str1 = "Bob";
Foo ( A2CW(psz1) );
Foo ( A2CW(str1.c_str()) );

// 示例2：将MBCS字符串和Unicode字符串送到Bar()
LPCSTR psz2 = "Bob";
LPCWSTR wsz = L"Bob";
BSTR bs1;
CComBSTR bs2;
bs1 = A2BSTR(psz2);         // 创建 BSTR
bs2.Attach ( W2BSTR(wsz) ); // 同上，分配到CComBSTR
Bar ( bs1 );
Bar ( bs2 );
SysFreeString ( bs1 );      // 释放bs1
// 不必释放bs2，由CComBSTR释放。
// 示例3：转换由Baz()返回的BSTR
BSTR bs3 = NULL;
string str2;
Baz ( &bs3 );          // Baz() 填充bs3内容
str2 = W2CA(bs3);      // 转换为MBCS字符串
SysFreeString ( bs3 ); // 释放bs3
}

{% endhighlight %}
可以看到，向一个需要某种类型参数的函数传递另一种类型的参数，用宏转换是非常方便的。


各种字符串类- MFC类
============================
MFC类 CString
--------------------------------------------------
`MFC`的`CString`含有`TCHAR`，它的实际字符类型取决于预处理标记的设置。通常，`CString`象`STL`字符串一样是不透明对象，只能用`CString`的方法来修改。`CString`比`STL`字符串更优越的是它的构造函数接受`MBCS`和`Unicode`字符串。并且可以转换为`LPCTSTR`，因此可以向接受`LPCTSTR`的函数直接传递`CString`对象，不必调用`c_str()`方法。
{% highlight cpp script %}
// 构造
CString s1 = "char string"; // 从LPCSTR构造
CString s2 = L"wide char string"; // 从LPCWSTR构造
CString s3 ( ' ', 100 ); // 预分配100字节，填充空格
CString s4 = "New window text";
// 可以在LPCTSTR处使用CString：
SetWindowText ( hwndSomeWindow, s4 );
// 或者，显式地做强制类型转换：
SetWindowText ( hwndSomeWindow, (LPCTSTR) s4 );
{% endhighlight %}
也可以从字符串表加载字符串。`CString`通过`LoadString()`来构造对象。用`Format()`方法可有选择地从字符串表读取一定格式的字符串。
{% highlight cpp script %}
// 从字符串表构造/加载
CString s5 ( (LPCTSTR) IDS_SOME_STR );  // 从字符串表加载
CString s6, s7;
// 从字符串表加载
s6.LoadString ( IDS_SOME_STR );
// 从字符串表加载打印格式的字符串
s7.Format ( IDS_SOME_FORMAT, "bob", nSomeStuff, ... );
{% endhighlight %}
第一个构造函数看上去有点怪，但它的确是文档标定的字符串加载方式。

注意，`CString`只允许一种强制类型转换，即强制转换为`LPCTSTR`。强制转换为`LPTSTR` (非常量指针)是错误的。按照老习惯，将`CString`强制转换为`LPTSTR`只能伤害自己。有时在程序中没有发现出错，那只是碰巧。转换到非常量指针的正确方法是调用`GetBuffer()`方法。

下面以往队列加入元素为例说明如何正确地使用`CString`：
{% highlight cpp script %}
CString str = _T("new text");
LVITEM item = {0};
item.mask = LVIF_TEXT;
item.iItem = 1;
item.pszText = (LPTSTR)(LPCTSTR) str; // 错！
item.pszText = str.GetBuffer(0);      // 正确
ListView_SetItem ( &item );
str.ReleaseBuffer();  // 将队列返回给str
{% endhighlight %}
`pszText`成员是`LPTSTR`，一个非常量指针，因此要用`str`的`GetBuffer()`。`GetBuffer()`的参数是`CString`分配的最小缓冲区。如果要分配一个1K的`TCHAR`，调用`GetBuffer(1024)`。参数为0，只返回指向字符串的指针。

上面示例的出错语句可以通过编译，甚至可以正常工作，如果恰好就是这个类型。但这不证明语法正确。进行非常量的强制类型转换，打破了面向对象的封装原则，并逾越了`CString`的内部操作。如果你习惯进行这样的强制类型转换，终会遇到出错，可你未必知道错在何处，因为你到处都在做这样的转换，而代码也都能运行。

知道为什么人们总在抱怨有缺陷的软件吗？不正确的代码就bug的滋生地。然道你愿意编写明知有错的代码让臭虫有机可乘？还是花些时间学习`CString`的正确用法让你的代码能够100%的正确吧。

`CString`还有二个函数能够从`CString`中得到`BSTR`，并在必要时转换成`Unicode`。那就是`AllocSysString()`和`SetSysString()`。除了`SetSysString()`使用`BSTR*`参数外，二者一样。
{% highlight cpp script %}
// 转换成BSTR
CString s5 = "Bob!";
BSTR bs1 = NULL, bs2 = NULL;
bs1 = s5.AllocSysString();
s5.SetSysString ( &bs2 );
// ...
SysFreeString ( bs1 );
SysFreeString ( bs2 );
{% endhighlight %}
`COleVariant` 与`CComVariant` 非常相似。`COleVariant` 继承于`VARIANT`，可以传递给需要`VARIANT`的函数。但又与`CComVariant` 不同，`COleVariant` 只有一个`LPCTSTR`的构造函数，不提供单独的`LPCSTR`和`LPCWSTR`的构造函数。在大多情况下，没有问题，因为总是愿意把字符串处理为`LPCTSTR`。但你必须知道这点。`COleVariant` 也有接受`CString`的构造函数。
{% highlight cpp script %}
// 构造
CString s1 = _T("tchar string");
COleVariant v1 = _T("Bob"); // 从LPCTSTR构造
COleVariant v2 = s1; // 从CString拷贝
{% endhighlight %}
对于`CComVariant`，必须直接处理`VARIANT`成员，用`ChangeType()`方法在必要时将其转换为字符串。但是，`COleVariant::ChangeType()` 在转换失败时会抛出异常，而不是返回`HRESULT`的出错码。
{% highlight cpp script %}
// 数据萃取
COleVariant v3 = ...; // 从某种类型构造v3
BSTR bs = NULL;
try
{
    v3.ChangeType ( VT_BSTR );
    bs = v3.bstrVal;
}
catch ( COleException* e )
{
    // 出错，无法转换
}
SysFreeString ( bs );
{% endhighlight %}
WTL类 CString、CLR 及 VC 7 类
--------------------------------------------------
> `WTL`的`CString`与`MFC`的`CString`的行为完全相同，参阅上面关于`MFC` `CString`的说明即可。

`System::String` 是`.NET`的字符串类。在其内部，`String`对象是一个不变的字符序列。任何操作`String`对象的`String`方法都返回一个新的`String`对象，因为原有的`String`对象要保持不变。`String`类有一个特性，当多个`String`都指向同一组字符集时，它们其实是指向同一个对象。`Managed Extensions C++` 的字符串有一个新的前缀S，用来表明是一个`managed string`字符串。
{% highlight cpp script %}
// 构造
String* ms = S"This is a nice managed string";
{% endhighlight %}
可以用`unmanaged string`字符串来构造`String`对象，但不如用`managed string`构造`String`对象有效。原因是所有相同的具有S前缀的字符串都指向同一个对象，而`unmanaged string`没有这个特点。下面的例子可以说明得更清楚些：
{% highlight cpp script %}
String* ms1 = S"this is nice";
String* ms2 = S"this is nice";
String* ms3 = L"this is nice";
Console::WriteLine ( ms1 == ms2 ); // 输出true
Console::WriteLine ( ms1 == ms3);  // 输出false
{% endhighlight %}
要与没有S前缀的字符串做比较，用`String::CompareTo()`方法来实现，如：
{% highlight cpp script %}
Console::WriteLine ( ms1->CompareTo(ms2) );
Console::WriteLine ( ms1->CompareTo(ms3) );
{% endhighlight %}
二者都输出0，说明字符串相等。

在`String`和`MFC 7`的`CString`之间转换很容易。`CString`可以转换为`LPCTSTR`，`String`有接受`char* `和 `wchar_t*` 的二种构造函数。因此可以直接把`CString`传递给`String`的构造函数：
{% highlight cpp script %}
CString s1 ( "hello world" );
String* s2 ( s1 );  // 从CString拷贝
{% endhighlight %}  
反向转换的方法也类似：
{% highlight cpp script %}
String* s1 = S"Three cats";
CString s2 ( s1 );
{% endhighlight %}
可能有点迷惑。从`VS.NET`开始，`CString`有一个接受`String`对象的构造函数，所以是正确的。
{% highlight cpp script %}
CStringT ( System::String* pString );
{% endhighlight %}
为了加速操作，有时可以用基础字符串(`underlying string`)：
{% highlight cpp script %}
String* s1 = S"Three cats";
Console::WriteLine ( s1 );
const __wchar_t __pin* pstr = PtrToStringChars(s1);
for ( int i = 0; i < wcslen(pstr); i++ )
    (*const_cast<__wchar_t*>(pstr+i))++;
Console::WriteLine ( s1 );
{% endhighlight %}
`PtrToStringChars()` 返回指向基础字符串的 `const __wchar_t*` 指针，可以防止在操作字符串时，垃圾收集器去除该字符串。

总结
============================
字符串类的打印格式函数
--------------------------------------------------
对字符串包装类使用`printf()`或其它类似功能的函数时要特别小心。包括`sprintf()`函数及其变种，以及`TRACE` 和`ATLTRACE` 宏。它们的参数都不做类型检验，一定要给它们传递C语言字符串，而不是整个`string`对象。

例如，要向`ATLTRACE()`传递一个`_bstr_t` 里的字符串，必须显式用(`LPCSTR`)或 (`LPCWSTR`)进行强制类型转换：
{% highlight cpp script %}
_bstr_t bs = L"Bob!";
ATLTRACE("The string is: %s in line %d\n", (LPCSTR) bs, nLine);
{% endhighlight %}
如果忘了用强制类型转换，直接把整个 `_bstr_t` 对象传递给`ATLTRACE`，跟踪消息将输出无意义的东西，因为`_bstr_t` 变量内的所有数据都进栈了。

所有类的总结
--------------------------------------------------
常用的字符串类之间的转换方法是：将源字符串转换为C类型字符串指针，然后将该指针传递给目标类的构造函数。下面列出将字符串转换为C类型指针的方法，以及哪些类的构造函数接受C类型指针。
| Class | string type  | convert to char*? | convert to const char*? | convert to wchar_t*? | convert to const wchar_t*? | convert to BSTR? | construct from char*? | construct from wchar_t*?|
| :--------:   | :-----:  | :-----:  | :-----:  | :-----:  | :-----:  | :-----:  | :-----:  | :-----:  |
| _bstr_t | BSTR | yes, cast | yes, cast | yes, cast | yes, cast | yes | yes | yes|
| _variant_t | BSTR | no | no | no | cast to _bstr_t | cast to _bstr_t | yes| yes|
| string | MBCS | no | yes, c_str() method | no | no | no | yes| no|
| wstring | Unicode | no | no | no | yes, c_str() method | no | no| yes|
| CComBSTR | BSTR | no | no | no | yes, cast to BSTR | yes, cast | yes| yes|
| CComVariant | BSTR | no | no | no | yes | yes | yes| yes|
| CString | TCHAR | no | in MBCS builds, cast | no | in Unicode builds | no | yes| yes|
| COleVariant | BSTR | no | no | no | yes | yes | in MBCS builds| in Unicode builds|


附注：
--------------------------------------------------
>*1.虽然 `_bstr_t` 可以转换为非常量指针，但对内部缓冲区的修改可能导致内存溢出，或在释放`BSTR`时导致内存泄露。*

> *2.`bstr_t` 的`BSTR`内含 `wchar_t*` 变量，所以可将`const wchar_t*` 转换到`BSTR`。但这个用法将来可能会改变，使用时要小心。*

> *3.如果转换到`BSTR`失败，将抛出异常。*

> *4.用`ChangeType()`处理`VARIANT`的`bstrVal`。在`MFC`转换失败将抛出异常。*

> *5.虽然没有`BSTR`的转换函数，但`AllocSysString()`可返回一个新的`BSTR`。*

> *6.用`GetBuffer()`方法可临时得到一个非常量`TCHAR`指针。*


参考
============================
>[1]*[C++字符串完全指南 - Win32字符编码](http://www.blogjava.net/neumqp/archive/2006/03/09/34504.html)*
