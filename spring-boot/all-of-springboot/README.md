
스프링 부트를 많이 안다고 생각했지만 아직 모르는 부분이 많다고 생각합니다. 그래서 이번 글에서는 김영한님의 '스프링 부트 - 핵심 원리와 활용' 강의를 듣고 정리한 내용에 대해 이야기해보고자 합니다. 

## 스프링 부트 

스프링 프레임워크는 의존성 주입 프레임워크로서 자바 개발자들에게 많은 편의성을 주었습니다. 하지만 여전히 개발자들을 어렵게 하는 것들이 있었습니다. 
- WAS를 설치하고 WAS에 빌드 파일을 업로드 해야 한다. 
- 버전관리가 어렵다. 
- 필요한 빈들을 일일히 등록해주어야 한다. 

위와 같이 스프링 프레임워크를 사용하더라도 해결되지 않는 부분들이 있었고, 이러한 문제들을 해결하는 것이 '스프링 부트'입니다.
지금 부터 어떻게 스프링 부트가 위에 나열한 문제점들을 해결하였는지에 대해 이야기해보겠습니다. 

## 내장 톰캣 

먼저 스프링에서 톰캣(WAS)으로 어떻게 어플리케이션을 구동하는지에 대해 설명하겠습니다. 

WAS를 사용하기 위해 필요한 기본적인 초기화 과정이 있습니다. 
- 서블릿 컨테이너 초기화 
- 어플리케이션 초기화 

서블릿 컨테이너 초기화를 하기 위해서 ```ServletContainerInitializer``` 인터페이스를 구현하고, 구현 클래스를 ````resources/META-INF/services/jakarta.servlet.ServletContainerInitializer```` 경로에 입력해야 합니다. 

spring-MVC도 서블릿 컨테이너 초기화를 위해 다음 경로에 org.springframework.web.SpringServletContainerInitializer 라는 서블릿 컨테이너 초기화 클래스를 지정했습니다.

![img_1.png](img_1.png)

WAS의 규칙에 의해, WAS가 구동하는 시점에 ```SpringServletContainerInitializer```의 onStartup 메서드를 호출합니다. 


서블릿 컨테이너 초기화 이후에 어플리케이션 초기화가 진행됩니다. 

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
    ...
}
```

방금전에 언급했던 ```SpringServletContainerInitializer``` 클래스 위에 ```@HandlesTypes(WebApplicationInitializer.class)``` 어노테이션을 볼 수 있을 것입니다. 
```@HandlerTypes``` 어노테이션에 어플리케이션 초기화 인터페이스를 지정하면 ```onStartup``` 메서드의 파라미터로 넘어오는 ```Set<Class<?>> c```에 애플리케이션 초기화 인터페이스의 구현체들을 모두 찾아서 클래스 정보로 전달합니다.

스프링에서는 ```WebApplicationInitializer``` 인터페이스를 어플리케이션 초기화 인터페이스로 지정했습니다. 따라서 서블릿 컨테이너 초기화 시점에 ```WebApplicationInitializer``` 인터페이스의 구현체들이 파라미터로 넘어옵니다. 

![img_2.png](img_2.png)

위의 내용은 모두 서블릿 컨테이너 위에서 동작하는 방식입니다. 따라서 항상 톰캣 같은 서블릿 컨테이너에 배포를 해야만 동작하는 방식입니다. 


### WAR 배포 방식의 단점 

WAS에 WAR 파일을 배포하는 방식은 번거롭습니다. 먼저 WAS를 설치해야 하고, WAR 파일을 그 위에 배포해야 합니다. 
스프링 부트는 이러한 번거로움을 제거하고 ```main()``` 메서드만 실행하면 웹 서버까지 실행되도록 프로세스를 단순화했습니다. 기존의 스프링 프레임워크는 톰캣에 WAR를 배포했다면, 스프링 부트는 JAR 파일안에 톰캣 라이브러리도 함께 포함된 형태입니다. 

![img_3.png](img_3.png)

스프링 부트에서는 main 메서드 호출을 통해 톰캣을 시작하기 때문에 jar 안에 ```META-INF/MANIFEST.MF``` 파일에 실행한 ```main()``` 메서드의 클래스를 지정해주어야 합니다.
```
task buildJar(type: Jar) {
     manifest {
         attributes 'Main-Class': 'com.example.StartApplicationClass'
     }
with jar }
```

```
Manifest-Version: 1.0
 Main-Class: com.example.StartApplicationClass
```
