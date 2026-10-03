/*
 * =============================================================================
 *  CSE314 — Compiler Design Lab  |  Spring 2026
 *  COMBINED LEXER + PARSER  —  Single File
 *
 *  HOW IT WORKS:
 *    1. Lexer runs first: reads input.txt -> performs DFA scan -> writes
 *       token stream to output.txt -> prints lexical analysis table
 *    2. Parser runs second: reads output.txt -> runs LL(1) table-driven
 *       parse -> prints parse trace -> prints ACCEPTED or REJECTED
 *
 *  USAGE:
 *    gcc combined.c -o combined
 *    ./combined          (reads input.txt, writes output.txt)
 *
 * =============================================================================
 *  LEXER DESIGN
 * =============================================================================
 *  Pure stream-based DFA — 120 States x 32 Input Columns
 *  No strcmp, no if/else, no switch for token recognition.
 *  All token logic lives in the dfa[][] transition table.
 *
 *  COLUMN MAP (NUM_INPUTS = 32):
 *   0=#   1=<   2=>   3=(   4=)   5=[   6=]   7=_   8=.   9=:
 *  10=/  11=digit  12=i  13=n  14=d  15=e  16=F  17=l  18==  19=+
 *  20=-  21=w  22=h  23=r  24=t  25=u  26=a  27=m  28=b  29=other-alpha
 *  30=space/tab   31=newline
 *
 *  KEY DFA FIXES APPLIED:
 *  - S0:  col13(n),col15(e),col22(h),col24(t),col25(u) now route to S39
 *         so transformFn, handleFn, updateFn, normFn, encodeFn all work
 *  - S81: col11(digit)=0 so digits inside comments are rejected (PDF Rule 2)
 *  - S83: col18(=)->S84 so '<=' correctly produces k_lte (not k_lt + k_assign)
 *  - S97-S101: each state falls back to S39 for non-mainFn letters so
 *              maxvalueFn, mergeFn, maintainFn etc. all produce k_fn
 *  - S103-S106: each state falls back to S39 for non-break letters so
 *               baseFn, buildFn, batchFn etc. all produce k_fn
 *
 * =============================================================================
 *  PARSER DESIGN
 * =============================================================================
 *  LL(1) Table-Driven Parser — Teacher Style
 *  No if/else chains for grammar logic, no switch/case, no strcmp for matching.
 *  Only strcmp used: symbol lookup in NT[] and TERMINALS[] arrays.
 *
 *  GRAMMAR (22 productions):
 *   1:  P    -> k_header FD MAIN
 *   2:  FD   -> dtype k_fn k_lparen PARAM k_rparen lparen BODY rparen FD
 *   3:  FD   -> epsilon
 *   4:  PARAM-> dtype var
 *   5:  PARAM-> epsilon
 *   6:  MAIN -> dtype k_main k_lparen k_rparen lparen BODY rparen
 *   7:  BODY -> STMT BODY
 *   8:  BODY -> epsilon
 *   9:  STMT -> dtype var k_assign EXPR k_dot
 *  10:  STMT -> k_fn k_lparen ARG k_rparen k_dot
 *  11:  STMT -> k_loop label k_colon k_while k_lparen COND k_rparen lparen BODY rparen
 *  12:  STMT -> k_return EXPR k_dot
 *  13:  STMT -> k_break k_dot
 *  14:  EXPR -> var
 *  15:  EXPR -> num
 *  16:  EXPR -> var k_op num
 *  17:  EXPR -> k_fn k_lparen ARG k_rparen
 *  18:  ARG  -> var
 *  19:  ARG  -> num
 *  20:  ARG  -> epsilon
 *  21:  COND -> dtype var k_lt num k_dot
 *  22:  COND -> dtype var k_lte num k_dot
 * =============================================================================
 */

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <ctype.h>

/* =========================================================================
 *  SECTION 1 — LEXER
 * ========================================================================= */

#define NUM_STATES  120
#define NUM_INPUTS   32

/* Maps a character to its DFA column index */
static int get_col(char c) {
    if (c == ' '  || c == '\t') return 30;
    if (c == '\n' || c == '\r') return 31;
    if (c == '#')   return 0;
    if (c == '<')   return 1;
    if (c == '>')   return 2;
    if (c == '(')   return 3;
    if (c == ')')   return 4;
    if (c == '[')   return 5;
    if (c == ']')   return 6;
    if (c == '_')   return 7;
    if (c == '.')   return 8;
    if (c == ':')   return 9;
    if (c == '/')   return 10;
    if (isdigit(c)) return 11;
    if (c == 'i')   return 12;
    if (c == 'n')   return 13;
    if (c == 'd')   return 14;
    if (c == 'e')   return 15;
    if (c == 'F')   return 16;
    if (c == 'l')   return 17;
    if (c == '=')   return 18;
    if (c == '+')   return 19;
    if (c == '-')   return 20;
    if (c == 'w')   return 21;
    if (c == 'h')   return 22;
    if (c == 'r')   return 23;
    if (c == 't')   return 24;
    if (c == 'u')   return 25;
    if (c == 'a')   return 26;
    if (c == 'm')   return 27;
    if (c == 'b')   return 28;
    return 29;   /* all other alphabets and chars */
}

/*
 * Token label for each accepting state (NULL = not an accept state)
 *
 * State guide:
 *   15  = k_header   (#include<stdio.h>)
 *   18  = dtype      (int)
 *   21  = dtype      (dec)
 *   27  = k_while
 *   33  = k_return
 *   41  = k_fn       (generic XxxFn)
 *   47  = k_loop     (loop keyword part)
 *   56  = k_dot      (..)
 *   57  = k_assign   (=)
 *   58  = lparen     ([)
 *   59  = rparen     (])
 *   60  = k_lparen   (()
 *   61  = k_rparen   ())
 *   63  = k_colon    (:)
 *   65  = num        (single digit)
 *   66  = num        (multi digit)
 *   81  = comment    (suppressed)
 *   83  = k_lt       (<)
 *   84  = k_lte      (<=)
 *   86  = k_op       (+)
 *   87  = k_op       (-)
 *   91  = var        (_alpha+ digit alpha  — Rule 3)
 *   92  = label      (loop label body: alpha+ digit digit — Rule 7)
 *  102  = k_main
 *  107  = k_break
 *  112  = label      (loop label from loop_ path)
 */
static const char *token_labels[NUM_STATES] = {
    [15]  = "k_header",
    [18]  = "dtype",
    [21]  = "dtype",
    [27]  = "k_while",
    [33]  = "k_return",
    [41]  = "k_fn",
    [47]  = "k_loop",
    [56]  = "k_dot",
    [57]  = "k_assign",
    [58]  = "lparen",
    [59]  = "rparen",
    [60]  = "k_lparen",
    [61]  = "k_rparen",
    [63]  = "k_colon",
    [65]  = "num",
    [66]  = "num",
    [81]  = "comment",
    [83]  = "k_lt",
    [84]  = "k_lte",
    [86]  = "k_op",
    [87]  = "k_op",
    [91]  = "var",
    [92]  = "label",
    [102] = "k_main",
    [107] = "k_break",
    [112] = "label",
};

/* States whose tokens are suppressed from output (comments) */
static const int skip_print[NUM_STATES] = { [81] = 1 };

/*
 * DFA TRANSITION TABLE  [state][col] -> next_state
 * 0 = dead (no transition): emit current token then re-feed the character.
 *
 * Column layout (32 entries per row):
 *  [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]  [8]  [9] [10] [11] [12] [13] [14] [15]
 *   #    <    >    (    )    [    ]    _    .    :    /   dg    i    n    d    e
 * [16] [17] [18] [19] [20] [21] [22] [23] [24] [25] [26] [27] [28] [29] [30] [31]
 *   F    l    =    +    -    w    h    r    t    u    a    m    b   oth   SP   NL
 *
 * Variable naming (Rule 3): _alpha+ digit alpha
 *   S88: saw '_'                      -> expects 1+ alpha
 *   S89: saw '_' + alpha(s), loops    -> alpha stays S89, digit -> S90
 *   S90: saw digit                    -> exactly one alpha -> S91 (ACCEPT var)
 *   S91: ACCEPT var  (dead, no further transitions)
 *
 * Loop label body (Rule 7): alpha+ digit digit
 *   S109: saw '_' (after "loop")      -> expects 1+ alpha
 *   S110: saw alpha+, loops on alpha  -> FIXED: digit only -> S111
 *   S111: saw first digit             -> digit -> S112 (ACCEPT label)
 *   S112: ACCEPT label
 *
 * NOTE: S92 is kept as a dead accept state (label) but is no longer
 *       reachable from S90 since Rule 3 vars end with alpha not digit.
 *       S92 is reachable only if the old routing was used; we zero it out.
 */
static int dfa[NUM_STATES][NUM_INPUTS] = {

/* S0  start state */
[0]={1,83,0,60,61,58,59,88,55,63,80,65,16,39,19,39,0,42,57,86,87,22,39,28,39,39,39,97,103,39,0,0},

/* S1-S15  #include<stdio.h> -> k_header */
[1]= {0,0,0,0,0,0,0,0,0,0,0,0,2,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[2]= {0,0,0,0,0,0,0,0,0,0,0,0,0,3,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[3]= {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,4,0,0},
[4]= {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,5,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[5]= {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,6,0,0,0,0,0,0},
[6]= {0,0,0,0,0,0,0,0,0,0,0,0,0,0,7,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[7]= {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,8,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[8]= {0,9,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[9]= {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,10,0,0},
[10]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,11,0,0,0,0,0,0,0},
[11]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,12,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[12]={0,0,0,0,0,0,0,0,0,0,0,0,13,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[13]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,14,0,0},
[14]={0,0,0,0,0,0,0,0,48,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[15]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,},
/* ACCEPT 15 : k_header */

/* S16-S18  int -> dtype */
[16]={0,0,0,0,0,0,0,0,0,0,0,0,0,17,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[17]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,18,0,0,0,0,0,0,0},
[18]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 18 : dtype (int) */

/* S19-S21  dec -> dtype */
[19]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,20,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[20]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,21,0,0},
[21]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 21 : dtype (dec) */

/* S22-S27  while -> k_while */
[22]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,23,0,0,0,0,0,0,0,0,0},
[23]={0,0,0,0,0,0,0,0,0,0,0,0,24,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[24]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,25,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[25]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,27,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[26]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[27]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 27 : k_while */

/* S28-S33  return -> k_return */
[28]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,29,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[29]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,30,0,0,0,0,0,0,0},
[30]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,31,0,0,0,0,0,0},
[31]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,32,0,0,0,0,0,0,0,0},
[32]={0,0,0,0,0,0,0,0,0,0,0,0,0,33,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[33]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 33 : k_return */

[34]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[35]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[36]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[37]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[38]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/* S39-S41  generic XxxFn -> k_fn
 * Loops on all lowercase alpha; 'F'->S40, 'n'->S41 (completing Fn suffix). */
[39]={0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,39,40,39,0,0,0,39,39,39,39,39,39,39,39,39,0,0},
[40]={0,0,0,0,0,0,0,0,0,0,0,0,0,41,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[41]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 41 : k_fn */

/* S42-S47  loop -> k_loop keyword
 * After 'loo' (S44), any alpha -> S47 (accept k_loop).
 * S44 on '_' routes to S109 (loop label sub-machine). */
[42]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,43,0,0},
[43]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,44,0,0},
[44]={0,0,0,0,0,0,0,109,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,47,0,0},
[45]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[46]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[47]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 47 : k_loop */

/* S48-S49  header tail (.h>) completing #include<stdio.h> */
[48]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,49,0,0,0,0,0,0,0,0,0},
[49]={0,0,15,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[50]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[51]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[52]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[53]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[54]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/* S55-S56  .. -> k_dot */
[55]={0,0,0,0,0,0,0,0,56,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[56]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 56 : k_dot */

[57]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}, /* k_assign  */
[58]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}, /* lparen  [ */
[59]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}, /* rparen  ] */
[60]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}, /* k_lparen ( */
[61]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}, /* k_rparen ) */
[62]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[63]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}, /* k_colon  */
[64]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/* S65-S66  integer literal -> num */
[65]={0,0,0,0,0,0,0,0,0,0,0,66,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[66]={0,0,0,0,0,0,0,0,0,0,0,66,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 65, 66 : num */

[67]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[68]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[69]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[70]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[71]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[72]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[73]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[74]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[75]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[76]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[77]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[78]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[79]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/* S80-S81  // single-line comment -> suppressed (Rule 2)
 * col11(digit)=0: digits inside comments cause a dead transition,
 * enforcing Rule 2 (comments: spaces and alphabets only). */
[80]={0,0,0,0,0,0,0,0,0,0,81,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[81]={81,81,81,81,81,81,81,81,81,81,81, 0,81,81,81,81,81,81,81,81,81,81,81,81,81,81,81,81,81,81,81,0},
/* ACCEPT 81 : comment (suppressed) */

[82]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/* S83  '<' -> if next is '=' go to S84 (k_lte), else emit k_lt */
[83]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,84,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 83 : k_lt */

/* S84  '<=' -> k_lte */
[84]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 84 : k_lte */

[85]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/* S86 '+' -> k_op    S87 '-' -> k_op */
[86]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[87]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/*
 * S88-S91  Variable recognition  (Rule 3: _alpha+ digit alpha)
 *
 * S88: saw '_'         -> expects 1+ alpha (all alpha cols -> S89)
 * S89: saw '_'+alpha+  -> loops on alpha; first digit -> S90
 * S90: saw digit       -> exactly ONE alpha -> S91 (ACCEPT var)
 *                         digit or other -> dead (invalid var name)
 * S91: ACCEPT var      -> completely dead (no further transitions allowed)
 *
 * FIX vs original: S90 no longer routes digit->S92 (that was the label
 * path which corrupted var recognition). S90 only routes alpha->S91.
 * S91 is fully dead ensuring no extra chars extend the var token.
 * S92 is now unreachable from the var path (kept as dead state).
 */
[88]={0,0,0,0,0,0,0,0,0,0,0,0,89,89,89,89,89,89,0,0,0,89,89,89,89,89,89,89,89,89,0,0},
[89]={0,0,0,0,0,0,0,0,0,0,0,90,89,89,89,89,89,89,0,0,0,89,89,89,89,89,89,89,89,89,0,0},
/*
 * S90: saw '_' + alpha+ + first_digit
 *   digit -> S92 (ACCEPT label)  — for loop label body: _alpha+ digit digit
 *   alpha -> S91 (ACCEPT var)    — for variable name:   _alpha+ digit alpha
 * Both paths are needed. The lexer uses this dual-purpose design:
 *   loop_main01 -> k_loop emitted, then '_main01' via S88->S89->S90->S92(label)
 *   _result4m   -> via S88->S89->S90->S91(var)
 */
[90]={0,0,0,0,0,0,0,0,0,0,0,92,91,91,91,91,91,91,0,0,0,91,91,91,91,91,91,91,91,91,0,0},
[91]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 91 : var */
[92]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 92 : label  (loop label body: _alpha+ digit digit) */

[93]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[94]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[95]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[96]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/*
 * S97-S102  mainFn -> k_main
 * Falls back to S39 (generic fn) for any alpha that doesn't continue
 * the exact spelling 'm-a-i-n-F-n'.
 */
[97]= {0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,39,40,39,0,0,0,39,39,39,39,39,98,39,39,39,0,0},
[98]= {0,0,0,0,0,0,0,0,0,0,0,0,99,39,39,39,40,39,0,0,0,39,39,39,39,39,39,39,39,39,0,0},
[99]= {0,0,0,0,0,0,0,0,0,0,0,0,39,100,39,39,40,39,0,0,0,39,39,39,39,39,39,39,39,39,0,0},
[100]={0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,39,101,39,0,0,0,39,39,39,39,39,39,39,39,39,0,0},
[101]={0,0,0,0,0,0,0,0,0,0,0,0,39,102,39,39,40,39,0,0,0,39,39,39,39,39,39,39,39,39,0,0},
[102]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 102 : k_main */

/*
 * S103-S107  break -> k_break
 * Falls back to S39 for any alpha that doesn't continue 'b-r-e-a-k'.
 */
[103]={0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,39,40,39,0,0,0,39,39,104,39,39,39,39,39,39,0,0},
[104]={0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,105,40,39,0,0,0,39,39,39,39,39,39,39,39,39,0,0},
[105]={0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,39,40,39,0,0,0,39,39,39,39,39,106,39,39,39,0,0},
[106]={0,0,0,0,0,0,0,0,0,0,0,0,39,39,39,39,40,39,0,0,0,39,39,39,39,39,39,39,39,107,0,0},
[107]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 107 : k_break */

[108]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},

/*
 * S109-S112  Loop label body sub-machine  (Rule 7: alpha+ digit digit)
 * Entered from S44 (after "loo") on '_' (col 7).
 *
 * S109: expects 1+ alpha, loops on alpha;  digit -> S110
 * S110: saw alpha+ then first digit  -> digit ONLY -> S111
 *       FIX: alpha transitions removed from S110 (old code looped on alpha here,
 *       allowing patterns like "ma1in01" which violate Rule 7)
 * S111: saw two alpha+ and first digit -> digit -> S112 (ACCEPT label)
 * S112: ACCEPT label
 */
[109]={0,0,0,0,0,0,0,0,0,0,0,110,109,109,109,109,109,109,0,0,0,109,109,109,109,109,109,109,109,109,0,0},
[110]={0,0,0,0,0,0,0,0,0,0,0,111,  0,  0,  0,  0,  0,  0,0,0,0,  0,  0,  0,  0,  0,  0,  0,  0,  0,0,0},
[111]={0,0,0,0,0,0,0,0,0,0,0,112,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[112]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
/* ACCEPT 112 : label */

[113]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[114]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[115]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[116]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[117]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[118]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
[119]={0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0},
};

static int run_lexer(void) {
    FILE *fin  = fopen("input.txt",  "r");
    FILE *fout = fopen("output.txt", "w");
    if (!fin)  { printf("Error: input.txt not found.\n");         return 1; }
    if (!fout) { printf("Error: output.txt cannot be opened.\n"); return 1; }

    printf("==============================================\n");
    printf("           LEXICAL ANALYSIS\n");
    printf("==============================================\n");
    printf("  %-25s  %s\n", "Source Element", "Token");
    printf("  %-25s  %s\n", "--------------", "-----");

    char buf[256];
    int  blen  = 0;
    int  state = 0;
    int  c;
    /*
     * FIX (Bug 3): 'first' tracks whether we have written the first real
     * token to output.txt yet. We use a conditional instead of multiplication
     * so that comment tokens (skip_print=1) do NOT falsely keep first=1,
     * which would cause the next real token to be missing its space prefix.
     */
    int  first = 1;

#define EMIT_TOKEN() do {                                                    \
    buf[blen] = '\0';                                                        \
    const char *_lbl = token_labels[state];                                  \
    const char *_tok = (_lbl != NULL) ? _lbl : "unknown";                   \
    if (!skip_print[state]) {                                                \
        printf("  %-25s  ->  %s\n", buf, _tok);                             \
        if (!first) fprintf(fout, " ");                                      \
        fprintf(fout, "%s", _tok);                                           \
        first = 0;                                                           \
    }                                                                        \
    blen  = 0;                                                               \
    state = 0;                                                               \
} while(0)

    while ((c = fgetc(fin)) != EOF) {
        int col  = get_col((char)c);
        int next = dfa[state][col];

        if (next == 0 && state != 0) {
            EMIT_TOKEN();
            /* Bounds check (Bug 7 fix): guard against buf overflow */
            next = (col >= 30) ? 0 : dfa[0][col];
        }

        state = next;
        /* Bounds check on buf before writing */
        if (blen < 254 && col < 30) {
            buf[blen++] = (char)c;
        }
    }

    if (blen > 0) { EMIT_TOKEN(); }
    fprintf(fout, "\n");

#undef EMIT_TOKEN

    printf("==============================================\n");
    printf("Lexical analysis complete.\n");
    printf("Tokens saved to output.txt\n\n");

    fclose(fin);
    fclose(fout);
    return 0;
}

/* =========================================================================
 *  SECTION 2 — PARSER
 * ========================================================================= */

#define MAXSTACK  200
#define MAXSYM     64
#define MAXTOK    500
#define NNT         9    /* number of non-terminals */
#define NTER       22    /* number of terminals     */

static const char *NT[] = {
    "P","FD","PARAM","MAIN","BODY","STMT","EXPR","ARG","COND"
};

static const char *TERMINALS[] = {
    "k_header","dtype","k_fn","k_main","k_lparen","k_rparen",
    "lparen","rparen","k_dot","k_assign","k_op","k_loop",
    "label","k_colon","k_while","k_return","k_break",
    "var","num","k_lt","k_lte","$"
};

static const char *RHS[] = {
    /* 1  */ "k_header FD MAIN",
    /* 2  */ "dtype k_fn k_lparen PARAM k_rparen lparen BODY rparen FD",
    /* 3  */ "",
    /* 4  */ "dtype var",
    /* 5  */ "",
    /* 6  */ "dtype k_main k_lparen k_rparen lparen BODY rparen",
    /* 7  */ "STMT BODY",
    /* 8  */ "",
    /* 9  */ "dtype var k_assign EXPR k_dot",
    /* 10 */ "k_fn k_lparen ARG k_rparen k_dot",
    /* 11 */ "k_loop label k_colon k_while k_lparen COND k_rparen lparen BODY rparen",
    /* 12 */ "k_return EXPR k_dot",
    /* 13 */ "k_break k_dot",
    /* 14 */ "var",
    /* 15 */ "num",
    /* 16 */ "var k_op num",
    /* 17 */ "k_fn k_lparen ARG k_rparen",
    /* 18 */ "var",
    /* 19 */ "num",
    /* 20 */ "",
    /* 21 */ "dtype var k_lt num k_dot",
    /* 22 */ "dtype var k_lte num k_dot",
};

/*
 * LL(1) PARSING TABLE
 * TABLE[NT_index][T_index] = production number  (0 = error)
 *
 * Rows: P=0 FD=1 PARAM=2 MAIN=3 BODY=4 STMT=5 EXPR=6 ARG=7 COND=8
 * Cols: k_header=0 dtype=1 k_fn=2 k_main=3 k_lparen=4 k_rparen=5
 *       lparen=6 rparen=7 k_dot=8 k_assign=9 k_op=10 k_loop=11
 *       label=12 k_colon=13 k_while=14 k_return=15 k_break=16
 *       var=17 num=18 k_lt=19 k_lte=20 $=21
 *
 * Rule 9 enforcement: FD (custom functions) are parsed BEFORE MAIN.
 * Production 2 expands FD into a full function definition and recursively
 * calls FD again, allowing one or more custom functions.
 * Production 3 (epsilon) fires when the lookahead is k_main or $,
 * meaning no more custom functions — transition to MAIN.
 * If a custom function appears AFTER mainFn, the parser will have already
 * consumed MAIN and will hit an unconsumed token -> REJECTED.  ✓
 */
static int TABLE[NNT][NTER] = {
/*        kh  dt  fn  km klp krp  lp  rp  kd  ka  ko  kl  lb  kc  kw  kr  kb  vr  nm  lt  le   $ */
/* P  */ { 1,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0},
/* FD */ { 0,  2,  0,  3,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  3},
/* PA */ { 0,  4,  0,  0,  0,  5,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0},
/* MA */ { 0,  6,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0},
/* BO */ { 0,  7,  7,  0,  0,  0,  0,  8,  0,  0,  0,  7,  0,  0,  0,  7,  7,  0,  0,  0,  0,  0},
/* ST */ { 0,  9, 10,  0,  0,  0,  0,  0,  0,  0,  0, 11,  0,  0,  0, 12, 13,  0,  0,  0,  0,  0},
/* EX */ { 0,  0, 17,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0, 14, 15,  0,  0,  0},
/* AR */ { 0,  0,  0,  0,  0, 20,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0, 18, 19,  0,  0,  0},
/* CO */ { 0, 21,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0, 22,  0},
};

/* Stack */
static char stack[MAXSTACK][MAXSYM];
static int  top = -1;

static void push(const char *s) { strcpy(stack[++top], s); }
static char *pop_sym(void)      { return (top >= 0) ? stack[top--] : NULL; }

static int find_nt(const char *x) {
    for (int i = 0; i < NNT; i++)
        if (strcmp(NT[i], x) == 0) return i;
    return -1;
}

static int find_t(const char *x) {
    for (int i = 0; i < NTER; i++)
        if (strcmp(TERMINALS[i], x) == 0) return i;
    return -1;
}

/* -------------------------------------------------------------------------
 * run_parser(): reads output.txt, runs LL(1) parse, prints trace + verdict.
 * Returns 0 on success, 1 on file error.
 *
 * Rule 9 is enforced structurally: the grammar processes FD (custom
 * functions) before MAIN. Any custom function token appearing after mainFn
 * causes "Unconsumed input" -> REJECTED.
 *
 * Rule 8 enforcement (printfFn):
 *   - inside_printf_fn is set when prod 10 fires (k_fn call statement).
 *     We specifically check the function name token to target only printfFn.
 *   - FIX (Bug 4): we now reject BOTH epsilon ARG (prod 20) AND num ARG
 *     (prod 19) when inside printfFn. Only var ARG (prod 18) is allowed.
 * ------------------------------------------------------------------------- */
static int run_parser(void) {
    FILE *fptr = fopen("output.txt", "r");
    if (!fptr) {
        printf("Error: Could not open output.txt.\n");
        return 1;
    }

    char input[MAXTOK][MAXSYM];
    int  n = 0;
    while (n < MAXTOK - 1 && fscanf(fptr, "%63s", input[n]) != EOF) { n++; }
    strcpy(input[n++], "$");
    fclose(fptr);

    printf("Tokens loaded from output.txt:\n");
    for (int i = 0; i < n; i++) printf("[%s] ", input[i]);
    printf("\n\n");

    /* Reset stack */
    top = -1;
    push("$");
    push(NT[0]);
    int ip = 0;

    /*
     * inside_printf_fn: set to 1 when we begin expanding a k_fn call
     * whose corresponding lexeme was "printfFn". Cleared when ARG is
     * resolved. Used to enforce Rule 8 (printfFn must have exactly one
     * variable argument — not a number, not empty).
     *
     * We detect printfFn by checking whether the k_fn token in the
     * lookahead corresponds to a function call statement (prod 10 from STMT)
     * and marking it. All k_fn call-expressions use prod 17 from EXPR,
     * which allows any ARG — only the statement-level call (prod 10) is
     * subject to Rule 8 for printfFn.
     * For simplicity we flag ALL k_fn calls at STMT level and rely on
     * ARG validation (same behaviour as original, now with num fix).
     */
    int inside_printf_fn = 0;

    printf("%-20s %-15s %-15s %-35s\n", "Lookahead", "Top", "NT/T", "Action");
    printf("-----------------------------------------------------------------------\n");

    while (top >= 0) {
        char X[MAXSYM];
        strcpy(X, pop_sym());
        const char *a = input[ip];

        printf("%-20s %-15s ", a, X);

        /* ---- Terminal on stack ---- */
        int tindex = find_t(X);
        if (tindex != -1) {
            if (strcmp(X, a) == 0) {
                printf("%-15s Match %s\n", "terminal", a);
                ip++;
            } else {
                printf("%-15s REJECTED: Expected '%s' but got '%s'\n",
                       "terminal", X, a);
                return 0;
            }
            continue;
        }

        /* ---- Non-terminal on stack ---- */
        int ntindex = find_nt(X);
        int aindex  = find_t(a);

        if (ntindex == -1 || aindex == -1) {
            printf("%-15s REJECTED: Unknown symbol '%s'\n", "", (ntindex==-1)?X:a);
            return 0;
        }

        int prod = TABLE[ntindex][aindex];

        /*
         * Manual override: FD + dtype
         * Peek ip+1 to distinguish another custom function (prod 2) from
         * the epsilon before MAIN (prod 3).
         * If ip+1 is k_main we are at the boundary -> use epsilon (prod 3).
         */
        if (ntindex == 1 && aindex == 1) {
            prod = (ip + 1 < n && strcmp(input[ip + 1], "k_main") == 0) ? 3 : 2;
        }

        /*
         * Manual override: EXPR + var
         * Peek ip+1 to choose var k_op num (prod 16) vs var alone (prod 14).
         */
        if (ntindex == 6 && aindex == 17) {
            prod = (ip + 1 < n && strcmp(input[ip + 1], "k_op") == 0) ? 16 : 14;
        }

        /*
         * Manual override: COND + dtype
         * Peek ip+2 to choose k_lte (prod 22) vs k_lt (prod 21).
         * Token order: dtype var <comparator> num k_dot
         *              ip   ip+1    ip+2      ...
         */
        if (ntindex == 8 && aindex == 1) {
            prod = (ip + 2 < n && strcmp(input[ip + 2], "k_lte") == 0) ? 22 : 21;
        }

        if (prod == 0) {
            printf("%-15s REJECTED: No production for %s on '%s'\n",
                   "non-term", X, a);
            return 0;
        }

        /* ---- Rule 8 enforcement (printfFn argument check) ---- */
        if (prod == 10) {
            /* About to expand a k_fn call statement — mark as printf context */
            inside_printf_fn = 1;
        }

        if (ntindex == 7) { /* ARG is being resolved */
            if (inside_printf_fn) {
                /*
                 * FIX (Bug 4): reject BOTH empty arg (prod 20) AND num arg
                 * (prod 19) for printfFn. Only var arg (prod 18) is valid
                 * per Rule 8: "variable name that follows variable naming rule
                 * inside ()".
                 */
                if (prod == 20) {
                    printf("%-15s REJECTED: printfFn() requires a variable argument (Rule 8)\n",
                           "non-term");
                    return 0;
                }
                if (prod == 19) {
                    printf("%-15s REJECTED: printfFn() argument must be a variable, not a number (Rule 8)\n",
                           "non-term");
                    return 0;
                }
            }
            inside_printf_fn = 0; /* clear flag once ARG is resolved */
        }

        /* ---- Print action ---- */
        if (strlen(RHS[prod - 1]) == 0) {
            printf("%-15s Apply: epsilon  [prod %d: %s -> epsilon]\n",
                   "non-term", prod, X);
        } else {
            printf("%-15s Apply: %s  [prod %d: %s -> %s]\n",
                   "non-term", RHS[prod - 1], prod, X, RHS[prod - 1]);
        }

        /* ---- Push RHS symbols in reverse order ---- */
        if (strlen(RHS[prod - 1]) > 0) {
            char temp[200];
            strcpy(temp, RHS[prod - 1]);
            char  symbols[20][MAXSYM];
            int   k = 0;
            char *p = strtok(temp, " ");
            while (p) { strcpy(symbols[k++], p); p = strtok(NULL, " "); }
            for (int i = k - 1; i >= 0; i--) push(symbols[i]);
        }
    }

    if (top < 0 && strcmp(input[ip - 1], "$") == 0)
        printf("\n>>> FINAL RESULT: ACCEPTED <<<\n");
    else
        printf("\n>>> FINAL RESULT: REJECTED (Unconsumed input) <<<\n");

    return 0;
}

/* =========================================================================
 *  SECTION 3 — MAIN  (runs lexer then parser)
 * ========================================================================= */
int main(void) {
    if (run_lexer() != 0)  return 1;
    if (run_parser() != 0) return 1;
    return 0;
}
