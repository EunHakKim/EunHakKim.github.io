---
title: "Swagger로 API 문서 자동화하기"
date: 2024-11-24 02:03:00 +09:00
categories: [백엔드, 스프링]
tags: [백엔드, 자바, 스프링, Swagger, API문서]
image:
  path: /assets/img/thumbnail/swaggerlogo.jpg
  alt: thumbnailLogo
---

## Swagger
- 개발자가 REST 웹 서비스를 설계, 빌드, 문서화, 소비하는 일을 도와주는 대형 도구 생태계의 지원을 받는 오픈 소스 소프트웨어 프레임워크
- `Spring-Fox`
    - 오래전에 나온 라이브러리이다.
    - 2020년 이후로 업데이트가 없다.
- `Spring-Doc`
    - 2019년에 나온 라이브러리이다.
    - 꾸준하게 업데이트가 되고 있다.
- 스프링부트 `build.gradle`에 아래와 같은 의존성을 추가하면 바로 사용 가능 (스프링부트 3.0 부터는 `Spring-fox`가 지원을 종료해서 아래와 같은 명령어로만 가능)

```groovy
// Swagger
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.4.0"
```

- SwaggerConfig 클래스

```java
import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;
import lombok.RequiredArgsConstructor;
import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@OpenAPIDefinition(
        info = @Info(title = "API 명세서 제목",
                description = "설명",
                version = "v1"))
@RequiredArgsConstructor
@Configuration
public class SwaggerConfig {
    @Bean
    public GroupedOpenApi chatOpenApi() {
        return GroupedOpenApi.builder()
                .group("그룹명")
                .pathsToMatch("/**")
                .build();
    }
}
```

- 아래와 같은 주소로 접근 가능하다.

```
http://localhost:8080/swagger-ui.html
```

- 컨트롤러에서 여러 어노테이션을 통해서 추가적인 정보를 기록할 수 있다.

```java
@Tag(name = "Story API", description = "동화 API")
public class StoryController
```

```java
@Operation(summary = "동화 생성")
@PostMapping
public ResponseEntity<StoryDto.StoryCreationResponse> generateStory
```

---

## Swagger-Editor

- API 명세서 작성용 에디터
- 작성하면서 실시간으로 확인 가능
- 아래와 같은 코드로 도커를 통해 실행 가능 (http://localhost:80/으로 접속하면 쓸 수 있다.)

```bash
docker pull swaggerapi/swagger-editor
docker run -d -p 80:8080 swaggerapi/swagger-editor
```

- 혹은 아래 공식 주소로 바로 사용 가능하다.

```
https://editor.swagger.io/
```