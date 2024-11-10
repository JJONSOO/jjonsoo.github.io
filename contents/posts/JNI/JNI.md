---
title: "JNI 사용하기"
description:
date: 2024-07-21
update: 2021-07-12
tags:
  - Java
  - JNI
---
JNI(Java Native Interface)는 자바와 Native Code(C,C++)를 연결하는 데 사용되는 프레임워크이다.
JNI를 통해 자바 Application에서 Natice Library를 호출하거나, Native Application에서 Java Method를 호출 할 수 있다.

이를 통해, 성능 향상, 기존 Native Library 재사용을 할 수 있다.

## 자바에서 사용하는 방법

### Native method

자바는 Method 구현이 native code에서 제공될 것임을 나타내는데 사용되는 native keyword를 제공한다.

공유 라이브러리 : Java 코드 내에서 라이브러리에 대한 참조만 있다. 실행 파일을 실행하는 환경이 프로그램에서 사용하는 libs의 모든 파일에 액세스 할 수 있어야 한다. 

공유 라이브러리는 클래스의 일부가 아닌 .so .ddl .dylib 파일 내에 Native Code를 별도로 보관한다.

### 필요 구성요소

Java Code

- Native Keyword : Native로 표시된 모든 method는 native 공유 라이브러리에서 구현되어야 한다.
- System.loadLibary(String libname) - 공유 라이브러리를 파일 시스템에서 메모리로 로드하고 내보낸 함수를 Java 코드에서 사요할 수 있도록 하는 정적 method

### C/C++ Code

- JNIEXPORT - 함수를 공유 라이브러리에 내보내기 기능으로 표시하여 함수 테이블에 포함되므로 JNI가 찾을 수 있다.
- JNICALL - JNIEXPORT와 결합 하여 JNI 프레임워크에서 method를 사용할 수 있도록 한다.
- JNIEnv - Native Code를 사용하여 Java 요소에 액세스 할 수 있는 method가 포함된 구조
- JavaVM - 실행 중인 JVM을 조작할 수 있는 구조에 Thread를 run/interrupt

이제 코드로 작성해보자

## “Hello World” 출력해보기

### 1. Java Code

```java
public class HelloJNI {
    static {
        System.loadLibrary("JNI_Hello"); // 네이티브 라이브러리를 로드
    }
    
    public native void printHello(); // 네이티브 메서드 선언
    
    public static void main(String[] args) {
        new HelloJNI().sayHello(); // 네이티브 메서드 호출
    }
}
```

method를 선언할 때 `native`  keyword 사용

`printHello`가 호출될 때 자바가 아닌 native로 구현된 method를 찾게 된다.

static block은 class가 사용될 때, `JNI_Hello` 이름을 가진 라이브러리를 메모리에 Load한다.

### 2. Java Compiler를 사용하여 Class를 Compile한 후, `javah`를 사용하여 헤더 파일을 생성한다.

```bash
javac HelloJNI.java
javah -jni HelloJNI
```

javah를 사용하여 만들어진 header 파일을 살펴보면 아래 코드와 같다.

```c
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class simple_Hello */

#ifndef _Included_simple_Hello
#define _Included_simple_Hello
#ifdef __cplusplus
extern "C" {
#endif

/*
 * Class:     simple_Hello
 * Method:    printHello
 * Signature: ()V
 */

JNIEXPORT void JNICALL Java_simple_Hello_printHello(JNIEnv *, jobject);

#ifdef __cplusplus
}
#endif
#endif
```

C파일에 `JNIEXPORT void JNICALL Java_simple_Hello_printHello(JNIEnv *, jobject);`
Method를 구현하면 된다. Method이름은 패키지-클래스-메소드 이름으로 되어있다. 따라서 패키지를 java에서 수정하면 native method를 사용하지 못한다.

### 3. C method

```c
#include <jni.h>
#include <stdio.h>
#include "HelloJNI.h"

JNIEXPORT void JNICALL Java_simple_Hello_printHello(JNIEnv *env, jobject obj) {
    printf("Hello from C!\n");
}
```

생성된 헤더 파일을 포함하여 C/C++ 파일을 작성하고 네이티브 메서드를 구현합니다.

### 4. Native Method Compile

작성한 네이티브 코드를 컴파일하여 공유 라이브러리를 생성합니다.

```bash
gcc -shared -o libhello.so -fPIC HelloJNI.c -I${JAVA_HOME}/include -I${JAVA_HOME}/include/linux
```

### 5. Java Application 실행

```bash
java -Djava.library.path=. HelloJNI
```

## Native Library를 Jar파일에 포함하여 Native Method를 사용해보기

위 예제는 서버 배포과정이 아닌 하나의 Java Application을 실행하는데 옵션을 추가하는 건 별 일 아닐 수 있다. 하지만 JNI를 사용하기 위해 서버 배포시 옵션을 추가해주는 일은 번거로운 일이다.

이를 해결하기 위해선 두 가지 문제점을 해결해야 한다.

1. 옵션을 사용하지 않고 Native Library 사용하기
2. 하나의 Jar 파일만 사용하기

해결 방법은 resources 폴더 내에 Native Library 포함하여 Jar File을 만든 후 Native static block에서 Native Library를 메모리에 Load 하기 전, resource 폴더에서 Native Library 파일을 copy하여 해당 파일을 Native Library를 메모리에 Load 한다. 코드는 아래와 같다.

[How to bundle a native library and a JNI library inside a JAR?](https://stackoverflow.com/questions/2937406/how-to-bundle-a-native-library-and-a-jni-library-inside-a-jar?rq=1)

```java
String libName = "libhello.so "; // The name of the file in resources/ dir
URL url = MyClass.class.getResource("/" + libName);
File tmpDir = Files.createTempDirectory("my-native-lib").toFile();
tmpDir.deleteOnExit();
File nativeLibTmpFile = new File(tmpDir, libName);
nativeLibTmpFile.deleteOnExit();
try (InputStream in = url.openStream()) {
    Files.copy(in, nativeLibTmpFile.toPath());
}
System.load(nativeLibTmpFile.getAbsolutePath());
```

위 코드와 같이 File에 임시로 Native Library를 copy 한 후 load 하는 방식으로 두 가지 문제점을 해결할 수 있었다.

생성된 File은 JVM이 종료되면 삭제된다.