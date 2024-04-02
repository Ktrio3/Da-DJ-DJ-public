# Da DJ DJ

This is the DJ DJ, mixing up your DJ compiler with the sickest of ~~beats~~ features.

This python script and JSON configuration file allows you to create a DJ compiler
with any of the different features you need.

Dr. Ligatti wants the compiler for this year to have multiple parameters AND booleans?
DISM has less than, and DJ has greater than?  No problem!  Just enable the desired
features in options.json, run the python script, and voila!  The output folder will
contain a compiler with all of the selected features!

Most features are a simple True/False option, but the options are described below.

Unless otherwise stated, features will not conflict with any other feature.  For example,
while DJ compilers in class only ever have greater than OR less than, you may enable
both features here to get a compiler that supports both if you want.

The main in dj.y includes an extra argument that will run only the relevant parts for each assignment (see help/usage message from program). Note that
I could not think of a good, clean a way to programmatically enable printing for tokens in dj.l, so that assignment still requires setting the debug in dj.l and recompiling.

The code also includes a printST.c file that prints out the symbol table. This is also useful when debugging student programs; simply modify their main in dj.y to call printST, then include printST.c when compiling. 

## Special Notes

The FINAL feature added in 2021 raises an interesting issue as it is the first time since 2009 that the AST changed (the original 2007 version had a different DOT_ID structure). If another new feature is added that would be similar, there would presumably be several conflicting DECLs, and implementing all these features at the same time would require changing the AST structure even further.  For example, if a private method feature was added, then there would likely be a METHOD_DECL, FINAL_METHOD_DECL, and a PRIVATE_METHOD_DECL. To implement BOTH features, Da DJ DJ would need a special FINAL_PRIVATE_METHOD_DECL. This would not be difficult to add, but would require ADDING features, not just SWAPPING in existing features.

## Pure DJ

The Da DJ DJ can take an optional parameter "--pure". If present, the output DJ program will not contain any of the convenience factors such as DISMantle (even if it is enabled in the config file) or the ability to switch which assignment the compiler should build for. It will not contain any of the potential tokens, or any other oddities. The files will be functionally identical to those
the students are expected to provide. 

In 2024, I tested these files output to ensure student submissions could be compiled using the object files they produce (e.g., for as3, you can copy the lex.yy.c and compile a student's dj.y without any issues). This should be the case for
all DJ features, but I only tested this that year.

## Testing

Da DJ DJ has been tested (with the correct configurations) to run on as5/as6 for 2020, 2021, 2022, and 2023, and with all of the features enabled at once. I did not test on assignments 2,3, or 4 (correctness on as5 and as6 means these should also be correct; if anything is wrong on those assignments, it should just be some missing printing in ast.c or dj.l). 

Note that this just means it passed all of the tests when compared to MY compiler, which just means it's outputting an equivalent compiler to mine. There could be some small edge cases
that either aren't covered by the test cases. After 4 years, I'm really confident in my code, but can't make any promises :)

### Building Tests for Grading

The folder `all-tests` contains every single test file that I, Dr. Ligatti, or previous TAs have written. This means 2013
to 2024. I've gone through and (painfully) added comments to each test file detailing what it is for, renamed some of them
to make their purpose clear, and even wrote some new ones.

The assignment that each test impacts will depend on the compiler settings. For example, for DJ without multiple method params the COMMA will be a lex error; one of LESS and GREATER will also similarly be a lex error.
Thus, I wrote a quick Python script that will try to compile every single test file and will sort files by assignment.

To get all of the test files for the assignments, you can run `python3 sort_tests.py X` and it will find all of the 
appropriate tests for assignment X using the current compiler version. This is determined using the following:

1. Runs the program for all of the assignments since assignment 1 doesn't have tests like this.
2. Will have all programs; programs with a lex error will be a "bad" test, all others will be good.
3. Excludes programs with a lex error; programs with a syntax error will be a "bad" test, all others will be good.
4. Same as 3.
5. Excludes programs with a lex/syntax error; programs with a semantic error will be a "bad" test, all others will be good.
6. Exludes programs with any compilation error; programs with a runtime error will be a "bad" test, all others will be good.

The sorted test files will be under `output/tests/asX`, where X is the assignment number.

For assignment 5 and 6, the programs will instead be sorted by their level. This will be determined by printing and reading the symbol table for the program. These levels correspond to the levels in the as6 handout. They are **NOT** sorted by good/bad to avoid
making the testing process overly complex (although this could be easily added if desirable; the script already determines which tests are bad in as5 to ignore for as6).

1. Level 1 includes programs with a symbol table that only contains the class Object and nothing under main.
2. Level 2 includes programs with a symbol table that only contains the class Object, and has one more local main variable.
3. Level 3 includes all of the remaining programs.
4. Note that for each level, assignment 6 may also include a "input" subcategory, which contains any program that makes a call to readNat.

Note that `all-tests` does not include all of the good/bad test files Dr. Ligatti gives every year. If you want to add them, 
you can run the following:

```bash
python3 sort_tests.py 1 -d /path/to/directory/with/additional/tests
```

You can provide `-d` as many times as you like.

**Finally, instead of recreating every test file twice, once with bools and one without, every test file
has been written assuming bools are allowed**. As part of the comments at the top of the test file, there
can be an optional `DJ-test-builder-replace-bools` option. If present, 
all instances of `bool` will be replaced with `nat`, `true` with `1`, and `false` with `0`.
The replacement is done with the 
regex `(\btrue\b)`, where \b is a word boundary (thus, something like `trueVar` is not caught but `true;` is caught). As an example, see `and_basic.dj`.

NOTE THIS MEANS YOUR TEST FILES CANNOT DECLARE VARIABLES NAMED `true` or `false` AND HAVE
`DJ-test-builder-replace-bools` ENABLED! If you need a test file with such vars, you'll need
two versions. 

Note also that this is not performed for >/< and &&/||. While possible, it would not be a simple swap. These symbols are not as commonly used, though, so not nearly as many tests will end up repeated (46 bool files vs 26 for &&/<) and most of them were already correctly labeled.

Instead of making the tests and copying them to the grading directory, I recommend just
linking to the output directory in your run scripts. See as3-6 in 2024 for example.

Stress tests (tests that pushed the upper limits of DJ memory) were originally 
kept in their own category as they took forever to run. However, with the 
speed improvements to DISM in DISMantle (instruction lookup went from O(n) to O(1), see DISMantle readme for more detail), this is no longer necessary. If desirable, they could be sorted out by adding a new flag in the test file comments.

**If running on your local machine**, you likely need to ensure there is a new line
character at the end of all of your tests to avoid an issue with Flex hanging. You can use the following to make sure all of the test files have a new line char:

```bash
sed -i '$a\' *.txt
```

## DISMantle

Da DJ DJ is designed to work with DISMantle, the version of DISM that includes a break command for debugging. If the DISMantle option is set to true, 
the compiler will include a breakDISM option.

The instruction was named breakDISM in case Dr. Ligatti adds a future option for a break instruction to exit while/for loops early (this seems fairly straightforward; whenever entering a for/while loop, the compiler stores an exit label number; the most recent number is jumped to when a break is encountered).

The breakDISM command evaluates to a 0, allowing it to be used anywhere a natural number can appear.

Here is DJ code similar to the example from DISMantle, where you can see the local variable be incremented without printing.

    main{
        nat x;
        while(x < 10){
            breakDISM;
            x = x + 1;
        };
    }

To make debugging methods easier, Da DJ DJ compilers label methods using the form "#cXmY", where X is the class number, and Y is the method number. So, "b #c1m0" would break on the first method in the first user-defined class.  If you want to skip the prologue, you can append "start"; "b #c1m0start" would
place a breakpoint after the prologue of class 1, method 0.

Da DJ DJ compilers also set a break point at the beginning of each line number (this isn't perfect, but works pretty well). By running "b #LINE1", you can break at 
line 1, assuming linenumber 1 actually contains an expression. Note that WHILE, FOR, and IF exprs are not included, since their line number would be the end, not beginning (although you can use line numbers inside the body of these expressions). If a line contains multiple expressions, it will stop at the first one.

Note that if a label doesn't exist, the debugger will halt.  

## Da DJ DJ "language"

Da DJ DJ uses a simple command system to generate the correct DJ code inline.  The commands are KEEPIF, ENDKEEPIF, and CALL.
Da DJ DJ commands are marked with a %%DJDJ in the DJ files.  The commands take the following formats:

    %%DJDJ KEEPIF [OPTION] [value]
    %%DJDJ ENDKEEPIF 
    %%DJDJ CALL [method]

KEEPIF and ENDKEEPIF commands are used for keeping large chunks of code depending on an enabled option in options.json. Take the following example from dj.l:

    %%DJDJ KEEPIF DJ_BOOLEANS true
        ommitted in this version to avoid leaking answers
    %%DJDJ ENDKEEPIF

The boolean code is only kept if the DJ_BOOLEANS in options.json evaluates to true. The value may be a boolean or a string matching [_-a-zA-Z0-9]. The type of value must match the type of the option (for example, DJ_BOOLEANS is a boolean, but DISM_BRANCH is either the string bgt or blt).

If the KEEPIF evaluates to false, the *ENITRE* line that the KEEPIF is on is dropped.  A line containing
ENDKEEPif is always dropped. As such, I recommend keeping KEEPIF and ENDKEEPIF on their own lines.

KEEPIF has a limited form of nesting.  Take this example from dj.y.

    %%DJDJ KEEPIF DJ_FINAL_CLASS true
        ommitted in this version to avoid leaking answers
    %%DJDJ KEEPIF DJ_STATIC true
        ommitted in this version to avoid leaking answers
    %%DJDJ ENDKEEPIF

The line containing `appendToChildrenList($$, $9);` will only be printed if both DJ_FINAL_CLASS
is true and DJ_STATIC is true. Note that the one ENDKEEPIF is enough for both KEEPIF.

Finall, the CALL command is used for small, dynamic replacements in the middle of a line. It takes
a method name to be called (the method is looked up in either lexer.py, parser.py, symbol.py, typechecker.py or codegen.py depending on the current phase).  The following example is from dj.y.

    $$ = newAST(%%DJDJ CALL nonfinal_class, $2, 0, NULL, yylineno);

The CALL command calls the nonfinal_class method in the lexer.py file, shown below.

    def nonfinal_class(self, options):
      if options["DJ_FINAL_CLASS"]:
        return "NONFINAL_CLASS_DECL"
      return "CLASS_DECL"
    
If the DJ_FINAL_CLASS is false, the following is the result in dj.y:

    $$ = newAST(CLASS_DECL, $2, 0, NULL, yylineno);
    
## Current Da DJ DJ Options

The following is a list of currently supported DJ features.

### MULTIPLE_METHOD_PARAMETERS

True or false.

When true, methods in DJ take a PARAM_LIST, and may have 0 or 1 or more parameters (separated by commas). 

When false, methods in DJ take exactly 1 param.  No more, no less.

### DJ_GREATER_THAN

True or false.

When true, DJ will have a greater than symbol (>).

### DJ_LESSER_THAN

True or false.

When true, DJ will have a lesser than symbol (<).

### DJ_WHILE

True or false.

When true, DJ will have a while operator.

### DJ_FOR

True or false.

When true, DJ will have a for operator.

### IDENTIFIERS_UNDERSCORE

The strings "start", "after", or "none".

If "start", DJ identifiers may contain an underscore character anywhere, including as the first character.

If "after", DJ identifiers must start with a letter, and may then contain underscores.

If "none", DJ identifiers may never contain underscores.

### NAT_LEADING_ZEROS

True or false.

If true, nat literals may start with one or more leading 0s.

### DJ_AND

True or false.

If true, DJ has an AND operator.

### DJ_OR

True or false.

If true, DJ has an OR operator.

### DJ_STATIC

True or false.

If true, DJ classes may have static variables.

### DJ_INSTANCE

True or false.

If true, DJ has an instance operator.

### ASSOC_PREC

An array defining the associativity and precedence.

There is no good way to implement a swapping system for this, so they must be defined manually.
The array is printed verbatim, with items in the array joined by new lines.

The list MAY include operators that are unused, but MUST include all operators that ARE used.
The program will check that all the necessary operators are included, but does no other error checking.

For example,

    ommitted in this version to avoid leaking answers

### DISM_BRANCH

A string, either "bgt" or "blt".

Determines which instruction should be used during codegen. For example, if 
DJ has greater than, the codegen phase will be correctly output to use bgt or to
inverse blt

### DJ_BOOLEANS

True or false.

Adds a boolean type to DJ.  Note that the type numbers will correctly change to accomodate.  For example, without booleans, -2 refers to any object type.  When booleans are introduced, 
bools are -2, any object becomes -3, and superclass of Object becomes -4.  This is done using defines in symtbl.c and typecheck.c. 

### DJ_OBJ_TYPE

True or false.

If true, Object is added to the lexer and parsed as an OBJ_TYPE.  If false, Object is treated as a normal id.
