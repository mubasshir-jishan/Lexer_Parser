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
