# MSA 기반 게임 키 유통 플랫폼
- 본 프로젝트는 Spring Boot 기반의 마이크로서비스 아키텍처(MSA)로 설계된 게임 구매 및 키 발급 API 서버입니다.
- 게임사와의 제휴 시나리오를 바탕으로, 결제 완료 시 해당 플랫폼의 활성화 키를 즉시 제공하는 비즈니스 로직을 구현했습니다.

---

# 사용한 메인 라이브러리
- **Framework**: Spring Boot v3.5.9, Spring Cloud 2025.0.1

- **Database & Cache**: MariaDB v10.6 (Sharding), Redis

- **Message Broker**: Kafka (Core Event), RabbitMQ (Spring Cloud Bus)

- **Data Pipeline**: Kafka Connect (CDC for Shared DB Integration)

- **Monitoring & Tracing**: Prometheus, Grafana, Zipkin (Micrometer Tracing)

- **Resilience**: Resilience4j (Circuit Breaker)

---

# 아키텍처 & 배포 구조도
<img src="https://github.com/hyuk-s-ecommerce-msa-project/.github/blob/main/images/%EB%B0%B0%ED%8F%AC%EA%B5%AC%EC%A1%B0%EB%8F%84.png?raw=true">

- Azure 클라우드 환경에서 AKS(Azure Kubernetes Service)를 활용하여 서비스 계층을 컨테이너화하고, 가용성과 확장성을 확보한 MSA 환경을 구축함
- 애플리케이션(K8s)과 데이터/메시징 인프라(VM)를 물리적으로 격리하여 인프라의 독립적인 확장성(Scalability)을 확보하고, 유지보수 효율성을 높인 멀티 인스턴스 아키텍처를 설계함

---

<img src="https://github.com/hyuk-s-ecommerce-msa-project/.github/blob/main/images/Gameinfo.png?raw=true">

---

# 서비스 핵심 기능

## 회원
- Argon2id 비밀번호 해싱: 기존 BCrypt보다 강력한 보안을 위해 메모리 하드(Memory-hard) 특성을 가진 Argon2 알고리즘을 적용하여 GPU 병렬 공격 방어
- JWT : 짧은 주기의 AccessToken과 긴 주기의 RefreshToken을 운용하여 사용자 편의성과 보안성 사이의 균형 확보
- MSA 가용성 및 장애 내성 (Resilience)
  - Circuit Breaker 적용: Order-Service 호출 시 Spring Cloud Circuit Breaker를 적용하여, 주문 서비스 장애가 유저 서비스 전체로 확산되는 '장애 전파(Cascading Failure)' 방지
  - Fallback 로직 구현: 타 서비스 장애 시에도 빈 리스트를 반환하는 등의 Fallback 처리를 통해 사용자에게 최소한의 기능(유저 기본 정보 조회 등)은 중단 없이 제공

## 주문
- 주문(Order) 서비스는 프로젝트 내에서 가장 높은 트래픽과 쓰기 부하가 집중되는 핵심 도메인
- 데이터베이스 샤딩 적용
  - ShardContext 기반 부하 분산: userId의 해시값을 이용한 샤딩 로직을 구현하여 데이터를 물리적으로 분산 저장함으로써 단일 DB의 성능 병목 현상 해결
  - ThreadLocal 활용: ShardContextHolder를 통해 트랜잭션 범위 내에서 동적으로 데이터 소스를 결정하고, 작업 완료 후 컨텍스트를 클리어하여 자원 격리 보장
- 메시지 기반 최종 일관성
  - Transactional Outbox Pattern: 주문 생성/취소 시 DB 트랜잭션 내에서 Outbox 테이블에 이벤트를 함께 저장하여, 서비스 장애 시에도 메시지 발행의 원자성 보장
  - 상태 동기화: 주문의 생명주기(CREATED, COMPLETED, CANCELED) 변화를 Kafka 이벤트를 통해 전파하여 타 서비스와의 데이터 정합성 유지
- 실시간 데이터 동기화
  - Source Connector 활용: MariaDB Shard 0, 1 각각에 커넥터를 배치하여 주문 및 결제 데이터의 변경 사항을 실시간으로 감지하고 Kafka 토픽으로 발행
  - 중앙 통합 데이터베이스 구축: 분산된 샤드 데이터를 Sink Connector를 통해 하나의 중앙 DB로 통합하여, 복잡한 통계 쿼리 및 전체 주문 이력 조회 성능을 최적화함
- 시스템 안정성 및 확장성
  - 장애 내성(Fault Tolerance): 개별 샤드 DB나 서비스에 장애가 발생하더라도, Kafka Connect의 오프셋 관리를 통해 복구 시 누락 없는 데이터 재처리가 가능하도록 설계
- 비즈니스 가치
  - 이벤트 기반 아키텍처의 핵심: 데이터의 흐름을 서비스 간 직접 호출이 아닌 '데이터 변경 이벤트'로 처리함으로써 서비스 간 결합도를 낮추고 데이터 활용성을 높임

## 결제
- 외부 결제 프로세스 관리
  - 2-Phase Commit 시뮬레이션: 결제 준비(ready)와 승인(approve) 단계를 분리하여 외부 결제 엔진과의 동기화 상태를 정교하게 관리함
  - 결제 무결성 검증: 클라이언트의 결제 요청 금액과 서버의 실제 주문 금액(Total Amount)을 이중 검증하여 데이터 위변조 및 부정 결제 방지
- 보상 트랜잭션
  - 원자성 보장: 카카오페이 승인이 완료된 후, 내부 서비스(Order, Key)의 후속 처리가 실패할 경우를 대비한 예외 처리 로직 구축
  - 자동 환불 프로세스: 서비스 간 통신 오류 발생 시, 즉시 KakaoPay Cancel API를 호출하여 사용자의 결제 금액을 실시간으로 환불 처리
  - 재고 데이터 정합성: 전체 프로세스 실패 시 할당되었던 게임 키를 즉시 회수(revokeKeys)하여 시스템 전체의 데이터 일관성 유지
    
## 게임 키 관리
- 게임 활성화 키의 생애주기를 관리하며, 대량의 결제 요청 상황에서도 데이터 경합 없이 안전하게 재고를 할당하도록 설계
- 고성능 재고 할당 및 동시성 제어
  - 동시성 최적화: 다수의 사용자가 동시에 동일한 상품을 구매할 때 발생하는 데이터 경합을 방지하기 위해 Database 특화 쿼리(SKIP LOCKED)를 활용하여 성능 저하 없는 안전한 재고 할당 로직 구현
  - 상태 기반 생애주기 관리: AVAILABLE -> RESERVED -> SOLD로 이어지는 엄격한 상태 관리를 통해 중복 판매 방지
- 자원 회수 및 데이터 이력 관리
  - 자동 재고 복구: 결제 실패나 주문 취소 시, 예약된 키를 즉시 AVAILABLE 상태로 복구하여 재고 가용성을 실시간으로 확보
  - 히스토리 추적: KeyHistoryEntity를 통해 각 키의 상태 변화 이력을 모두 기록하여, 추후 고객 응대나 데이터 감사에 활용 가능한 데이터 구조 설계
- 외부 데이터 연동 및 관리 효율화
  - 동적 재고 로드: Google Sheets CSV API를 연동하여 관리자가 별도의 DB 접근 없이도 대량의 게임 키를 실시간으로 업로드하고 시스템에 반영할 수 있는 편의 기능 구현

## 게임 상품
- 복합 상품 메타데이터 관리
  - 다중 상품 조회 인터페이스: 주문 서비스 등에서 대량의 상품 정보를 한 번에 요청할 때 부하를 줄이기 위한 일괄 조회(findByProductIdIn) 로직 구현
- 재고 및 상품 상태 동기화
  - 유연한 재고 관리: 결제 취소나 주문 실패 시 상품의 가용 재고를 실시간으로 복구(increaseStock)하여 판매 기회 손실을 방지
- Steam API
  - 외부 게임 플랫폼의 데이터를 손쉽게 수용할 수 있도록 장르, 카테고리 등을 독립적인 엔티티로 분리하여 유연한 상품 확장성 확보

## 인프라 및 배포
- 모든 마이크로서비스를 컨테이너화하여 로컬 개발 환경과 Azure 클라우드 배포 환경 사이의 격차를 해소하고, 서비스 독립적인 확장성을 확보
- 서비스 컨테이너화 및 이미지 최적화
  - Docker 이미지 빌드: 각 서비스(User, Order, Pay 등)를 독립적인 Docker 이미지로 빌드하여 OS 및 라이브러리 의존성 문제를 원천 차단
- 환경 분리 및 가용성 확보
  - 인스턴스 분리 배포: Azure 인프라 활용 시 서비스 서버 인스턴스와 데이터베이스/메시지 브로커 전용 인프라 인스턴스를 분리하여 배포함으로써 시스템 안정성 강화
- 클라우드 환경과의 결합
  - Docker 컨테이너를 기반으로 Azure 클라우드 환경에 서비스를 배포하여 서비스 간 네트워크 격리 및 보안 그룹을 통한 접근 제어 구현
- 별도의 인스턴스로 인프라와 서비스 분리
  - 상태 보존의 안정성 : 서비스 서버는 Stateless하여 자유로운 스케일링이 가능하지만, DB나 Kafka 같은 Stateful 서비스는 K8s 내에서 Volume 관리 및 파드 재시작 시 데이터 안정성 리스크가 존재. 이를 방지하기 위해 데이터 인프라는 물리적으로 격리된 전용 인스턴스에서 운영
  - 성능 최적화: Disk I/O 부하가 큰 Kafka와 MariaDB를 K8s 노드 자원과 분리함으로써, 애플리케이션 연산과 데이터 처리가 서로의 성능에 간섭하지 않는 환경을 구축
  - 유연한 확장성(Agility): 트래픽 변동이 심한 서비스 서버(Order, Pay 등)를 컨테이너 단위로 관리하여 필요한 순간에 즉각적으로 스케일 아웃할 수 있는 구조를 확보

