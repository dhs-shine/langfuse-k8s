# Langfuse Helm ClickHouse 마이그레이션 작업 계획

## 배경
현재 `charts/langfuse`는 Bitnami `clickhouse` 차트(`oci://registry-1.docker.io/bitnamicharts`)에 의존하고 있으며,
기본 이미지도 `bitnamilegacy/clickhouse`, `bitnamilegacy/zookeeper`를 사용하도록 오버라이드되어 있습니다.
Bitnami의 ClickHouse 차트/이미지 유지보수 중단 이슈를 고려해, 유지보수 가능한 대안으로 전환이 필요합니다.

## 목표
- Bitnami ClickHouse 차트 의존성 제거.
- 유지보수되는 ClickHouse 차트/이미지(공식 또는 커뮤니티 활성 프로젝트)로 전환.
- 기존 사용자 값(`clickhouse.*`)과의 호환성 최대 유지.
- 외부 ClickHouse 사용 시나리오(`clickhouse.deploy=false`)는 기존처럼 유지.

## 대안 비교 (결정 필요)

### 옵션 A: ClickHouse 공식 Helm 차트 기반 전환 (우선 검토)
- 장점
  - ClickHouse 프로젝트 생태계와 정합성 높음.
  - 공식 이미지(`clickhouse/clickhouse-server`) 사용 가능.
- 단점
  - Bitnami 차트와 values 구조가 달라 매핑 작업 필요.
  - 기존 테스트/문서 대규모 수정 필요 가능.

### 옵션 B: Altinity Operator/Chart 기반 전환
- 장점
  - 운영환경에서 많이 쓰는 패턴(CHI, operator 기반) 지원.
  - 클러스터 운영 기능이 상대적으로 강함.
- 단점
  - 의존성 복잡도 증가(Operator + CRD).
  - 현재 단일 dependency 패턴과 차이가 커서 마이그레이션 부담 큼.

## 권장 방향
1) **1차 릴리스는 옵션 A(공식 ClickHouse 차트 + 공식 이미지)**로 진행.
2) 옵션 B는 엔터프라이즈/대규모 운영 요구가 명확할 때 별도 트랙으로 검토.

## 구현 단계

### Phase 0. 사전 조사
- 새 차트의 values 구조, 서비스 이름 규칙, 인증/스토리지 파라미터 조사.
- 현재 Langfuse 템플릿에서 실제로 사용하는 키(`clickhouse.auth.*`, `replicaCount`, `shards`, host 계산 등)를 매핑표로 정리.
- 산출물: `docs/clickhouse-values-mapping.md` (신규)

### Phase 1. 차트 의존성 교체
- `charts/langfuse/Chart.yaml`
  - Bitnami `clickhouse` dependency 제거.
  - 대체 ClickHouse 차트 dependency 추가.
- 필요시 차트 alias 유지(`clickhouse`)로 상위 values 경로 변화 최소화.

### Phase 2. values 스키마/기본값 업데이트
- `charts/langfuse/values.yaml`
  - Bitnami 특화 설정(`resourcesPreset`, `zookeeper.image.repository`) 제거 또는 대체.
  - 기본 이미지를 공식 이미지 계열로 전환.
  - 사용자 호환성을 위해 deprecated 키는 한 릴리스 동안 유지 + 경고 문구 추가.

### Phase 3. 템플릿/헬퍼 로직 조정
- `charts/langfuse/templates/_helpers.tpl`
  - deploy 시 기본 host 계산 로직(서비스명) 대체 차트에 맞게 수정.
  - `CLICKHOUSE_URL`, `CLICKHOUSE_MIGRATION_URL` 계산 검증.
- `charts/langfuse/templates/validations.yaml`
  - 신규 차트 구조에 맞는 유효성 검사로 업데이트.

### Phase 4. 테스트 갱신
- `charts/langfuse/tests/clickhouse-cluster_test.yaml` 및 연관 테스트 갱신.
- 시나리오
  - `clickhouse.deploy=true` + 단일 replica
  - `clickhouse.deploy=true` + 다중 replica
  - `clickhouse.deploy=false` + 외부 ClickHouse
- 가능하면 CI에 template 렌더링 스모크 테스트를 추가.

### Phase 5. 문서/마이그레이션 가이드
- `charts/langfuse/README.md` 의 dependency table, values 설명 전면 업데이트.
- `UPGRADE.md`에 Bitnami -> 신규 차트 마이그레이션 섹션 추가.
- breaking/deprecated 항목 명시.

## 위험요소 및 대응
- 서비스명 변경으로 인해 애플리케이션 접속 host 깨짐
  - 대응: helper에서 host override 우선 적용 + 테스트 강화.
- 인증 키 이름 변경으로 기존 secret 호환성 저하
  - 대응: `existingSecret` 경로를 우선 보장하고 매핑 문서 제공.
- HA 토폴로지 차이(shard/replica semantics)
  - 대응: 1차 릴리스는 현재 동작과 동일한 최소 토폴로지만 보장.

## 작업 브랜치 전략
- 브랜치명: `feat/clickhouse-chart-migration`
- 실제 구현은 아래 단위 커밋 권장
  1. dependency 교체
  2. values/helper/validation 수정
  3. 테스트 수정
  4. 문서/업그레이드 가이드

## 완료 기준 (Definition of Done)
- Bitnami ClickHouse dependency 제거.
- 기본 ClickHouse 이미지가 비-Bitnami 유지보수 이미지로 교체.
- 헬름 템플릿 테스트 통과.
- README/UPGRADE 가이드 반영.
