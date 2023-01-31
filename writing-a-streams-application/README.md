# STREAMS 애플리케이션 작성

Apache Kafka Streams 애플리케이션을 작성하는 방법을 설명합니다.   
<br />

Kafka Streams Docs 를 참고하면서 작성하였습니다.   
<a href="https://kafka.apache.org/33/documentation/streams/developer-guide/write-streams.html">Kafka Streams Docs</a>
<br />
<br />
<br />
<br />



## 목차
- <a href="#maven">라이브러리 및 Maven 아티팩트</a>   
- <a href="#kafka-streams">애플리케이션 코드 내에서 Kafka Streams 사용</a>   
- <a href="#test">Streams 애플리케이션 테스트</a>   

<br />
<br />
<br />
<br />

Kafka Streams 라이브러리를 사용하는 모든 Java 또는 Scala 애플리케이션은 Kafka Streams 애플리케이션으로 간주됩니다.   
Kafka Streams 애플리케이션의 연산 논리는 스트림 프로세서(노드)와 스트림(에지)의 그래프인 프로세서 토폴로지로 정의됩니다.   
<br />

Kafka Streams API 를 사용하여 프로세서 토폴로지를 정의할 수 있습니다.
- Kafka Streams DSL
  - `map`, `filter`, `join`, `aggregations`와 같은 가장 일반적인 데이터 변환 작업을 제공하는 높은 수준의 API 입니다.   
  DSL 은 Kafka Streams 을 처음 접하는 개발자에게 권장되는 시작점이며 많은 사용 사례와 스트림 처리 요구 사항을 처리해야 합니다.   
  Scala 애플리케이션을 작성하는 경우 Java DSL 로 직접 작업하는 대신 Java/Scala 커플링의 상당 부분을 제거하는 Scala 라이브러리용 Kafka Streams DSL 을 사용할 수 있습니다.   
<br />

- Processor API
  - 프로세서를 추가 및 연결하고 상태 저장소와 직접 상호 작용할 수 있는 저수준 API 입니다.   
  Processor API 는 DSL 보다 훨씬 더 많은 유연성을 제공하지만 애플리케이션 개발자 측에서 더 많은 수작업(예: 더 많은 코드 라인)을 요구합니다.   
<br />
<br />
<br />
<br />

<div id="maven"/>

## 라이브러리 및 Maven 아티팩트
Kafka Streams 애플리케이션 작성에 사용할 수 있는 Kafka Streams 관련 라이브러리입니다.   
애플리케이션에 대해 다음 라이브러리에 대한 종속성을 정의할 수 있습니다.   
<br />

|Group ID|Artifact ID|Version|Description|
|----|-----|-----|-----|
|`org.apache.kafka`|`kafka-streams`|`3.3.1`| (필수) Kafka Streams용 기본 라이브러리입니다.|
|`org.apache.kafka`|`kafka-clients`|`3.3.1`| (필수) Kafka 클라이언트 라이브러리. 내장 직렬 변환기/역직렬 변환기를 포함합니다.|
|`org.apache.kafka`|`kafka-streams-scala`|`3.3.1`| (선택) Scala Kafka Streams 애플리케이션을 작성하기 위한 Scala 라이브러리용 Kafka Streams DSL 입니다. <br/>SBT를 사용하지 않는 경우 애플리케이션에서 사용 중인 올바른 Scala 버전( `_2.12`, `_2.13`) 으로 Artifact ID Suffix 를 추가해야 합니다. |
<br />

> `Serializer`, `Deserializer`에 대한 자세한 내용은 <a href="">데이터 유형 및 직렬화</a>을 참조하십시오 .
<br />

Maven 사용 시 `pom.xml` 예시
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.3.1</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.3.1</version>
</dependency>
<!-- (선택) Kafka Streams DSL for Scala 2.13을 포함합니다 -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams-scala_2.13</artifactId>
    <version>3.3.1</version>
</dependency>
```
<br />
<br />
<br />
<br />

<div id="kafka-streams"/>

## 애플리케이션 코드 내에서 KAFKA STREAMS 사용
애플리케이션 코드의 어느 곳에서나 Kafka Streams 을 호출할 수 있지만 일반적으로 이러한 호출은 애플리케이션의 `main()` 메서드 또는 일부 변형 내에서 이루어집니다.   
애플리케이션 내에서 프로세싱 토폴로지를 정의하는 기본 요소는 다음과 같습니다.   
<br />

먼저 `KafkaStreams` 의 인스턴스를 생성해야 합니다.   
- `KafkaStreams` 생성자의 첫 번째 인수는 토폴로지를 정의하는 데 사용되는 `Topology`(DSL 용 `StreamsBuilder#build()` 또는 Processor API 용 `Topology`)를 사용합니다.   
<br />

- `KafkaStreams` 생성자의 두 번째 인수는 특정 토폴로지에 대한 구성을 정의하는 속성으로, `java.util.Properties`의 인스턴스 입니다.   
<br />

```java
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.kstream.StreamsBuilder;
import org.apache.kafka.streams.processor.Topology;

// 빌더를 사용하여 실제 처리 토폴로지를 정의합니다. 
// 예를 들어, 읽을 입력 항목, 호출할 스트림 작업(filter, map 등) 등을 지정합니다.
class Example {
    StreamsBuilder builder = ...;  // DSL 을 사용할 경우
    Topology topology = builder.build();
    //
    // 또는
    //
    Topology topology = ...; // Processor API 를 사용할 경우

    // Configuration 을 사용하여 애플리케이션에 Kafka 클러스터의 위치, 
    // 기본적으로 사용할 Serializers/Deserializers 를 알려주고 
    // 보안 설정을 지정하는 등의 작업을 수행합니다.
    Properties props = ...;
    
    KafkaStreams streams = new KafkaStreams(topology, props);   
}
```
<br />

이 시점에서 내부 구조는 초기화되지만 처리는 아직 시작되지 않습니다.   
`KafkaStreams#start()` 메서드를 호출하여 Kafka Streams 스레드를 명시적으로 시작해야 합니다.   
```java
// Kafka Streams 스레드 시작
streams.start();
```
다른 곳(ex: 다른 머신)에서 이 스트림 처리 애플리케이션의 다른 인스턴스 실행 중인 경우 Kafka Streams 은 기존 인스턴스에서 방금 시작한 새 인스턴스로 작업을 투명하게 재할당합니다.   
자세한 내용은 <a href="https://github.com/jeongwon201/kafka-streams/tree/main/kafka-streams-architecture#stream-partition-task">스트림 파티션과 작업</a> 및 <a href="https://github.com/jeongwon201/kafka-streams/tree/main/kafka-streams-architecture#threading-model">스레딩 모델</a> 을 참조하세요 .   
<br />

예기치 않은 예외를 포착하려면 애플리케이션을 시작하기 전에 `java.lang.Thread.UncaughtExceptionHandler`를 설정할 수 있습니다.   
이 핸들러는 스트림 스레드가 예기치 않은 예외로 인해 종료될 때마다 호출됩니다.   
```java
// 자바 8 이상, 람다 표현식 사용
streams.setUncaughtExceptionHandler((Thread thread, Throwable throwable) -> {
    // 여기서 Throwable 또는 Exception 을 검토하고 적절한 조치를 수행해야 합니다!
});


// 자바 7
streams.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
    public void uncaughtException(Thread thread, Throwable throwable) {
        // 여기서 Throwable 또는 Exception 을 검토하고 적절한 조치를 수행해야 합니다!
    }
});
```
<br />

애플리케이션 인스턴스를 중지하려면 `KafkaStreams#close()`메소드를 호출하십시오.   
```java
// Kafka Streams 스레드 중지
streams.close();
```
<br />

SIGTERM 에 응답하여 응용 프로그램을 정상적으로 종료하려면 종료 후크를 추가하고 `KafkaStreams#close`를 호출하는 것이 좋습니다.   
- Java 8 이상
  ```java
  // Add shutdown hook to stop the Kafka Streams threads.
  // You can optionally provide a timeout to `close`.
  Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
  ```
<br />

- Java 7
  ```java
  // Add shutdown hook to stop the Kafka Streams threads.
  // You can optionally provide a timeout to `close`.
  Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
      @Override
      public void run() {
          streams.close();
      }
  }));
  ```
<br />

애플리케이션이 중지된 후 Kafka Streams 은 인스턴스에서 실행 중이던 모든 작업을 사용 가능한 나머지 인스턴스로 마이그레이션합니다.   
<br />
<br />
<br />
<br />

<div id="test"/>

## STREAMS 애플리케이션 테스트
Kafka Streams 은 다음 링크에서 애플리케이션을 테스트하는 데 도움이 되는 `test-utils` 모듈과 함께 제공됩니다.   
<a href="">Streams 애플리케이션 테스트</a>