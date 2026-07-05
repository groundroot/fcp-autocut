# FCPXML 처리 재설계 계획서

## 배경 (왜 이 계획이 필요한가)
베타 v0.1 → v0.10.7까지 오면서 무음제거·자막 배치에서 반복적으로 import 오류가
발생했다: 프레임 경계 이탈, audio-channel-source 순서, Invalid edit(미디어 없음),
어저스트먼트 레이어 증식·삭제, 갭 생성. 매번 개별 증상을 땜질했으나 근본 구조가
취약해 새 프로젝트 유형마다 새 버그가 나온다. "버전별로 다른 로직이 필요한가"라는
질문에서 출발해 FCPXML 포맷을 제대로 학습하고 재설계한다.

## 핵심 조사 결과 (검증됨)

### 1. FCPXML은 버전 1.9~1.14 구조가 사실상 동일하다
FCP 내장 DTD(1.0~1.14, `/Applications/Final Cut Pro.app/.../Interchange.framework`)를
직접 비교한 결과:
- `spine` 콘텐츠 모델: 1.9~1.14 **완전 동일** — `(%clip_item; | transition)*`
- `clip_item` 엔티티: 1.14에 `live-drawing`만 추가, 나머지 동일
  (`audio|video|clip|title|mc-clip|ref-clip|sync-clip|asset-clip|audition|gap`)
- `asset-clip` 콘텐츠 모델: 1.9~1.14 **완전 동일**
- `anchor_item`(커넥티드/어저스트먼트 레이어가 lane으로 붙는 요소): 안정적

**결론: 우리가 겪은 오류는 버전 차이가 아니라 "생성 로직의 견고성 문제"다.**
따라서 버전별 분기 로직이 아니라, **모든 버전에 통하는 견고한 생성기 + 버전별 DTD
검증**이 정답이다.

### 2. 시간 표현 규칙 (모든 버전 공통)
- 시간은 유리수 `분자/분모s` (Core Media 기반). 정수 초는 축약(`5s`).
- **자막·클립 offset/duration은 프레임 배수여야 FCP가 "프레임 경계"로 인정.**
  축약분수(`11/10`)는 FCP 12.3+가 거부 → 프레임 타임베이스(`3300/3000`)로 출력해야 함.
  (이미 v0.9.3/v0.10.5에서 `_frame_time_str`로 부분 적용)
- `offset`=부모 내 위치, `start`=소스 TC, `duration`=길이.
  asset-clip의 `start`+`duration`은 asset의 미디어 가용 범위를 벗어나면
  "Invalid edit with no respective media".

### 3. 무음제거의 근본 문제 — 파괴적 재구성
현재 `_build_silence_removed`는 **스파인을 비우고 asset-clip만 재구성**한다.
이 방식은 구조적으로 다음을 삭제/훼손한다:
- `<clip>`/`<video>`/`<gap>` (b롤, 스틸, 공백)
- 커넥티드 클립(lane)·어저스트먼트 레이어 → 증식 또는 삭제
- clips2와 clip_records의 인덱스 zip 매칭 → 음성 없는 클립이 중간에 있으면 오정렬

**사용자 요구(확정): "타임라인의 어떤 오브젝트도 삭제 금지. 무음컷은 오디오/비디오
클립만 컷."** 현재 아키텍처로는 불가능 → 재설계 필요.

## 재설계 방향

### A. 무음제거 = 파괴적 재구성 → **비파괴 시간 리매핑(Time-Remapping)**
스파인을 새로 짓지 않는다. 대신:
1. **제거할 무음 구간을 타임라인 절대 좌표로 수집** (각 오디오 클립의 발화 사이 침묵 +
   완전 무음 클립). 오디오/비디오(오디오 있는) 클립에서만 계산.
2. **구간 선형 리매핑 함수** 구축: `타임라인시간 → 압축시간` (무음 구간을 건너뜀).
3. **스파인의 모든 오브젝트를 순회**하며 offset을 리매핑으로 이동.
   - 무음 구간에 걸친 오디오 클립은 그 지점에서 **분할**(split)하여 무음 부분만 제거.
   - b롤·어저스트먼트 레이어·커넥티드 클립·갭은 **삭제하지 않고 위치만 이동**,
     무음 구간을 관통하면 그 부분만 잘라 길이 조정(트림).
4. 자막은 리매핑된 시간에 배치.
결과: **모든 오브젝트 보존 + 무음만 제거 + 전체 싱크 유지.**

이 접근은 FCP의 네이티브 무음 제거가 커넥티드 클립을 유지하는 방식과 동일 계열이며,
"오브젝트 삭제 금지" 요구를 만족한다.

### B. 견고한 시간 유틸리티 (버전 무관)
- `_frame_time_str`을 **모든** 클립/자막/갭 offset·duration에 일관 적용 (일부만 적용된
  현재 상태 정리). 프레임 타임베이스는 시퀀스 frameDuration에서 축약 없이 획득.
- asset 미디어 범위 클램핑을 공통 유틸로.
- 요소 삽입 순서(anchor item → marker → audio-channel-source → filter → metadata)를
  DTD 기반 정렬 유틸로 통일 (audio-channel-source 순서 버그 재발 방지).

### C. 버전별 DTD 검증을 안전망으로
- 출력 fcpxml의 version에 맞는 내장 DTD(`FCPXMLv1_N.dtd`)로 **생성 직후 자동 검증**.
  통과 못하면 로그에 정확한 위반 요소·라인을 남김 (현재 `_verify_fcpxml`는 자체 검사만).
- 출력 버전은 입력 버전 유지 (caption 등 요구 시에만 최소 1.8로 승격 — 기존 로직 유지).

### D. 지원 범위 명확화 + 안전한 폴백
- 지원: 단일/다중 소스, 커넥티드 클립, 싱크 클립, 어저스트먼트 레이어, b롤(비파괴 리매핑).
- 리매핑으로 안전히 처리 못하는 구조(멀티캠 mc-clip, 컴파운드 ref-clip 내부)는
  자막본만 생성 + 명확한 안내 (기존 정책 유지).

## 현재 코드 취약점 (전수 감사 결과, 파일:라인)

대상: `Silence-Cutter/silence_cutter/retranscribe.py`(파싱·수정 핵심), `fcpxml.py`(무에서 생성).

**약점 1 — 버전 인지 부재**
- 생성 경로는 `version="1.13"` **하드코딩** (`fcpxml.py:182, 461`).
- 파싱 경로는 `_ensure_min_version`으로 **1.8 하한만** 처리, 원본 버전 보존
  (`retranscribe.py:1465-1479`). 버전→(요소 순서·지원 태그) 매핑 테이블 없음.
- 자막 삽입 순서 규칙이 **경험적 화이트리스트**(`_TRAILING_TAGS` `retranscribe.py:397-401`,
  title 내 `param` before `text` `retranscribe.py:501`) — 신버전 규칙 변화에 취약.

**약점 2 — 전역 가변 상태 (재진입/동시성 위험)**
- `_FRAME_TB_NUM/_DEN`(`retranscribe.py:420-421`)과 `_TITLE_FONT`를 `retranscribe()`가
  실행 중 **global mutate**(`retranscribe.py:997`). 컨텍스트 객체 주입으로 전환 필요.
- 시간 표현이 float(`_parse_time`)와 Fraction(`_parse_time_fraction`) **이중 갈래** —
  규율이 주석에만 의존. 전면 Fraction 단일화 권장.
- 시퀀스 duration은 여전히 축약 `_rational_str`(`retranscribe.py:773`) — 프레임 정합 필요 시 잠재 버그.

**약점 3 — 무삭제가 "회피"로만 달성됨 (핵심)**
- `_build_silence_removed`(`retranscribe.py:616-785`)가 스파인을 비우고 재구성.
  lane 자식(커넥티드/어저스트먼트)을 **제외**(`retranscribe.py:702-703`)해 삭제.
- 상위 `_silence_safe` 게이트(`retranscribe.py:1085-1091, 1501-1505`)로 그런 프로젝트는
  무음제거본 **생략** → 오브젝트는 지키지만 기능이 꺼짐. **진짜 무삭제 무음제거를 하려면
  in-place 컷 + 후속 오브젝트 일괄 시프트로 근본 전환 필요.**

**약점 4 — 미지원 구성이 넓음**
- mc-clip(멀티캠), ref-clip(컴파운드), 중첩 커넥티드, asset-clip 없는 sync-clip이
  에러/제외(`retranscribe.py:1102-1119, 1197-1200`).

**약점 5 — 로직 이중화 + DTD 실검증 없음**
- `fcpxml.py`(생성)와 `retranscribe.py`(수정)가 시간·자막 로직 부분 중복.
- 사후 검증 `_verify_fcpxml`(`retranscribe.py:545-613`)은 자막 타이밍만 점검, **실제 DTD
  검증은 없음**. FCP 내장 DTD(1.0~1.14 보유)로 검증 계층 추가 권장.

**강점 (보존할 것)**: 실전 FCP 검증 실패를 하나씩 방어한 성숙한 방탄 코드(각 방어에 이유
주석). `_frame_time_str`의 프레임 경계 통과, 다중소스 per-asset VAD, `_project_sequence`,
미디어 클램핑, UID 재부여, 언어별 폰트 폴백 — 재설계에서 유틸로 계승한다.

## 구현 단계 (TDD)
1. **테스트 픽스처 + 골든 테스트** — 프로젝트 유형별 합성 fcpxml(단일/다중소스/커넥티드/
   어저스트먼트/b롤 clip/gap/무음클립 중간삽입) + 실제 커피국수. 각 유형에 대해
   "결과물이 해당 버전 DTD 통과 + 오브젝트 수 입력=출력" 자동 검증.
   ASR 없이도 돌도록 **무음 구간을 주입하는 구조 테스트**로 설계(미디어/NAS 불필요).
2. **포맷 컨텍스트 객체** — `_FRAME_TB_*`·`_TITLE_FONT` 전역을 `FormatContext`(frameDuration,
   타임베이스, 폰트, 버전)로 대체. 시간 표현 Fraction 단일화. `_rational_str` 잔여 사용처를
   `_frame_time_str`로 통일(시퀀스 duration 포함).
3. **버전별 DTD 검증 계층** — 출력 version에 맞는 내장 `FCPXMLv1_N.dtd`로 생성 직후
   `xmllint --dtdvalid` 검증, 위반 요소·라인 로그. 요소 삽입 순서를 DTD content-model에서
   도출하는 정렬 유틸(`_TRAILING_TAGS` 수동목록 대체).
4. **비파괴 무음제거 리매핑 엔진** (`_build_silence_removed` 대체):
   - 오디오 있는 클립에서만 무음 구간 수집 → 타임라인 절대 좌표.
   - `remap(t)` 구간 선형 함수(무음 skip) 구축.
   - 스파인 전체 오브젝트를 **삭제 없이** 순회: offset 리매핑, 무음 관통 클립 분할·트림,
     커넥티드/어저스트먼트/gap/b롤 함께 이동. `_silence_safe` 게이트 제거(진짜 무삭제 달성).
5. **회귀 스위트** — 8개 유형 × (DTD 통과 + 오브젝트 보존 + 자막 정합 + 프레임 경계) 자동화.

## 검증 기준 (완료 정의)
- 커피국수(다중소스+어저스트먼트+b롤) 무음제거본이 **모든 오브젝트 보존** + FCP 1.13
  DTD 통과 + 실제 FCP import 무경고.
- 8개 프로젝트 유형 전부 DTD 통과 + 오브젝트 수(어저스트먼트/b롤/커넥티드) 입력=출력.
- 프레임 경계·Invalid edit·audio-channel-source 회귀 0.

## 리스크
- 비파괴 리매핑은 복잡도가 높다 → 테스트 픽스처를 먼저 세우고 TDD로 진행.
- 실제 미디어(NAS) 없이는 ASR이 안 도므로, VAD/리매핑 단계는 **오디오 없이도 검증
  가능한 구조 테스트**(무음 구간을 주입)로 분리 설계.

## 참고 자료
- FCP 내장 DTD 1.0~1.14 (버전별 정확 검증에 사용)
- fcp.cafe FCPXML 개발자 가이드 (시간 rational·구조)
- Apple Legacy FCPXML DTD 문서
