---
layout: post
title: "VSCode로 uftrace 디버깅하기"
subtitle: "vscode로 uftrace 디버깅하기"
category: profiling
tags: uftrace
---
## VSCode로 uftrace 디버깅하기
2024 오픈소스 컨트리뷰션 아카데미의 uftrace팀 리드 멘토로 활동하며 우리 팀에 참여하신 멘티 분들을 위해
정리했던 문서 중 하나로, 리눅스의 콘솔 화면에서 gdb를 사용한 디버깅 방법에 대해 익숙하지 않은 분들을 위해
작성한 문서입니다.

윈도우 환경에 설치된 VSCode에 몇 가지 확장 프로그램을 설치하면 원격으로 리눅스 환경에서 동작하는 프로그램을
디버깅할 수 있으며, 이 기능을 활용해 리눅스에서 동작하는 uftrace 프로그램을 디버깅하기 위한 환경을 구축한
문서입니다.

## 1. 원격 디버깅에 필요한 VSCode Extension 설치하기

### 1-1. Remote Development Extension 설치
- 화면 왼쪽에서 `Exensions` (윈도우 기준 단축키는 Ctrl+Shift+X) 클릭
- `remote development` 입력 후 엔터
- 검색 결과 중 `Remote Development`의 `Install` 버튼 클릭 

  ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled.png)

### 1-2. C/C++ Extension Pack 설치

- 화면 왼쪽에서 `Exensions` (윈도우 기준 단축키는 Ctrl+Shift+X) 클릭
- `c/c++` 입력 후 엔터
- 검색 결과 중 `C/C++ Extension Pack` 의 `Install` 버튼 클릭 

  ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%201.png)

## 2. VSCode로 원격 리눅스 환경에 접속

### 2-1. 원격 리눅스 환경에 접속하기 위한 SSH 설정하기
  - `Ctrl+Shift+P` 키를 눌러 창을 하나 띄우고, `ssh`를 입력해 검색해 `Remote-SSH: Add New SSH Host…`를 선택합니다.
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/48a55afc-e3d4-4e33-a8c3-0f5b9ec8e0c8.png)

  - 아래와 같이 `[계정명]@[원격 리눅스 IP]` 형식으로 입력한 뒤 엔터
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/806344e5-e4f6-4acb-b3ff-b60d3cc1fb50.png)

  - 해당 부분에서는 어떤 항목을 눌러도 상관없으며, 여기서는 상단에 있는 항목을 선택합니다.
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%202.png)

### 2-2. 원격 리눅스 환경에 접속하기
  - `Ctrl+Shift+P` 키를 눌러 창을 하나 띄우고, `ssh`를 입력해 검색해 `Remote-SSH: Connect Current Window to Host…`를 선택합니다.
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%203.png)

  - 위에서 추가했던 원격 리눅스 환경의 IP를 선택합니다.
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%204.png)

  - 원격 리눅스 환경의 계정 비밀번호를 입력합니다. (비밀번호를 입력하는 과정은 여러번 반복될 수 있음)
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%205.png)


# 3. 원격 gdb로 uftrace를 디버깅 하기 위한 lauhch.json 생성
## 3-1. lauhch.json 생성하는 방법
  - VSCode 화면 왼쪽의 디버깅 아이콘 클릭
  - `create a launch.json file` 링크 클릭
![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%206.png)

## 3-2. launch.json에 아래 내용 추가
  - 아래 json 내용을 launch.json에 덮어씁니다.
    ```json
    {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [{
            "name" : "uftrace debugging",
            "type" : "cppdbg",
            "request" : "launch",
            "program" : "${workspaceRoot}/uftrace",
            "args" : [],
            "stopAtEntry" : true,
            "cwd" : "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [{
                "description": "gdb pretty print",
                "text" : "-enable-pretty-printing",
                "ignoreFailures": true
            }]
        }]
    }
    ```

## 3-3. launch.json의 몇몇 인자 소개
- **"args"**
    - `"args"` 는 프로그램 실행 시 입력하는 인자를 넣기 위해 사용하는 옵션으로, 인자를 여러개 입력할 수 있도록 리스트 형식으로 구성되어 있습니다.
    - 해당 옵션은 `uftrace` 실행 시 입력할 수 있는 Command 목록인 `record, live, …` 등을 입력하는데 사용할 수 있습니다.

- **"stopAtEntry"**
    - `“stopAtEntry”`는 브레이크 포인트를 걸지 않고 시작해도 무조건 프로그램의 시작 위치인 메인 함수에 멈추도록 하는 옵션으로, 브레이크 포인트 없이 시작할 때 유용하게 사용할 수 있습니다.

# 4. 디버깅 시작하기
## 4-1. 브레이크 포인트 없이 uftrace 디버깅 시작해보기

- `“stopAtEntry”`를 `true`로 설정하고 키보드의 `F5` 를 눌러 디버깅을 시작하면 아래 화면과 같이 `uftrace`의 `main()` 함수 첫 부분에서 멈추는 것을 확인할 수 있습니다.  

    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%207.png)

- 화면 상단의 디버깅을 위한 버튼을 눌러도 되고, 아래의 단축키를 사용해 디버깅을 진행할 수 있다.
    - **Continue (F5) :** 브레이크 포인트를 만날 때 까지 프로그램 실행
    - **Step Over (F10) :** 함수를 만나면 함수로 들어가지 않고 한 줄씩 따라가기
    - **Step Into (F11) :** 함수를 만나면 함수로 들어가 한 줄씩 따라가기
    - **Step Out (Shift + F11) :** 현재 함수를 빠져나가기
    - **Restart (Ctrl + Shift + F5) :** 디버깅하고 있는 프로그램을 재실행
    - **Stop (Shift + F5) :** 디버깅하고 있는 프로그램 종료  



- 아래와 그림과 같이 변수 명 위에 마우스 커서를 올리거나 VSCode 화면의 왼쪽에 있는 `VARIABLES`에서 현재 변수에 들어가 있는 값을 확인할 수 있습니다.
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%208.png)

- 현재는 디버깅을 실행해도 `uftrace`의 command에 해당하는 인자를 주지 않은 상태이기 때문에 몇 줄 따라가다 보면 프로그램이 바로 종료됩니다.

## 4-2. uftrace의 특정 command 디버깅 해보기

### 4-2-1. uftrace로 추적에 사용할 타겟 프로그램 빌드하기

- 터미널 창에서 `uftrace`의 소스 코드가 위치한 곳으로 이동 후 아래 명령을 실행합니다.

  ```bash
  $ gcc -pg -g -o t-abc tests/s-abc.c
    
  $ ls -l t-abc
  -rwxrwxr-x 1 gichoel gichoel 18160  8월  6 02:57 t-abc
  ```

- `-pg` 옵션은 `uftrace`의 `mcount()`를 사용한 추적 기능을 활용하기 위함이고, `-g` 옵션은 우리가 사용할 디버거가 디버깅 정보를 활용하기 위함입니다.

### 4-2-2. uftrace로 타겟 프로그램을 추적하기 위해 launch.json 수정하기

- 여기서는 `uftrace`의 여러 명령 중 `live` 명령을 사용해 타겟 프로그램을 추적하는 과정을 디버깅 할 예정입니다.
- 위에서 빌드한 타겟 프로그램의 경우 `t-abc`라는 실행 파일 명으로 빌드하였고, 이를 `uftrace`의 `live` 명령으로 추적하기 위해 실행하는 명령어는 아래와 같습니다.

  ```bash
  $ uftrace live t-abc
  ```


- 위 명령에서 `live t-abc` 부분을 `“args”` 에 넣으면 되고, 전체 `launch.json`의 내용은 아래와 같습니다.
    - 만약, `record` 명령을 사용해 추적을 진행하고 한다면 `“live”`를 `“record”`로 변경하면 됩니다.

  ```json
  {
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [{
        "name" : "uftrace debugging",
        "type" : "cppdbg",
        "request" : "launch",
        "program" : "${workspaceRoot}/uftrace",
        **"args" : ["live", "t-abc"],**
        "stopAtEntry" : true,
        "cwd" : "${workspaceFolder}",
        "environment": [],
        "externalConsole": false,
        "MIMode": "gdb",
        "miDebuggerPath": "/usr/bin/gdb",
        "setupCommands": [{
            "description": "gdb pretty print",
            "text" : "-enable-pretty-printing",
                "ignoreFailures": true
            }]
        }]
    }
  ```

### 4-2-3. uftrace 디버깅 실행해보기

- 위에서 설정한 `launch.json`에 의해 이번에는 한 줄씩 따라가다 보면 이번에는 `cmds/live.c` 파일의 `command_live()` 함수로 따라가는 것을 확인할 수 있습니다.
- 한 줄씩 쭉 따라가다 보면 몇 줄 안따라갔는데 생각보다 일찍 `uftrace`가 종료되는 것을 확인할 수 있고, 이는 정상적인 현상이며 아래에서 별도로 설명하도록 합니다.

  ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%209.png)


### 4-2-4. 왜 uftrace가 생각보다 일찍 끝나죠?
- 위에서 디버깅을 해보면 uftrace의 코드 규모보다 너무 일찍 끝나는 것을 확인할 수 있고, 그에 대한 이유는 아래와 같습니다.
- 중간에 따라가며 확인했을 수 있겠지만, `uftrace`는 내부적으로 `fork()` 함수를 실행해 자식 프로세스를 실행하는 것을 확인할 수 있고 **gdb는 별도의 설정이 없는 경우 fork()로 생성한 자식 프로세스의 코드를 따라가지 않습니다**.
- gdb가 `fork()`로 생성한 자식 프로세스의 코드를 따라가도록 하려면 `set follow-fork-mode child` 라는 별도의 설정이 필요하며, `VSCode`에서는 아래와 같이 `launch.json`의 `setupCommand`에 항목을 추가해 해당 옵션을 설정해 줄 수 있습니다.
    
    ```json
    {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [{
            "name" : "uftrace debugging",
            "type" : "cppdbg",
            "request" : "launch",
            "program" : "${workspaceRoot}/uftrace",
            "args" : ["record", "t-abc"],
            "stopAtEntry" : true,
            "cwd" : "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [{
                "description": "gdb pretty print",
                "text" : "-enable-pretty-printing",
                "ignoreFailures": true
            },
            **{   "description": "The new process is debugged after a fork. The parent process runs unimpeded.",
                "text": "-gdb-set follow-fork-mode child",
                "ignoreFailures": true
            }**]
        }]
    }
    ``` 

### 4-2-5. mcount_init() 함수부터 디버깅해서 따라가보기

- `libmcount`의 시작 지점이 되는 `mcount_init()` 함수부터 따라가기 위해서 `mcount_init()` 내부의 `mcount_startup()` 함수에 브레이크 포인트를 설정합니다.

    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%2010.png)


- 이후, `F5`키를 눌러 디버거를 시작하고 따라가면 아래와 같이 설정한 브레이크 포인트에 걸리는 것을 확인할 수 있고 원하는대로 `libmcount` 코드를 따라가며 디버깅을 수행할 수 있습니다.
    ![Untitled](/assets/posts/uftrace/2024-11-23/vscode_debug/Untitled%2011.png)


# 5. QnA

## 5-1. libmcount 디렉토리의 코드 수정 후 브레이크 포인트가 정상 동작하지 않는 경우

- libmcount 디렉토리 내부의 소스코드를 수정 후 디버거가 브레이크 포인트가 아닌 다른 위치에서 멈추는 경우 아래 해결책을 사용하면 된다.
- 해당 이슈는 `make install` 명령을 사용하여 이미 시스템에 uftrace를 설치한 경우 발생하며, 디버거가 실행하며 참조하는 소스 코드는 수정한 최신 버전이나 `libmcount.so` 파일을 이미 시스템에 설치된 파일을 바라보기 때문에 발생한 현상입니다.

- 아래와 같이 `args`에 `"--libmcount-path=."` 을 추가해 현재 빌드한 소스코드의 디렉토리 경로의 `libmcount`를 사용하도록 설정합니다.
    
    ```json
    {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [{
            "name" : "uftrace debugging",
            "type" : "cppdbg",
            "request" : "launch",
            "program" : "${workspaceRoot}/uftrace",
            **"args" : ["live", "--libmcount-path=.", "t-abc"],**
            "stopAtEntry" : true,
            "cwd" : "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "setupCommands": [{
                "description": "gdb pretty print",
                "text" : "-enable-pretty-printing",
                "ignoreFailures": true
            }]
        }]
    }
    ```
