# 개발자를 위한 코드 스타일 가이드라인

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

> 누구든 비어 있는 캐치(`catch`) 구문을 만들 때는 뭔가 섬뜩한 기분이 들어야 한다. 물론 그렇게 하는 것이 옳은
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

* 예외를 캐치한 뒤 새로운 `RuntimeException`을 발생시킨다. 위험한 방법이므로 해당 에러가 발생했을 때
올바른 처리 방법이 크래시(crash)라고 확신이 있을 때만 사용하라.

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