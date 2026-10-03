# CSE314 — Compiler Design Lab

## Combined Lexer and LL(1) Parser

This project is a single-file C implementation of a DFA-based lexical analyzer and a table-driven LL(1) parser. The program reads a source program from `input.txt`, performs lexical analysis using a DFA transition table, generates the corresponding token stream, and saves the tokens to `output.txt`. The generated token stream is then processed by the LL(1) parser for syntax analysis.

## Project Description

The main purpose of this project is to demonstrate the basic phases of a compiler, particularly lexical analysis and syntax analysis. The lexer scans the input source code and identifies different types of tokens such as keywords, data types, functions, variables, numbers, operators, labels, and other language-specific symbols. Comments are also handled during lexical analysis and are ignored when generating the token stream.

After lexical analysis is completed, the parser reads the generated tokens from `output.txt`. It uses a stack-based LL(1) parsing algorithm along with a set of grammar productions and an LL(1) parsing table. During parsing, the program displays a trace showing the current lookahead token, stack top, whether the symbol is a terminal or non-terminal, and the action performed by the parser.

## Features

The project includes DFA-based lexical analysis, token generation, comment handling, variable and label recognition, function recognition, `printfFn()` argument validation, table-driven LL(1) parsing, parsing trace generation, syntax validation, and a final `ACCEPTED` or `REJECTED` result.

## Project Files

The main source code is stored in `combined.c`. The source program to be analyzed is provided through `input.txt`. After lexical analysis, the generated token stream is stored in `output.txt`. The project documentation is maintained in `README.md`.

## Program Flow

The program first reads the source code from `input.txt`. The lexer then scans the source code using the DFA transition table and generates tokens. These tokens are written to `output.txt`. The parser subsequently reads the token stream and performs LL(1) syntax analysis using the predefined grammar and parsing table. Finally, the program displays the parsing result as either `ACCEPTED` or `REJECTED`.

## Grammar

The parser uses 22 grammar productions covering program structure, function definitions, parameters, the main function, statements, expressions, arguments, and conditions. These productions define the syntax that an input program must follow in order to be successfully parsed.

## Compilation

The program can be compiled using GCC with the command `gcc combined.c -o combined`. For additional compiler warnings and C11 standard support, the command `gcc -Wall -Wextra -std=c11 combined.c -o combined` can be used.

## Running the Program

After compilation, place the source program that needs to be analyzed inside `input.txt`. Then run the program using `./combined`. The program will perform lexical analysis, generate `output.txt`, read the generated tokens, perform LL(1) parsing, display the parsing trace, and show the final result.

## Example Input

An example input program can contain a header such as `#include<stdio.h>`, function definitions, variable declarations, assignments, return statements, loops, and `printfFn()` statements according to the grammar defined in the project.

## Requirements

The project requires GCC or another C11-compatible compiler and the standard C library. No external libraries are required.

## Author

CSE314 — Compiler Design Lab  
Spring 2026

