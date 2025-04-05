---
title: "SpringBoot에서 Swagger 도입부터 트러블슈팅에 배포까지...(?)"
date: "2025-04-04"
draft: false
---

## 서론

API명세서는 프론트엔드와 백엔드간의 협업을 위해서 가장 필요한 도구입니다.
백엔드 개발자는 API를 설계하고 공유할때 API명세서를 만듭니다. API 명세서를 다루는 대표적인 툴로는 Swagger가 있습니다.
이번에는 Swagger 을 도입하는 과정과 중간에 겪은 트러블슈팅 해결과정 그리고 더나아가서 직접 github-action으로 배포하는 과정을 공유해보도록하겠습니다.

---

## 스프링부트에 스웨거 도입하기

- Java: 17
- SpringBoot: 3.4.4

### build.gradle 에 추가하기

```groovy
dependencies {

 // Swagger UI
 implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.0.2'

    ...생략...
}
```

### application.yaml 에 추가하기

```yaml
spring:
    application:
        name: hhplus-concert
    ...(생략)

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operationSorter: method
```

---

## 도입중 문제 발생

로컬호스트에서 실행후, 브라우저의 URL에 `localhost:8080/swagger-ui/index.html` 에 접속했더니 아래이미지와 같은 에러가 발생했습니다.

![swagger-error](../images/swagger-error.png)

이 에러에 대한 문제점 원인은 GlobalHandlerException 핸들러의 `@RestControllerAdvice` 데코레이터를 붙였기 때문에 발생했습니다.

```java
// GlobalExceptionHandler.java

@RestControllerAdvice
public class GlobalExceptionHandler {

 // 유효성검사 실패로 발생한 예외처리: 400 에러반환
 @ExceptionHandler(InvalidValidationException.class)
 @ResponseStatus(HttpStatus.BAD_REQUEST)
 public ErrorResponse handleInvalidValidationException(InvalidValidationException e) {
  return ErrorResponse.of(HttpStatus.BAD_REQUEST.value(), e.getMessage());
 }
 // 서버내부 오류로 발생한 예외처리: 500 에러반환
 @ExceptionHandler(Exception.class)
 @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
 public ErrorResponse handleException(Exception e) {
  return ErrorResponse.of(HttpStatus.INTERNAL_SERVER_ERROR.value(), e.getMessage());
 }
}
```

실행로그에서 발생한 오류문구는 아래와 같이 기재되어있습니다.
이 에러에 대한 해석을 해보자면, "현재 실행중인 코드가 ControllerAdviceBean(Object) 라는 생성자 메서드가 있는줄 알았는데 실제로는 그 메서드가 없는 Spring 라이브러리를 참조하고 있어서 발생하는 오류이기 때문입니다.

> java.lang.NoSuchMethodError:
> 'void org.springframework.web.method.ControllerAdviceBean.<init>(java.lang.Object)'

Swagger UI는 내부적으로 `[GET] /v3/api-docs` URL을 호출하는데 이 요청을 처리하는 과정에서 `@ControllerAdvice`를 스캔하거나 초기화할때 오류가 발생하여 서버 자체에서 500에러를 리턴하여 스웨거문서 로딩에 실패됩니다.

GlobalExceptionHandler에 부여된 `@RestControllerAdvice` 데코레이터를 제거함으로써 이 문제를 해결했습니다.

- 참고: [[트러블슈팅] Swagger 500에러: Failed to load API definition](https://dev-meung.tistory.com/entry/%ED%95%B4%EC%BB%A4%ED%86%A4-HY-THON-%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-Swagger-500-%EC%97%90%EB%9F%AC-Failed-to-load-API-definition)

---

## 로컬호스트에 있는 swagger API명세를 swaggerHub에 옮길 수 없을까?

### 1. 로컬호스트의 swagger(`localhost:8080/swagger-ui/index.html`)에 접속하기

![swagger-local](../images/swagger-local.png)

### 2. `/v3/api-docs`에 접속해서 json코드를 얻기

위의 이미지처럼 작은 글씨로 `/v3/api-docs` 링크에 들어가게되면 로컬에 띄운 swaggerUI에 대한 JSON 파일을 얻을 수 있습니다. 참고로 pretty 포맷팅보다는 raw 포맷팅으로 json파일을 복사하기를 권장합니다.

![swagger-ui-to-json](../images/swagger-ui-to-json.png)

### 3. 코드 포맷 변경 사이트에 들어가서 json 복붙하여 yaml 로 코드를 변환하기

- 코드 포맷 변경 사이트: [BairesDev](https://www.bairesdev.com/tools/json2yaml/)

### 4. 만들어둔 SwaggerHub 프로젝트에 복붙후 이상없으면 저장하기

SwaggerHub 프로젝트는 SmartBear의 SwaggerHub에 회원가입 후에 프로젝트를 만들 수 있습니다. 신규계정은 30일동안 유료티어 기능을 사용할 수 있어요~!

![swaggerhub-api-hub](../images/swaggerhub-api-hub.png)

- 참고: [Github에 Swagger API 문서를 공유하기](https://velog.io/@gdhi/GitHub%EC%97%90-Swagger-API-%EB%AC%B8%EC%84%9C%EB%A5%BC-%EA%B3%B5%EC%9C%A0%ED%95%98%EA%B8%B0)

---

## github-action을 이용해서 swagger 서버를 배포해보자

직접 localhost에 있는 스웨거 json코드를 yaml로 변환해서 swaggerHub에 직접 복붙하는 방식은 손이 많이 가기때문에 귀찮고 불편했습니다. API 응답결과나 응답상태값이 추가되거나 변경될 수 있을텐데 유지보수 측면에서는 비효율을 해결하고 싶었습니다.

여러가지 방법들이 존재 했습니다. github-action이 가장쉬운방법이긴한데.. 아쉽게도 github-pages를 이용한 방법을하려면 깃헙계정 1개당 1개의 페이지를 만들 수 있는 제약이 있습니다. 저의 경우에는 이 블로그가 github-page를 이용한 방법이기때문에 다른 방법을 찾아봐야했습니다. 고민하던중 깃헙액션의 워크플로우로 작성하는 방법으로 진행해보기로했습니다.

### 워크플로우 파일 작성 (1차)

> deploy-openapi-swaggerhub.yaml

```yaml
name: Deploy OpenAPI Spec to SwaggerHub

on:
  push:
    branches: ["main"]

jobs:
  deploy-openapi:
    runs-on: ubuntu-latest
    env:
      OWNER: sampleswagger-f17
      API_NAME: hh-08-concert
      VERSION: 1.0.0
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Give execute permission to gradlew
        run: chmod +x ./gradlew
      # application 빌드
      - name: Build the application
        run: ./gradlew bootJar

      # application 백그라운드에서 실행
      - name: Run the application in background
        run: |
          java -jar build/libs/concert-0.0.1-SNAPSHOT.jar &
          echo $! > pid.txt
      # 스프링부트가 완전히 올라올때까지 대기
      #
      # http://localhost:8080/actuator/health 는 health-check이며
      # 서버가 잘 동작하는지 확인할 수 있습니다.
      - name: Wait for server to be ready
        run: |
          for i in {1..30}; do
            curl --silent http://localhost:8080/actuator/health && exit 0
            echo "⏳ Waiting for server ..."
            sleep 2
          done
          echo "❌ Server did not start in time"
          cat pid.txt | xargs kill
          exit 1
      # 로컬호스트내 OpenAPI 스펙다운로드
      - name: Download OpenAPI spec
        run: curl http://localhost:8080/v3/api-docs -o openapi.json

      - name: Install SwaggerHub CLI
        run: npm install -g swaggerhub-cli

      - name: Push to swaggerHub
        run: |
          swaggerhub api:push <OWNER>/<API_NAME>/<VERSION> \
            --file openapi.json \
            --token ${{ secrets.SWAGGERHUB_API_KEY }} \
            --visibility public
```

### swaggerHub의 정보 파악하기

스웨거헙은 OWNER / API_NAME / VERSION 으로 구성되어있습니다.
아래의 이미지의 URL을 확인해보시면 쉽게 이해할 수 있습니다.

![swaggerhub-info](../images/swaggerhub-info.png)

### 스웨거 API키 발급과 깃헙시크릿키 값에 넣기

1. swaggerHub의 API 시크릿키 찾기

   ![swaggerhub-secret-1](../images/swaggerhub-secret-1.png)
   ![swaggerhub-secret-2](../images/swaggerhub-secret-2.png)

2. swaggerHub API 시크릿값을 Github 래포지토리 시크릿값에 넣기

   ![swaggerhub-secret-3](../images/swaggerhub-secret-3.png)
   ![swaggerhub-secret-4](../images/swaggerhub-secret-4.png)

### 반복되는 빌드 실패...

![fail-workflow-build](../images/fail-workflow-build.png)

결국은 워크플로우를 했지만 실패가 나왔습니다. 그에대한 이유는 아무래도 스프링부트를 실행하려면 데이터베이스와의 연결이 필요하기 때문입니다. 이 프로세스가 되려면 원격으로 데이터베이스가 필요합니다. 하지만 스웨거만을 배포해야되므로 굳이 클라우드의 DB를 붙일 필요가 없으므로 TestContainers를 하기로 했습니다.

## TestContainer 설정하기

이 프로젝트에서는 Mysql을 사용하기 때문에 TestContainer도 동일한 데이터베이스를 사용해야합니다. build.gradle에 TestContainer 관련해서 추가해야합니다.

```groovy
// build.gradle

dependencies {
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:mysql'
}
```

### 워크플로우파일 작성 (2차)

> application-ci.yml

깃헙액션에서 테스트컨테이너를 실행할때 사용하는 설정값입니다.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
  server:
    port: 8080

spring:
  dataSource:
    url: jdbc:tc:mysql:8.0:///testdb
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
    username: test
    password: test
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    database-platform: org.hibername.dialect.MySQL8Dialect

logging:
  level:
    root: DEBUG
```

> deploy-openapi-swaggerhub.yaml

```yaml
name: Deploy OpenAPI Spec to SwaggerHub

on:
  push:
    branches: ["main", "step05", "step06"]

jobs:
  deploy-openapi:
    runs-on: ubuntu-latest
    env:
      OWNER: sampleswagger-f17
      API_NAME: hh-08-concert
      VERSION: 1.0.0
      JAVA_TOOL_OPTIONS: "-Dorg.slf4j.simpleLogger.defaultLogLevel=debug"
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      # Java 설치
      - name: Set up java
        uses: actions/setup-java@v3
        with:
          java-version: "17"
          distribution: "temurin"

      - name: Give execute permission to gradlew
        run: chmod +x ./gradlew
      # application 빌드
      - name: Build the application
        run: ./gradlew bootJar

      # application 백그라운드에서 실행
      - name: Run the application in background
        run: |
          java -jar build/libs/concert-0.0.1-SNAPSHOT.jar --spring.profiles.active=ci &
          echo $! > pid.txt
      # TestContainer 로그 확인 추가
      - name: Debug logs if server fails
        if: failure()
        run: |
          echo "🧩 Showing last 100 lines from app logs (if any)"
          tail -n 100 build/libs/*.log || echo "No log file"
          echo "🐳 Running Docker containers"
          docker ps -a || echo "Docker info not available"
      # 서버가 통신되는지 확인
      - name: Wait for server to be ready
        run: |
          for i in {1..30}; do
            curl --silent http://localhost:8080/actuator/health && exit 0
            echo "⏳ Waiting for server ..."
            sleep 2
          done
          echo "❌ Server did not start in time"
          echo "📜 ==== APP LOG START ===="
          cat app.log || echo "No log file found"
          echo "📜 ==== APP LOG END ===="
          cat pid.txt | xargs kill
          exit 1
      # 로컬호스트내 OpenAPI 스펙다운로드
      - name: Download OpenAPI spec
        run: curl http://localhost:8080/v3/api-docs -o openapi.json

      - name: Install SwaggerHub CLI
        run: npm install -g swaggerhub-cli

      - name: Push to swaggerHub
        run: |
          swaggerhub api:push $OWNER/$API_NAME/$VERSION \
            --file openapi.json \
            --token ${{ secrets.SWAGGERHUB_API_KEY }} \
            --visibility public
```

하지만 아쉽게도 `Wait for server to be ready` 단계에서 계속 똑같은 실패에서 발생을합니다. 연결이 되지 않았기 때문인데요. ( 이 문제에 대한 자세한 솔루션을 찾은경우에는 dmsrkd1216@gmail.com 으로 이메일 보내주시면 감사하겠습니다! )

지금 과제를 해야되는데 과제와 상관없는 삽질을 해버리고말았네요 ㅠㅠ 시간이 많이 소요된 관계로 실패로 끝났습니다. 약 3시간 39분정도 시도를 해봤습니다. 결과를 못이뤄낸건 아쉽지만 그래도 후회는 없습니다.

일단은 불편하더라도 직접 제가 복붙해서 사용하는 방안으로 진행해야될거같습니다.
아무래도 테스트컨테이너가 잘 지원한다해도 워크플로우에서의 서버와의 통신이 어려울거같습니다.

만일 성공이 됐다하더라도, 추후에 레디스나, kafka 이 추가된다면 이 워크플로우에도 연결을 해서 반영을 해야될겁니다. TestContainers처럼 워크플로우 실행중에서만 사용되는 소프트웨어가 있는지 직접 찾아봐야될거 같습니다.

- 참고: [협업을 위한 Swagger 배포하기(1)](https://jihyun-devstory.tistory.com/41)
