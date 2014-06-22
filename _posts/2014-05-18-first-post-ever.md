---
layout: post
keywords: 工作
description: 关于我
title: 开篇
categories: [life]
tags: [about me]
group: archive
icon: coffee
---

留白，有空再写

| Left-Aligned  | Center Aligned  | Right Aligned |
| :------------ |:---------------:| -----:|
| col 3 is      | some wordy text | $1600 |
| col 2 is      | centered        |   $12 |
| zebra stripes | are neat        |    $1 |


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
