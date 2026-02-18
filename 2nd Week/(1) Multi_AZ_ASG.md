# 1. AWS에서 멀티 AZ 환경에서 ASG가 한쪽 가용 영역 장애 시 인스턴스 분산 및 트래픽 집중을 방지하는 방법은 무엇인가?   
## 1) ASG의 인스턴스 분산 로직과 헬스체크 방식의 내부 동작 원리는?
### 1.1 ASG의 인스턴스 분산 로직
- ASG는 기본적으로 AZ 리밸런싱을 통해 트래픽을 분신 시킨다.(ASG AZ Rebalance Process)
    + Launch Template 기반으로 인스턴스 스펙을 결정
    + ELB 연동 시 Healthy한 인스턴스로만 트래픽 전달
    + AZ 별로 인스턴스 수 비교하여 가장 적은 AZ에 우선 배치
        * On-Demand: 가용성 기반으로 배치
        * Spot: 가격 + 가용성 기반으로 배치
- 트래픽을 분산 시키기 위한 설계
    + 최소 2~3개의 AZ에 트래픽을 분산
    + Capacity Rebalancing 활성화(Spot 사용 시 필수)
    + ELB Cross-Zone Load Balancing 활성화하(ALB는 기본)
- AZ Rebalancing의 발생 조건
    + AZ 장애 후 복구 시
    + AZ를 신규로 ASG에 추가 시
    + Spot 인스턴스 중단 시
    + 수동으로 인스턴스를 AZ에서 종료 시
- 신규로 인스턴스를 생성한 후, 기존 인스턴스를 삭제하기 때문에 max >= desired + 1이 되어야 함
### 1.2 헬스 체크 방식의 내부 동작 원리
- EC2 헬스 체크
    + 확인 대상: 인스턴스 상태(하이퍼바이저 레벨)ks
    + 감지 범위: 하드웨어 장애, 네트워크 단절
    + 반응 속도: 빠름
    + 권장 여부: 기본 설정
- ELB 헬스 체크
    + 확인 대상: 애플리케이션 응답(HTTP, TCP)
    + 감지 범위: 앱 크래시, 메모리 부족
    + 반응 속도: 설정에 따라 다름
    + 권장 여부: 프로덕션 환경 필수
- Grace Period
    + ELB 헬스 체크 기간 동안, 앱 부팅 시간을 고려하여 헬스 체크 결과를 무시하는 기간이며 이 기간이 짧으면 서버가 자주 교체되는 현상 발생
## 2) 멀티 AZ 환경에서 네트워크 파티셔닝이 발생할 때 트래픽 라우팅을 어떻게 최적화할 수 있는가?
- 네트워크 파티셔닝: 가용영역은 살아있으나 네트워크 단절이 발생한 상태
### 2.1 해결 전략
- ALB로 확인하는 법
    + 짧은 인터벌(5s)로 여러번(2~3회) 헬스 체크 시도 후 응답 없으면 애플리케이션 수준에서 확인
- Route53 Healthcheck + Failover Routing
    + AZ 혹은 리전 단위 failover 발생 시 조치 가능
    + DNS 엔드포인트를 변경
- AWS Global Accelerator
    + Anycast 기반으로 Health Check 수행하여 트래픽 자동 우회
    + DNS TTL 영향 없음
## 3) AWS Route53, ALB, NLB의 장애 조치(페일오버) 동작 차이점은?    
### 3.1) AWS Route53
- Failover, Latency Routing이 가능하며 리전 단위(글로벌 수준) 장애 조치로 적합함
- DNS를 활용하여 TTL의 영향을 받으므로 탐지, 조치에 시간이 추가 소요됨
### 3.2) Application Load Balancer(ALB)
- L7 애플리케이션 계층(HTTP, HTTPS)의 장애 탐지 가능
- Cross-Zone 로드 밸런싱을 통해 가용영역 간의 장애 탐지, 조치 가능
- 빠른 장애 탐지 및 조치가 가능함
### 3.3) Network Load Balancer(NLB)
- L4에서 TCP, UDP를 활용해서 장애 탐지 가능
- 초저지연으로 탐지 가능하나 Cross-zone이 기본 비활성화이므로 활성화 필요
- 애플리케이션 수준의 장애 탐지 불가
## 4) ASG와 Spot Instance, On-Demand Instance 혼합 시 장애 복구 전략은?  
- Mixed Instances Policy를 사용하여 가격과 성능을 최적화 할 수 있다.
    + Base Capacity 설정: 최소한의 가용성을 위하여 일정 수량은 On-Demand로 설정하여야 한다.
    + Capacity Rebalancing 활성화: Spot Interruption 발생 시 2분전에 신규 인스턴스를 생성한 후, 기존 인스턴스를 삭제
    + 여러 Spot Pool 사용: Instance Type을 다변화하고 'capacity-optimized' 전략을 사용하면 중단 확률이 가장 낮은 용량 풀에서 스팟 인스턴스를 할당받아 중단 리스크를 최소화한다.
## 5) AZ 장애 시 상태 저장 서비스(RDS, ElastiCache 등)의 고가용성 보장 방법은?
### 5.1) RDS
- Primary + Secondary 구조로, Primary에서 Commit 되면 그대로 Secondary에서도 데이터 적용
- Primary에서 장애 발생 시, DNS 엔드포인트는 변경하지 않고 IP만 변경됨
- Standby는 읽기 불가이며, 단순 장애 대비용임
- RPO를 최대한 0에 가깝게 유지하는 것이 목표이며 최대한 데이터 손실이 없어야 함
### 5.2) ElastiCache
- Primary + Replica 구조로, Primary에서 Replica로 비동기 전파됨
- 약간의 Replica lag이 존재함
- Replica로 읽기 트래픽을 분산 가능함
- 캐시 특성 상, 데이터를 유실을 허용하더라도 빠른 성능 확보가 우선임