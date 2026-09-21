# shell programming

For management purposes,shell is excellent and great tool.

Thanks to that portability,shell script in the vast majority shell can be used .

shell still as a popular skill being learned by a huge number of Ops engineers  right now.

editor writing note intended to assist want to quick start shell programming,lacking some theoretical knowledge.

## shell bang

Q:what is shell bang and way learn shell bang?

A:The shebang must be the absolute first line of the file,tell which interpreter should execute that file.

*No use allow blank lines,any spaces or space before!*

>   #!interpreter [optional-argument]
>
> - **Bash/Shell**: `#!/bin/bash` or `#!/usr/bin/env bash`
>
> - **Python**: `#!/usr/bin/env python3`
>
> - **Node.js**: `#!/usr/bin/env node`
>
> - **Perl**: `#!/usr/bin/perl`
>
>   ##### The "Env" Trick (Best Practice)
>
>   You will often see `#!/usr/bin/env python3` instead of `#!/usr/bin/python3`.
>
>   - `#!/usr/bin/python3` assumes Python is exactly located at that path. If a user has Python installed elsewhere, the script breaks.
>   - `#!/usr/bin/env python3` searches the user's `$PATH` environment variable to find `python3`. This makes your script **much more portable** across different Linux/macOS systems.



## bash features

Q:what the relationship between bash and shell?

**A "shell" is a general concept. Bash is one specific piece of software that fulfills that concept.**

Think of it like this:

Shell** = **"Car"** (the general category of a vehicle with wheels and an engine).

Bash** = **"Toyota Camry"** (a specific make and model of a car).

Generally speaking,bash perfect for your ight now while your are reading 

- *bash is a command processor ,running in text window,can execute user the command entered directly.*
- *bash can read Linux command in file,called script.*
- *bash support wildcards,pipe,command substitution,logical reasoning control statements* 

![image-20260828114402884](C:\Users\a\AppData\Roaming\Typora\typora-user-images\image-20260828114402884.png)

`shell keep user submitted command which had executed in conversation`

>   history #command and parameters
>
> -c :clear command history in memory
>
> -r:recover history command in file
>
> -number :according number display recently  command
>
> command !! :execute last one history command





## shell execute environment(parent shell &child shell)

Q:why shell has environment and different with execute methods.

A:

1. different bash create child shell,thus didn't keep shell variable.(use  Pstree command check process tree )
2. call source is loading script in right now shell,keep  variable.



#### parent shell &child shell

*shell depend on how execute the script*

- source and dot (.), executing a script, takes effect only in the current shell environment
- Specifying the bash or sh interpreter to run a script opens a subshell, running the script commands in a child shell
- ./script, specifies the shebang, runs via the interpreter, and also runs commands in a subshell

##### parent shell(view the parent shell )

How to view the parent shell are two methods as follows.

> pstree
>
> ps -ef --forest   #-f diaplay uid,pid and ppid ; -e list all process information,same as -A

##### child shell(create process list/create child shell)

execute a series of shell command 

> ls;cd;pwd;echo "love" #list means List

shell process list concept,need use ( ) parentheses,as follows named process list

> (cd ~;pwd;ls ;cd /tmp/;pwd ;ls;)

test whether the subshell exists

> Linux default variable relate with shell
>
>  $BASH_SUBSHELL                     # $value=0 means parent shell otherwise is subshell
>
> (cd ~;pwd;ls ;cd /tmp/;pwd ;ls;echo $BASH_SUBSHELL)   #test whether the subshell exists



##### shell structure

> parent shell
>
> ​				->	child shell
>
> ​									->	child shell
>
> ​													->		child shell

input exit ,can exit child shell

create child shell complete task,exit shell interrupt shell,

##### nested running of subshell

subshell can nested running,Utilizing parentheses ,open subshell concept and test. In shell script development, subshells are frequently used for multi-process handling to improve program concurrency and execution efficiency.

> (pwd;(pwd;(echo $BASH_SUBSHELL)))

![image-20260828200835620](C:\Users\a\AppData\Roaming\Typora\typora-user-images\image-20260828200835620.png)



`shell depend variable  loading order ,father and son shell certain rules`

Before talking about inheritance, you need to know the two types:

- **Local (Shell) Variables**: Defined with `myvar="hello"`. These are **private** to the current shell.
- **Environment (Exported) Variables**: Defined with `export myvar="hello"` (or `export` after defining). These are marked to be **copied** into any child shells.



environment variable normally means use "export" Built-in commands  exported variable ,mainly used for define shell runtime environment,keep shell command execute correctly.

shell determined by environment variable check login use name ,path ,file system and so on.

environment variable can temporarily create,but exit shell terminal will lost it.

![image-20260828202702195](C:\Users\a\AppData\Roaming\Typora\typora-user-images\image-20260828202702195.png)



![image-20260828204931224](C:\Users\a\AppData\Roaming\Typora\typora-user-images\image-20260828204931224.png)

#### The .or source command

. and source load script in present shell,load variable in shell,if want learn more usage please read shell function section .

other addition:

If you are writing a script with `#!/bin/bash`, use **source**. It is much more obvious to anyone reading your code that you are loading an external file.
If you are writing a script with `#!/bin/sh` (for maximum compatibility), you **must** use the **`.`**, because `source` isn't guaranteed to exist there.

## shell variable

- set define and value,make sure no spaces between  variable and values

> name="we1l"
>
> variable name
>
> variable types,bash as all variable as string by default
>
> bash variable weak typing,no need pre-declared type,declaration and assignment happen at the same time

- variable substitution & variable reference

> name="we1l"
>
> echo ${name}
>
> echo $name         #parentheses can be omitted

- variable naming convention
  - name need correct and precise,prohibit the use of reserved keywords 
  - contains only  numbers,letters,underscores
  - no numbers at the beginning
  - no punctuation allow
  - strictly distinguish between uppercase and lowercase letters

> effective writing style
>
> Name_me
>
> NameMe
>
> invalid writing style
>
> %me
>
> me-name

- variable scope
  - local variable only applicable to the current shell process
  - environment variable ,or global variable ,applicable to current shell and other child shell. Also divided into custom and build-in environment variable
  - local variable applicable to shell function or script

- position parameter variable : applicable to transferred   parameter shell script 

- special variable : shell build-in special efficacy variable

  - $?

  ​         0:execute success

  ​         1-255:error code

  ```
  
  $? Return status of the last command execution, 0 is success, non-0 is failure
  $$ Process ID of the current shell script
  $! PID of the last background process
  $_ The last command executed before, or the last parameter
  Lookup method: man bash
  Search for Special Parameters
  ```

  

- custom variable 

  -  variable assignment : varName=value

  -  variable reference :  ${varName}

     *the use off quotation marks has great effect in reference*

    '' : stronger reference,close parsing output original text

    "" : weak reference,output all content in " ",will recognize special symbols

    no quotation :not recommended,consecutive variable can use this,exist blank will make ambiguity

    backtick: command substitution,refer command execute result,equal with $()

    > n1=1
    >
    > n2=2
    >
    > n3="$n1" #replace variable value             -""
    >
    > n4='$n1'  #recognized as regular string   -''



## shell String manipulation



#### shell string basic grammar

*before learn how string manipulate we need understand string basic grammar*



> ${variable}  return variable value
>
> ${#variable}  variable="apple" return variable and string length
>
> ${variable:start}  return variable  string after the offset number
>
> ${variable:start:length}    return variable  string after the offset by length limited
>
> ${variable#word}    Delete the shortest matching word substring from the beginning of the variable
>
> ${variable##word} Delete the longest matching  word f rom the beginning of the variable
>
> ${variable%word} Delete the shortest matching word from the end of the variable
>
> ${variable%%word}  Delete the longest matching word from the end of the variable
>
> ${variable/pattern/string} Replace the first matching  pattern  with  string
>
> ${variable//pattern/string} Replace all matching pattern occurrences with  string



#### String slice

There are generally two ways to slice strings from Shell variables: slicing from a specified position and slicing from a specified character (substring).

##### **Slicing from a Specified Position**

This method requires two parameters: in addition to specifying the starting position, the slice length is also needed to finally determine the substring to be extracted.

Since a starting position needs to be specified, the issue of counting direction arises: whether to count from the left side of the string or from the right side. The answer is that Shell supports both counting methods simultaneously.

###### **1) Counting from the Left Side of the String**

If you want to count from the left side of the string, the specific format for slicing the string is as follows:

```
${string: start :length}
```

Here, `string` is the string to be sliced, `start` is the starting position (starting from the left, counting begins at 0), and `length` is the length to be sliced (if omitted, it indicates until the end of the string).

###### 2) Counting from the right Side of the String

If you want to count from the right side of the string, you need to enclose the starting offset in parentheses, otherwise Shell will interpret it as an arithmetic subtraction operation. The specific format for slicing a string from the right is shown below: `${string: -start :length}`

> ⚠️ Note: There **must be a space before the minus sign `-`**.

- `string`: the target string to slice
- `-start`: starting position counted from the right‑hand end of the string; counting starts at `1` (rightmost character is position 1)
- `length`: the number of characters to extract. If omitted, extraction continues all the way to the end of the string.

Quick example

```
str="ABCDEFG"
# Take 3 characters starting from 3rd position on the right
echo ${str: -3:3}
# Output: EFG
```

Summary comparison table

| Syntax                  | Direction     | Start index origin | Example        |
| ----------------------- | ------------- | ------------------ | -------------- |
| `${str:start:length}`   | Left‑to‑right | `0`                | `${str:0:2}`   |
| `${str: -start:length}` | Right‑to‑left | `1`                | `${str: -3:2}` |

Common pitfall

`${str:-3}`  wrong, this means **default value substitution**, not string slicing. 

`${str: -3}`  correct, space before minus sign for right‑hand offset slice.



#### string length statistics

There are several ways to count the length of the string ,but them have the different efficiency

```shell
1. `${#variable}` : Bash built‑in, high performance (Recommended)
2. `expr length "$str"` : POSIX compatible, external command
3. `echo -n "$str" | wc -c` : count bytes, beware newline character
4. `awk '{print length($0)}'` : for batch text processing
```

`Editor advise as much as possible use build-in command in shell programming`



#### Variable Processing


Variable Processing

- If the parameter variable value is empty, return the word string


> ${parameter:-word}

- 
  If the para variable is empty, then word substitutes the variable value, and returns its value


> ${parameter:=word}

- If the para variable is empty, word is output as stderr, otherwise output the variable value
  Used for returning error messages when an empty variable causes an error

> ${parameter:?word}

- 
  If the para variable is empty, do nothing, otherwise return word


> ${parameter:+word}

## shell command

#### **Built-in Commands, External Commands**

shell

linux commands

What are built-in commands, what are external commands

**Built-in command**: Loaded into memory when the system starts, resident in memory, higher execution efficiency, but occupies resources, e.g.,`cd`

**External command**: The system needs to read the program file from the hard disk, then read it into memory to load

External commands, also known as, file system commands downloaded separately by themselves, are programs outside of the bash shell.



```bash
/bin
/usr/bin
/sbin
/usr/sbin

[root@chaogelinux tmp]# which cd
/usr/bin/cd
```

For example, the `ps` command

> Use the linux `type` command to verify whether it is a built-in or external command

###### characteristics of external commands :

it will definitely start subshell process execution

###### characteristics of built-in command:

built-in command didn't create subshell to executing

built-in command and shell is integrated,is a part of shell,no need read other file,executing in  after system start working

you can use type command validate

```bash
cpmpgen -b
```

#### common commands

> read is built-in commands,for use in read parameter
>
> -p  #set up reminder message
>
> -t  number  # wait timeout 
>
> input a series of parameter need blank separate
>
> last use variable receive values

## shell  script development

Q:Do shell script programming have dependencies or library functions?

A:Shell does **not** have compiled‑style shared libraries like `.so` / `.dll` in C, or import‑style module systems like Python. But it still has equivalent concepts: built‑ins, external dependencies, and reusable script libraries.



### shell  calculate

*in integer  calculate in shell has native methods,but floating-point calculate need refer other tool*

#### Common Arithmetic Operation Commands in Shell

| Arithmetic Operators and Commands | Meaning                                                      |
| --------------------------------- | ------------------------------------------------------------ |
| (())                              | Commonly used operator for integer arithmetic, very efficient |
| let                               | Used for integer arithmetic, similar to (())                 |
| expr                              | Can be used for integer arithmetic, but also has many other additional functions |
| bc                                | A calculator program under Linux (suitable for integer and floating-point arithmetic) |
| $[]                               | Used for integer arithmetic                                  |
| awk                               | awk can be used for both integer and floating-point arithmetic |
| declare                           | Defines variable values and attributes; the -i parameter can be used to define integer variables for arithmetic operations |

###### **Common Arithmetic Operators in Shell**

| Arithmetic Operator               | Meaning (* indicates commonly used)                          |
| --------------------------------- | ------------------------------------------------------------ |
| `+`, `-`                          | Addition (or positive sign), Subtraction (or negative sign) * |
| `*`, `/`, `%`                     | Multiplication, Division, Remainder (Modulo) *               |
| ``                                | Exponentiation *                                             |
| `++`, `--`                        | Increment and Decrement, can be prefix or postfix to variable * |
| `!`, `&&`, `||`                   | Logical NOT (Negation), Logical AND (and), Logical OR (or) * |
| `<`, `<=`, `>`, `>=`              | Comparison operators (Less than, Less than or equal to, Greater than, Greater than or equal to) |
| `==`, `!=`, `=`                   | Comparison operators (Equal, Not equal; for strings "=" can also mean equivalent) * |
| `<<`, `>>`                        | Left shift, Right shift                                      |
| `~`, `|`, `&`, `^`                | Bitwise NOT, Bitwise XOR, Bitwise AND, Bitwise OR            |
| `=`, `+=`, `-=`, `*=`, `/=`, `%=` | Assignment operators, e.g., `a+=1` is equivalent to `a=a+1`, `a-=1` is equivalent to `a=a-1` * |

##### (( ))

![image-20260829224040330](C:\Users\a\AppData\Roaming\Typora\typora-user-images\image-20260829224040330.png)

##### let 

qual with (( )) but have higher efficiency 

##### expr

a simple calculator executes commands

```
expr -- help #Find mmore usage
```

example

```bash
expr 5 + 3 #expr commmand not use-friendly,basic in blank input parameters
expr 5 \* 3 #special character escaping through '\'
expr 5 \> 3 
expr length apple #use expr parameter do obtain different results
```

###### expr pattern matching

> expr support pattern matching
>
> two special symbols
>
> `:` : count string character
>
> `.*` : any string string repeat 0 or any number of times
>
> grammar: expr String  ":"  ".*"

### shell test

> shell provides syntax for conditional testing
> test command
> [ ] square brackets

####  Common Syntax for Conditional Test

| Conditional Test Syntax             | Description                                                  |
| :---------------------------------- | :----------------------------------------------------------- |
| Syntax 1: `test <test expression>`  | This is the method of using the `test` command to perform conditional test expressions. There is at least one space between the `test` command and "`<test expression>`". |
| Syntax 2: `[ <test expression> ]`   | This is the method of performing conditional test expressions using `[]` (single brackets), and its usage is the same as the `test` command; this is the method recommended by Old Boy. There is at least one space between the boundary and the content of `[]`. |
| Syntax 3: `[[ <test expression> ]]` | This is the method of performing conditional test expressions using `[[]]` (double brackets), which is a newer syntax format than `test` and `[]`. There is at least one space between the boundary and the content of `[[]]`. |
| Syntax 4: `(( <test expression> ))` | This is the method of performing conditional test expressions using `(())` (double parentheses), generally used within `if` statements. Spaces are not required at both ends of `(())` (double parentheses). |

#### test condition test

test command evaluate a expression ,true return 1,else if return 0,utilize $? get value

##### test command parameter 

> 
>
> 1.Regarding 'type' detection (existence or not) of a certain filename, e.g., test -e filename
>
> -e: Does the 'filename' exist? (Common)
> -f: Is the 'filename' a file? (Common)
> -d: Is the 'filename' a directory? (Common)
> -b: Is the 'filename' a block device?
> -c: Is the 'filename' a character device?
> -S: Is the 'filename' a Socket file?
> -p: Is the 'filename' a FIFO (pipe) file?
> -L: Is the 'filename' a symbolic link?
>
> 2.Regarding file permission detection, e.g., test -r filename
>
> -r: Detect if the filename has 'readable' attributes?
> -w: Detect if the filename has 'writable' attributes?
> -x: Detect if the filename has 'executable' attributes?
> -u: Detect if the filename has 'SUID' attributes?
> -g: Detect if the filename has 'SGID' attributes?
> -k: Detect if the filename has 'Sticky bit' attributes?
> -s: Detect if the filename is a 'non-empty file'?
>
> 3.Comparison between two files, e.g.: test file1 -nt file2
>
> nt (newer than) Determine if file1 is newer than file2
> -ot (older than) Determine if file1 is older than file2
> -ef Determine if file2 and file2 are the same file, useful for hard link determination. The main significance is to determine if two files point to the same inode!
>
> 4.Regarding the judgment between two integers, e.g., `test n1 -eq n2`
>
> Comparison judgment on the magnitude of variable values
>
> -eq Two values are equal (equal)
> -ne Two values are not equal (not equal)
> -gt n1 is greater than n2 (greater than)
> -lt n1 is less than n2 (less than)
> -ge n1 is greater than or equal to n2 (greater than or equal)
> -le n1 is less than or equal to n2 (less than or equal)
>
> 5.Determine string data
>
> `test -z string` Determine if the string is 0? If string is an empty string, then true
> `test -n string` Determine if the string is non-zero? If string is an empty string, then false.
> Note: -n can be omitted
> `test str1 = str2` Determine if str1 is equal to str2, if equal, return true
> `test str1 != str2` Determine if str1 is not equal to str2, if equal, return false
>
> 6.Multiple condition judgment, e.g.: `test -r filename -a -x filename`
>
> -a (and) Both conditions hold simultaneously! E.g., `test -r file -a -x file`, returns true only if file has both r and x permissions.
> -o (or) Either of the two conditions holds! E.g., `test -r file -o -x file`, returns true if file has r or x permission.
> ! Negation state, e.g., `test ! -x file`, returns true when file does not have x permission.

#### bracket condition test [ ]

in script perform condition testing the most commonly used is [ ] 

test and [ ] has same usage

it is necessary addition that blank bracket is integrant

> |
>
> note
>
> in condition use variable must add " "
>
> [ -n "$filename"]

`It is worth mentioning that [[ ]]`

[[ ]] is supplement of [],support regular processing

in [[ ]] not need escape character 

In daily work, single square brackets are used most frequently, while double brackets belong to extended syntax for special scenarios.

Moreover, double square brackets also support `-eq`, `-lt`, `<`, `>`, `=`. 

#### String comparison test 

| Common String Test Operators | Description                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| `-n "string"`                | If the length of the string is not 0, it is true, meaning the test expression holds. `n` can be understood as `no zero`. |
| `-z "string"`                | If the length of the string is 0, it is true, meaning the test expression holds. `z` can be understood as an abbreviation for `zero`. |
| `"str1" = "str2"`            | If string 1 equals string 2, it is true, meaning the test expression holds. `==` can be used as a replacement. |
| `"str1" != "str2"`           | If string 1 is not equal to string 2, it is true, meaning the test expression holds. However, `!==` cannot be used as a replacement. |

Comparing the values of two string variables, checking for equality or inequality scenarios.

> - `=` : Judge whether equal
> - `!=` : Judge whether not equal
> - `!` : Take the inverse of the result, reverse true and false

> Note
> Regarding string variable comparison
> Always remember to add double quotes to variables
> Check if variable values are equal
>

#### numerical comparison test

| Comparison symbols used in`[]`and`test` | Comparison symbols used in`(())`and`[[]]` | Description                                              |
| :-------------------------------------- | :---------------------------------------- | :------------------------------------------------------- |
| `-eq`                                   | `==` or `=`                               | Equal, full spelling is equal                            |
| `-ne`                                   | `!=`                                      | Not equal, full spelling is not equal                    |
| `-gt`                                   | `>`                                       | Greater than, full spelling is greater than              |
| `-ge`                                   | `>=`                                      | Greater than or equal to, full spelling is greater equal |
| `-lt`                                   | `<`                                       | Less than, full spelling is less than                    |
| `-le`                                   | `<=`                                      | Less than or equal to, full spelling is less equal       |

1.a Usage of numerical testing in brackets and `test`

In brackets, when using mathematical comparison symbols, please add escape symbols

```
# \>
#in test and [ ] grammar support -eq and < > = !=
```

#### logical operation symbol

> && - a     AND    operation, the result is true only if both sides are true
>
> ||   - o      OR     operation, the result is true if one of the sides is true

| Operators used in [ ] and test | Operators used in [[ ]] and (()) | Description                                          |
| :----------------------------- | :------------------------------- | :--------------------------------------------------- |
| -a                             | &&                               | and, both ends are true, then the result is true     |
| -o                             | \|\|                             | or, one end is true, then the result is true         |
| !                              | !                                | not, both ends are opposite, then the result is true |

### shell  function

#### if statement



```
#Single-branch if
if  <condition expression>
	then
        code..
fi

#Simplified
if <condition expression>; then
    code....
fi


#nested if
if <condition expression>
then
    code 1....
      if <condition expression>
    	then
        code 2...
    fi
fi

#if-else
if <condition expression>

    then

        When condition is true, execute me.... (Command Set 1)

else

    Otherwise, execute me.... (Command Set 2)

fi

#multi branch if
#Syntax
#if: If... then
#elif: Else if... then
#else: Otherwise... does not need 'then'

if <condition expression>
    then
        Code 1
elif <condition expression 2>
    then
   		Code2 
elif <condition expression 3>
		Code3
fi		
```



#### case statement

The basic structure of a `case` statement is as follows:

```bash
case "$variable" in
    pattern1)
        # Commands to execute if pattern1 matches
        ;;
    pattern2)
        # Commands to execute if pattern2 matches
        ;;
    *)
        # Default commands (executed if no other pattern matches)
        ;;
esac
```

**Key Syntax Elements:**

- **`case`**: The keyword that starts the statement.
- **`$variable`**: The expression or variable to be evaluated. It is highly recommended to enclose it in double quotes (e.g., `"$variable"`) to prevent errors if the variable is empty or contains spaces.
- **`in`**: A keyword separating the variable from the patterns.
- **`pattern)`**: The matching condition. The right parenthesis `)` marks the end of the pattern.
- **`;;`**: The terminator for each branch. It acts like the `break` keyword in other programming languages, causing the script to exit the entire `case` structure after executing the matched commands.
- **`\*)`**: The default wildcard pattern. It matches anything that hasn't been matched by previous patterns, functioning similarly to an `else` block in an `if` statement.
- **`esac`**: The keyword that ends the statement (it is simply `case` spelled backward).

#### **Shell Function Development**

The characteristics of functions are similar to `alias`; they can simplify Linux command operations, making the entire command more readable and easier to use.

- A function is simply combining the shell commands you need to execute into a **function body**.
- You also need to give this function body a name, which is called the **function name**.
- Function Name + Function Body
- In the future, if you want to execute this function, you just use this function name.

##### **What are the benefits of using functions?**

usage

> 1. Define the function first
> 2. Use the function

- Encapsulating identical programs and definitions into a function can reduce the amount of code in a program and improve development efficiency.
- Using functions allows you to write less code. Finishing coding earlier means you can go home and rest sooner, which is great.
- Functions can enhance program readability and maintainability (container management).

###### Syntax for shell function definition



```
###### Standard shell function definition

###### function function_name() {

    Function body
    The linux commands you want to execute...
    return return_value
}
bash



###### Lazy syntax

When using the function keyword, parentheses can be omitted

function function_name {
    Function body
    The commands you want to execute...
    return return_value
}
bash



###### Super lazy syntax, when you are a shell pro

Parentheses are required

function_name() {

}
```

#### **Basic Concepts of Function Execution**

- To execute a shell function, simply write the function name; no other content needs to be added.
- Functions must be defined before execution, as shell scripts are loaded from top to bottom.
- Variables defined within the function body are called local variables.
- A `return` statement needs to be added inside the function body. Its purpose is to exit the function and assign a return value to the program calling the function, which is the shell script (in shell scripts, after defining and using a function, once the script execution finishes, you can retrieve its return value via `$?`).
- The `return` statement is different from `exit`:
  - `return` ends the execution of the function and returns a value (exit value, return value).
  - `exit` ends the shell environment and returns a value (exit value, return value) to the current shell.
- If a function is written into a separate file, it needs to be read using `source`.
- Inside a function, use the `local` keyword to define local variables.