

#summary Quick ref to non C extensions of Eppic as well as APIs.

# Introduction #

Eppic is a C interpreter that permits easy access to the symbol and type information stored in a executable image like a coredump or live memory interfaces (e.g. /dev/kmem, /dev/mem). Although it has a strong association with live or postmortem kernel analysis, it is not constraint to it and can be embedded in any tools that is C friendly.

It support a FULL C syntax and the same variable and function scope and type. Symbols and type information in the image become standard variables and types during the execution of Eppic code. Addition variables and types can be defined in Eppic code to match missing types information (e.g. stripped kernel modules).

> This README focuses on the differences between eppic and a C compiler so, for the complete C syntax , please refer to a C reference manual.

Also explained here, are the mechanisms of the API that allow eppic to be inserted into any debugging tool that deals with objects and can feed back symbol and type information to eppic through the API.

# Preprocessor commands #

All preprocessor commands I knew of are supported.  #define, #undef, #ifdef, #if, #ifndef, #else, #elif, #endif and #include.

These are ignored:  #ident #pragma

Eppic has a builtin secondary parser for preprocessor expression evaluation.

# Target symbols (target variables) #

The symbols from the target image and their associated value are available from within eppic.

Because the type of the symbol may sometime not be known (for example, when the debug/dwarf information is missing), Eppic enables access to symbols in two ways. The first, is a more natural mode in that it makes symbols available as fully typed entities. The second, is generic and treats all symbols as an address to data which then needs to be cast to the proper type explicitly.  The former obviously enables an easier cut&pasting of target code into Eppic code.

> Note: It is up to the API function `int getval(char *, ull *, VALUE_S *);` to determine if symbols are access in a fully typed manner or in a generic manner.

Below are 2 examples of how accessing a simple (int) value nproc from an image might look like.

## Fully typed symbol access ##

In this mode (which will soon be the default mode for redHat's crash tool), the symbols are access in a natural way.

For example, accessing ` int nproc; ` would look like typical target C code:
```
    void showprocs()
    {
    int i;
        for(i=0;i<nproc;i++) {
            do something...
        }
    }
```

And accessing the ` struct proc *procs; ` would also look like typical target C code:
```
    void showprocs()
    {
        struct proc* p;
        for(p=procs; p; p=p->p_next)
            do something...
        }
    }
```
## Generic symbol access ##
In this mode Eppic effectively adds a level of reference to the symbol and makes it typed as a `(void*)` reference.  So, if the symbol is an (int), one would have to access it in this way (possibly assigning it to a local (int) variable, but not necessarily):
```
    void
    showprocs()
    {
        int i,np;
        np=*(int*)nproc;
        for(i=0;i<np;i++) {
            do something...
        }
    }
```
Accessing a pointer as type `(struct proc*)` in the target would look like:
```
    void showprocs()
    {
        struct proc* p;
        for(p=*(struct proc**)procs; p; p=p->p_next)
            do something...
        }
    }
```

Notice the  level of dereference to that procs in the above example.

# Variables #

All C variables types are supported.

## Variable initialization ##

Normal C variable initialization is supported for non complex types. Structures and unions initialization is not supported at this time. Structures or union instances are not supported in general at this point.

Variables can be initialized at compile time if their scope is _global_ or if their storage type is _static_. Any values that can be resolved at the time of the compile, like preprocessor macros or sizeof(), or integer values, can be used for these compile time initialization.

Automatic variables can be initialized using the set of values valid for static variables as well as any values or variables active in the run-time context of that initialization.

For more complex initialization needs, the function init() will be executed, if it is defined within a file that has just been compiled.

Using an uinitialized variable will generate a run time error.

## Variable types ##

All types made available through the API can be used. These are the types already defined in the executable image.

Floating point types are not supported. There are no plans to support them at this time.

Declaration of arrays is not supported. To access a array from the image, use a pointer.

Unions and structures type declarations are supported and can be used to create additional types that become available within the same macro file.

Typedefs are supported.

Function pointers are not supported (use 'string' type instead,	see "Operators" below)

Eppic defines a 'string' types to handle ansi C strings within the interpreter. This string type also support some of the operator (+, ==, =, !=, etc... see below)

## Scope ##

All symbols available in the system image become global variable in the eppic context.

Variable declared within eppic can be given one of 3 different scopes like in normal C.

### Global Scope ###
A variable that is declared outside a function that is not marked as static. this variable if available to all compile units.

Eppic specific note:

> You do not have to declare extern variables.
> Variable lookups are preformed at runtime with variables declared in
> Eppic files being search first and with the target variables being
> search second.

Ex:
> file1:
```
 int global; 
 int func()
 { 
 }
```
> file2:
```
 func2()
 {
    int i=global;
 }
```

Since eppic currently validates variable existence only at run time there is no need to declare a 'global' as an 'extern' variable. At run time, if none of the currently compiled files  **AND** none of the target image variables define a global variable _global_, then `i=global` will fail with a _'unknown variable'_ error.

### File scope ###

A static variable that is declared outside any functions.
Same as normal C file scope (or compile unit scope).

The init() function can be used to initialize status dynamic arrays.

Ex:

file1:
```
    static int array;

    __init() 
    {
        int i;
        for(i=0;i<10;i++) array[i]=-1;
    }

    void func1(int i)
    {
	if(array[i] != -1) ...
    }
    int getmaxproc() { return maxproc; }
```
> Notes:
  * init() is used to initialized the static 'array'.
  * Array is a dynamic array. See below.
  * Both func1() and getmaxproc() use 'maxproc'.
  * getmaxproc() makes the local static variable 'maxproc' available to other, external functions.

### Other scopes ###

**Function**, **statement block** and **function parameters** are all supported normal C scopes.

## Storage classes ##

In the context of eppic the register or volatile storage classes do not make any sense and are ignored by the interpreter.

Like in true C, a variable declared within a function will be automatic (stack) by default. The 'static' keyword can be used to make a variable persistent. Outside of any function, all variables are considered persistent and the 'static' keyword can be used to change the variable scope from 'global' to 'file' (see above).

# Operators #

## Normal operators for simple types ##

> All standard C operators are supported for interger types (int, short, char etc...).

## Special _string_ type operators ##
### Basic operators ###
Eppic support the following operators for the 'string' type.
```
        +, !=, <, >, <=, >=
```
Examples:
```
        s = "hello" + "World";

        if("hello" == "world" ) { ... }
```

### Using _string_ variable as function pointer ###
Functions pointers are not supported for now. Ans a cheap substitute has been used to supply such important functionality in Eppic code. The value of a a _string_ variable can be used as the function name in a function call statement.

When eppic is about to perform a call, it will look at the type of the variable used to name the function. If the type is 'string' it will use the value that string and call that function instead.

Example:
```
        func0(int i, int j)
        {
                printf("i=%d j=%d\n", i,j);
        }
        func1(string func)
        {
                func(1,2);
        }
        main()
        {
                func1("func0");
        }
```

In the above example, func1() ends up calling func0() not func.  This can be used as a callback mechanism, specially useful for creating generating function that call s linked list of objects and calls a variable function for each object. Like a function that walks tasks or procs.

## Operator IN for associative arrays ##

The 'in' operator is supported on dynamic arrays.
```
        if(i in array) { ... }
        str=i in array ? "yes" : "no";
```

## Operator sizeof() ##

The sizeof() operator is evaluated at execution time or compile time depending on the use of it.

When used as part of a static or global variable initialization statement, it will be evaluated at compile time. When used in any other context, it will be evaluated at run time.

If the sizeof() target is not a type, but a executable statement instead, the expression **will** be executed, so be careful. Eppic does not have a "type only" statement execution mode which would enable evaluation of the type yielded by statement execution without actual execution of that statement.

# Statements #


All C statements except 'goto' are supported.
The 'for(i in array)' is supported on dynamic arrays.

# Associative arrays #

When indexing through a variable that is anything put a pointer. you end up creating a associative array. Associative arrays can be useful in stashing away various lists of typed items. Associative arrays can have many levels, i.e. each element of an array can be associated it own array as well.

Associative arrays can be scanned using the _in_ operator combined with the _for_ statement.

Example:
```
        int func()
        {
        char *cp, c;
        int array, i;

                cp=(char *)symbol;
                c=cp[10];

                array[0]="one string";
                array[12]="second string";

                for(i in array) { 

                        printf("array[%d]=%s\n", i, array[i]);
                }
                
        }
```
In the 'c=cp[10](10.md)' statement, eppic goes to the system image to get one 'char' at the address symbol+10. This is thus  the case of a simple pointer indexing, and does not create an associative array.

In the second case, 'array' is not a pointer, it's a 'int'. So eppic threats all indexing through it as dynamic.

Additionally, eppic supports multi levels of dynamic index, which makes possible to create random trees of indexed values.

Example:
```
        int func()
        {
        int array, i, j;

                array[10]="array10";
                array[10][3]="array10,3";
                array[20]="array20";
                array[20][99]="array20,99";

                for(i in array) {

                        printf("array[%d]=%s\n", i, array[i]);

                        for(j in array[i]) {

                              printf("array[%d][%d]=%s\n", i, j, array[i][j]);

                        }

                }
        }
```

This feature of Eppic is valuable in the context of system image access and analysis, since they require frequent lists search. For example, with dynamic arrays, one can walk the proc list taking note of the proc`*`, then walking a user thread list taking note of the thread`*` and gathering some metrics for each of these threads. In order to get to these metrics at some later point in the execution, something like this could be used:

Example:
```
        func()
        {
        proc *p;

                for(p in procs) {

                        thread *t;

                        for(t in procs[p]) {

                                int rss, size;

                                /* we use index 0 for rss and 1 for size */
                                printf("proc %p, thread %p, rss:size = %d:%d\n"
                                      , p, t, procs[p][t][0], procs[p][t][1]);
                        }
                }
        }
```

Arrays are always passed by reference. On creation the reference count is set to one. So this array will exist untill the variable it's assigned to dies.

Arrays can be created by dso's. See the DSO section below for more information and examples of this.


# Tool integration API #

This is the API though which Eppic is integrated into some target tool. Versus the runtime API which is the API used during compile and execution of Eppic code within that same tool (defined further below).

Eppic can be integrated into any tool that needs to access symbol and type information from some object. Currently it is integrated in lcrash and icrash (tools that access Linux and Irix kernel images respectively), but it should be possible to use it, for example, in dbx or gdb. The API gives a simple interface through which the host application sends symbol and type (including member) information and gives access to the image itself so that eppic can read random blocks of data from the image.

## ` eppic_builtin(bt *bt) ` ##

Install some set of builtin function. See below (builtin api).


## ` eppic_chkfname(char *fname, int silent) ` ##

Will check for the exsistance of a function in eppic. Typically used to check xtra entry points before the application registers a new command (see eppic\_setcallback).

## eppic\_open() ##

The first function that should be called is eppic\_open(). eppic\_open() will return a value of 1 if everything is ok or 0 in case of some problem. This call initializes internal date for the eppic package.

## ` eppic_setapi(apiops* ops, int nbytes) ` ##

This function will setup the callbacks that eppic will use to get information from the application during compile and execution of Eppic code.

> See 'runtime API' below.

## ` eppic_load(char *name) ` ##

To have eppic load and compile a macro or a set of macro use eppic\_load().  Parameter name gives the name of the file to compile. If name points to a directory instead, then all the files in this directory will be load.  So an application would call eppic\_load() when it first
starts up specifying some well known files or directories to load. For example $HOME/.xxx and /etc/xxx would be loaded, ~/.xxx containing user defined macros, and /etc/xxx containing system macros.

## ` eppic_unload(char *funcname) ` ##

To unload the a macro file use this function. "funcname" is the name of any global function in the file you want to unload.

## ` void eppic_setcallback(void (*scb)(char *))` ##

To be called prior to any load calls. After each loads, eppic will call this function back with the name of each functions compiled. Typicly, the application will then perform checks and potencially install a new command for this function.

Example:
```
	void
	reg_callback(char *name)
	{
	char fname[MAX_SYMNAMELEN+sizeof("_usage")+1];
	_command_t cmds[2];

        	snprintf(fname, sizeof(fname), "%s_help", name);
        	if(!eppic_chkfname(fname, 0)) return;
        	snprintf(fname, sizeof(fname), "%s_usage", name);
        	if(!eppic_chkfname(fname, 0)) return;

        	cmds[0].cmd=strdup(name);
        	cmds[0].real_cmd=0;
        	cmds[0].cmdfunc=run_callback;
        	cmds[0].cmdparse=parse_callback;
        	cmds[0].cmdusage=usage_callback;
        	cmds[0].cmdhelp=help_callback;
        	cmds[1].cmd=0;
        	unregister_cmd(cmds[0].cmd);
        	(void)register_cmds(cmds);
	}
```
## `` eppic_setipath(char `*`path) `` ##

When eppic processes a #include directive it will use the specified path as a search path. The usual PATH format is supported ex: "/etc/include:/usr/include".

## `` eppic_setmpath(char `*`path) `` ##

When eppic\_load() is called with a relative path name or just the name of a file, it will use a search PATH to locate it. The path parameter to eppic\_set,path() sets this path. The usual PATH format is supported ex: "/etc/xxx:/usr/lib/xxx".

## ` eppic_setofile(FILE *ofile) ` ##

All output of eppic commands will be send to file ofile.

## ` eppic_cmd(char *cmd, char **argv, int nargs) ` ##

This is the way to execute a eppic command that as been loaded.  'cmd' is the name of function to call.  'argv' are the argument to this function.  'nargs' is the number of argument in array 'argv'.

Eppic\_cmd() will process argv and make the corresponding values available to the function by creating global variables that the function can test and use.

## ` eppic_showallhelp() ` ##

This command will send a complete list of the commands long with the usage and help for each one of them. This function should be called when the user request something like 'help all'.

## ` eppic_showhelp(char *func) ` ##

This will display the help information for a particular function loaded in eppic.

# Rutime API #

This is the API used by the Eppic compiler and execution engine.

Everytime Eppic needs some piece of information, it will call the application back for it. THe eppic\_apiset() function is used to install this callback interface into eppic. Here is the list of callback functions:

Eppic\_apiset() passes this structure to eppic:

```
        typedef unsigned long long ull;
	typedef struct {

        	int (*getmem)(ull, void *, int);
        	int (*putmem)(ull, void *, int);
        	int (*member)(char *, ull, type * , member *);
        	int (*getctype)(int ctype, char * , type*);
        	char* (*getrtype)(ull, type *);
        	int (*alignment)(ull);
        	int (*getval)(char *, ull *);
        	enum_t* (*getenum)(char *name);
        	def_t*  (*getdefs)();
        	uint8_t (*get_uint8)(void*);
        	uint16_t (*get_uint16)(void*);
        	uint32_t (*get_uint32)(void*);
        	uint64_t (*get_uint64)(void*);
	} apiops;
```

The apiops`*` struct defines the following member and function pointers:

## ` getmem(ull addr, void *buffer, int nbytes) ` ##

Read nbytes from image at virtual address addr (32 or 64 bit) to buffer.

## ` putmem(ull addr, void *buffer, int nbytes) ` ##

Write nbytes from buffer to image at virtual address addr (32 or 64 bit).

## ` member(char *name, ull pidx, type *tm, member *m) ` ##


Get information on a structure member called name. Pidx is a unique type index for the parent structure. The getctype() function should fill in this index in it's type`*`. The dwarf model uses unique indexes (die offsets) that can be used here.  'tm' will hold information on the type of the member.  'm' will hold information on the member specific stuff (bit sizes, bit offset etc.).

> Note:
> Use the eppic\_member_...() functions to setup m.
> Use the eppic\_type_...() functions to setup t.

## ` getctype(int ctype, char *name, type *tout) ` ##

Get type information for a complex type.  Ctype specifies that name is a type of type struct/union or enum.  tout contain the returned type information.

## ` getrtype(ull idx, type *t) ` ##

Gets the type string linked to a typedef. For example, the gettdeftype(<idx of type ull>) would return "unsigned long long". This enables eppic to drill down a typedef (a typedef can be build from a typedef itself) in order to perform proper type validation for assignment or function parameters or return values.

## ` getval(char *sname, ull *value) ` ##

Returns the value of symbol "sname" from the image. The value is returned in 'value'.  On any image this is address of the symbol within the image itself, not the value of the symbol itself. See explanation of this above.

## ` getenum(char *name); ` ##

Return a list of enum values. Eppic will make these available as symbol for the duration of the compile.

## ` getdefs() ` ##

Return a list of #defines to be active througout the eppic session.

## ` get_uint8/16/32/64() ` ##

Return converted unsigned integers. As parameters are passed pointers to unsigned int values in dump representation. The return values are the corresponding unsigned int values in the representation of the 	host architecture, where eppic is running.

# The builtin C API #
## Intro ##
Sometime it is necessary to create a C function that will handle some piece of the work, that a macro cannot do. Eppic's builtin function are implemented this way. Generic function like 'printf' or 'getstr' can get some parameter input from the macros and do something (printf) or they get some information, map it to a eppic value and return it to a macro (getstr).


Eppic can load new functiosn from DSOs. If the extension of a file name is ".so" then eppic opens it and gets a list of function specs from it. Unload of that file will uninstall these functions.

The API between the dso and eppic is quite simple at this time. It has not been exercised as must as it would need to, so it might get more flexible and thus complex in the future.

## Example : Hello World function ##

This is an example of a simple extension. An equivalent os the "hello world" C program, but this one gets 2 parameters, one int and one string and returns the received int.

Examples:

```
	#include "eppic_api.h"

	value *
	helloworld(value *vi, value *vs)
	{
	int i=eppic_getval(vi);
	char *s=(char*)eppic_getval(vs);

		eppic_msg("Hello to the world![%d] s=[%s]\n", i, s);
		return eppic_makebtype(1);
	}

	BT_SPEC_TABLE = {
		{ "int hello(int i, string s)",	helloworld},
		{ 0, 0}
	};

	static char *buf;

	BT_INIDSO_FUNC()
	{
		eppic_msg("Hello world being initialized\n");
		buf=eppic_alloc(1000);
		return 1;
	}

	BT_ENDDSO_FUNC()
	{
		eppic_msg("Hello world being shutdown\n");
		eppic_free(buf);
	}
```
The BT\_SPEC\_TABLE is scanned. It's a simple table with 2 entries per functions and terminated with a NULL prototype.

The DSO initializer function is called. If it returns 0 then installtion is terminates. If it returns 1 we proceed forward.

The prototype is compiled and a syntax error will send the error message to the application output file (stdout usually).

When the prototype as compiled with no errors the function is installed and ready to be used from eppic code.

Type checking is performed by eppic at execution time on both, the function parameters and the function return.

## Example: associative arrays ##

DSO's can also receive, create and manipulate dynamic arrays.
Here is an example of this:

```
	#include "eppic_api.h"

	#ifdef ARRAY_STATIC
	static value *v;
	#endif

	value *
	mkarray(value* vi)
	{
	int i=eppic_getval(vi);
	#ifndef ARRAY_STATIC
	value *v=eppic_makebtype(0);
	#endif

		eppic_msg("Received value [%d]\n", i);
		/* build an array indexed w/ int w/ 2 string values */
		eppic_addvalarray(v, eppic_makebtype(0)
			, eppic_makestr("Value of index 0"));
		eppic_addvalarray(v, eppic_makebtype(2)
			, eppic_makestr("Value of index 2"));
	#if ARRAY_STATIC
		/* 
	   	For a static array use :
		Then the array will persist until you Free it.
		*/
	   	eppic_refarray(v, 1);
	#endif
		return v;
	}

	value *
	showstrarray(value* va)
	{
	value *v1=eppic_strindex(va, "foo");
	value *v2=eppic_strindex(va, "goo");

		printf("array[1]=%d\n", eppic_getval(v1));
		printf("array[2]=%d\n", eppic_getval(v2));
		eppic_addvalarray(va, eppic_makestr("gaa"),                  eppic_makebtype(3));
		eppic_addvalarray(va, eppic_makestr("doo"), eppic_makebtype(4));
		eppic_freeval(v1);
		eppic_freeval(v2);
		return eppic_makebtype(0);
	}

	value *
	showintarray(value* va)
	{
	value *v1=eppic_intindex(va, 1);
	value *v2=eppic_intindex(va, 2);

		printf("array[1]=%d\n", eppic_getval(v1));
		printf("array[2]=%d\n", eppic_getval(v2));
		eppic_freeval(v1);
		eppic_freeval(v2);
		return eppic_makebtype(0);
	}

	BT_SPEC_TABLE = {
		{ "int mkarray(int i)",	mkarray},
		{ "void showintarray(int i)",showintarray},
		{ "void showstrarray(int i)",showstrarray},
		{ 0, 0}
	};

	static char *buf;

	BT_INIDSO_FUNC()
	{
		eppic_msg("mkarray initialized\n");
	#ifdef ARRAY_STATIC
		/* we will need a static value to attach the
		   array too */
		v=eppic_makebtype(0);
	#endif
		return 1;
	}

	BT_ENDDSO_FUNC()
	{
		eppic_msg("mkarray being shutdown\n");
	#ifdef ARRAY_STATIC
		eppic_freeval(v);
		/* freing the value decrements the reference
		   count by one. So, if none of the calling
		   macros copied the value to a static
		   eppic variable, it will free the array */
	#endif
	}

```

# Builtin functions #

Here is a description of the builtin functions that ship with Eppic.
```
	unsigned long long 
	atoi(string value [, int base])
```

Convert a string value to a long long. Base is the base
that should be used to process the string e.g. 8, 10 or
16. If not specified, then the standard numeric format
will be scnanned for ex:
```
			0x[0-9a-fA-F]+ : hexadecimal
			0[0-7]+        : octal
			[1-9]+[0-9]*   : decimal
```
This function is used when converting command line
arguments to pointers.

> Example:
```
		void
		mycommand()
		{
		int i;
		
			for(i=1;i<argc;i++) {
			
				struct proc *p;
				
				p=(struct proc*)atoi(argv[i],16);
				
				...
			}
		}
		
```

## ` int exists(string name) ` ##

Checks for the existance of a variable. Returns 1 if
the variables does exist and 0 otherwise. This function
is mostly used to test if some options were specified
on when the macro was executed from command line.

It can also be used to test for image variable.

> Example:

```
		void
		mycommand()
		{
			if(exists("vmemlist")) {

				// this version of the target
                                // does use vmemlist variable

			}
		}
```

## `	void exit() ` ##

Terminate macro excution now.

## `	int getchar() ` ##

Get a single character from tty.

## `	string gets() ` ##

Get a line of input from tty.

## `	string getstr(void *) ` ##

Gets a null terminated string from the image at the address specified. Eppic will read a series of 16 byte values from the image untill it find the \0 character. Up to 4000 bytes will be read this way.

## `	string getnstr(void *, int n) ` ##

Gets n characters from the image at the specified address and returns the corresponding string.

## `	string itoa(unsigned long long) ` ##

Convert a unsigned long long to a decimal string.

## `	void printf(char *fmt, ...); ` ##

Send a formatted message to the screen or output file. For proper allignment of output on 32 and 64 bit systems one can use the %> sequence along with the %p format.

On a 32 bit system %p will print a 8 character hexadecimal value and on a 64 bit system it will print a 16 character value. So, to get proper alignment on both type of systems use the %> format which will print nothing on a 64 bit system but will print 8 times the following character on a 32 bit system.

> Example:
```
		struct proc *p;

		  printf("Proc	%>	uid 	pid\n");
		  printf("0x%p		%8d	%8d\n"
			, p, p->p_uid,p_p_pid);
```
### `	int eppic_depend(string file) ` ###

Loads a macro or directory of macros called 'file'. Contrary to eppic\_load() it will not give any error messages. Returns 1 on success 0 otherwise.

## `	int eppic_load(string file) ` ##

Loads and compiles a eppic macro file. returns 1 if successful or 0 otherwise.

## `	void eppic_unload(string file) ` ##

Unload's a eppic macro file

## `	string sprintf(string format, ...) ` ##

Creates a string from the result of a sprintf.

> Example:
```
	void
		mycommand()
		{
		
		string msg;
		
			msg=sprintf("i=%d\n", i);
		}
```
The result will be truncated to maxbytes if it would be longer.

## `	int strlen(string s) ` ##

Return the length of string s.

## `	string substr(string s, int start, int len) ` ##

Creates a string from the substring starting a charcater
'start' of 's' for 'len' characters.

> Example:
```
	s=substr("this is the original", 6, 2);
```
So 's' will become "is".