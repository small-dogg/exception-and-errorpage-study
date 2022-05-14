# 예외처리

## 서블릿 예외 처리

**서블릿은 다음 2가지 방식으로 예외 처리를 지원한다**

- `Exception`(예외)
- `response.sendError(HTTP 상태 코드, 오류 메시지)`

### Exception(예외)

**웹 애플리케이션 예외 발생**
웹 애플리케이션은 사용자 요청 별로 별도의 Thread가 할당되고, 서블릿 컨테이너 안에서 실행된다.
애플리케이션에서 예외가 발생했는데 예외를 처리하지 못하고, 서블릿 밖으로 까지 예외가 전달되면
WAS 수준까지 예외가 전달된다.

```text
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
```

예외를 WAS 이전까지 처리하지 못하고, WAS까지 전달되면 WAS가 제공하는 오류 화면이 출력된다.

`Exception`의 경우, 서버 내부에서 처리할 수 없는 경우 Internal Server Error(error code: 500)가 발생한다.

서블릿에서 처리할 수 없는 화면을 요청할 경우에는 404 오류 화면이 출력된다.

### response.sendError(HTTP 상태 코드, 오류 메시지)

Controller 단에서 ServletResponse 객체의 sendError()를 호출하면, WAS에서는 이 송신된 Error를 확인하고,
서블릿 컨테이너가 기본으로 제공하는 오류 화면을 볼 수 있다.

```text
WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError() 호출)
```

### 서블릿 예외 처리 - 오류 화면 제공

서블릿 컨테이너가 제공하는 기본 예외 처리 화면이 아닌 서블릿이 제공하는 오류 화면 기능을 사용한다.
서블릿은 `Exception`이 발생해서 서블릿 밖으로 전달되거나 또는 `response.sendError()`가 호출 되었을 때 각각의
상황에 맞춘 오류 처리 기능을 제공한다.

과거에는 web.xml에 error-page를 code에 따라 대상 화면을 맵핑하여 보여주곤 했다.

지금은 스프링 부트를 통해서 서블릿 컨테이너를 실행하기 때문에, 스프링 부트가 제공하는 기능을 사용해서 서블릿 오류 페이지를 등록하면 된다.

WebServerFactoryCustomizer를 구현하여, 코드에 따른 에러페이지를 ErrorPage인스턴스로 만들어 전달하면 된다.

```java
package hello.exception;

import org.springframework.boot.web.server.ConfigurableWebServerFactory;
import org.springframework.boot.web.server.ErrorPage;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

@Component
public class WebServerCustomizer implements
        WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/errorpage/404");
        ErrorPage errorPage500 = new
                ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");
        ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "/errorpage/500");
        factory.addErrorPages(errorPage404, errorPage500, errorPageEx);
    }
}
```

## 서블릿 예외 처리 - 오류 페이지 작동 원리
서블릿은 `Exceoption`(예외)가 발생해서 서블릿 밖으로 전달되거나 또는 `response.sendError()`가 호출 되었을 때 설정된 오류 페이지를 찾는다.
Spring Boot Application이 구동되면서 위에서 작성한 Custom한 에러 페이지를 WAS에게 전달하기 때문에, 기존 WAS에서 제공하는 에러페이지가 아닌,
에러코드에 따른 커스텀한 에러 페이지를 출력해줄 수 있게 된다.

**예외 발생 흐름**
```text
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외 발생)
```
**sendError 흐름**
```text
WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())
```

WAS는 전달된 예외 또는 sendError를 확인하고, 오류 페이지 정보를 기반으로 대상 경로를 다시 요청한다.

**오류 페이지 요청 흐름**
```text
WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

전체 흐름은 아래와 같다.
**예외 발생과 오류페이지 요청 흐름**
```text
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외 발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

** 서블릿 예외 처리 - 필터
예외가 발생한 상황에서 전체 흐름에서 아래의 2번과 같이 다시 요청을 하게되면, 필터를 두번이나 호출하게 된다.
**예외 발생과 오류페이지 요청 흐름**
```text
1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외 발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View
```

필터는 이런 경우를 위해서 `DispatherTypes`라는 옵션을 제공한다.
처음 요청이 들어왔을 떄는 이 `DispatherTypes`가 `DispatherTypes.REQUEST`로 작성되어 있고,
오류 요청의 경우에는 `DispatherTypes.ERROR`로 작성된다.

그밖에도 DispatherTypes는
- `REQUEST` : 클라이언트 요청
- `ERROR` : 오류 요청
- `FORWARD` : MVC에서 배웠던 서블릿에서 다른 서블릿이나 jsp를 호출할 때 `RequestDispather.forword(request,response):`
- `INCLUDE` : 서블릿에서 다른 서블릿이나 JSP의 결과를 포함할 때 `RequestDispather.include(request, response);`
- `ASYNC` : 서블릿 비동기 호출

이 존재한다.

하여, Filter를 등록할 때, `FilterRegistrationBean<Filter>` 객체에 `setDispatherTypes(타입...)`을 지정하여, 해당 필터가 호출되는
DispatherTypes를 지정하면, Error의 경우 호출되지 않도록 처리하거나 할 수 있다.

물론, 이 `setDispatherTypes()`의 기본값은 REQUEST이기 때문에, 위에서의 경우에는 필터에 특별히 ERROR 타입을 지정하지 않으면
필터를 수행하지 않는다.

## 서블릿 예외 처리 - 인터셉터
인터셉터는 중복 호출을 제거하기 위해, 별도의 `FilterRegistrationBean`과 같이 `DispatherTypes`를 지정하지는 않는다.
기본적으로 제공하는 Interceptor의 `excludePathPatterns` 메서드를 사용하여, 대상 error 파일 경로를 넣어주면
인터셉터를 호출하지 않고 컨트롤러로 넘길 수 있다.

## 스프링 부트 - 오류 페이지
스프링 부트는 위에서 수행한 복잡한 과정을 자동으로 처리해준다.
- `ErrorPage`를 자동으로 등록한다. 이때 `/error`라는 경로로 기본 오류 페이지를 설정한다.
  - `new ErrorPage("/error")` 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용된다.
  - 서블릿 밖으로 예외가 발생하거나, `response.sendError(...)`가 호출되면 모든 오류는 `/error`를 호출하게 된다.
- `BasicErrorController`라는 스프링 컨트롤러를 자동으로 등록한다.
  - `ErrorPage`에서 등록한 `/error`를 매핑해서 처리하는 컨트롤러이다.

**뷰 템플릿 우선순위**
`BasicErrorController`는 출력해줄 뷰에 대한 우선순위가 존재한다.

1. 뷰 템플릿
   - `resources/templates/error/500.xml`
   - `resources/templates/error/5xx.xml`
2. 정적 리소스(`static`,`public`)
   - `resources/static/error/400.html`
   - `resources/static/error/404.html`
   - `resources/static/error/4xx.html`
3. 적용 대상이 없을 때 뷰 이름(`error`)
   - `resources/templates/error.html`