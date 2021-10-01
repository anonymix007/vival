# VIVAL

Command line app to validate standard I/O applications on series of tests.

# Installation

Everything should work on Linux, Windows and MacOS with Python 3.6+ versions. Also having C/C++ compiler installed is recommended but isn't required.

## Using Pip

```bash
  $ pip install vival
```
## Manual

```bash
  $ git clone https://github.com/ViktorooReps/vival
  $ cd vival
  $ python setup.py install
```

# Usage

## Testing

VIVAL supports compiling C/C++ source code with gcc/g++ compilers. Make sure you have them installed and available through CLI.

`vival <executable or source code> -t <path/tests.txt>`

Here are some flags you may find useful:
* `-t <path/tests.txt>` to specify path to text file with tests (required).
* `-nt <INTEGER>` to set the number of failed tests displayed.
* `-o <path/output.txt>` if specified, will write all tests to output.txt (recommended in fill mode).

## Creating your own tests

All the tests file structure condenses to pairs __(tag, tagged text)__, where tag specifies the use of it's text.

Tags and their meaning (some tags might not be available on earlier versions of VIVAL) for v3.1:

Tag         | Category | Tagged text meaning
----------- | -------- | -------------------
INPUT       | Test     | What will executed program get on stdin. 
CMD         | Test     | Command line arguments with which the program will be executed.
OUTPUT      | Test     | Expected contents of stdout.
COMMENT     | Test     | Some commentary on test that will be displayed if program fails it.
MAIN        | File     | Some auxillary code (usually just `int main() { ... }`) that will be compiled and linked with given source code (ignored if executable is given as argument). 
FLAGS       | File     | Flags that will be passed to compiler.
DESCRIPTION | File     | Description of tests file.
STARTUP     | Test     | Shell commands that will be executed before test.
CLEANUP     | Test     | Shell commands that will be executed after test.
TIMEOUT     | File     | Sets time limit in seconds for all tests in the file (default is 2.0 sec).
SEPARATOR   | File     | Sets separator for INPUT and OUTPUT tokens

The body of tests file consists of repeating sections of "wild space" and bracketed text: <...WS...>__/{__<...text...>__}/__ . Wild space is mostly skipped apart from tags that will define meaning of text in brackets. The text in brackets stays unformatted.

So, for example:
```
The text that is going to be skipped
INPUT
This will be skipped as well /{1 2 3}/
```
Will be converted to pair `(INPUT, "1 2 3")`.

If there are multiple tags only the last one will define bracketed text's meaning.

There are two categories of tags: the ones that define some features of a particular test and the ones that define features of an entire tests file. Only one definition of each feature is allowed, so when multiple File feature definitions are present in file, the last one is used. Redifinitions of Test features mark the beginning of a new test.

For example, file with such contents: 
```
DESCRIPTION /{Description 1}/
DESCRIPTION /{Description 2}/ <- DESCRIPTION is redefined

COMMENT /{Empty test}/

MAIN /{int sum(int a, int b) { return a + b; }}/

COMMENT <- the start of a new test
/{Some other test}/
INPUT
/{1}/
OUTPUT
/{}/
```
Will be translated to File features: `{(DESCRIPTION, "Description 2"), (MAIN, "int sum(int a, int b) { return a + b; }")}`, and two tests: `{(COMMENT, "Empty test")}` and `{(COMMENT, "Some other test"), (INPUT, "1"), (OUTPUT, "")}`

Tests without `OUTPUT` feature are considered unfilled and VIVAL wont't run given program on them. These tests can be filled in manually or using another executable.

## INPUT and OUTPUT features

Important to note that `INPUT` and `OUTPUT` features are preprocessed by splitting them into tokens. You can set the separator 
used to split source text and then join separated tokens by configuring `SEPARATOR` feature:

```
SEPARATOR /{<SEP>}/

INPUT /{1}/ /{2}/ /{3<SEP>4}/
will be converted to 1<SEP>2<SEP>3<SEP>4
```

The default value of `SEPARATOR` is new line.   
This feature is especially important with tests created for programs with ambiguous output. Note that it is not recommended
to set `SEPARATOR` value to "$" as this symbol is used to recognize output variables.

## OUTPUT variables

These features deal with ambiguous outputs. Output variables are tokens with runtime defined value. They are mentioned in 
text of `OUTPUT` feature as `$VARIABLE_NAME`. Once program is run on test with output variables, and it's
output got split into tokens using `SEPARATOR` feature (it's important to note that variable will be recognized only
if there is no other text in token other than variable definition present in token's text), VIVAL starts comparing expected 
and program's outputs **token by token**. The first time it encounters variable with specific name, it takes the value of 
corresponding token from program's output and substitutes all the occurrences of this variable with mentioned token.

Example:
```
let's say we have OUTPUT /{3 reps: $V1 $V1 $V1}/ and SEPARATOR /{ }/
```
Then this test's expected output will be converted to `[3, reps:, $V1, $V1, $V1]` tokens and any program that outputs
text `"3 reps:"` and then any string repeated 3 times and separated by space symbol will pass this test.

**Why would you need these variables?**

The answer is simple: to account for some runtime-defined values (such as PID) in program's output.  

Example:
```
Test for a task defined as follows:
'Create N processes, each of them should count from A and up to B in specified format: 

PID A
PID A + 1
...
PID B - 1
PID B

N, A, B are passed as command line arguments.'

SHUFFLED /{True}/ 

SEPARATOR /{ }/ 

CMD /{3 1 3}/

OUTPUT 
/{$PID1 }/
```

## Filling in

Fill mode activates with flag `-m`:

`vival <executable or source code> -t <path/tests.txt> -m fill`

VIVAL in fill mode runs given executable on unfilled tests, saving the output from stdout. \
But it will be lost unless `-o <path/output.txt>` is specified:

`vival <executable or source code> -t <path/tests.txt> -m fill -o <path/output.txt>`

In which case executable's output on a test will be saved as `OUTPUT` feature of test and then resulting filled tests will be written to `output.txt`.
