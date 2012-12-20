---
layout: tutorial
title: COM interface tutorial
categories: ['tutorials', 'AutoHotkey', 'AutoHotkey_L', 'IronAHK', 'COM']
permalink: '/tutorials/COM-Interfaces-m.html'

ahk_versions: [['AutoHotkey', false], ['AutoHotkey_L', true], ['AutoHotkey v2', true]]
required_os: 'None'
skills: 'medium / high'

css_files: ['tutorial', 'syntax']
---

## Introduction
COM is a very powerful way to automate things on your computer. This tutorial will show you a way to use the very powerful COM technology even in situations whre it seems impossible at first sight.

You may know the [COM.ahk library by Sean](http://www.autohotkey.com/community/viewtopic.php?t=22923) or have used COM to automate IE, Word, Excel etc. But this is only *a very small subset* of COM.

If you look around [msdn](http://msdn.com), you will find plenty COM ***interfaces***. If you know [object oriented programming](http://en.wikipedia.org/wiki/Object-oriented_programming), you will know that interfaces define the methods and properties a class must have. Regarding COM, especially when it comes to system management, Windows often also includes implementations of these interfaces.

Some of them, like IE or Office products, can be used easily from within AutoHotkey - either through COM.ahk or, as of AutoHotkey_L, natively. In this tutorial we will look into how to use the others - which is a little nasty.

## Native COM in AutoHotkey_L and AutoHotkey v2
With AutoHotkey_L's native COM functions, you can do quite a lot: You can create class instances with `ComObjCreate()`, you can call these instance's methods, and you can interact with them in other ways. However, you're limited to a subset of classes: classes that implement the `IDispatch` interface. This interface allows the caller, in this case the AutoHotkey interpreter, to get a method by name instead of the method calls being defined at compile-time (as, for example, in C++). A lot of classes don't do this, so AutoHotkey cannot call their methods like this.

But that doesn't mean an AutoHotkey version with native COM support is useless here:
since AutoHotkey_L v1.0.96.00, `ComObjCreate()` does not only accept a ***ProgID*** like `"InternetExplorer.Application"`, but also a ***CLSID*** and an ***IID*** to create an instance. Users with AutoHotkey classic must use the COM Standard Library to do this.
**So let's start now!**

## Investigating on the interface and class
To use a COM class that does not implement `IDispatch`, we first need some information on the class and the interface we want to work with. We need:

* the ***IID*** (***I***nterface ***Id***entifier) of the interface
This is a [GUID](http://en.wikipedia.org/wiki/Globally_unique_identifier) (a ***g***lobally ***u***nique ***id***entifier, stored as 128-bit integer) that identifies the interface.
* the ***CLSID*** (***Cl***as***s*** ***Id***entifier) of the class implementing that interface
Similar to the IID, this GUID uniquely identifies the class. These IDs are used internally by COM instead of the class / interface name, so that theoretically, there could be several classes or interfaces with the same name - as long as the IDs differ.
* the interface(s) our interface inherits from
All COM interfaces inherit `IUnknown`. Some interfaces derive directly from it. But inheritance can also go several layers deep.
* a list of the interface's methods in vTable order
This special order will be covered later in the tutorial. It is absolutely necessary that it fits exactly, otherwise the method calls go wrong.

For this tutorial, we will use the `ITaskbarList` interface to manipulate the Windows taskbar. It is implemented by the system's `TaskbarList` class.

As first step, we search the interface on msdn. This leads us to [this site](http://msdn.microsoft.com/en-us/library/windows/desktop/bb774652). Unfortunately, neither IID nor CLSID are mentioned there (look out, for other interfaces they are mentioned!). Also **pay attention:** the method order on msdn is almost always NOT the order we require. But at least it tells us that `ITaskbarList` directly inherits from `IUnknown`.

For the missing information, we can check out several other internet resources: the [Win32 programmer reference](http://winapi.freetechsecrets.com/win32/index.htm) or the [OLE programmer reference](http://winapi.freetechsecrets.com/ole/) sometimes tell us the method order. And of course, google is your friend. :)

## Using the Windows header file
However, if you can't find what you're looking for, the [Microsoft Windows SDK](http://www.microsoft.com/en-us/download/details.aspx?id=3138) has it all. If you have it installed, perfect! Otherwise, consider it: it contains hundreds of definitions necessary for advanced Windows programming in many programming languages (although primarily for C and C++). Now, if the SDK is available, we'll have a look in a ***header file***. These are files used by the C and C++ languages, and they contain definitions of constants as well as of our interface. On the msdn page, you could see the header file (in the table on the bottom). In our case, it's *Shobjidl.h*.
We open this file and search for `ITaskbarList` until we find something like below:

```cpp
EXTERN_C const IID IID_ITaskbarList;

#if defined(__cplusplus) && !defined(CINTERFACE)

MIDL_INTERFACE("56FDF342-FD6D-11d0-958A-006097C9A090")
ITaskbarList : public IUnknown
```

The string in the 5th line is the IID of the interface we're looking for. For use in AutoHotkey, we put curly braces around it: `IID := "{56FDF342-FD6D-11d0-958A-006097C9A090}"`. That was easy!

Finding the CLSID is often more complicated - but not in our example. We just search for something like below:

```cpp
class DECLSPEC_UUID("56FDF344-FD6D-11d0-958A-006097C9A090")
TaskbarList;
```

The string in the first line is what we were looking for. And again, we put braces around it: `CLSID := "{56FDF344-FD6D-11d0-958A-006097C9A090}"`. Sometimes you won't find this declaration and it will be quite complicated to find the CLSID. Try using google, that often helps.

Now we go back to the interface declaration. A few lines underneath we find something like this:

```cpp
typedef struct ITaskbarListVtbl
{
	BEGIN_INTERFACE
	
	HRESULT ( STDMETHODCALLTYPE *QueryInterface )( 
		__RPC__in ITaskbarList * This,
		/* [in] */ __RPC__in REFIID riid,
		/* [annotation][iid_is][out] */ 
		__RPC__deref_out  void **ppvObject);
	
	ULONG ( STDMETHODCALLTYPE *AddRef )( 
		__RPC__in ITaskbarList * This);
	
	ULONG ( STDMETHODCALLTYPE *Release )( 
		__RPC__in ITaskbarList * This);
	
	HRESULT ( STDMETHODCALLTYPE *HrInit )( 
		__RPC__in ITaskbarList * This);
	
	HRESULT ( STDMETHODCALLTYPE *AddTab )( 
		__RPC__in ITaskbarList * This,
		/* [in] */ __RPC__in HWND hwnd);
	
	HRESULT ( STDMETHODCALLTYPE *DeleteTab )( 
		__RPC__in ITaskbarList * This,
		/* [in] */ __RPC__in HWND hwnd);
	
	HRESULT ( STDMETHODCALLTYPE *ActivateTab )( 
		__RPC__in ITaskbarList * This,
		/* [in] */ __RPC__in HWND hwnd);
	
	HRESULT ( STDMETHODCALLTYPE *SetActiveAlt )( 
		__RPC__in ITaskbarList * This,
		/* [in] */ __RPC__in HWND hwnd);
	
	END_INTERFACE
} ITaskbarListVtbl;
```

Now this is the method list in vTable-order we were searching for. It includes all methods inherited from other interfaces, in this case `IUnknown`'s `QueryInterface()`, `AddRef()` and `Release()`.

## Creating an instance
Now we have all the information to start coding! As with all classes and interfaces, we first create an instance. We do so by using the Windows `CoCreateObject()` API, but instead of calling it directly, we use the AutoHotkey_L builtin wrapper `ComObjCreate()`:

```ahk
ptr := ComObjCreate(CLSID, IID)
```

Users of AutoHotkey classic and COM.ahk use this:

```ahk
ptr := COM_CreateObject(CLSID, IID)
```

This is one way to create the instance. But there are cases where this is not possible (for example the class and therefore the CLSID is unknown) or useless (objects representing special data we don't have). In these cases, we need some alternate API (mostly `DllCall()`) to create or retrieve an instance.

## Pointers and the vTable
Now we need to analyse what we got from this operation: we **did not get an AHK object** that we can use like any other AHK_L object. If that was the case, there would be no point in this tutorial :) Instead, we got a ***pointer***: an integer pointing to some place in memory. You would get the equivalent running the code below:

```ahk
ptr := ComObjUnwrap(ComObjCreate("Scripting.Dictionary")) ; requires AHK_L or v2. AHK classic users will ALWAYS get a pointer using COM_CreateObject()
```

This code creates an AHK-usable object and directly passes it to `ComObjUnwrap()` - which returns the pointer the object represents.

This pointer points to the object in memory. And in the first place in this object, at memory location `ptr + 0`, there's another pointer we need:

```ahk
vtbl := NumGet(ptr + 0, 0, "Ptr")
```

This code retrieves the mentioned pointer (adding 0, so that AHK doesn't try to read the memory of the `ptr` variable itself, which would result in garbage). This new pointer now holds the location of a memory table - a table holding more pointers! This is the so-called *vTable*. And the pointers inside the table are pointers to methods. Probably you know the AutoHotkey function `RegisterCallback()`: it returns a pointer to a function. This pointer can be used to call the function. And such function pointers are in the vTable - for every interface method. We can use these to call the methods!

## Investigating on a method
But at the moment, we only have a pointer to the beginning of the table - not to the methods themselves. To get such, we need the index of the method in the table. We have looked up the method listing in the vTable-order previously, so we just have to look it up now. Things to consider:

* The index is **zero-based**. So the first method has index 0, the second has index 1 etc.
* Inherited methods are included in the table. The list we can obtain from the header file also includes those, so no problem. But if you got the list from somewhere else, ensure you don't forget to put the inherited methods at the top.

Additionally to the index, we need information on what parameters and return value the method has, what type they have and what meaning. For windows, msdn is the best resource here.

## Getting a pointer to a method
Now think of a real table, printed on paper, lying in front of you. If you want to get to the second row (zero-based index 1), you start from the top and go down once the height of one row. For the third row (index 2), you go down twice the height of one row and so on.

The same thing is what we're doing now. The row height in the vTable depends on your system: 32bit systems have 4 bytes, 64bit systems have 8 bytes. But we don't have to care: AutoHotkey_L has the builtin variable `A_PtrSize` that holds exactly this value. And AutoHotkey classic users can just use 4, as there's no 64bit version.

To get the pointer to the `ITaskbarList::HrInit()` method, we will use the following:
```ahk
hrInit := NumGet(vtbl + 0, 3 * A_PtrSize, "Ptr")
```
Remember: `vtbl` was the top of the table, 3 is the zero-based index of `HrInit()`. We read the pointer value at that position using `NumGet()`.

## Calling a method
Finally! We have a pointer to a method! Now let's call it!
But wait: you can't do `hrInit()` now - it's only a pointer, an integer!

Instead, we use `DllCall()`. Yeah, you read correctly. `DllCall`. Although we don't directly call a function in a DLL now. Check out the manual.
[It wrote](http://l.autohotkey.net/docs/commands/DllCall.htm)

> `[DllFile\]Function` - In v1.0.46.08+, this parameter may also consist solely of an an integer, which is interpreted as the address of the function to call.

So now we call it:

```ahk
hr := DllCall(hrInit, "Ptr", ptr) ; AutoHotkey classic users: use "UInt" instead of "Ptr"
```

`HrInit()` has no parameters. But why are we passing the `ptr` parameter then? Remember: `ptr` is the pointer to the instance. We must pass this to the function so that it can work with it. Because most likely it contains data related to the instance - which can then be modified. We must always pass this as first parameter.

Now we have created an instance of `ITaskbarList` and we initialized it. Now we're gonna do some magic:

```ahk
addTab := NumGet(vtbl + 0, 4 * A_PtrSize, "Ptr")
deleteTab := NumGet(vtbl + 0, 5 * A_PtrSize, "Ptr")

WinGet, windows, List ; get a list of all windows
Loop %windows%
{
	DllCall(deleteTab, "Ptr", ptr, "Ptr", windows%A_Index%) ; remove all entries from the taskbar
}
sleep 3000
Loop %windows%
{
	DllCall(addTab, "Ptr", ptr, "Ptr", windows%A_Index%) ; re-add all entries to the taskbar
}
```

A last thing to mention is the return value. Most COM methods return a `HRESULT` error code, where `S_OK := 0x00` means success. But it should always be documented.

## The entire code
```ahk
IID := "{56FDF342-FD6D-11d0-958A-006097C9A090}", CLSID := "{56FDF344-FD6D-11d0-958A-006097C9A090}"
, S_OK := 0x00

ptr := ComObjCreate(CLSID, IID)
vtbl := NumGet(ptr + 0, 0, "Ptr")

hrInit := NumGet(vtbl + 0, 3 * A_PtrSize, "Ptr")
addTab := NumGet(vtbl + 0, 4 * A_PtrSize, "Ptr")
deleteTab := NumGet(vtbl + 0, 5 * A_PtrSize, "Ptr")

if (DllCall(hrInit, "Ptr", ptr) != S_OK)
{
	MsgBox Error!
	ExitApp
}

WinGet, windows, List ; get a list of all windows
Loop %windows%
{
	DllCall(deleteTab, "Ptr", ptr, "Ptr", windows%A_Index%) ; remove all entries from the taskbar
}
sleep 3000
Loop %windows%
{
	DllCall(addTab, "Ptr", ptr, "Ptr", windows%A_Index%) ; re-add all entries to the taskbar
}
```

## More...
COM wrappers like this fit perfectly to the AutoHotkey_L / AutoHotkey v2 class syntax. Check out the [COM Classes Framework](http://www.autohotkey.com/forum/viewtopic.php?t=71201), an approach to collect standardized and convenient COM wrappers.

Some COM APIs which do not implement `IDispatch` can be called as if they would. This requires a COM Type Library to be available (*.tlb files, can be created from *.idl files). There's an (not yet complete) library available for this: [ImportTypeLib](http://www.autohotkey.com/community/viewtopic.php?t=83708).