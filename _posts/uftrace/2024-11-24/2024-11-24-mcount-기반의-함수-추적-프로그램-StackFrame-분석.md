---
layout: post
title: "mcount 기반의 함수 추적 프로그램 StackFrame 분석"
subtitle: "mcount 기반의 함수 추적 프로그램 StackFrame 분석"
category: profiling
tags: uftrace
---

# mcount 기반의 함수 추적 프로그램 StackFrame 분석
해당 문서에서는 Child Address와 Parent Location을 구할 수 있는 이유를 분석하기 위해 아래와 같이 2가지 시도를 진행해볼 예정입니다.

- mcount() 기반의 함수 추적 코드를 적용하지 않은 예제 코드의 Stack Frame을 분석
- 동일한 예제 코드에 mcount() 기반의 함수 추적 코드를 적용한 예제 코드의 Stack Frame 분석

# mcount() 없이 빌드 된 예제 코드의 Stack Frame 분석하기

## 1. 해당 예제를 이해하기 위해 필요한 배경 지식

### 1-1. System V AMD64 psABI의 Stack Frame 레이아웃

- `ABI`는 `Application Binary Interface`의 약자로, 바이너리 형태의 프로그램이 생성되기 위한 규칙이라고 볼 수 있다.
- `Windows` 운영체제는 `Linux`와 다른 `ABI`를 사용하며, `Linux`의 경우 `64bit` 커널은 `System V AMD64 psABI`를 사용한다.
- 아래 그림에서 볼 수 있듯이 `Linux 64bit`의 경우 `Stack Frame`은 `return address`로 시작해서 밑으로 내려가는 것을 확인할 수 있다.
- 그림 URL : [https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
    ![Untitled](/assets/posts/uftrace/2024-11-24/mcount-stackframe/Untitled%2011.png)

## 2. mcount()를 사용하지 않도록 예제 코드 빌드

- 예제 코드를 아래 명령어를 사용해 빌드를 수행한다.
    - 디버깅을 위한 `-g` 옵션과 디버깅 시 편의성을 위해 컴파일러 최적화를 막아주는 `-O0`를 넣어준다.
    
    ```bash
    $ gcc -g -O0 -rdynamic -o mcount_exam2 mcount.S prof.h prof.c main.c
    ```
    

## 3. gdb 실행 및 브레이크 포인트 설정

- 아래와 같이 gdb를 실행 후 `b [브레이크 포인트를 걸 함수명]` 명령을 사용해 브레이크 포인트를 설정한다.
    
    ```bash
    $ gdb mcount_exam2
    
    GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
    Copyright (C) 2022 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.
    Type "show copying" and "show warranty" for details.
    This GDB was configured as "x86_64-linux-gnu".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    <https://www.gnu.org/software/gdb/bugs/>.
    Find the GDB manual and other documentation resources online at:
        <http://www.gnu.org/software/gdb/documentation/>.
    
    For help, type "help".
    Type "apropos word" to search for commands related to "word"...
    Reading symbols from mcount_exam2...
    
    (gdb) b main
    Breakpoint 1 at 0x12dd: file main.c, line 32.
    
    (gdb) b foo
    Breakpoint 2 at 0x1267: file main.c, line 7.
    
    (gdb) b bar
    Breakpoint 3 at 0x1281: file main.c, line 13.
    
    (gdb) 
    ```
    

## 4. 프로그램 실행

- 아래와 같이 `r` 명령을 사용해 프로그램을 실행하고, 아래와 같이 설정했던 브레이크 포인트 중 하나인 `main()` 함수에 도달하는지 확인한다. 
    ```bash
    (gdb) r
    Starting program: /home/gichoel/workdir/test2/mcount_exam2 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
    
    Breakpoint 1, main () at main.c:32
    32              char ch = 'a';
    ```
    

## 5. main() 함수의 디스어셈블 결과 확인 및 Stack Frame 분석

### 5-1. main() 함수의 디스어셈블 결과 확인

- 아래와 같이 `disassemble` 명령을 사용해 `main()` 함수의 디스어셈블 결과를 확인한다.
    
    ```bash
    (gdb) disassemble
    Dump of assembler code for function main:
       0x00005555555552d1 <+0>:     endbr64 
       0x00005555555552d5 <+4>:     push   %rbp
       0x00005555555552d6 <+5>:     mov    %rsp,%rbp
       0x00005555555552d9 <+8>:     sub    $0x10,%rsp
    => 0x00005555555552dd <+12>:    movb   $0x61,-0x1(%rbp)
       0x00005555555552e1 <+16>:    mov    $0x0,%eax
       0x00005555555552e6 <+21>:    call   0x555555555275 <bar>
       0x00005555555552eb <+26>:    mov    $0x3,%edi
       0x00005555555552f0 <+31>:    call   0x5555555552a6 <recursive>
       0x00005555555552f5 <+36>:    mov    $0x0,%eax
       0x00005555555552fa <+41>:    leave  
       0x00005555555552fb <+42>:    ret    
    End of assembler dump.
    ```
    

### 5-2. main() 함수의 Stack Frame 분석

- **`bt` 명령으로 Stack Frame 출력해보기**
    - `main()` 함수는 이전 `Stack Frame`이 존재하지 않아 `bt` 명령 실행 시 `main`의 `Stack Frame`만 출력 되는 것을 확인 할 수 있음
        
        ```bash
        (gdb) bt
        #0  main () at main.c:32
        ```
        
- **`info f` 명령으로 return address 주소 확인해보기**
    - `info f`는 `info frame` 명령을 줄인 단축어로, 현재 `Stack Frame` 정보를 조금 더 자세히 출력해주는 명령어이다.
    - 해당 명령어의 출력 결과 중 `saved rip = [주소값]` 으로 된 부분이 있는데, 해당 부분이 현재 `Stack Frame`의 `Return Address`에 해당한다.
        
        ```bash
        (gdb) info f
        Stack level 0, frame at 0x7fffffffddb0:
         rip = 0x5555555552dd in main (main.c:32); saved rip = 0x7ffff7c29d90
         source language c.
         Arglist at 0x7fffffffdda0, args: 
         Locals at 0x7fffffffdda0, Previous frame's sp is 0x7fffffffddb0
         Saved registers:
          rbp at 0x7fffffffdda0, rip at 0x7fffffffdda8
        ```
        

## 6. bar() 함수의 디스어셈블 결과 확인 및 Stack Frame 분석

### 6-1. bar() 함수의 디스어셈블 결과 확인

- 아래와 같이 `c` 명령을 사용해 다음 브레이크 포인트에 도달할 때 까지 프로그램을 실행하고, 이후 도달한 `bar()` 함수의 디스어셈블 결과를 확인한다.
    
    ```bash
    (gdb) c
    Continuing.
    
    Breakpoint 3, bar () at main.c:13
    13              int a = 3;
    
    (gdb) disassemble
    Dump of assembler code for function bar:
       0x0000555555555275 <+0>:     endbr64 
       0x0000555555555279 <+4>:     push   %rbp
       0x000055555555527a <+5>:     mov    %rsp,%rbp
       0x000055555555527d <+8>:     sub    $0x10,%rsp
    => 0x0000555555555281 <+12>:    movl   $0x3,-0xc(%rbp)
       0x0000555555555288 <+19>:    movl   $0x9,-0x8(%rbp)
       0x000055555555528f <+26>:    movl   $0x7,-0x4(%rbp)
       0x0000555555555296 <+33>:    movb   $0x62,-0xd(%rbp)
       0x000055555555529a <+37>:    mov    $0x0,%eax
       0x000055555555529f <+42>:    call   0x55555555525f <foo>
       0x00005555555552a4 <+47>:    leave  
       0x00005555555552a5 <+48>:    ret    
    End of assembler dump.
    ```
    

### 6-2. bar() 함수의 Stack Frame 분석

- **`bt` 명령으로 Stack Frame 출력해보기**
    - `bar()` 함수는 이전 main() 함수의 출력 결과와는 다르게 main() 함수의  `Stack Frame`이 존재하기 때문에 여러개가 출력 되는 것을 확인 할 수 있다.
        
        ```bash
        (gdb) bt
        #0  bar () at main.c:13
        #1  0x00005555555552eb in main () at main.c:34
        ```
        
- **`info f` 명령으로 return address 주소 확인해보기**
    - `bar()` 함수의 `saved rip =` 결과를 `main()` 함수의 디스어셈블 결과와 비교해보면 `call   0x555555555275 <bar>` 바로 다음 줄인 것을 확인할 수 있다.
    - `bar()` 함수가 `return`을 호출한 이후 `main()` 함수의 해당 부분으로 돌아갈 수 있도록 StackFrame에 저장된 것이다.
        
        ```bash
        (gdb) info f
        Stack level 0, frame at 0x7fffffffdd90:
         rip = 0x555555555281 in bar (main.c:13); **saved rip = 0x5555555552eb**
         called by frame at 0x7fffffffddb0
         source language c.
         Arglist at 0x7fffffffdd80, args: 
         Locals at 0x7fffffffdd80, Previous frame's sp is 0x7fffffffdd90
         Saved registers:
          rbp at 0x7fffffffdd80, rip at 0x7fffffffdd88
        ```
        

### 6-3. bar() 함수가 호출된 시점의 전체 Stack Frame 구조도

![Untitled](/assets/posts/uftrace/2024-11-24/mcount-stackframe/Untitled%2012.png)

## 7. foo() 함수의 디스어셈블 결과 확인 및 Stack Frame 분석

### 7-1. foo() 함수의 디스어셈블 결과 확인**

- 아래와 같이 `c` 명령을 사용해 다음 브레이크 포인트에 도달할 때 까지 프로그램을 실행하고, 이후 도달한 `bar()` 함수의 디스어셈블 결과를 확인한다.
    
    ```bash
    (gdb) c
    Continuing.
    
    Breakpoint 2, foo () at main.c:7
    7               int b = 7;
    
    (gdb) disassemble
    Dump of assembler code for function foo:
       0x000055555555525f <+0>:     endbr64 
       0x0000555555555263 <+4>:     push   %rbp
       0x0000555555555264 <+5>:     mov    %rsp,%rbp
    => 0x0000555555555267 <+8>:     movl   $0x7,-0x4(%rbp)
       0x000055555555526e <+15>:    mov    $0x1,%eax
       0x0000555555555273 <+20>:    pop    %rbp
       0x0000555555555274 <+21>:    ret    
    End of assembler dump.
    ```
    

### 7-2. foo() 함수의 Stack Frame 분석

- **`bt` 명령으로 Stack Frame 출력해보기**
    - `foo()` 함수는 이전 main() 함수의 출력 결과와는 다르게 main() 함수의  `Stack Frame`이 존재하기 때문에 여러개가 출력 되는 것을 확인 할 수 있다.
        
        ```bash
        (gdb) bt
        #0  foo () at main.c:7
        #1  0x00005555555552a4 in bar () at main.c:18
        #2  0x00005555555552eb in main () at main.c:34
        ```
        
- **`info f` 명령으로 return address 주소 확인해보기**
    - `foo()` 함수의 `saved rip =` 결과를 `bar()` 함수의 디스어셈블 결과와 비교해보면 `0x000055555555529f <+42>:    call   0x55555555525f <foo>` 바로 다음 줄인 것을 확인할 수 있다.
        
        ```bash
        (gdb) info f
        Stack level 0, frame at 0x7fffffffdd70:
         rip = 0x555555555267 in foo (main.c:7); saved rip = 0x5555555552a4
         called by frame at 0x7fffffffdd90
         source language c.
         Arglist at 0x7fffffffdd60, args: 
         Locals at 0x7fffffffdd60, Previous frame's sp is 0x7fffffffdd70
         Saved registers:
          rbp at 0x7fffffffdd60, rip at 0x7fffffffdd68
        ```
        

### 7-3. foo() 함수가 호출된 시점의 전체 Stack Frame 구조도**

![Untitled](/assets/posts/uftrace/2024-11-24/mcount-stackframe/Untitled%2013.png)

## 8. 해당 예제의 결론

- `Stack Frame`은 `Return Address → Previous Func’s Frame Pointer → local variables → …` 순으로 레이아웃이 구성된다.
- 현재 `Stack Frame`의 `Frame Pointer`는 `rbp` 레지스터의 값이다.
- 현재 `Stack Frame`의 `Stack Pointer`는  `rsp` 레지스터의 값이다.

---

# mcount() 사용되도록 빌드 된 예제 코드의 Stack Frame 분석하기

해당 부분에서는 대부분 4장의 내용과 동일하기 때문에 `bar()` 함수에서 `mcount()` 함수가 호출되는 과정에 대한 부분만 분석하도록 한다.

## 1. mcount()를 사용하도록 예제 코드 빌드

- 예제 코드를 아래 명령어를 사용해 빌드를 수행한다.
    
    ```bash
    $ gcc -g -O0 -rdynamic -o mcount_exam2 **mcount.S** prof.h prof.c main.c
    ```
    

## 2. gdb 실행 및 브레이크 포인트 설정

- 아래와 같이 gdb를 실행 후 `b [브레이크 포인트를 걸 함수명]` 명령을 사용해 브레이크 포인트를 설정한다.
    
    ```bash
    $ gdb mcount_exam2
    
    GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
    Copyright (C) 2022 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.
    Type "show copying" and "show warranty" for details.
    This GDB was configured as "x86_64-linux-gnu".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    <https://www.gnu.org/software/gdb/bugs/>.
    Find the GDB manual and other documentation resources online at:
        <http://www.gnu.org/software/gdb/documentation/>.
    
    For help, type "help".
    Type "apropos word" to search for commands related to "word"...
    Reading symbols from mcount_exam2...
    
    (gdb) b bar
    Breakpoint 1 at 0x1311: file main.c, line 13.
    
    (gdb) b mcount
    Breakpoint 2 at 0x11d9: file mcount.S, line 6.
    
    (gdb) 
    ```
    

## 3. 프로그램 실행

- 아래와 같이 `r` 명령을 사용해 프로그램을 실행하고, `main()` 함수 내부에서 호출된 `mcount()`를 빠져나가기 위해 `c` 명령을 실행한다.
- 이후, `bt` 명령을 사용해 `bar()` 함수 내부에서 `mcount()` 함수가 호출되었는지 확인한다.
    
    ```bash
    (gdb) r
    Starting program: /home/gichoel/workdir/test2/mcount_exam2 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
    
    Breakpoint 2, mcount () at mcount.S:6
    6               sub $48, %rsp
    
    (gdb) c
    Continuing.
    ((null)) : 0x7ffff7c29d90 -> (main) : 0x555555555379
    
    Breakpoint 2, mcount () at mcount.S:6
    6               sub $48, %rsp
    
    (gdb) bt
    #0  mcount () at mcount.S:6
    #1  0x0000555555555311 in bar () at main.c:12
    #2  0x0000555555555387 in main () at main.c:34
    ```
    

## 4. bar() 함수의 디스어셈블 결과 확인 및 Stack Frame 분석

### 4-1. bar() 함수의 디스어셈블 결과 확인

- 우리는 bar() 함수에도 브레이크 포인트를 걸었음에도 위에서 `bt` 명령을 확인했을 때 `bar()` 함수가 아닌 `mcount()` 내부까지 들어가진 것을 확인할 수 있다.
- 이것은 정상적인 결과로 `bar()` 함수의 어셈블리어 코드를 확인해보면 Stack Frame을 구성하는 부분의 어셈블리 명령어만 실행된 이후 바로  `mcount()`가 먼저 실행되도록 하기 때문이다.
- 이 상태에서 바로 `disassemble` 명령을 실행해도 `mcount()`에 대한 디스어셈블 결과만 나오기 때문에 우리는 아래와 같이 `up`이라는 `gdb`의 명령을 사용해 `Stack Frame`을 한 단계 위로 이동해야 한다.

    ```bash
    (gdb) disassemble
    Dump of assembler code for function bar:
    0x00005555555552ff <+0>:     endbr64 
    0x0000555555555303 <+4>:     push   %rbp
    0x0000555555555304 <+5>:     mov    %rsp,%rbp
    0x0000555555555307 <+8>:     sub    $0x10,%rsp
    0x000055555555530b <+12>:    addr32 call 0x5555555551d9 <mcount>
    => 0x0000555555555311 <+18>:    movl   $0x3,-0xc(%rbp)
    0x0000555555555318 <+25>:    movl   $0x9,-0x8(%rbp)
    0x000055555555531f <+32>:    movl   $0x7,-0x4(%rbp)
    0x0000555555555326 <+39>:    movb   $0x62,-0xd(%rbp)
    0x000055555555532a <+43>:    mov    $0x0,%eax
    0x000055555555532f <+48>:    call   0x5555555552df <foo>
    0x0000555555555334 <+53>:    leave  
    0x0000555555555335 <+54>:    ret    
    End of assembler dump.
    ```

### 4-2. mcount() 함수의 디스어셈블 결과 확인
    ```bash
    (gdb) disassemble
    Dump of assembler code for function mcount:
    => 0x00005555555551d9 <+0>:     sub    $0x30,%rsp
    0x00005555555551dd <+4>:     mov    %rdi,0x28(%rsp)
    0x00005555555551e2 <+9>:     mov    %rsi,0x20(%rsp)
    0x00005555555551e7 <+14>:    mov    %rdx,0x18(%rsp)
    0x00005555555551ec <+19>:    mov    %rcx,0x10(%rsp)
    0x00005555555551f1 <+24>:    mov    %r8,0x8(%rsp)
    0x00005555555551f6 <+29>:    mov    %r9,(%rsp)
    0x00005555555551fa <+33>:    mov    0x30(%rsp),%rdi
    0x00005555555551ff <+38>:    mov    0x8(%rbp),%rsi
    0x0000555555555203 <+42>:    push   %rax
    0x0000555555555204 <+43>:    call   0x55555555522c <print_instrument_func_info>
    0x0000555555555209 <+48>:    pop    %rax
    0x000055555555520a <+49>:    mov    (%rsp),%r9
    0x000055555555520e <+53>:    mov    0x8(%rsp),%r8
    0x0000555555555213 <+58>:    mov    0x10(%rsp),%rcx
    0x0000555555555218 <+63>:    mov    0x18(%rsp),%rdx
    0x000055555555521d <+68>:    mov    0x20(%rsp),%rsi
    0x0000555555555222 <+73>:    mov    0x28(%rsp),%rdi
    0x0000555555555227 <+78>:    add    $0x30,%rsp
    0x000055555555522b <+82>:    ret    
    End of assembler dump.
    ```

### 4-3. mcount() 함수의 Stack Frame 분석

- **`bt` 명령으로 Stack Frame 출력해보기**
    
    ```bash
    (gdb) bt
    #0  mcount () at mcount.S:6
    #1  0x0000555555555311 in bar () at main.c:12
    #2  0x0000555555555387 in main () at main.c:34
    ```
    

- **`info f` 명령으로 return address 주소 확인해보기**
    - `bar()` 함수의 디스어셈블 결과를 확인해보면 Return Address는 `0x000055555555530b <+12>:    addr32 call 0x5555555551d9 <mcount>` 바로 다음 줄인 것을 확인할 수 있다.
        
        ```bash
        (gdb) info f
        Stack level 0, frame at 0x7fffffffdd70:
         rip = 0x5555555551d9 in mcount (mcount.S:6); saved rip = 0x555555555311
         called by frame at 0x7fffffffdd90
         source language asm.
         Arglist at 0x7fffffffdd60, args: 
         Locals at 0x7fffffffdd60, Previous frame's sp is 0x7fffffffdd70
         Saved registers:
          rip at 0x7fffffffdd68
        ```
        

### 4-4. bar() 함수 내부에서 mcount() 함수가 호출된 시점의 전체 Stack Frame 구조도

- C언어로 구현된 함수의 경우 자동으로 Stack Frame 레이아웃에 맞게 구성되도록 컴파일러가 처리하도록 코드를 삽입하지만, mcount() 함수의 경우 어셈블리어로 구현되었기 때문에 직접 Calling Convention을 지키도록 함수를 구현해야 한다.
    - **//TODO: 어셈블리어는 직접 Calling Convention을 지키도록 함수를 구현해야 한다는 내용을 어디서 본 것 같은데 링크를 찾지 못해서 나중에  찾아서 넣기**
- 아래 그림을 이해하기 위해서는 `bar()` 함수와 `mcount()` 함수의 디스어셈블 결과를 참고해야 하며, 확인할 부분은 아래와 같다.
    - `mcount()`로 넘어갔음에도 `rbp`레지스터가 가리키는 주소가 `bar()` 함수인 이유는 `mcount()`에서 `rbp` 레지스터 값을 스택에 `push`하고 `rsp` 값을 `rbp` 레지스터에 넣는 과정이 없기 때문인 것으로 보임
    - `Stack Frame`에 `rbp` 레지스터를 `push` 하는 부분이 없기 때문에 `mcount()` 함수의 `rsp + 48` 지점은 `Return Address`에 해당한다.

![Untitled](/assets/posts/uftrace/2024-11-24/mcount-stackframe/Untitled%2014.png)

- **메인으로 참조한 URL**
    - Ref URL 1 : [https://github.com/namhyung/uftrace/blob/master/arch/x86_64/mcount.S](https://github.com/namhyung/uftrace/blob/master/arch/x86_64/mcount.S)
    - Ref URL 2 : [https://www.bhral.com/post/linux-kernel-ftrace-간단한-원리](https://www.bhral.com/post/linux-kernel-ftrace-%EA%B0%84%EB%8B%A8%ED%95%9C-%EC%9B%90%EB%A6%AC)
- **부가적으로 참조한 URL**
    - Sub Ref URL 1 : [https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)
    - Sub Ref URL 2 : [https://unix.stackexchange.com/questions/466389/where-are-the-kernel-stack-frames-for-c-run-time-startup-functions-and-fr](https://unix.stackexchange.com/questions/466389/where-are-the-kernel-stack-frames-for-c-run-time-startup-functions-and-fr)
    - Sub Ref URL 3 : [https://stackoverflow.com/questions/72649142/difference-between-amd64-and-intel-x86-64-stack-frame](https://stackoverflow.com/questions/72649142/difference-between-amd64-and-intel-x86-64-stack-frame)
    - Sub Ref URL 4 : [https://velog.io/@msung99/시스템-프로그래밍-4-2주차-4ngkpnns](https://velog.io/@msung99/%EC%8B%9C%EC%8A%A4%ED%85%9C-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-4-2%EC%A3%BC%EC%B0%A8-4ngkpnns)
    - Sub Ref URL 5 : [https://it-eldorado.tistory.com/37](https://it-eldorado.tistory.com/37)
    - Sub Ref URL 6 : [https://koharinn.tistory.com/224](https://koharinn.tistory.com/224)
