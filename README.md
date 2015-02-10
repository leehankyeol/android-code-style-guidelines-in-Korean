# 안드로이드 개발자를 위한 코드 스타일 가이드라인

아래에 적힌 규칙들은 단순한 가이드라인이나 권장사항이 아닌, 엄격한 규칙이다.
해당 규칙을 지키지 않은 안드로이드 프로젝트에 대한 코드 기여는 일반적으로 받아들여지지 않을 것이다.

현재 존재하는 모든 코드가 이 규칙을 따르지는 않지만, 새로운 코드들은 이 규칙들을 따를 것을 기대한다.

## 자바 언어 규칙

몇 가지 규칙을 추가한 자바의 표준 코딩 컨벤션을 따른다.

### 예외(exception)를 무시하지 마라

코드를 짜다 보면 때로는 아래처럼 예외를 완전히 무시하고 싶어질 때가 있을 것이다.

```java
void setServerPort(String value) {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) { }
}
```

절대 이런 식으로 코드를 짜서는 안 된다. 당신의 코드가 절대로 오류가 발생할 조건을 만족하지 않을 거라고,
또는 발생하는 에러를 처리하는 것이 중요하지 않다고 생각할 수도 있으나 위의 예제와 같이 예외를 무시하는 것은
언젠가 당신의 코드를 다룰 누군가에게 지뢰를 깔아주는 것과 다름없다. 따라서 코드에서 발생할 수 있는 모든 예외를
정해진 규칙에 따라 처리해주어야 한다. 처리 방법은 각각의 경우마다 다양할 것이다.

> 누구든 비어 있는 캐치`catch` 구문을 만들 때는 뭔가 섬뜩한 기분이 들어야 한다. 물론 그렇게 하는 것이 옳은
> 때가 분명히 있지만, 적어도 다시 한 번 생각해볼 필요가 있다. 자바를 짤 때는 그 섬뜩한 기분을 무시해선 안 된다.
> [제임스 고슬링](https://source.android.com/source/code-style.html#java-language-rules)

허용할 만한 대안들은 아래와 같다. 목록의 위에 있을수록 선호도가 높은 방법이다.

* 예외를 해당 메소드의 콜러(caller)에 발생시킨다.

	```java
	void setServerPort(String value) throws NumberFormatException {
	    serverPort = Integer.parseInt(value);
	}
	```

* 추상(abstraction) 정도에 맞는 새로운 예외를 발생시킨다.

	```java
	void setServerPort(String value) throws ConfigurationException {
	    try {
	        serverPort = Integer.parseInt(value);
	    } catch (NumberFormatException e) {
	        throw new ConfigurationException("Port " + value + " is not valid.");
	    }
	}
	```

* 캐치 블록 안에서 적당한 값으로 대체한다.

	```java
	/** Set port. If value is not a valid number, 80 is substituted. */

	void setServerPort(String value) {
	    try {
	        serverPort = Integer.parseInt(value);
	    } catch (NumberFormatException e) {
	        serverPort = 80;  // default port for server 
	    }
	}
	```

* 예외를 캐치한 뒤 새로운 런타임예외`RuntimeException`를 발생시킨다. 위험한 방법이므로 해당 에러가 발생했을 때
올바른 처리 방법이 크래시(crash)라 확신이 있을 때만 사용하라.

	```java
	/** Set port. If value is not a valid number, die. */

	void setServerPort(String value) {
	    try {
	        serverPort = Integer.parseInt(value);
	    } catch (NumberFormatException e) {
	        throw new RuntimeException("port " + value " is invalid, ", e);
	    }
	}
	```

컨스트럭터(constructor)에게 런타임예외로 전해질 때 원래 발생한 예외가 같이 전달되는 것에 주목하자.
자바 1.3 미만의 버전에서 컴파일을 해야 한다면 원인이 되는 예외를 코드에서 제외해야 할 것이다.

* 최후의 보루: 예외를 무시하는 것이 올바르다고 확신이 든다면 그렇게 해도 좋다. 다만 합당한 이유를 꼭 코멘트로 남기자.

	```java
	/** If value is not a valid number, original port number is used. */
	void setServerPort(String value) {
	    try {
	        serverPort = Integer.parseInt(value);
	    } catch (NumberFormatException e) {
	        // Method is documented to just ignore invalid user input.
	        // serverPort will just be unchanged.
	    }
	}
	```

### 일반적 예외(generic exception)를 캐치하지 마라

우리는 종종 예외를 잡아내는 것에 게을러져 아래와 같은 코드를 짜고 싶을 때가 있다.

```java
try {
    someComplicatedIOFunction();        // may throw IOException 
    someComplicatedParsingFunction();   // may throw ParsingException 
    someComplicatedSecurityFunction();  // may throw SecurityException 
    // phew, made it all the way 
} catch (Exception e) {                 // I'll just catch all exceptions 
    handleError();                      // with one generic handler!
}
```

하지만 이렇게 코드를 짜면 안 된다. 거의 모든 경우에 일반적인 예외나 쓰로어블(throwable)을 캐치하는 것은 적절하지 못한 선택이다. (preferably not Throwable, because it includes Error exceptions as well.) 매우 위험한 코드다. 전혀 예상하지 못한 예외가, 예를 들어 ClassCastException 같은 RuntimeExceptions 등, 애플리케이션 레벨의 에러 핸들링에서야 캐치되게 될 것이다. 이런 방식은 코드가 다양한 실패 상황에 대처하지 못하게끔 하는데, 이는 누군가 당신의 코드가 호출하는 곳에 새로운 종류의 예외를 만들었을 때 컴파일러가 그 새로운 에러를 다른 방식으로 다뤄야 한다는 메시지를 당신에게 전달하지 못한다는 의미다. 게다가 어쨌든 그렇게 다른 종류의 예외를 같은 방식으로 처리하는 것 또한 해서는 안 되는 일이다.

이 규칙에는 아주 드문 예외가 있다. UI에 나타나는 것을 방지한다든지, 배치 잡(batch job)을 계속 돌려야 한다든지 하는 경우에 모든 종류의 에러를 캐치할 최상위 코드나 테스트 코드가 필요할 때다. 이런 경우엔 일반적인 예외나 쓰로어블을 캐치해서 올바르게 에러를 처리해도 된다. 하지만 이와 같은 처리는 언제나 주의해야 하며 왜 이 상황에 이런 처리가 안전한지 설명하는 주석을 꼭 달아야 한다.

일반적 예외를 캐치하는 대안들:

* 하나의 트라이`try`문 뒤에 각각 다른 여러 개의 캐치 블록을 작성한다. 보기에 어색할 수는 있지만 여전히 한 번에 모든 예외를 캐치하는 것보다 낫다. 각 캐치 블록 안에 너무 많은 중복 코드를 작성하지 않도록 주의하자.

* 더 조밀하게 에러 처리를 할 수 있도록 더 많은 트라이 블록으로 코드를 리팩토링(refactoring)하라. Split up the IO from the parsing, handle errors separately in each case.

* 예외를 다시 쓰로우(rethrow)하라. 많은 경우 어차피 해당 단계에서 예외를 캐치할 필요가 없기 때문이다.

기억하자. 예외는 우리의 친구다! 컴파일러가 예외를 캐치하지 않고 있다고 불만을 하면 울지 말고 웃어라. 컴파일러는 당신의 코드에서 발생할 수 있는 런타임 문제를 더 쉽게 해결해주고자 할 뿐이다.

### 파이널라이저(finalizer)를 사용하지 말라

파이널라이저는 객체가 가비지 콜렉션될 때 실행될 코드를 짜두는 방법 중 하나다.

장점: 특히 외부 리소스를 포함해 여러모로 리소스 정리를 할 때 편리하다.

단점: 언제 파이널라이저가 호출될지, 심지어 파이널라이저가 호출이 되는지 안 되는지도 정확히 알기가 어렵다.

결론: 파이널라이저를 사용하지 말자. 대부분의 경우 파이널라이저가 할 일은 예외 처리를 잘 함으로써 해결이 가능하다. 정말 필요하다면, `close()` 메소드나 그 비슷한 것을 정의하고 정확히 언제 해당 메소드가 호출되어야 하는지를 문서로 작성하라. See InputStream for an example. In this case it is appropriate but not required to print a short log message from the finalizer, as long as it is not expected to flood the logs.

### 임포트`import`는 끝까지 명시하라

패키지 `foo`의 클래스 `Bar`를 사용하고 싶다면 임포트를 할 때 두 가지 선택지가 있다.

1. `import foo.*;`

장점: 임포트 문의 수를 줄일 수 있다.

2. `import foo.Bar;`

장점: 정확히 어떤 클래스가 사용되는지 알 수 있다. 유지보수에 유리하다.

결론: 모든 안드로이드 프로젝트에서는 후자의 방법을 사용하라. 특별한 예외를 둔다면 자바 표준 라이브러리(`java.util.*`, `java.io.*`, etc.)나 유닛 테스트 코드(`junit.framework.*`) 정도다.

## 자바 라이브러리 규칙

안드로이드의 자바 라이브러리와 툴을 사용할 때에도 지켜야 할 컨벤션이 있다. 컨벤션의 핵심적인 사항이 바뀌어 이전의 코드가 더 이상 사용되지 않고 사라질(deprecated) 패턴이나 라이브러리를 사용할 때가 있다. 그렇게 짜여져 있는 코드를 다룰 때는 현재 유지되고 있는 스타일을 계속 사용하는 것도 괜찮으나 새로운 코드를 짤 때는 절대 사라질 라이브러리를 사용하지 말라.

## 자바 스타일 규칙

### 자바독(Javadoc) 표준 주석을 사용하라

모든 파일은 저작권에 대한 기술로 시작해야 한다. 그 뒤로 패키지와 임포트`import`에 관한 문구가 이어지는데 각 블록은 한 줄을 띄운다. 그 아래로 클래스와 인터페이스를 선언한다. 자바독 주석에는 각 클래스와 인터페이스가 무엇을 하는지를 기술하라.

```java
/*
 * Copyright (C) 2013 The Android Open Source Project 
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at 
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software 
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and 
 * limitations under the License.
 */

package com.android.internal.foo;

import android.os.Blah;
import android.view.Yada;

import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * Does X and Y and provides an abstraction for Z.
 */

public class Foo {
    ...
}
```

당신이 만든 모든 클래스와 이름으로 봤을 때 기능이 자명하지 않은(nontrivial) 퍼블릭 메소드에는 각 클래스와 메소드가 무슨 일을 하는지 적어도 한 줄 이상의 자바독 주석을 달자. 이 때 각 주석문은 3인칭 단수형 동사로 시작해야 한다.

예:

```java
/** Returns the correctly rounded positive square root of a double value. */
static double sqrt(double a) {
    ...
}
```

또는

```java
/**
 * Constructs a new String by converting the specified array of 
 * bytes using the platform's default character encoding.
 */
public String(byte[] bytes) {
    ...
}
```

주석의 내용이 단지 "Foo를 설정한다(sets Foo)" 등 자명한 이름의 겟(get)과 셋(set) 메소드에는 자바독 주석을 달 필요가 없다. 하지만 해당 메소드가 조금 더 복잡한 일, 예를 들어 제약 조건을 강화한다거나 중대한 부작용을 야기할 수 있는 경우엔 꼭 문서화를 해야 한다. 또한 "Foo"라는 속성이 의미하는 바가 자명하지 않을 경우에도 문서화가 필요하다.

당신이 작성하는 모든 메소드들에, 그것이 퍼블릭한 것이든 아니든, 자바독을 사용하는 편이 좋다. API의 일부가 되는 퍼블릭 메소드의 경우는 당연히 자바독을 필요로 한다.

안드로이드는 현재 자바독 주석을 사용함에 있어 특정한 스타일을 제시하고 있진 않지만, [자바독 툴을 위한 문서화(How to Write Doc Comments for the Javadoc Tool)](http://www.oracle.com/technetwork/java/javase/documentation/index-137868.html)의 지시 사항을 따르는 것이 좋다.

### 메소드를 짧게 유지하라

가능한 한도 내에서 메소드는 한 주제에 맞게 짧게 작성되어야 한다. 어떤 경우에는 긴 메소드를 작성하는 것이 옳기 때문에 메소드 길이에 정확히 정해진 제한은 없다. 한 메소드의 길이가 40줄을 넘어가게 되면 전체 프로그램의 구조를 해치지 않는 선에서 코드를 나눌 수 있는지 생각해보라.

### 정해진 위치에 필드를 정의하라

필드는 파일의 상단이나, 그 필드를 사용하는 메소드 바로 직전에 정의되어야 한다.

### 변수의 범위(scope)를 제한하라

지역(local) 변수의 범위는 최소한으로 유지되어야 하는데 그럼으로써 코드의 가독성과 유지보수의 편이성이 높아질 뿐만 아니라 에러의 가능성도 낮출 수 있기 때문이다. 모든 변수는 그 변수가 사용되는 것을 모두 감쌀 수 있는 가장 안쪽의 블록에서 선언되어야 한다.

지역 변수는 처음 사용되는 시점에 선언되어야 한다. 거의 모든 지역 변수의 선언은 초기화가 동반되는 것이 옳다. 변수를 선언하는 시점에서 어떤 값으로 초기화할지 충분한 정보가 없다면 그런 정보가 주어지는 시점까지 변수의 선언을 미뤄야 한다.

이 규칙에 한 가지 예외가 있다면 트라이`try`-캐치`catch` 문을 사용할 때다. 만약 어떤 변수가 예외 처리되는 메소드의 반환값으로 초기화될 경우 트라이 블록 안에서 초기화되어야 할 것이다. 만약 다음 예와 같이 그 변수가 트라이 블록 바깥에서 사용되어야 한다면 해당 변수는 트라이 블록 이전에 초기화 없이 선언될 수 있다.

```java
// Instantiate class cl, which represents some sort of Set 
Set s = null;
try {
    s = (Set) cl.newInstance();
} catch(IllegalAccessException e) {
    throw new IllegalArgumentException(cl + " not accessible");
} catch(InstantiationException e) {
    throw new IllegalArgumentException(cl + " not instantiable");
}

// Exercise the set 
s.addAll(Arrays.asList(args));
```

하지만 이런 경우에도 트라이-캐치 블록 자체를 메소드에 인캡슐레이션함으로써 변수를 미리 선언하는 것을 피할 수 있다.

```java
Set createSet(Class cl) {
    // Instantiate class cl, which represents some sort of Set 
    try {
        return (Set) cl.newInstance();
    } catch(IllegalAccessException e) {
        throw new IllegalArgumentException(cl + " not accessible");
    } catch(InstantiationException e) {
        throw new IllegalArgumentException(cl + " not instantiable");
    }
}

...

// Exercise the set 
Set s = createSet(cl);
s.addAll(Arrays.asList(args));
```

반복문에 사용되는 변수는 꼭 그러지 말아야 할 이유가 없는 한 반복문 내부에서 선언되어야 한다.

```java
for (int i = 0; i < n; i++) {
    doSomething(i);
}
```

같은 예다.

```java
for (Iterator i = c.iterator(); i.hasNext(); ) {
    doSomethingElse(i.next());
}
```

### 임포트`import`문의 순서

임포트문은 다음의 순서로 정렬한다.

1. 안드로이드 임포트

2. 써드파티 임포트(`com`, `junit`, `net`, `org`)

3. `java`와 `javax`

IDE 설정과 정확히 맞추기 위한 순서는 다음과 같다:

* 각각의 그룹 내에서는 알파벳순으로 정렬한다. 대문자가 소문자 앞에 온다. (즉 Z가 a보다 앞이다.)

* 주요 그룹은 한 줄 띄어준다. (`android`, `com`, `junit`, `net`, `org`, `java`, `javax`).

원래 임포트의 순서에는 정해진 스타일이 없었다. IDE가 자동적으로 임포트문의 순서를 바꾸거나, 아니면 개발자들이 자동 관리 옵션을 비활성화하고 손수 관리하는 방법밖에 없었다. 이런 방식이 썩 좋아보이지 않았다. 온갖 방식을 두고 자바만의 스타일을 정해보기로 했고 결국 "원칙을 정하고 일관성을 유지하자"는 기조에 맞춘 결정을 내렸다. 그래서 한 가지 스타일을 정하고, 스타일 가이드를 개선한 뒤 IDE에게 그 스타일을 따르게끔 했다. 별도의 노력없이 IDE 사용자들이 앞으로 코드를 짜나감에 따라 모든 패키지의 임포트문이 이 패턴에 맞춰지길 기대한다.

그렇게 정해진 스타일은 다음의 목적을 만족시키고자 했다:

* 사람들이 먼저 보기 원하는 임포트는 상위에 위치한다. (예: `android`)

* 그다지 볼 필요가 없는 임포트는 하단에 위치한다. (예: `java`)

* 사람들이 스타일을 쉽게 따를 수 있다.

* 다양한 IDE도 스타일을 쉽게 따를 수 있다.

스태틱 임포트의 위치는 다소 논란의 여지가 있었다. 여러 임포트 사이에 배치하자는 의견도 있었고 다른 임포트들의 최상단이나 최하단에 놓자는 의견도 있었다. 또한, 모든 IDE가 이 순서를 지키게 하는 방법도 아직 찾지 못했다.

많은 사람들이 그렇게 중요하게 생각하지 않는 문제인 만큼 알아서 판단하라. 일관성을 유지하는 것만 지키면 된다.

### 들여쓰기에는 스페이스를 사용하라

블록 들여쓰기에는 4개의 스페이스를 사용한다. 탭은 사용하지 않는다. 어떻게 해야 할지 잘 모르겠다면 당신이 다루고 있는 코드가 기존에 유지하던 방식을 고수하라.

함수 호출이나 값을 할당할 때 등 줄바꿈을 할 때는 8개의 스페이스를 사용한다. 아래는 올바른 사용 예다.

```java
Instrument i =
        someLongExpression(that, wouldNotFit, on, one, line);
```

그리고 아래는 잘못된 예다.

```java
Instrument i =
    someLongExpression(that, wouldNotFit, on, one, line);
```

### 필드 네이밍 컨벤션을 따르라

* 퍼블릭이나 스태틱이 아닌 필드의 이름은 m으로 시작한다.

* 스태틱 필드 이름은 s로 시작한다.

* 다른 필드는 소문자로 시작한다.

* 퍼블릭 스태틱 필드(상수)는 모두 대문자에 언더스코어를 사용한다.

예:

```java
public class MyClass {
    public static final int SOME_CONSTANT = 42;
    public int publicField;
    private static MyClass sSingleton;
    int mPackagePrivate;
    private int mPrivate;
    protected int mProtected;
}
```

### 정해진 중괄호 스타일을 사용하라

다음과 같이, 혼자서 한 줄을 차지하는 중괄호를 사용하지 말고 이전 줄의 코드에 붙여 사용하라.

```java
class MyClass {
    int func() {
        if (something) {
            // ...
        } else if (somethingElse) {
            // ...
        } else {
            // ...
        }
    }
}
```

조건문을 사용할 때는 중괄호를 사용하는 것이 원칙이다. 단, 조건문의 모든 내용이 한 줄로 기술이 가능하면, (의무는 아니지만) 한 줄에 쓰는 것도 가능하다. 아래는 가능한 예다.

```java
if (condition) {
    body(); 
}
```

아래의 예도 가능하다.

```java
if (condition) body();
```

하지만 아래의 예는 잘못되었다.

```java
if (condition)
    body();  // bad!
```

### 행 길이를 제한하라

코드의 한 행의 길이는 최대 100글자로 제한되어야 한다.

많은 토론이 있었지만 최종적으로 최대 100글자에서 제한해야 한다는 결론을 내렸다.

예외: 커맨드나 URL을 포함한 주석의 경우 한 줄에 100글자가 넘더라도 복사, 붙여넣기 등의 편이를 위해 길이를 제한하지 않는다.

예외: 임포트`import` 구문의 경우 사람이 직접 코드를 읽는 경우가 적기 때문에 100글자 이상의 코드를 허용한다. This also simplifies tool writing.

### 자바 표준 주석(annotation)을 사용하라

주석은 다른 모디파이어(modifier)보다 선행한다. `@Override` 같은 단순한 주석은 다른 언어 요소(language element)와 같은 줄에 위치해도 된다. 주석이 여러 개이거나 파라미터를 받는 주석인 경우, 알파벳 순서대로 한 줄에 하나씩 적는 것이 원칙이다.

자바에서 선정의된(predefined) 세 가지 주석에 대해 안드로이드가 정한 기준은 다음과 같다.

* `@Deprecated`: @Deprecated 주석은 주석처리된 요소의 사용이 자제되어야 할 때 사용된다. 만약 @Deprecated 주석을 사용하겠다면 @deprecated 자바독(Javadoc) 태그를 만들어 대안으로 사용될 메소드를 명시해야 한다. 추가적으로, @Deprecated 처리된 메소드들은 여전히 동작해야 한다는 사실도 잊지 마라.
지난 코드에서 @deprecated 자바독 태그를 본다면 @Deprecated 주석을 달아두자.

* `@Override`: @Override 주석은 슈퍼클래스에 구현 또는 선언되어 있는 메소드를 오버라이드(override)하는 경우에 사용된다.
예를 들어, `@inheritdocs` 자바독 태그를 사용하고, 인터페이스가 아닌 클래스로부터 상속을 받을 경우엔 각 메소드가 부모 클래스의 메소드를 오버라이드한다는 것을 주석으로 표시해야 한다.

* `@SuppressWarnings`: @SuppressWarnings 주석은 경고(warning)을 없애는 것이 불가능한 상황에서 사용되어야 한다. 발생하는 모든 경고가 코드 상에 실재하는 문제를 반영하기 위해 필요한 주석이다.
@SuppressWarnings 주석은 "경고를 제거하는 것이 불가능한" 상황을 설명하는 `TODO` 주석과 같이 작성되어야 하는데 이를 통해 어떤 클래스가 이상한 인터페이스를 가지고 있는지 파악할 수 있다. 예를 들면 다음과 같다.

```java
// TODO: The third-party class com.third.useful.Utility.rotate() needs generics 
@SuppressWarnings("generic-cast")
List<String> blix = Utility.rotate(blax);
```

@SuppressWarnings 주석이 들어가야 하는 경우엔, 주석이 적용되는 범위의 코드는 리팩토링(refactoring)되어야 한다.

### 두문자어(acronym)을 일반 단어로 취급하라

변수와 메소드, 클래스의 이름을 지을 때 두문자어와 약어(abbreviation)를 일반적인 단어로 취급하라. 더 가독성이 좋은 이름을 지을 수 있다.

| 좋은 예          | 나쁜 예          |
|----------------|----------------|
| XmlHttpRequest | XMLHTTPRequest |
| getCustomerId  | getCustomerID  |
| class Html     | class HTML     |
| String url     | String URL     |
| long id        | long ID        |

JDK와 안드로이드 코드는 모두 이 두문자어와 관련된 문제에서 굉장히 일관적이지 못한 편이다. 따라서 사실상 당신이 접할 모든 코드에서 이 규칙을 지키는 것이 불가능하다. 그런 만큼, 어금니를 질끈 물고, 모든 두문자어를 일반 단어로 취급하자.

### `TODO` 주석을 사용하라

TODO 주석은 일시적이거나 단기적인 해결책, 완벽하진 않지만 우선은 괜찮은 코드 등에 사용할 수 있다.

TODO는 모두 대문자로 작성되어야 하며 항상 콜론(:)이 뒤따라야 한다.

```java
// TODO: Remove this code after the UrlTable2 has been checked in.
```

또 다른 예다.

```java
// TODO: Change this to use a flag instead of a constant.
```

"언제까지 어떤 것을 해야 한다"는 식으로 TODO를 작성할 경우 아주 자세한 일정이나("2005년 11월까지 수정") 자세한 할 일("모든 관계자들이 7버전의 프로토콜을 이해한 뒤에야 이 코드를 삭제")을 병기한다.

### 로그는 아껴 써라

로그는 필수적인 작업이지만 짜임새있게 코딩하지 않으면 성능에 부정적인 영향을 주고 그 순기능을 쉽게 잃어버리게 된다. 자바의 로그 퍼실리티(log facility)는 5단계의 로그를 지원한다.

* `ERROR`: 일반 사용자가 바로 발견할 수 있는 결과가 나거나 따로 어떤 데이터를 지우거나, 애플리케이션을 삭제하거나, 데이터 파티션을 싹 날리거나 아니면 기기를 초기화하거나(아니면 그보다 더 나쁜 상황이거나) 하는 정도로 아주 치명적인 일이 발생했을 때 사용되는 로그 단계다. 이 단계의 로그는 어느 상황에서든 필수적이며 일반적으로 상황을 통계 서버에 보고하는 것이 좋다.

* `WARNING`: 심각하거나 예상하지 못한 일이 발생했을 때, 즉 사용자에게 가시적인 결과가 있지만 별도의 액션과(좀 기다리거나 앱을 재시작하거나 새 버전으로 업그레이드하거나 기기를 재시작하는 정도) 데이터의 손실이 없이 복구가 가능한 상황 등에 사용되는 로그 단계다. 이 단계의 로그 또한 어느 상황에서든 필수적이며 통계 서버에 보고를 고려해볼 만한 대상이다.

* `INFORMATIVE`: 꼭 에러까지는 아니더라도 효과가 다방면으로 퍼지는 상황처럼 많은 수의 사람들이 주지하고 있어야 하는 상황에 사용되는 로그 단계다. 다른 모듈에 의한 중복 로그를 방지하기 위해, 이 단계의 로그는 해당 영역에서 로그를 하기에 가장 적절한 권한을 가지는(most authoritative) 모듈에 의해 작성되는 것이 좋다. 마찬가지로 항상 작성되어야 하는 로그다.

* `DEBUG`: 예상하지 못한 결과를 디버그하거나 조사할 때 관련이 있을 수도 있는 부분에 대한 기록이 필요할 때 사용되는 로그 단계다. 해당 지점에 대해 충분한 정보를 얻는데 필요한 정도만 로그해야 하며 전체 로그 양에 비해 이 단계의 로그가 너무 많아질 경우엔 `VERBOSE` 단계의 로그를 사용하는 것을 고려해야 한다.
릴리즈 빌드에서도 필요하다면 기록되어야 하지만 `if (LOCAL_LOG)` 또는 `if (LOCAL_LOGD)` 블록 등으로 감싸서 `LOCAL_LOG[D]`를 정의하는 클래스나 서브컴포넌트 등이 원한다면 모든 로그를 비활성화할 수 있게 해야 한다. 따라서 `if (LOCAL_LOG)` 블록 안에는 유효한 로직이 들어가서는 안 되고 로그에 사용되는 모든 문자열은 `if (LOCAL_LOG)` 블록 안에 들어가 있어야 한다. The logging call should not be re-factored out into a method call if it is going to cause the string building to take place outside of the if `(LOCAL_LOG)` block. 아직도 `if (localLOGV)` 같은 코드가 존재하는데 비록 표준에서 벗어나는 네이밍이지만 허용 가능한 코드다.

* `VERBOSE`: 나머지 모든 것을 로그할 때 쓰는 로그 단계다. 이 로그는 디버그 빌드에서만 기록되어야 하고 기본적으로 컴파일에서 제외되어야 하기 때문에 `if (LOCAL_LOGV)` 블록 또는 그와 동치인 블록 등으로 둘러싸여야 한다. 마찬가지로 릴리즈 빌드에서는 모든 문자열이 제외되기 때문에 로그와 관련된 문자열들은 `if (LOCAL_LOGV)` 블록 내부에 작성되어야 한다.

*노트*:

* 하나의 모듈 안에서는 VERBOSE 단계가 아닌 이상 가능한 한 하나의 에러는 단 한 번만 보고되어야 한다. 한 모듈 내부에서 일련의 함수 호출 과정 안에서는 가장 내부에 위치하는 함수만이 에러를 반환해야 하고 그 함수를 호출하는 동일 모듈 내의 다른 부분에서는, 그렇게 하는 것이 문제를 고립하는데 충분한 도움이 되는 경우에만, 필요한 로그를 덧붙이는 형식이 되어야 한다.

* 모듈 체인에서 상위 레벨의 모듈에서 온 데이터가 하위 레벨 모듈에서 유효하지 않은 것으로 확인될 때엔, 로그가 되지 않고서는 함수를 호출한 대상이 그 사실을 알 수 없는 경우에만 `DEBUG` 단계의 로그가 필요하다. (`VERBOSE` 로그는 문제 없다.) 구체적으로, 예외(exception)가 발생하는 경우(예외 자체가 모든 관련된 정보를 담고 있다.), 로그 정보가 오직 에러 코드를 담고 있는 경우에는 로그를 할 필요가 없다. 이 규칙은 프레임워크와 애플리케이션 사이의 상호작용에 있어 특히 중요한데, 써드파티 애플리케이션에서 발생하는 예외 상황을 프레임워크가 올바르게 처리를 하는 상황에선 `DEBUG` 단계를 넘는 로그가 필요하지 않다. `INFORMATIVE` 이상의 로그가 필요한 유일한 상황은 모듈이나 애플리케이션이 자신이 속한 레벨 또는 그보다 하위 레벨에서 오는 오류를 발견했을 때이다.

* 어떤 로그가 다발적으로 발생할 가능성이 정당화되는 상황이라면 같은 (또는 아주 비슷한) 정보가 여러 번 기록되어 로그가 넘쳐나는 현상을 방지하기 위해 비율 제한(rate-limiting) 메커니즘을 구현하는 것을 고려하면 좋다.

* 네트워크와 연결이 끊기는 상황은 자주 발생하는 일이기 때문에 불필요하게 로그되어서는 안 된다. 네트워크의 연결이 끊김으로써 앱 내부에서 어떤 결과를 가져오는 경우에 `DEBUG` 또는 `VERBOSE` 단계의 로그가 적절하다. 릴리즈 빌드에서도 로그될 필요가 있을 만큼 심각하거나 예상하지 못한 정도에 따라 단계를 설정하면 된다.

* 써드파티 애플리케이션을 위해 존재하는, 또는 써드파티 애플리케이션이 접근 가능한 파일시스템에 대해서는 `INFORMATIVE`보다 상위 단계에서 로그되면 안 된다.

* 신뢰하지 못하는 모든 소스(공유 저장소의 모든 파일들과 네트워크를 통한 모든 수신 데이터를 포함한다.)로부터 온 데이터가 유효하지 않은 것은 충분히 발생할 가능성이 있는 상황이기 때문에 `DEBUG`보다 상위 단계에서 로그되어서는 안 되며 로그를 하는 경우에도 최대한 제한되어야 한다.

* `+` 연산자가 문자열`String`에 사용되면 내부적으로 16글자의 기본 버퍼 사이즈를 갖는 `StringBuilder`를 포함해 임시적으로 사용되는 다른 여러 문자열 객체를 만든다는 사실에 유념하라. 사실 기본 `+` 연산자를 사용하는 것보다 명시적으로(explicitly) `StringBuilder`를 정의하는 것이 훨씬 더 효율적일 수 있다. 또한 `Log.v()`를 호출하는 코드들은 그 로그가 실제로 발생하지 않는다고 해도 문자열을 생성하는 것을 포함, 릴리즈 빌드에서 컴파일되고 실행된다는 사실에도 유념하라.

* 다른 사람들이 보도록 릴리즈 빌드에서 발생시키는 모든 로그는(최소한 `DEBUG` 단계의 로그까지는) 그 의미를 숨기지 않으면서 다른 사람들이 이해할 수 있게끔 간결해야 한다.

* 가능하다면 로그는 한 줄에서 끝나는 것이 좋다. 80~100자 정도의 로그까지는 충분히 허용될 만하며 태그 길이를 포함해 130~160자 정도되는 로그는 되도록 피해야 한다.

* 성공(successes)을 보고하는 로그는 `VERBOSE`보다 상위의 단계에서 사용되어서는 안 된다.

* 재생산하기 어려운 이슈를 점검하기 위한 일시적인 로그는 `DEBUG` 또는 `VERBOSE` 단계에 머물러야 하며, 컴파일 타임에 전부 비활성화될 수 있게끔 조건문`if` 블록으로 감싸야 한다.

* 로그를 통한 보안 문제에 신경써야 한다. 보호되어야 하는 정보는 물론 개인 정보 역시 로그에 포함되어서는 안 된다. 이 규칙은 특히나 프레임워크 코드를 짤 때 더욱 중요한데 프레임워크 코드를 짤 때는 어떤 정보가 개인 정보, 또는 보호되어야 하는 정보인지 미리 알기가 쉽지 않기 때문이다.

* `System.out.println()` 또는 `printf()` 등은 절대 사용하지 말아야 한다. `System.out`과 `System.err`은 `/dev/null`로 리다이렉트되기 때문에 아무런 가시적 효과를 얻을 수 없음에도 여전히 유효하게 실행되는 문장들이기 때문이다.

* *로그의 황금률은, 다른사람들의 로그가 당신의 로그를 방해해도 안 되듯이 당신의 로그가 다른 사람들의 로그를 불필요하게 가로막아서는(push out of the buffer) 안 된다는 것이다.*

### 일관성을 유지하라

마지막으로, 항상 코드의 일관성을 유지하자. 코드를 수정할 일이 생긴다면 몇 분이라도 투자해 코드의 이곳저곳을 살펴보고 코드를 수정할 때 어떤 스타일을 사용할 것인지 결정하라. 어떤 코드가 `if`문 주변으로 스페이스를 사용했다면 당신도 그렇게 해야 한다. 그 코드의 주석이 별표로 만들어진 상자 안에 들어 있다면, 당신의 주석 또한 별표 상자 안에 작성하라.

스타일 가이드라인의 포인트는 코딩에 있어 공통적인 어휘를 사용함으로써 사람들이 당신이 어떻게 말하고 있는지보다 무엇을 말하고 있는지에 집중하게끔 하는 것이다. 이 전반적인 스타일 규칙으로 공통적으로 사용될 어휘를 제시했지만 특정한 경우의 스타일 또한 중요한 법이다. 새롭게 짜넣은 코드가 기존의 코드와 심하게 다른 스타일이라면 다른 사람들이 코드를 읽어내려가는데 적지 않은 불편을 겪게 될 것이다. 그런 상황을 되도록 피하자.

## 자바테스트 스타일 규칙

### 테스트 메소드 네이밍 컨벤션을 따르라

테스트 메소드에 이름을 붙일 때는 무엇을 테스트하고 있는지와 테스트의 대상이 되는 특정 경우를 구분하기 위해 언더스코어를 사용하면 된다. 이 스타일을 따르면 어떤 케이스를 테스트하는지 알아보기가 쉽다.

예: 

```java
testMethod_specificCase1 testMethod_specificCase2

void testIsDistinguishable_protanopia() {
    ColorMatcher colorMatcher = new ColorMatcher(PROTANOPIA)
    assertFalse(colorMatcher.isDistinguishable(Color.RED, Color.BLACK))
    assertTrue(colorMatcher.isDistinguishable(Color.X, Color.Y))
}
```