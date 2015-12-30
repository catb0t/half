# half
half is an extremely simple language.

# [the original spec, cloned here:](http://codegolf.stackexchange.com/posts/22958/edit)

Half (interpreter/translator in Windows Batch)
==============================================

I don't know why I'm answering so many puzzles in windows batch, for some sick reason I think I'm enjoying it :P Anyways, this is similar to something I worked on for fun a while ago, a basic language that's translated to windows batch by a script that is also written in windows batch. It's not particularly amazing, but it works.

99 Bottles of Beer
------------------
    # Initialize variables
    bottles ~ 99
    # You can't directly compare a literal value
    zero ~ 0

    # This makes a point 'loop' that can be jumped to or used as a subroutine
    mark loop
        write $ bottles
    # You only need quotes when you have leading or trailing spaces
        print ~ " bottles of beer on the wall,"
        write $ bottles
        print ~ " bottles of beer."
        print ~ Take one down and pass it around,
        bottles @ bottles-1
        if
        bottles equ zero
            jump none
        endif
        write $ bottles
        print ~ " bottles of beer on the wall."
        print ~
    jump loop
    
    mark none
        print ~ no more bottles of beer on the wall.
        print ~
        print ~ No more bottles of beer on the wall,
        print ~ No more bottles of beer.
        print ~ Go to the store and buy some more,
        print ~ 99 bottles of beer on the wall.

Syntax
------
Only three tokens are recognized on each line, separated by spaces.

\# is a comment.

In most cases where a value is needed, a `$` in the second token signifies that the third should be treated as a variable name, whereas a `~` denotes a literal value.
General instructions take the form `<instruction> [$~] <name>`. Setting a variable takes the same form, but is implemented whenever <instruction> is not recognized.

Defined commands:

 * `print` and `write` both write output, but `write` does not add a newline. Needs $ or ~.
 * `mark` creates a point that can be jumped to or called as a subroutine.
 * `jump` equivalent of goto in batch (or any language for that matter).
 * `proc` calls a subroutine. Equivalent of `call :label`.
 * `return` returns from a subroutine. Will exit the program when not inside one.
 * `if` conditional instruction. Takes comparison from the next line, in the form `<var1> <operator> <var2>`. Operators are the same as `if`'s in batch, ie. `EQU, NEQ, LSS, LEQ, GTR, GEQ`. Will execute instructions after it only if the comparison is true.
 * `endif` ends an if statement.
 * `cat` concatenates two variables. `cat a b` will store the value of ab in a.


When none of these commands are found, the expression is treated as a variable assignment, using the first token as the variable name. `$` and `~` behave the same as in `print`, but there is also a `@` identifier. This treats the last token as a mathematical expression, passed to `set /a`. It includes most operators. If none of the three identifiers are found, this is a syntax error and the interpreter exits.

Interpreter (Windows Batch)
---------------------------
The interpreter actually translates the code into windows batch, places it in a temporary file and executes it. While it recognizes syntax errors in the Half language, the resulting batch script may cause issues, especially with special characters like parentheses, vertical bars etc.

    @echo off
    
    REM Half Interpreter / Translator
    
    if exist ~~.bat del ~~.bat
    if not exist "%1" call :error "File not found: '%1'"
    set error=
    setlocal enabledelayedexpansion
    call :parse "%1" 1>~~.bat
    if exist ~~.bat if not "error"=="" ~~.bat 2>nul
    goto :eof
    
    :parse
    set ifstate=0
    echo @echo off
    echo setlocal
    echo setlocal enabledelayedexpansion
    for /f "eol=# tokens=1,2* delims= " %%a in (%~1) do  (
        if "!ifstate!"=="1" (
            if /i not "%%b"=="equ" if /i not "%%b"=="neq" if /i not "%%b"=="lss" if /i not "%%b"=="leq" if /i not "%%b"=="gtr" if /i not "%%b"=="geq" call :error "Unknown comparator: '%%b'"
            echo if "^!%%a^!" %%b "^!%%c^!" ^(
            set ifstate=0
        ) else (
            if "%%a"=="print" (
                if "%%b"=="$" (
                    echo echo.^^!%%c^^!
                ) else if "%%b"=="~" (
                    echo echo.%%~c
                ) else call :error "Unknown identifier for print: '%%b'"
            ) else if "%%a"=="write" (
                if "%%b"=="$" (
                    echo echo^|set/p="^!%%c^!"
                ) else if "%%b"=="~" (
                    echo echo^|set/p="%%~c"
                ) else call :error "Unknown identifier for write: '%%b'"
            ) else if "%%a"=="mark" (
                if not "%%c"=="" call :error "Unexpected token: %%c"
                echo :%%b
            ) else if "%%a"=="jump" (
                if not "%%c"=="" call :error "Unexpected token: %%c"
                echo goto :%%b
            ) else if "%%a"=="proc" (
                if not "%%c"=="" call :error "Unexpected token: %%c"
                echo call :%%b
            ) else if "%%a"=="return" (
                if not "%%c"=="" call :error "Unexpected tokens: %%b %%c"
                if not "%%b"=="" call :error "Unexpected token: %%b"
                echo goto :eof
            ) else if "%%a"=="if" (
                if not "%%c"=="" call :error "Unexpected tokens: %%b %%c"
                if not "%%b"=="" call :error "Unexpected token: %%b"
                set ifstate=1
            ) else if "%%a"=="endif" (
                if not "%%c"=="" call :error "Unexpected tokens: %%b %%c"
                if not "%%b"=="" call :error "Unexpected token: %%b"
                echo ^)
            ) else if "%%a"=="cat" (
                echo set "%%b=^!%%b^!^!%%c^!"
            ) else (
                if "%%b"=="$" (
                    echo set "%%a=!%%c!"
                ) else if "%%b"=="~" (
                    echo set "%%a=%%~c"
                ) else if "%%b"=="@" (
                    echo set/a"%%a=%%c"
                ) else call :error "Unknown tokens '%%a %%b %%c'"
            )
        )
    )
    echo endlocal
    goto :eof
    
    :error
    echo.Parse Error: %~1 1>&2
    set error=1
    goto :eof
