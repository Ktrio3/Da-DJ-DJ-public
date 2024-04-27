# DISMantle

- [DISMantle](#dismantle)
  - [Debugging with DISMantle](#debugging-with-dismantle)
    - [Better Errors in 2024](#better-errors-in-2024)
    - [A Simple Example -- Finding a Null Dereference](#a-simple-example----finding-a-null-dereference)
  - [DISM Improvements](#dism-improvements)
    - [AST Printing](#ast-printing)
    - [Instruction Array](#instruction-array)
  - [Building DISMantle](#building-dismantle)
    - [Step 1: BGT or BLT](#step-1-bgt-or-blt)
    - [Step 2: Add the DJ Parser](#step-2-add-the-dj-parser)
    - [Step 3: Compilation](#step-3-compilation)
    - [Step 4: Run it](#step-4-run-it)
    - [Important changes](#important-changes)

DISM, but with extra breakage.

DISMantle is a modified version of the sim-dism code that adds a break instruction (brk) and debugging tools similar to those found in gdb. Some features include:

- Faster execution of DISM programs
- A brk instruction to set break points in the DISM source file
- Adding or removing breakpoints by line number or label
- Stepping through the execution of a program
- Loading a DJ file's symbol table
- Printing memory sections (with corresponding symbol info)
- Disabling or enabling the overwhelmingly verbose step-by-step debug information from normal sim-dism

The DISM code is based on the DISM code provided in 2023. I don't believe this code changes besides switching from bgt or blt, but just an FYI.

## Debugging with DISMantle

DISMantle provides commands very similar to those found in GDB (however, unlike GDB, the commands are always abbreviated).  The commands are:

- The continue command, "c", continues running the program until the next breakpoint is hit.  
- The step command, "s", runs the next instruction, and then breaks. 
- The breakpoint command, "b", takes a label or instruction number, and sets a breakpoint
at that instruction.
  - If no instruction number is provided, the list of set breakpoints is printed (note no ordering is done).
- The delete command, "d", takes a label or instruction number and deletes that breakpoint.  
  - Providing an asterisk deletes all breakpoints.
- The file command, "f", takes a filename (up to 100 chars in length) and parses that file as a 
DJ program. The symbol table is then available during debugging.
- The verbose command, "v", toggles the very verbose, step=by-step, output typical of sim-dism.
- The info command, "i", prints information about the current debugging session.
  - Context c
  - Memory m
  - Heap h
  - Stack s
- The reset command, "r", clears memory and resets the program, ready to run again
- The write command, "w", changes a memory location
  - The first arg is type: r for register, m for memory, and p for PC
  - The second arg is location (or value for PC): register 0-8 or memory 0-65535
  - The third arg is the value: any valid DISM nat value (an unsigned int)
- The quit command, "q", exits :(

Hitting enter (no whitespace, just a single newline char) will rerun the last command.

Note that placing a breakpoint on or stepping into a brk instruction will cause the code to break twice. A brk instruction also cannot be removed.

### Better Errors in 2024

DISMantle now has better error handling! It was extremely annoying to load the symbol table,
place breakpoints, and then lose all that work when the program crashed. Now all errors are 
caught and redirected to the debug console. Ready to run it again? Use the new "reset" command
to clear memory and set PC back to 0.

Any comments that follow an instruction are now displayed when printing the context. For example,
a common practice is to write the DISM as "mov 0 0 ; START METHOD HERE". That comment will
now be printed with the context, as well as the proper line number (not just the instruction number). Now "tagging" instructions in the DJ compiler with the expr type that produced it is
easy to quickly find. 

This also means you can debug in crash context. You can use the info command at the crash
point to debug memory as it was when it crashed. 

You can also use the write command to modify memory. Very niche, but technically you can now
recover any program (although any fixes must be repeated as you cannot add/edit/remove instructions). I'm like that astronaut using Lisp to save Deep Space 1 (except my REPL is terrible ¯\\_(ツ)_/¯ )

### A Simple Example -- Finding a Null Dereference

Take the following DJ code for a BST:

```txt
class Tree extends Object{
    static nat dummyStatic; //Just to show off static on heap
    Tree left;
    Tree right;
    nat data;

    nat add(nat a){
        nat b; nat c; dummyStatic = 1000; b = 100; c = 200; //Just to show off locals on stack
        if(data == 0){
            data=a;
        }
        else{
            if(data > a){
                addLeft(a);
            }else{
                addRight(a);
            };
        };   
    }
    nat addLeft(nat a){
        if(left == null){
            left = new Tree();
        }else{null;};
        left.add(a);
    }
    nat addRight(nat a){
        if(right == null){
            left = new Tree();  //ERROR: making new left, not right
        }else{null;};
        right.add(a);
    }
    nat printTree(){
        if(!(left == null)){
            left.printTree();
        }else{0;};
        printNat(data);
        if(!(right == null)){
            right.printTree();
        }else{0;};
    }
}

main{
    nat i;
    Tree t;
    t = new Tree();

    while(!((i = readNat())==0)){
        t.add(i);
    };
    t.printTree();
}
```

Running this code as-is will result in a null-dereference. For example, if we are going to insert
`10 2 4 15 1`, we get the following:

```shell
./DISMantle ./example.dism
Enter a natural number: 10
Enter a natural number: 2
Enter a natural number: 4
Simulation completed with code 88 at PC=753.
```

Let's pretend we don't know where the error is, and try to find the cause using DISMantle. Let's start
by placing a breakpoint at the instruction that halted.

```txt
./DISMantle ./example.dism a
DISMantle(0): b 753
DISMantle(0): b
Breakpoints: 
        753
DISMantle(0): c
Enter a natural number: 10
Enter a natural number: 2
Enter a natural number: 4
DISMantle(753): i c
******Next instruction at location 753 (in ?): 
0:HLT_AST
1:  INT_AST(1)
Register contents *BEFORE* executing this instruction:
  0:0 1:88 2:9 3:65535 4:2 5:14 6:65503 7:65511 PC:753
```

Printing the current context, we can see "(in ?)", which reminds us that we did not load the symbol table. Let's do that now.

```txt
DISMantle(753): f example.dj
Parsed in the following symbols: 
Main var i on 44 with type -1
Main var t on 45 with type 1

Class Object on -1 extends (-4)
Class Tree on 1 extends (Object)
         Static var dummyStatic on 2 with type -1
          var left on 3 with type 1
          var right on 4 with type 1
          var data on 5 with type -1
         Method add on 7 returns -1
                PARAMS
                  var a on 7 with type -1
                LOCALS
                  var b on 8 with type -1
                  var c on 8 with type -1
         Method addLeft on 20 returns -1
                PARAMS
                  var a on 20 with type -1
                LOCALS
         Method addRight on 26 returns -1
                PARAMS
                  var a on 26 with type -1
                LOCALS
         Method printTree on 32 returns -1
                PARAMS
                LOCALS

DISMantle(753): i c
******Next instruction at location 753 (in addRight of Tree called by 9): 
0:HLT_AST
1:  INT_AST(1)
Register contents *BEFORE* executing this instruction:
  0:0 1:88 2:9 3:65535 4:2 5:14 6:65503 7:65511 PC:753
```

With the symbol table loaded, we can now see we are in method addRight of class Tree, called
by the Tree object stored at address 9.

So now lets print the heap.

```text
DISMantle(753): i h
M[13] = 1 (start of Tree)
M[12] = 0 (left)
M[11] = 0 (right)
M[10] = 0 (data)
M[9] = 1 (start of Tree)
M[8] = 13 (left)
M[7] = 0 (right)
M[6] = 2 (data)
M[5] = 1 (start of Tree)
M[4] = 9 (left)
M[3] = 0 (right)
M[2] = 10 (data)
M[1] = 1000 (dummyStatic)
```

We can now see our data stored, with the 10 on the bottom of the heap, 2 in the middle,
and an empty one on the very top. From the context information, we know the method was called by
the Tree on address 9, which contains our 2. This makes sense, as inserting (10, 2, 4) will place
the 4 as the right child of the node containing 2. However, the Tree on address 9 has a
0 for the right child, and address 13 (our empty Tree on the top) for the left child. We have now determined 
the addRight method added a left child instead of a right child, and we can return to the DJ code to make the fix.

We've already solved, the issue, but let's show off the stack real quick just to show off
all the features.

```text
DISMantle(753): i s
M[65506] = 65519 (oldFP)
M[65507] = 4 (param a, type -1)
M[65508] = 2 (staticMethod addRight)
M[65509] = 1 (staticClass Tree)
M[65510] = 9 (dynCaller address)
M[65511] = 472 (RA to #57ret1c0m)
----Start new frame---
M[65512] = 65525 (oldFP)
M[65513] = 200 (local c, type -1)
M[65514] = 100 (local b, type -1)
M[65515] = 4 (param a, type -1)
M[65516] = 0 (staticMethod add)
M[65517] = 1 (staticClass Tree)
M[65518] = 9 (dynCaller address)
M[65519] = 631 (RA to #77ret1c1m)
----Start new frame---
M[65520] = 65533 (oldFP)
M[65521] = 4 (param a, type -1)
M[65522] = 1 (staticMethod addLeft)
M[65523] = 1 (staticClass Tree)
M[65524] = 5 (dynCaller address)
M[65525] = 410 (RA to #48ret1c0m)
----Start new frame---
M[65526] = 65535 (oldFP)
M[65527] = 200 (local c, type -1)
M[65528] = 100 (local b, type -1)
M[65529] = 4 (param a, type -1)
M[65530] = 0 (staticMethod add)
M[65531] = 1 (staticClass Tree)
M[65532] = 5 (dynCaller address)
M[65533] = 150 (RA to #12ret0c0m)
----Start new frame---
M[65534] = 5 (main local t, type 1)
M[65535] = 4 (main local i, type -1)
```

After fixing that error, we can now see the tree code works correctly.

```text
./DISMantle ./example.dism  
Enter a natural number: 10
Enter a natural number: 2
Enter a natural number: 4
Enter a natural number: 15
Enter a natural number: 1
Enter a natural number: 0
1
2
4
10
15
Simulation completed with code 0 at PC=207.
```

## DISM Improvements

The following are some general improvements made to sim-dism.

### AST Printing

By default, sim-dism prints out the AST tree when debug is enabled.  However, this is completely useless when you want to debug
the user program and not the simulator itself, so DISMantle only prints the AST if it is requested by providing anything as a 3rd argument.

### Instruction Array

The original version of sim-dism interpreted the instructions by looping over
a large linked list. For large DISM programs, such as those output by a DJ compiler,
this is very innefficient.  For example, if instruction 100 is `jmp 0 #LABEL`, sim-dism first
loops over the entire list to find instruction 100, then loops over the entire list again to find
the `#LABEL` at instruction 200, and then loops over it a third time to actually run instruction 200.

Here is the output of running the following DJ program (good23.dj, but prints i as well):

```txt
//takes a while to execute, but should work if the
//generated DISM code manages memory correctly
class C extends Object { 
  nat n;
  nat foo(nat unused) {this.n = 8;}
  nat bar(nat unused) {this.foo(7);}
}
main {
  nat i;
  C c;
  c=new C();
  i=0;
  while(!(i>66000)) {
    c.bar(6);
    i=i+1;
  };
  printNat(c.n);
  printNat(i);
}
```

```bash
time ./sim-dism good23.dism
8
66001
Simulation completed with code 0 at PC=207.
./sim-dism good23.dism  21.74s user 0.00s system 99% cpu 21.742 total
```

```bash
time ./DISMantle good23.dism 
8
66001
Simulation completed with code 0 at PC=207.
./DISMantle good23.dism  4.16s user 0.01s system 99% cpu 4.166 total
```

It's been a while since I took computer architecture and used Amdahl's Law, but if I used this
[calculator](https://www.omnicalculator.com/other/amdahls-law) correctly, that's 80.86% faster than the original with a speedup of 5.226.

By switching to an array, the instructions can now be accessed in O(1) time. Labels are also
stored in a symbol table, with the symbol table index being stored with the instruction after the first
encounter. For example, in the following code, the `#LOOP` symbol table index is stored with the `jmp`
instruction, meaning it only loops over the symbol table once.

```txt
mov 1 1
mov 3 10
#LOOP: add 2 2 1
beq 2 3 #END
jmp 0 #LOOP
#END: hlt 0
```

The switch to an array now means a set of pointers to the instruction structs must be located sequentially, so this could mean absolutely enormous DISM files now run out of space that might have fit using the scattered Linked List. This seems highly unlikely given this just needs to be contiguous in virtual memory, not physical memory, and DISM does not free much memory to create holes in the first place, but it is a theoretical downside worth mentioning.

The new instruction data structures are also much smaller. A single labelled add instruction, for example, used to need 4 list structs and 5 node structs (total of 4\*16+28\*5=204 bytes on a 64-byte machine with 8-byte pointers and 4-byte ints). Now, an instruction uses only a single instruction struct and 3 (possibly null) operand structs (max of 36+16\*3=84 bytes, min of 36+16\*0=36 bytes). The new ADD structure is thus 58.82% smaller than the original.

Note the original AST structure is still used for now during parsing and converted to an array
afterwards as the structure is well-suited to Bison. During conversion, the AST memory is freed.

Since DISM programs are just a single instruction list, a simple solution for working with Bison's
parsing might be to store the instructions in the array as they are parsed, and then reversing the
array to get the correct order (due to LLR parsing). I have not put much thought into it beyond
this, though.

## Building DISMantle

### Step 1: BGT or BLT

To build DISMantle, you should first decide if DISMantle should have a branch greater than (bgt) or breanch less than (blt) instruction. Sim-dism only
allows one or the other, and DISMantle does the same.  Open the
dism.l file under src and change the following to get the behavior you want.

```C
//To use BGT (BLT will throw an error):
#define BGT_ENABLED 1

//To disable BGT and use BLT (BGT will throw an error):
#define BGT_ENABLED 0
```

### Step 2: Add the DJ Parser

The build.py script can be used to automatically perform the following steps using the Da-DJ-DJ files in the parent directory, with
the resulting binary being located under the `bin` directory.

In order to build the symbol table when debugging a file compiled from DJ, DISMantle needs to parse the original DJ file.  First, get the dj2dism source code by running Da DJ DJ. Next, copy
the dj.l, dj.y, ast.c, ast.h, printST.c, symtbl.c, and symtbl.h files into the DISMantle directory.  The dj.y file needs to be altered slightly, so first open that file and change the includes in the "%code provides" section to the following:

```C
#include "lex.yy.c"
#include "ast.h"
#include "symtbl.h"
#include "break.h"
```

Next, we need to get rid of the main function in dj.y.  Replace the entire main function with the following:

```C
void parseDJFile(char *fileName) {

    yyin = fopen(fileName,"r");
    if(yyin==NULL) {
        printf("ERROR: could not open file %s\n",fileName);
        return;
    }
    /* parse the input program */
    yyparse();

    setupSymbolTables(pgmAST);
}
```

This new function calls the usual DJ functions to create the symbol table. Note that the program will not
halt if an non-existing file name is entered, but it **WILL** exit if the file exists, but is not a syntactically valid DJ program.  Note the program is not typechecked, as DISMantle assumes it's already been checked.
This isn't guranteed, of course, so if you want to typecheck, copy over the DJ typecheck files, add the include statements, add `typecheckProgram()` after `setupSymbolTables`, and include typecheck.c in the gcc command in the following step.

Note: For more information on using two parsers at once with Bison, see the [Bison Manual](https://www.gnu.org/software/bison/manual/html_node/Multiple-Parsers.html).

### Step 3: Compilation

Now we just need to compile everything.

```bash
flex dism.l; 
flex dj.l; 
bison dism.y;
bison dj.y;
sed -i '/extern YYSTYPE yylval/d' dj.tab.c;
sed -i '/extern DISMSTYPE dismlval/d' dism.tab.c;
gcc ast.c break.c dism.tab.c dism-ast.c dj.tab.c interp.c symtbl.c printST.c -oDISMantle;
```

### Step 4: Run it

```bash
./DISMantle FILENAME.dism [DEBUG] [PRINTAST]
```

### Important changes

*IMPORTANT:* This code slightly changes the way readNat works. Code
has been added to consume all characters until the new line char. In original DISM, you can technically chain numbers on a single line.

```txt
rdn 1
rdn 2
rdn 3
rdn 4

Inputting "1 1 1 1 \n" into the above DISM code will work;
registers 1, 2, 3, and 4 will all be filled.

With DISMantle, the three trailing ones in "1 1 1 1 \n" will be dropped. 
To fill all 4 registers, you must type "1 \n 1\n 1\n 1\n".
```

Here is some example code of a loop, where the break allows you to see the incrementing without printing.

```txt
mov 1 1
mov 3 10
#LOOP: add 2 2 1
beq 2 3 #END
jmp 0 #LOOP
#END: hlt 0
```
