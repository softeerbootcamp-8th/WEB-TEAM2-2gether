# 관측성/부하테스트 개선 계획

모니터링 서버(Grafana/Prometheus) 대시보드가 5개로 쪼개져 있어 한눈에 상태를
보기 어렵고, 성공/실패 레이턴시 구분, SSE 연결 수, 정책적 실패 분류 등 여러
관측 공백이 있다는 문제에서 출발했다. 부하테스트로 실측한 데이터를 근거로
SLO를 정하고, 그 SLO를 실제로 검증할 수 있게 계측을 보강한 뒤, 대시보드를
Four Golden Signals 기준으로 재구성한다. 동시에 Redis 도입 전 기준선을
남기고, 그 기준선을 재현 가능하게 측정할 k6 시나리오를 설계한다.

## 배경

- 부하테스트 중 backend/DB 서버(`t4g.micro`, 2vCPU/903MB, swap 사용 중)의
  물리적 한계, HikariCP/Tomcat 커넥터 설정, wallet_holds 데드락, SSE
  브로드캐스트 fan-out 비용 등을 조사하며 다수의 관측 공백을 발견했다.
- SSE 아키텍처 자체는 [`../realtime/1-sse-architecture.md`](../realtime/1-sse-architecture.md)에서
  이미 "MVP는 전체 연결, 트래픽 문제가 실제로 발생하면 그때 보이는 항목만
  구독"하기로 팀이 합의했다. 이번 부하테스트가 바로 그 트리거 조건에
  해당하는지를 데이터로 확인하는 게 이 계획의 목적 중 하나다.
- 입찰 처리 구간 메트릭(`dbidding_bid_*`)은 이미 `docs/eunki/2026-08-09-bid-load-test-metrics-plan.md`로
  구현되어 있다. 이 계획은 그 위에 SSE/Tomcat/실패분류 쪽 공백만 추가로 메운다.

## 실행 순서

| 순서 | 상태 | 문서 | 완료 결과 |
|---|---|---|---|
| 1 | 대기 | [SLO/Error Budget 정의](1-slo-error-budget.md) | 엔드포인트별 목표치, 정책적 실패 분류 기준 |
| 2 | 대기 | [관측 공백 및 계측 보강](2-metrics-gap-and-instrumentation.md) | SSE 게이지/타이머, Tomcat 커넥터별 메트릭, DiscardPolicy 카운터 |
| 3 | 대기 | [Grafana 대시보드 재구성](3-grafana-dashboard-redesign.md) | Four Golden Signals + Business 구조의 통합 대시보드 |
| 4 | 대기 | [Redis 도입 전 기준선](4-redis-baseline-comparison.md) | 재현 가능한 pre-Redis 베이스라인 데이터 |
| 5 | 대기 | [k6 부하테스트 시나리오 설계](5-k6-scenario-design.md) | 순수 처리량/실사용 패턴 시나리오, 정책적 실패 처리 통일 |

문서 2·3은 문서 1의 목표치를 전제로 하고, 문서 5는 문서 1의 SLO를 pass/fail
기준으로 그대로 사용한다. 문서 4(베이스라인 측정)는 문서 5의 시나리오가
먼저 정리돼야 실행 가능하다.

## 공통 원칙

- 이 폴더의 계측 변경은 기존 `dbidding_bid_*`, `dbidding_wallet_*`,
  `dbidding_auction_*` 메트릭 이름/태그를 바꾸지 않는다 — 새 메트릭만 추가한다.
- 메트릭 태그에 auction/user/bid/request ID를 넣지 않는다(카디널리티 제약,
  `docs/eunki` 계획과 동일한 제약 유지).
- SLO 윈도우는 30일 rolling 같은 상시 운영 개념 대신 **부하테스트 세션
  단위**로 잡는다 — 이 프로젝트는 상시 트래픽이 아니라 세션 단위로 부하를
  가하는 구조이기 때문이다.
