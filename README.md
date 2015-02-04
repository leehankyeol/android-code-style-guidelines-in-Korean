# 안드로이드 개발자를 위한 코드 스타일 가이드라인

아래에 적힌 규칙들은 단순한 가이드라인이나 권장사항이 아닌, 엄격한 규칙이다.
해당 규칙을 지키지 않은 안드로이드 프로젝트에 대한 코드 기여는 일반적으로 받아들여지지 않을 것이다.

현재 존재하는 모든 코드가 이 규칙을 따르지는 않지만, 새로운 코드들은 이 규칙들을 따를 것을 기대한다.

## 자바 언어 규칙

몇 가지 규칙을 추가한 자바의 표준 코딩 컨벤션을 따른다.

### 예외(exception)를 무시하지 마라

코드를 짜다 보면 때로는 아래처럼 예외를 완전히 무시하고 싶어질 때가 있을 것이다.

	void setServerPort(String value) {
	    try {
	        serverPort = Integer.parseInt(value);
	    } catch (NumberFormatException e) { }
	}

절대 이런 식으로 코드를 짜서는 안 된다. 당신의 코드가 절대로 오류가 발생할 조건을 만족하지 않을 거라고,
또는 발생하는 에러를 처리하는 것이 중요하지 않다고 생각할 수도 있으나 위의 예제와 같이 예외를 무시하는 것은
언젠가 당신의 코드를 다룰 누군가에게 지뢰를 깔아주는 것과 다름없다. 따라서 코드에서 발생할 수 있는 모든 예외를
정해진 규칙에 따라 처리해주어야 한다. 처리 방법은 각각의 경우마다 다양할 것이다.

> 누구든 비어 있는 캐치`catch` 구문을 만들 때는 뭔가 섬뜩한 기분이 들어야 한다. 물론 그렇게 하는 것이 옳은
> 때가 분명히 있지만, 적어도 다시 한 번 생각해볼 필요가 있다. 자바를 짤 때는 그 섬뜩한 기분을 무시해선 안 된다.
> - [제임스 고슬링](https://source.android.com/source/code-style.html#java-language-rules)

허용할 만한 대안들은 아래와 같다. 목록의 위에 있을수록 선호도가 높은 방법이다.

* 예외를 해당 메소드의 콜러(caller)에 발생시킨다.

		void setServerPort(String value) throws NumberFormatException {
		    serverPort = Integer.parseInt(value);
		}

* 추상(abstraction) 정도에 맞는 새로운 예외를 발생시킨다.

		void setServerPort(String value) throws ConfigurationException {
		    try {
		        serverPort = Integer.parseInt(value);
		    } catch (NumberFormatException e) {
		        throw new ConfigurationException("Port " + value + " is not valid.");
		    }
		}

* 캐치 블록 안에서 적당한 값으로 대체한다.

		/** Set port. If value is not a valid number, 80 is substituted. */

		void setServerPort(String value) {
		    try {
		        serverPort = Integer.parseInt(value);
		    } catch (NumberFormatException e) {
		        serverPort = 80;  // default port for server 
		    }
		}

* 예외를 캐치한 뒤 새로운 런타임예외`RuntimeException`를 발생시킨다. 위험한 방법이므로 해당 에러가 발생했을 때
올바른 처리 방법이 크래시(crash)라 확신이 있을 때만 사용하라.

		/** Set port. If value is not a valid number, die. */

		void setServerPort(String value) {
		    try {
		        serverPort = Integer.parseInt(value);
		    } catch (NumberFormatException e) {
		        throw new RuntimeException("port " + value " is invalid, ", e);
		    }
		}

컨스트럭터(constructor)에게 런타임예외로 전해질 때 원래 발생한 예외가 같이 전달되는 것에 주목하자.
자바 1.3 미만의 버전에서 컴파일을 해야 한다면 원인이 되는 예외를 코드에서 제외해야 할 것이다.

* 최후의 보루: 예외를 무시하는 것이 올바르다고 확신이 든다면 그렇게 해도 좋다. 다만 합당한 이유를 꼭 코멘트로 남기자.

		/** If value is not a valid number, original port number is used. */
		void setServerPort(String value) {
		    try {
		        serverPort = Integer.parseInt(value);
		    } catch (NumberFormatException e) {
		        // Method is documented to just ignore invalid user input.
		        // serverPort will just be unchanged.
		    }
		}

### 일반적 예외(generic exception)를 캐치하지 마라

우리는 종종 예외를 잡아내는 것에 게을러져 아래와 같은 코드를 짜고 싶을 때가 있다.

	try {
	    someComplicatedIOFunction();        // may throw IOException 
	    someComplicatedParsingFunction();   // may throw ParsingException 
	    someComplicatedSecurityFunction();  // may throw SecurityException 
	    // phew, made it all the way 
	} catch (Exception e) {                 // I'll just catch all exceptions 
	    handleError();                      // with one generic handler!
	}

You should not do this. In almost all cases it is inappropriate to catch generic Exception or Throwable, preferably not Throwable, because it includes Error exceptions as well. It is very dangerous. It means that Exceptions you never expected (including RuntimeExceptions like ClassCastException) end up getting caught in application-level error handling. It obscures the failure handling properties of your code. It means if someone adds a new type of Exception in the code you're calling, the compiler won't help you realize you need to handle that error differently. And in most cases you shouldn't be handling different types of exception the same way, anyway.

There are rare exceptions to this rule: certain test code and top-level code where you want to catch all kinds of errors (to prevent them from showing up in a UI, or to keep a batch job running). In that case you may catch generic Exception (or Throwable) and handle the error appropriately. You should think very carefully before doing this, though, and put in comments explaining why it is safe in this place.

Alternatives to catching generic Exception:

* Catch each exception separately as separate catch blocks after a single try. This can be awkward but is still preferable to catching all Exceptions. Beware repeating too much code in the catch blocks.

* Refactor your code to have more fine-grained error handling, with multiple try blocks. Split up the IO from the parsing, handle errors separately in each case.

* Rethrow the exception. Many times you don't need to catch the exception at this level anyway, just let the method throw it.

Remember: exceptions are your friend! When the compiler complains you're not catching an exception, don't scowl. Smile: the compiler just made it easier for you to catch runtime problems in your code.

### Don't Use Finalizers

Finalizers are a way to have a chunk of code executed when an object is garbage collected.

Pros: can be handy for doing cleanup, particularly of external resources.

Cons: there are no guarantees as to when a finalizer will be called, or even that it will be called at all.

Decision: we don't use finalizers. In most cases, you can do what you need from a finalizer with good exception handling. If you absolutely need it, define a close() method (or the like) and document exactly when that method needs to be called. See InputStream for an example. In this case it is appropriate but not required to print a short log message from the finalizer, as long as it is not expected to flood the logs.

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

당신이 만든 모든 클래스와 이름으로 봤을 때 기능이 자명하지 않은(nontrivial) 퍼블릭 메소드에는 각 클래스와 메소드가 무슨 일을 하는지 적어도 한 줄 이상의 자바독 주석을 달자. 이 때 각 주석문은 3인칭 단수형 동사로 시작해야 한다.

예:

	/** Returns the correctly rounded positive square root of a double value. */
	static double sqrt(double a) {
	    ...
	}

또는

	/**
	 * Constructs a new String by converting the specified array of 
	 * bytes using the platform's default character encoding.
	 */
	public String(byte[] bytes) {
	    ...
	}

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

하지만 이런 경우에도 트라이-캐치 블록 자체를 메소드에 인캡슐레이션함으로써 변수를 미리 선언하는 것을 피할 수 있다.

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

반복문에 사용되는 변수는 꼭 그러지 말아야 할 이유가 없는 한 반복문 내부에서 선언되어야 한다.

	for (int i = 0; i < n; i++) {
	    doSomething(i);
	}

같은 예다.

	for (Iterator i = c.iterator(); i.hasNext(); ) {
	    doSomethingElse(i.next());
	}

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

	Instrument i =
	        someLongExpression(that, wouldNotFit, on, one, line);

그리고 아래는 잘못된 예다.

	Instrument i =
	    someLongExpression(that, wouldNotFit, on, one, line);

### 필드 네이밍 컨벤션을 따르라

* 퍼블릭이나 스태틱이 아닌 필드의 이름은 m으로 시작한다.

* 스태틱 필드 이름은 s로 시작한다.

* 다른 필드는 소문자로 시작한다.

* 퍼블릭 스태틱 필드(상수)는 모두 대문자에 언더스코어를 사용한다.

예:

	public class MyClass {
	    public static final int SOME_CONSTANT = 42;
	    public int publicField;
	    private static MyClass sSingleton;
	    int mPackagePrivate;
	    private int mPrivate;
	    protected int mProtected;
	}

### 정해진 중괄호 스타일을 사용하라

다음과 같이, 혼자서 한 줄을 차지하는 중괄호를 사용하지 말고 이전 줄의 코드에 붙여 사용하라.

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

조건문을 사용할 때는 중괄호를 사용하는 것이 원칙이다. 단, 조건문의 모든 내용이 한 줄로 기술이 가능하면, (의무는 아니지만) 한 줄에 쓰는 것도 가능하다. 아래는 가능한 예다.

	if (condition) {
	    body(); 
	}

아래의 예도 가능하다.

	if (condition) body();

하지만 아래의 예는 잘못되었다.

	if (condition)
	    body();  // bad!

### 행 길이를 제한하라

코드의 한 행의 길이는 최대 100글자로 제한되어야 한다.

많은 토론이 있었지만 최종적으로 최대 100글자에서 제한해야 한다는 결론을 내렸다.

예외: 커맨드나 URL을 포함한 주석의 경우 한 줄에 100글자가 넘더라도 복사, 붙여넣기 등의 편이를 위해 길이를 제한하지 않는다.

예외: 임포트`import` 구문의 경우 사람이 직접 코드를 읽는 경우가 적기 때문에 100글자 이상의 코드를 허용한다. This also simplifies tool writing.

### Use Standard Java Annotations

Annotations should precede other modifiers for the same language element. Simple marker annotations (e.g. @Override) can be listed on the same line with the language element. If there are multiple annotations, or parameterized annotations, they should each be listed one-per-line in alphabetical order.<

Android standard practices for the three predefined annotations in Java are:

* `@Deprecated`: The @Deprecated annotation must be used whenever the use of the annotated element is discouraged. If you use the @Deprecated annotation, you must also have a @deprecated Javadoc tag and it should name an alternate implementation. In addition, remember that a @Deprecated method is still supposed to work.
If you see old code that has a @deprecated Javadoc tag, please add the @Deprecated annotation.


* `@Override`: The @Override annotation must be used whenever a method overrides the declaration or implementation from a super-class.
For example, if you use the @inheritdocs Javadoc tag, and derive from a class (not an interface), you must also annotate that the method @Overrides the parent class's method.

* `@SuppressWarnings`: The @SuppressWarnings annotation should only be used under circumstances where it is impossible to eliminate a warning. If a warning passes this "impossible to eliminate" test, the @SuppressWarnings annotation must be used, so as to ensure that all warnings reflect actual problems in the code.
When a @SuppressWarnings annotation is necessary, it must be prefixed with a TODO comment that explains the "impossible to eliminate" condition. This will normally identify an offending class that has an awkward interface. For example:

	// TODO: The third-party class com.third.useful.Utility.rotate() needs generics 
	@SuppressWarnings("generic-cast")
	List<String> blix = Utility.rotate(blax);

When a @SuppressWarnings annotation is required, the code should be refactored to isolate the software elements where the annotation applies.

### 두문자어(acronym)을 일반 단어로 취급하라

변수와 메소드, 클래스의 이름을 지을 때 두문자어와 약어(abbreviation)를 일반적인 단어로 취급하라. 더 가독성이 좋은 이름을 지을 수 있다.

| 좋은 예          | 나쁜 예          |
|----------------|----------------|
| XmlHttpRequest | XMLHTTPRequest |
| getCustomerId  | getCustomerID  |
| class Html     | class HTML     |
| String url     | String URL     |
| long id        |                |

JDK와 안드로이드 코드는 모두 이 두문자어와 관련된 문제에서 굉장히 일관적이지 못한 편이다. 따라서 사실상 당신이 접할 모든 코드에서 이 규칙을 지키는 것이 불가능하다. 그런 만큼, 어금니를 질끈 물고, 모든 두문자어를 일반 단어로 취급하자.

### `TODO` 주석을 사용하라

TODO 주석은 일시적이거나 단기적인 해결책, 완벽하진 않지만 우선은 괜찮은 코드 등에 사용할 수 있다.

TODO는 모두 대문자로 작성되어야 하며 항상 콜론(:)이 뒤따라야 한다.

	// TODO: Remove this code after the UrlTable2 has been checked in.

또 다른 예다.

	// TODO: Change this to use a flag instead of a constant.

"언제까지 어떤 것을 해야 한다"는 식으로 TODO를 작성할 경우 아주 자세한 일정이나("2005년 11월까지 수정") 자세한 할 일("모든 관계자들이 7버전의 프로토콜을 이해한 뒤에야 이 코드를 삭제")을 병기한다.

### Log Sparingly

While logging is necessary, it has a significantly negative impact on performance and quickly loses its usefulness if it's not kept reasonably terse. The logging facilities provides five different levels of logging:

* `ERROR`: This level of logging should be used when something fatal has happened, i.e. something that will have user-visible consequences and won't be recoverable without explicitly deleting some data, uninstalling applications, wiping the data partitions or reflashing the entire phone (or worse). This level is always logged. Issues that justify some logging at the ERROR level are typically good candidates to be reported to a statistics-gathering server.

* `WARNING`: This level of logging should used when something serious and unexpected happened, i.e. something that will have user-visible consequences but is likely to be recoverable without data loss by performing some explicit action, ranging from waiting or restarting an app all the way to re-downloading a new version of an application or rebooting the device. This level is always logged. Issues that justify some logging at the WARNING level might also be considered for reporting to a statistics-gathering server.

* `INFORMATIVE`: This level of logging should used be to note that something interesting to most people happened, i.e. when a situation is detected that is likely to have widespread impact, though isn't necessarily an error. Such a condition should only be logged by a module that reasonably believes that it is the most authoritative in that domain (to avoid duplicate logging by non-authoritative components). This level is always logged.

* `DEBUG`: This level of logging should be used to further note what is happening on the device that could be relevant to investigate and debug unexpected behaviors. You should log only what is needed to gather enough information about what is going on about your component. If your debug logs are dominating the log then you probably should be using verbose logging.
This level will be logged, even on release builds, and is required to be surrounded by an `if (LOCAL_LOG)` or `if (LOCAL_LOGD)` block, where `LOCAL_LOG[D]` is defined in your class or subcomponent, so that there can exist a possibility to disable all such logging. There must therefore be no active logic in an `if (LOCAL_LOG)` block. All the string building for the log also needs to be placed inside the `if (LOCAL_LOG)` block. The logging call should not be re-factored out into a method call if it is going to cause the string building to take place outside of the if `(LOCAL_LOG)` block.
There is some code that still says `if (localLOGV)`. This is considered acceptable as well, although the name is nonstandard.

* `VERBOSE`: This level of logging should be used for everything else. This level will only be logged on debug builds and should be surrounded by an `if (LOCAL_LOGV)` block (or equivalent) so that it can be compiled out by default. Any string building will be stripped out of release builds and needs to appear inside the `if (LOCAL_LOGV)` block.

*Notes*:

* Within a given module, other than at the VERBOSE level, an error should only be reported once if possible: within a single chain of function calls within a module, only the innermost function should return the error, and callers in the same module should only add some logging if that significantly helps to isolate the issue.

* In a chain of modules, other than at the VERBOSE level, when a lower-level module detects invalid data coming from a higher-level module, the lower-level module should only log this situation to the DEBUG log, and only if logging provides information that is not otherwise available to the caller. Specifically, there is no need to log situations where an exception is thrown (the exception should contain all the relevant information), or where the only information being logged is contained in an error code. This is especially important in the interaction between the framework and applications, and conditions caused by third-party applications that are properly handled by the framework should not trigger logging higher than the DEBUG level. The only situations that should trigger logging at the INFORMATIVE level or higher is when a module or application detects an error at its own level or coming from a lower level.

* When a condition that would normally justify some logging is likely to occur many times, it can be a good idea to implement some rate-limiting mechanism to prevent overflowing the logs with many duplicate copies of the same (or very similar) information.

* Losses of network connectivity are considered common and fully expected and should not be logged gratuitously. A loss of network connectivity that has consequences within an app should be logged at the DEBUG or VERBOSE level (depending on whether the consequences are serious enough and unexpected enough to be logged in a release build).

* A full filesystem on a filesystem that is acceessible to or on behalf of third-party applications should not be logged at a level higher than INFORMATIVE.

* Invalid data coming from any untrusted source (including any file on shared storage, or data coming through just about any network connections) is considered expected and should not trigger any logging at a level higher then DEBUG when it's detected to be invalid (and even then logging should be as limited as possible).

* Keep in mind that the + operator, when used on Strings, implicitly creates a `StringBuilder` with the default buffer size (16 characters) and potentially quite a few other temporary String objects, i.e. that explicitly creating StringBuilders isn't more expensive than relying on the default '+' operator (and can be a lot more efficient in fact). Also keep in mind that code that calls `Log.v()` is compiled and executed on release builds, including building the strings, even if the logs aren't being read.

* Any logging that is meant to be read by other people and to be available in release builds should be terse without being cryptic, and should be reasonably understandable. This includes all logging up to the DEBUG level.

* When possible, logging should be kept on a single line if it makes sense. Line lengths up to 80 or 100 characters are perfectly acceptable, while lengths longer than about 130 or 160 characters (including the length of the tag) should be avoided if possible.

* Logging that reports successes should never be used at levels higher than VERBOSE.

* Temporary logging that is used to diagnose an issue that's hard to reproduce should be kept at the DEBUG or VERBOSE level, and should be enclosed by if blocks that allow to disable it entirely at compile-time.

* Be careful about security leaks through the log. Private information should be avoided. Information about protected content must definitely be avoided. This is especially important when writing framework code as it's not easy to know in advance what will and will not be private information or protected content.

* `System.out.println()` (or `printf()` for native code) should never be used. System.out and System.err get redirected to /dev/null, so your print statements will have no visible effects. However, all the string building that happens for these calls still gets executed.

* *The golden rule of logging is that your logs may not unnecessarily push other logs out of the buffer, just as others may not push out yours.*

### 일관성을 유지하라

마지막으로, 항상 코드의 일관성을 유지하자. 코드를 수정할 일이 생긴다면 몇 분이라도 투자해 코드의 이곳저곳을 살펴보고 코드를 수정할 때 어떤 스타일을 사용할 것인지 결정하라. 어떤 코드가 `if`문 주변으로 스페이스를 사용했다면 당신도 그렇게 해야 한다. 그 코드의 주석이 별표로 만들어진 상자 안에 들어 있다면, 당신의 주석 또한 별표 상자 안에 작성하라.

스타일 가이드라인의 포인트는 코딩에 있어 공통적인 어휘를 사용함으로써 사람들이 당신이 어떻게 말하고 있는지보다 무엇을 말하고 있는지에 집중하게끔 하는 것이다. 이 전반적인 스타일 규칙으로 공통적으로 사용될 어휘를 제시했지만 특정한 경우의 스타일 또한 중요한 법이다. 새롭게 짜넣은 코드가 기존의 코드와 심하게 다른 스타일이라면 다른 사람들이 코드를 읽어내려가는데 적지 않은 불편을 겪게 될 것이다. 그런 상황을 되도록 피하자.

## 자바테스트 스타일 규칙

### 테스트 메소드 네이밍 컨벤션을 따르라

테스트 메소드에 이름을 붙일 때는 무엇을 테스트하고 있는지와 테스트의 대상이 되는 특정 경우를 구분하기 위해 언더스코어를 사용하면 된다. 이 스타일을 따르면 어떤 케이스를 테스트하는지 알아보기가 쉽다.

예: 

	testMethod_specificCase1 testMethod_specificCase2

	void testIsDistinguishable_protanopia() {
	    ColorMatcher colorMatcher = new ColorMatcher(PROTANOPIA)
	    assertFalse(colorMatcher.isDistinguishable(Color.RED, Color.BLACK))
	    assertTrue(colorMatcher.isDistinguishable(Color.X, Color.Y))
	}