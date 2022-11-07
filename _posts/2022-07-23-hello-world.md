---
title: Hello World
author: Equinox134
date: 2022-07-23
categories: [Random]
tags: [Test,Writing]
excerpt: This is the first official post of this blog.
excerpt_separator: <!--more-->
---

<!--more-->
### Introduction

So I created blog using GitHub Pages. My plan is to use this blog as a place to write about topics that I studied/learned about, and other random stuff.
Most of the time I'll probably post stuff related to computer science and math, but I can't be sure.

Here are a few things I want to mention:
* I wasn't able to enable comments, but I am trying.
* English is not my native language, and there could be multiple grammatical errors. Please ignore for the tume being.

Currently I don't have anything interesting to post about, but at the same time want to write something, so I'll just show Hello World written in multiple differnt languages, in no particular order (I'll keep adding more if I feel like it).

Current Language Count: 50

### C
```c
#include <stdio.h>

int main(){
  printf("Hello World");
  return 0;
}
```

### C++
```cpp
#include <iostream>

int main(){
  std::cout << "Hello World";
  return 0;
}
```

### C#
```csharp
namespace HelloWorld
{
  class HW{
    static void Main(string[] args){
      System.console.WriteLine("Hello World");
    }
  }
}
```

### Python
```python
print("Hello World")
```

### Java
```java
class HelloWorld{
  public static void main(String[] args){
    System.out.println("Hello World");
  }
}
```

### JavaScript
```javascript
console.log("Hello World");
```

### Pascal
```pascal
program Hello;
begin
  writeln('Hello World');
  readln;
end.
```

### Ruby
```ruby
puts 'Hello World'
```

### Kotlin
```kotlin
fun main(args: Array<String>){
  println("Hello World");
}
```

### Haskell
```haskell
module Main where

main = putStrLn "Hello World"
```

### Swift
```swift
println("Hello World");
```

### Node.js
```javascript
var http = require('http');

http.createServer(function (req,res){
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('Hello World');
}).listen(8080);
```

### Text
```
Hello World
```

### Go
```go
package main

import "fmt"

func main(){
  fmt.Println("Hello World")
}
```

### D
```d
import std.stdio;

void main(){
  writeln("Hello World");
}
```

### PHP
```php
echo "Hello World";
```

### Rust
```rust
fn main(){
  println!("Hello World");
}
```

### Scala
```scala
object HelloWorld exends App{
  printIn("Hello World")
}
```

### Lua
```lua
print("Hello World")
```

### Perl
```perl
print("Hello World");
```

### F#
```fsharp
printfn "Hello World"
```

### Visual Basic
```
Import System

Module Module1
  Sub Main()
    Console.WriteLine("Hello World")
    Console.Readline()
  End Sub
End Module
```

### Objective-C
```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]){
    NSLog(@"Hello World");
}
```

### Objective-C++
```objc++
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]){
    NSLog(@"Hello World");
}
```

### Golfscript
```
'Hello World'
```

### TypeScript
```javascript
console.log('Hello World');
```

### Assembly
```
global _start

section .text

_start:
  mov rax, 1      ; write(
  mov rdi, 1      ;   STDOUT_FILENO,
  mov rsi, msg    ;   "Hello World",
  mov rdx, msglen ;   sizeof("Hello World")
  syscall         ; );
  
  mov rax, 60     ; exit(
  mov rdi, 0      ;   EXIT_SUCCESS
  syscall         ; );

sectio .rodata
  msg: db "Hello World", 10
  msglen: equ $ - msg
```

### Bash
```
echo "Hello World"
```

### Fortran
```
program helloworld
  print *, "Hello World"
end program helloworld
```

### Scheme
```scheme
(begin
  (display "Hello World")
  (newline))
```

### Ada
```
with Ada.Text_IO;

procedure Hello_World is
begin
  Ada.Text_IO.Put_Line ("Hello World");
end Hello_World;
```

### awk
```
awk 'BEGIN {print "Hello World"}'
```

### OCaml
```ocaml
print_string "Hello World"
```

### Brainfuck
```
++++++++++[>+++++++>++++++++++>+++<<<-]>++.>+.+++++++
..+++.>++.<<+++++++++++++++.>.+++.------.--------.>+.
```

### Whitespace

Space, Tab, Linefeed character is shown with the identifying comment "S","T", or "L" respectively.

```
S S S T	S S T S S S L
T	L
S S S S S T	T	S S T	S T	L
T	L
S S S S S T	T	S T	T	S S L
T	L
S S S S S T	T	S T	T	S S L
T	L
S S S S S T	T	S T	T	T	T	L
T	L
S S S S S T	S T	T	S S L
T	L
S S S S S T	S S S S S L
T	L
S S S S S T	T	T	S T	T	T	L
T	L
S S S S S T	T	S T	T	T	T	L
T	L
S S S S S T	T	T	S S T	S L
T	L
S S S S S T	T	S T	T	S S L
T	L
S S S S S T	T	S S T	S S L
T	L
S S S S S T	S S S S T	L
T	L
S S L
L
L
```

Without comments, the code above would look something like this:

```
           
  
              
  
              
  
              
  
                
  
            
  
           
  
                
  
                
  
               
  
              
  
              
  
            
  
  


```

### R
```R
print("Hello World", quote=FALSE)
```

### Tcl
```
puts "Hello World"
```

### Rhino
```javascript
print("Hello World")
```

### Cobol
```
IDENTIFICATION DIVISION.
PROGRAM-ID. Hello-world.
PROCEDURE DIVISION.
  DISPLAY "Hello World".
```

### Pike
```
int main(){
  write("Hello World");
  return 0;
}
```

### Sed
```
echo "Hello sed" | sed 's/sed/World/'
```

### INTERCAL
```
DO ,1 <- #13
PLEASE DO ,1 SUB #1 <- #238
DO ,1 SUB #2 <- #108
DO ,1 SUB #3 <- #112
DO ,1 SUB #4 <- #0
DO ,1 SUB #5 <- #64
DO ,1 SUB #6 <- #194
DO ,1 SUB #7 <- #48
PLEASE DO ,1 SUB #8 <- #22
DO ,1 SUB #9 <- #248
DO ,1 SUB #10 <- #168
DO ,1 SUB #11 <- #24
DO ,1 SUB #12 <- #16
DO ,1 SUB #13 <- #162
PLEASE READ OUT 1,
PLEASE GIVE UP
```

### bc
```
print "Hello World";
```

### Algol
```
BEGIN DISPLAY("Hello World") END.
```

### Befunge
```
"dlroW olleH"> , v
             | : <
             @
```

### FreeBASIC
```
Option Explicit
Cls
Print "Hello World"
Sleep
End
```

### Haxe
```haxe
class Text{
  static function main(){
    trace("Hello World");
  }
}
```

### LOLCODE
```
HAI 1.2
CAN HAS STDIO?
VISIBLE "HAI WORLD!"
KTHXBYE
```

### SystemVerilog
```verilog
module test;
  initial begin
    $display("Hello World");
  end
endmodule
```

### Piet
![Piet](https://github.com/Equinox134/equinox134.github.io/blob/master/assets/img/2022-07-23-hello-world/piet%20hello%20world.png)

### Notes and Final Thoughts
* There are a lot of programming languages in this world
* LOLCODE should be more well known and people should use it more often
* Never write in whitespace ever again
