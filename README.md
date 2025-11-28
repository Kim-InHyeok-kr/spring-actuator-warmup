# Warmup 및 서비스 준비 상태(Health / Readiness) 개선 가이드

## 개요

애플리케이션 기동 직후 발생할 수 있는 초기 지연 및 불안정한 상태를 최소화하기 위해  
다음과 같이 **Warmup 로직**, **Actuator Health 연동**, **Kubernetes Readiness Probe 연계**를 통해  
서비스 준비 상태를 정교하게 관리하도록 구성하였습니다.

---

## 1. 주요 개선 사항

### 1.1 Warmup 기능 도입
애플리케이션 기동 직후 발생할 수 있는 초기 지연을 최소화하기 위해  
DB, Redis 등 핵심 자원에 대한 사전 준비(Warmup) 로직을 구성하였습니다.

- 애플리케이션 시작 시 Warmup 로직 자동 실행
- 주요 인프라 자원(DB, Redis)의 연결 가능 여부 및 기본 동작을 사전에 검증
- 초기 트래픽 유입 시 오류 및 타임아웃 발생 가능성 감소

---

### 1.2 Actuator Health Check 연동
기존 Actuator Health 체크 Bean에 **Warmup 검증 Bean**을 추가하여  
애플리케이션의 **실제 서비스 가능 여부(Service Readiness)** 가 Health 결과에 반영되도록 개선하였습니다.

- Warmup이 완료되지 않은 상태에서는 `readiness` 상태를 `OUT_OF_SERVICE` 또는 `DOWN`으로 반환
- Warmup이 완료된 이후에만 `UP` 상태로 전환되도록 설계
- 운영/모니터링 시스템에서 보다 신뢰성 있는 상태 확인 가능

---

### 1.3 Kubernetes Readiness Probe 연계
Actuator Health 체크 결과를 Kubernetes Readiness Probe와 연동하여,  
**컨테이너가 완전히 준비되기 전까지는 실제 서비스 트래픽이 유입되지 않도록** 구성합니다.

- `readiness` 그룹 Health 엔드포인트를 Kubernetes Readiness Probe에 매핑
- 롤링 배포 시, 신규 파드에 대한 트래픽 전환 시점을 Warmup 완료 기준으로 제어
- 배포 직후 발생할 수 있는 일시적 오류 및 장애 가능성 최소화

---

## 2. 구성 요소

### 2.1 Maven 의존성 추가

Warmup 상태를 Actuator Health에 연동하기 위해 `spring-boot-starter-actuator` 의존성을 추가합니다.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

---

### 2.2 application.yaml 설정

Actuator Health 그룹 및 Readiness 구성을 위한 예시 설정입니다.

```yaml
# Actuator 설정
management:
  server:
    port: {포트번호}   # 예: 8081

  health:
    diskspace:
      enabled: false   # 필요 시 비활성화

    endpoint:
      # 필요 시 개별 endpoint 세부 설정 가능

  endpoint:
    health:
      group:
        readiness:
          include: db, rabbit, redis, warmup
          show-components: always
          show-details: always
      enabled: true

  endpoints:
    enabled-by-default: false
    web:
      exposure:
        include: health  # /actuator/health 및 /actuator/health/readiness 노출
```
### 참고
- readiness 그룹에 warmup 인디케이터를 포함시켜 Warmup 상태가 Readiness 판단에 반영되도록 합니다.
- 필요에 따라 db, rabbit, redis 구성 요소를 추가/제외할 수 있습니다.

---
## 3. Warmup 관련 Java 클래스 구조 (예시)

### 3.1 Warmup 상태 통합 HealthIndicator

```java
@RequiredArgsConstructor
public class WarmupHealthIndicator implements HealthIndicator {

    private final WarmupStatus warmupStatus;

    @Override
    public Health health() {
        if (!warmupStatus.isWarmedUp()) {
            return Health.outOfService()
                    .withDetail("resultMsg", "Application is warming up")
                    .build();
        }

        return Health.up()
                .withDetail("resultMsg", "Application warmed up")
                .build();
    }
}
```
- Actuator HealthIndicator를 구현하여 Warmup 상태를 Readiness에 반영
- Warmup 완료 여부에 따라 `UP / DOWN` 상태를 반환
- Kubernetes Readiness Probe에 직접 연결되는 핵심 컴포넌트

### 3.2 Application 기동 시 Warmup 작업 실행 Runner
```java
@Slf4j
@RequiredArgsConstructor
public class WarmupRunner implements ApplicationListener<ApplicationReadyEvent> {

    private final WarmupStatus warmupStatus;

    private final List<WarmupChecker> warmupCheckers;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("WarmupRunner::onApplicationEvent() - Warm-up started.");

        // Warmup 구현
        
        warmupStatus.markWarmedUp();

        log.info("WarmupRunner::onApplicationEvent() - Warm-up finished.");
    }
```
- 애플리케이션 기동 직후 실행되는 Warmup 로직 수행
- DB / Redis 등 주요 자원의 초기 연결 시도 및 준비 작업 실행
- Warmup 결과를 `WarmupStatus`에 기록

### 3.3 WarmupStatus : Warmup 진행 상태 관리 객체

```java
public class WarmupStatus {

    private final AtomicBoolean warmedUp = new AtomicBoolean(false);

    public boolean isWarmedUp() {

        return warmedUp.get();
    }

    public void markWarmedUp() {

        warmedUp.set(true);
    }
}
```
- Warmup 실행 상태를 저장/관리하는 상태 객체
- Warmup 완료 여부, 실패 여부, 에러 메시지 등을 보관
- HealthIndicator에서 Warmup 상태를 조회할 때 사용

---

## 4. 기대 효과
- 서비스 기동 직후의 초기 지연 감소
- 준비되지 않은 상태의 트래픽 유입 방지
- Health/Readiness 모니터링의 신뢰성 향상
- Kubernetes 롤링 배포 시 안정성 강화

