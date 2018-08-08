* 原文：[Adding your own Natives to warcraft III (c/c++)](http://www.wc3c.net/showthread.php?t=84417)
* 原作者：[PipeDream](http://www.wc3c.net/member.php?u=742043)
* 摘取时间：2018-8-9 01:40:13 (UTC+8)

# 添加你自己的Native函数到《魔兽争霸3》中

In this tutorial I'm going to show how to use xttoc's jAPI to write your own warcraft natives. You'll need to know C/C++ and have a compiler whose calling convention you know (IE, can get lucky with). I'll be using borland's bcc 5.60. Unfortunately I couldn't get anything free to work.

Theoretically with this technique we'll be able to do anything at all, although it may take quite a bit of research. Even then there'll be a serious limitation in that maps won't be bnet distributable. I'd like to specifically warn against someone writing a general library that maps could call, as this would be begging for security problems.

In any case, let's get started. One thing that would be particularly nice to have is C routines for data structures. We'll implement two: One, a heap- just dynamically allocated arrays. Then we'll do something a little fancier, a disjoint set for mazes.
First off grab jAPI Tool from this thread: [http://www.wc3campaigns.net/showthread.php?t=79652](http://www.wc3c.net/showthread.php?t=79652). Stuff all the files into your warcraft3 directory except for the jNatives folder-keep the folder but toss its contents. You'll probably want shortcuts for both the loaders, in particular, add the -window flag to the LoaderWar3.exe for testing, as in target: "D:\games\Warcraft III\LoaderWar3.exe" -window

The outline of what needs to happen is:
- Write a native
- Load native into warcraft
- Get custom declaration of native into common.j
- Write code that uses native
- Run game

We'll start with just a hello world. Here's our test.cpp:

```cpp
/*
The following routines are all exported by japi.dll:

char* MemStrSearch(unsigned char *start, unsigned char *end, char *str);
bool MemPatternCompare(unsigned char *address, unsigned char *pattern, unsigned char *mask, unsigned long length);
unsigned char* MemPatternSearch(unsigned char *start, unsigned char *end, unsigned char *pattern, unsigned char *mask, unsigned long length);

PIMAGE_SECTION_HEADER GetImageSectionHeaders(HMODULE hModule, WORD *count);
PIMAGE_SECTION_HEADER GetImageSectionHeader(HMODULE hModule, unsigned char name[8]);

void	jAPI jBindNative(void *routine, char *name, char *prototype);
void	jAddNative(void *routine, char *name, char *prototype);
jString jAPI jStrMap(char *str);
char*	jAPI jStrGet(jString strid);
*/
// - Andy Scott aka xttocs


#include <windows.h>
#include <stdio.h>

#define jNATIVE	__stdcall
#define jAPI	__msfastcall

#define FloatAsInt(f) (*(long*)&f)
#define IntAsFloat(i) (*(float*)&i)

typedef long jString;
typedef long jInt;
typedef long jReal;

typedef void	(jAPI *jpAddNative)(void *routine, char *name, char *prototype);
typedef jString (jAPI *jpStrMap)	(char *str);
typedef char *	(jAPI *jpStrGet)	(jString strid);

jpAddNative		jAddNative;
jpStrMap		jStrMap;
jpStrGet		jStrGet;

#pragma warning ( disable : 4996 )


jString jNATIVE test(jString js, char *fnname)
{
	char *s;
	s = jStrGet(js);
	s[0] = 'A';
	return jStrMap(s);
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD  ul_reason_for_call, LPVOID lpReserved)
{
	if (ul_reason_for_call == DLL_PROCESS_ATTACH)
	{
		DisableThreadLibraryCalls(hModule);

		HMODULE hjApi = GetModuleHandle("japi.dll");

		jAddNative	= (jpAddNative)GetProcAddress(hjApi, "jAddNative");
		jStrMap		= (jpStrMap)*GetProcAddress(hjApi, "jStrMap");
		jStrGet		= (jpStrGet)*GetProcAddress(hjApi, "jStrGet");

		jAddNative(test, "test", "(S)S");
	}
    return TRUE;
}
```

The two important bits here are:

```cpp
jString jNATIVE test(jString js, char *fnname)
{
	char *s;
	s = jStrGet(js);
	s[0] = 'A';
	return jStrMap(s);
}
```

and

```cpp
jAddNative(test, "test", "(S)S");
```

If stuff crashes and you're not using borland's compiler, try using __fastcall instead of __msfastcall.

Our test function takes a string and returns a string. It simply substitutes the first character in the string for the letter 'A', hoping that it wasn't passed a string of zero length. The engine also passes the name of the called function to the native so we include that fnname to make sure the stack gets cleaned up when we're done.

That third argument of jAddNative gives the type information. In parentheses comes the arguments. For example, (ISI) would mean takes integer x, string s, integer y. After the parenthesis comes the return type, in our case, S for string.

We need to compile a DLL. In a command prompt:

```cpp
bcc32 -WD -e"test.xjp" test.cpp
```

-WD means build a dll, -e is the output file and test.cpp is of course the input. The file extension is .xjp instead of .dll because this is what jAPI recognizes and links into warcraft. Now stuff the .xjp file into where jAPI will look, the jNatives folder in your warcraft directory (which should be otherwise empty).

Next we need to add the native to common.j. If you don't have a copy, extract it with your favorite mpq archiver from war3patch.mpq.

```jass
native test takes string s returns string
```

Now start LoaderWorldEditor.exe from the jAPI archive. If nothing at all happens, you may have registry issues. Try reinstalling warcraft. If it crashes, then something is probably wrong with the calling convention you set the functions to. Post the error message and the dll you generated and I can take a look. Now that the editor is running, create a new map and do whatever description fiddling you like. Import your modified common.j and change its base directory from war3mapImported\ to Scripts\ so that it overwrites the default blizzard copy. Save. Usually it'll fail the first time, before common.j imports properly. If after a couple saves you continue to get registered native declaration problems, check that the types you set in common.j match the types you set in the jAddNative call in test.cpp.

If everything is still going OK, write some JASS to try out your new test function. Try running call BJDebugMsg(test("Hello")) a few seconds into the game. Save your map. If the native doesn't seem to be declared, check that your common.j has the prototype line. If it saves ok, don't bother with the test map button. Instead run the jAPI LoaderWar3.exe, preferably through a shortcut with the -window flag. Head over to your map and start it up. If, upon clicking the map in the map selection screen, no player slots appear, you're probably missing some natives in your .xjp file. All the natives you added in common.j must have an entry in your custom dll.

If the game runs and displays "Aello", congratulations, you've written your first custom native. Now let's do something more interesting. One common lament of JASS programmers is the lack of dynamically allocated memory. We can't even pass arrays! In C, however, we can easily malloc() a new chunk of memory. This is a little dangerous, since it won't be cleaned up when the map finishes. We'll have to do it manually for now, but this is good practice anyway. For simplicity, and since we have the return bug for flexibility, we'll just write arrays of integers. So we're going to want four functions:

```jass
native ArrayAlloc takes integer length returns integer
native ArrayFree takes integer arr returns nothing
native ArrayGet takes integer arr, integer index returns integer
native ArraySet takes integer arr, integer index, integer val returns nothing
```

Go ahead and add them to common.j

Next copy test.cpp to array.cpp and replace the test native with some array natives:

```cpp
jInt jNATIVE ArrayAlloc(jInt len, char *fnname)
{
	return (jInt)malloc(sizeof(jInt)*len);
}

jInt jNATIVE ArraySet(jInt array,jInt index,jInt val, char *fnname)
{
	int *p = (int *)array;
	p[index] = val;
	return p[index];
}

jInt jNATIVE ArrayGet(jInt array, jInt index, char *fnname) {
	int *p = (int *) array;
	return p[index];
}

jInt jNATIVE ArrayFree(jInt array, char *fnname) {
	int *p = (int *)array;
	free(p);
	return 0;
}
```

I've had all of them return something since JASS seems to expect some sort of return value from every function, even if it's told that it returns nothing.

To register them, we'll need these lines in DllMain:

```cpp
jAddNative(ArrayAlloc,"ArrayAlloc","(I)I");
jAddNative(ArraySet,"ArraySet","(III)I");
jAddNative(ArrayGet,"ArrayGet","(II)I");
jAddNative(ArrayFree,"ArrayFree","(I)I");
```

Compile this with bcc32 -WD -e"array.xjp" array.cpp

Toss out your test.xjp and replace it in the jNatives folder with array.xjp.

Try some simple test code, perhaps:

```jass
local integer heap = ArrayAlloc(16)
call ArraySet(heap,5,42)
call BJDebugMsg(I2S(ArrayGet(heap,5)))
call ArrayFree(heap)
```

If you have problems, go back and check the troubleshooting tips listed for 'test'.

Exercise: Add bounds checking and dynamic resizing to your array.

Here's another native, this one's for blu and his maze generation map, a [disjoint set](http://en.wikipedia.org/wiki/Disjoint-set_data_structure):

```cpp
typedef struct {
	int parent;
	int rank;
} node;

jInt jNATIVE DJSNew(jInt len, char *fnname)
{
	node *set = (node *)malloc(sizeof(node)*len);
	int i;
	for(i=0;i<len;i++) {
		set[i].parent = -1;
		set[i].rank = 0;
	}
	return (int)set;
}

jInt jNATIVE DJSFind(jInt iset, jInt x, char *fnname) {
	node *set = (node *)iset;
	if (set[x].parent == -1) return x;
	set[x].parent = DJSFind(iset,set[x].parent);
	return set[x].parent;
}

jInt jNATIVE DJSUnion(jInt iset,jInt x,jInt y, char *fnname)
{
	node *set = (node *)iset;
	int xr = DJSFind(iset,x);
	int yr = DJSFind(iset,y);
	if(set[xr].rank > set[yr].rank)
		set[yr].parent = xr;
	else if(set[xr].rank < set[yr].rank)
		set[xr].parent = yr;
	else {
		set[yr].parent = xr;
		set[xr].rank++;
	}
	return 0;
}

jInt jNATIVE DJSFree(jInt iset, char *fnname) {
	free((node *)iset);
	return 0;
}
```

Or some lists for weaaddar:

```cpp
typedef struct {
	int car;
	int cdr;
} pair;

pair *allocpair() {
	return (pair *)malloc(sizeof(pair));
}

void freepair(pair *p) {
	free(p);
}

jInt jNATIVE cons(jInt x, jInt y, char *fnname) {
	pair *newpair = allocpair();
	newpair->car = x;
	newpair->cdr = y;
	return (jInt)newpair;
}

jInt jNATIVE car(jInt ipair, char *fnname) {
	pair *p = (pair *)ipair;
	return p->car;
}
jInt jNATIVE cdr(jInt ipair, char *fnname) {
	pair *p = (pair *)ipair;
	return p->cdr;
}
```

Prototype letters:

```
Integer		I
Real		R
String		S
Code		C
Boolean		B
Returns nothing	V
Handles		H; (?)
Handle ext.	H followed by ext followed by semicolon e.g. Hplayer;
```

Big thanks to xttocs for doing all the hard work and thanks for reading. If you manage to get this working with different compilers or how to work with handles or anything really, please post what/how! Good luck.

--------

## Known good compiler list

* Borland 5.60
* [Borland 5.5](http://www.borland.com/products/downloads/download_cbuilder.html) (free)

--------

Important update: Each native needs an additional "char *fnname" argument at the end. Warcraft passes the name of the called native in so we need to make sure the stack is cleaned up. Thanks xttocs.

If you have problems with the stock loader, try the one in loader.rar