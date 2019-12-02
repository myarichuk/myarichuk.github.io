---
title: Understandable errors in ANTLR4
date: 2019-11-21 16:47:39
tags:
  - ANTLR
  - C#
  - Parsers
categories:
  - Programming
  - Parsers
author: Michael Yarichuk
top_img: top.jpg
cover: /2019/11/21/antlr-error-handling/antlr.jpg
---
### There is more than one way to peel an orange!
Once a colleague told me: "you can't really generate user-friendly error messages with ANTLR. This didn't seem right - serious parser generators must have ways to generate proper errors...  
Online searching has shown approaches to error handling mostly revolve around either various implementations of [ANTLRErrorStrategy](https://www.antlr.org/api/Java/org/antlr/v4/runtime/ANTLRErrorStrategy.html) or "fail fast" strategy that involves overriding implementation of [DefaultErrorStrategy](https://www.antlr.org/api/Java/org/antlr/v4/runtime/DefaultErrorStrategy.html) to throw **ParseCancellationException**, which would cause parsing to stop at the first syntax error.  
Those approaches were nice, but I wanted to find a way that would allow me to control both error messages and the "offending token" - syntax token to be highlighted in UI when showing syntax errors.  
  
Consider the following ANTLR grammar:
``` antlr
grammar TestStrings;

testExpression: stringExpression EOF;
stringExpression: QUOTE CHAR_SEQUENCE QUOTE;

QUOTE: '"';
CHAR_SEQUENCE: (~('"'| '\\'))*;

SPACES: [ \u000B\t\r\n] -> channel(HIDDEN);
```
This combined grammar parses character sequence and detects C-style quoted strings. Now, in order to add custom errors in a declarative way, we will add a custom error to be thrown if the string is missing a closing quote. 
``` antlr
grammar TestStrings;

testExpression: stringExpression EOF;
stringExpression:       
        QUOTE CHAR_SEQUENCE QUOTE #String
    |   QUOTE CHAR_SEQUENCE { NotifyErrorListeners(_input.Lt(-1), "Missing a '\"' at the end of the string!", null); } #UnclosedString  
    ;


QUOTE: '"';
CHAR_SEQUENCE: (~('"'| '\\'))*;

SPACES: [ \u000B\t\r\n] -> channel(HIDDEN);
```
{% note info %}
Note how **_input.Lt(-1)** is used to specify which token in the lexer stream is the "problematic" one, by using offset from current position in the stream. 
{% endnote %}
Coupled with the following simple error listener, specifying errors like this provided me with what I wanted.

``` cs
public readonly struct SyntaxError
{
    public readonly IRecognizer Recognizer;
    public readonly IToken OffendingSymbol;
    public readonly int Line;
    public readonly int CharPositionInLine;
    public readonly string Message;
    public readonly RecognitionException Exception;

    public SyntaxError(IRecognizer recognizer, IToken offendingSymbol, int line, 
                       int charPositionInLine, string message, RecognitionException exception)
    {
        Recognizer = recognizer;
        OffendingSymbol = offendingSymbol;
        Line = line;
        CharPositionInLine = charPositionInLine;
        Message = message;
        Exception = exception;
    }
}

public class SyntaxErrorListener : BaseErrorListener
{
    public readonly List<SyntaxError> Errors = new List<SyntaxError>();
    public override void SyntaxError(IRecognizer recognizer, IToken offendingSymbol, int line, int charPositionInLine, string msg,
        RecognitionException e)
    {
        Errors.Add(new SyntaxError(recognizer, offendingSymbol, line, charPositionInLine, msg, e));
    }
}
```

As I am not an expert on ANTLR, if you think this can be done in a better way or you think this is not a good way of handling errors in ANTLR, do let me know!<br/>There is always a place for improvement.