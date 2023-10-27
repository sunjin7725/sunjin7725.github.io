---
title: Go 언어란?
categories: [Go]
tags: [Go, Language]
comments : true
---

# Go 언어란?

![GoLang](/assets/img/post/golang.png)  

Go Lang은 2007년 구글에서 개발한 오픈소스 프로그래밍 언어로, 간결하고 효율적으로 신뢰성있는 소프트웨어를 개발하기
위해 개발되었다.

Go Lang은 다음과 같은 특징을 가진다.

- 간결한 문법
- 빠른 컴파일 속도
- 동시성 제어
- 가비지 컬렉션

## 간결한 문법

Go Lang은 타 언어들에 간결한 문법을 가지고 있다고 한다. 당연히 맨처음 모든 언어를 배우다 보면 마찬가지로 해맬 수는
있지만, 기존 C, Java, Python 등과 같은 프로그래밍 언어를 다루어 보았던 사람이라면, 한번에 이해할 수 있는 문법을
가지고 있다.

실제로, 필자가 공부해보고 간단한 API를 만들어 보았을 때 느낌점은, 많은 언어가 그렇듯 개발 이전에 나온 언어들의
사용하기 좋으면서, 명료한 문법들을 모아 놓은 느낌이였다. 또한 문법 Keyword가 25가지 밖에 없다고 한다.

## 빠른 컴파일 속도

Go Lang은 Python과 JS와는 달리, C와 Java와 같은 컴파일 언어이다. Java는 직접 경험해보진 않았지만, C를 할 때의
컴파일 속도를 본다면 굉장히 답답하다고 느낄 때가 많다(학부에서 배우는 과정에서 조차).

하지만, Go Lang은 컴파일 속도가 굉장히 빠르다. 그 이유는 C처럼 모든 사용되지 않는 헤더들을 일일이 찾는 컴파일 방식이 아니라,
컴파일 대상이 사용하는 패키지 목록만을 찾고 컴파일 해주기 때문에 굉장히 빠르다. 이를, Go에서는 간결한 의존성 해석 알고리즘이라고도
표현하는 듯 하다.

## 동시성 제어

고루틴(Goroutine)은 Golang이 사용하는 경량 쓰레드를 의미한다. Golang에서 실행되는 모든 프로그램은 고루틴에서 실행된다.
즉, 메인 함수도 고루틴에서 실행이 된다.

다시 말하면, Golang의 모든 프로그램은 반드시 하나 이상의 고루틴을 가지고 있음을 의미합니다.

Golang에서는 go 키워드를 사용하여 함수를 호출하여 고루틴을 실행할 수 있다.

### 예제

```go
package main

import (
    "fmt"
    "time"
)

func PrintAlphabet() {
    alphabet := "abcdefghijklmnopqrstuvwxyz"

    for _, v := range alphabet {
        time.Sleep(200 * time.Millisecond)
        fmt.Printf("%c", v)
    }
}

func PrintNumbers() {
    for i := 1; i <= 10; i++ {
        time.Sleep(200 * time.Millisecond)
        fmt.Printf("%d", i)
    }
}

func main() {
    PrintAlphabet()
    fmt.Println("")

    PrintNumbers()
    fmt.Println("")

    go PrintAlphabet()
    go PrintNumbers()
    fmt.Println("")

time.Sleep(3 * time.Second)
}
```

위의 코드는 고루틴을 활용하는 예시를 보여준다. go 키워드를 사용하지 않은 두 개의 함수는 동기식 방식으로 실행결과가 출력되고,
go 키워드를 사용하여 호출한 아래쪽 함수들은 비동기식으로 두개의 함수의 출력값들이 번갈아가면서 출력된다.

따라서, 아래의 두개의 함수 출력값들은 실행결과 마다 다를 수 있다. 아래 실행결과를 참고하자.

### 실행결과

```bash
abcdefghijklmnopqrstuvwxyz
12345678910

1ab23c4de56fg78hi910jklmn
```

## 가비지 컬렉션

어느 언어에서나, 최근 차용하고 있는 가비지 컬렉션은 우리가 개발을 함에 있어 메모리 관리에 신경을 쓰지 않고도,
개발언어 자체에서 메모리를 관리해주는 작업을 의미한다.

Java와 최근 개발되고 있는 언어들은 모두 해당 작업을 차용하고 있는데, 그 이유를 보면 C나 C++의 경우 가비지 컬렉터가
존재하지 않는다. 그래서 개발을 함에 있어, 동적 메모리 영역을 할당하고 사용 이후에는 항상 해제해주는 코드가 필요하다.

```c
void main(){
    int array_size = 300;
    int* int_array = (int*)malloc(sizeof(int)*size);
    
    for(int i = 0; i < array_size; i++){
        int_array[i] = i;
    }
    free(int_array);
}
```

C에서는  위의 코드처럼 동적으로 메모리를 할당해주고 꼭!!! free라는 함수를 사용하여 해당 메모리를 해제해줘야한다.

하지만, Java나 Python, 또 Go에서는 가비지 컬렉터가 들어가있어 일정시간 이후에, 더 이상 프로그램에서 사용되지 않는
메모리는 자동으로 취합하여 해제해주는 작업을 진행한다. 더이상 free를 해주지 않아 메모리가 누수되는 일은 없는것이다.

**P.S 하지만, 가비지 컬렉션도 만능은 아니다. 일부 라이브러리나, 함수에서는 동작하지 않는 경우도 프로젝트를 하다보면
생기기 마련이다.
