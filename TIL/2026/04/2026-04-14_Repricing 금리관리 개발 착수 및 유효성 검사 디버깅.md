# 2026-04-14_Repricing 금리관리 개발 착수 및 유효성 검사 디버깅

## 오늘 한 것

* 국가필수선대 지정/해지 이력관리 작업 인계 내용 정리 및 잔여 작업 파악
* Git merge commit 정상 여부 확인
* Repricing 금리관리 페이지 유효성 검사 오류 원인 분석
* 기존 금리 서버단 로직(InrtInfr) 파악
* Repricing 금리관리 파일 구조 / API / 명칭 확정

---

## 1. 국가필수선대 잔여 작업

| 항목 | 상태 |
|------|------|
| otherwise 지정/해지일자 제거 여부 | 사수 확인 필요 |
| BeanUtils import Spring vs Apache 확인 | 미확인 |
| Git 푸시 완료 여부 | 미확인 |
| 이력 그리드 바인딩 최종 확인 | 미확인 |

> BeanUtils Apache Commons면 인자 순서 `copyProperties(dest, src)` — 반드시 `org.springframework.beans.BeanUtils` 임포트 확인

---

## 2. Git merge commit 정상 여부

`git pull` 시 원격에 새 커밋이 있으면 merge commit 자동 생성됨.  
54개 파일 포함 → **다른 팀원 작업물이 내려받아진 것**, 내 코드 손실 아님.  
conflict 없이 merge 됐으면 완전 정상.

- `feat: 국가필수선대...` 커밋은 히스토리에 살아있음 ✅
- 향후 방지: `git pull --rebase`

---

## 3. WebSquare 캘린더 유효성 검사 오류

### 증상
날짜 입력했는데 "적용일자 시작일" 필수입력 알럿이 계속 뜸.  
콘솔: `rpcgAplcnBgngYmd ===` (빈값)

### 원인 1 (주원인) — ref 바인딩 누락
```xml
<!-- ❌ -->
<w2:calendar id="cal_rpcgAplcnBgngYmd" ... />

<!-- ✅ -->
<w2:calendar id="cal_rpcgAplcnBgngYmd"
    ref="data:dma_rprcInrtReqVO.rpcgAplcnBgngYmd" ... />
```
`ref` 없으면 화면에 값이 보여도 DataCollection에 값이 안 들어감.

### 원인 2 — focus() 변수명 오타
```javascript
cal_rpcgAplcnBngnYmd.focus(); // ❌ Bngn
cal_rpcgAplcnBgngYmd.focus(); // ✅ Bgng
```
→ `ReferenceError: cal_rpcgAplcnBngnYmd is not defined`

---

## 4. Repricing 금리관리 개발

### 개발 전략
금리관리 폴더 하위 `금리 등록` 페이지(`InrtInfr` 계열) 구조가 동일  
→ **새로 만들지 않고 `금리 등록` 관련 파일 복사 후 치환** 전략 채택  
→ 컨벤션 유지 + 개발 속도 모두 유리

### 약어 컨벤션
| 약어 | 의미 |
|------|------|
| `Inrt` | Interest Rate (금리) |
| `Infr` | 기준금리 인프라 |
| `Rprc` | Repricing |

### 설계서 API 경로 (원문)
```
/api/sf/cs/cmsf/selectRepricingTrgt
/api/sf/cs/cmsf/saveRepricingAplcn
/api/sf/cs/cmsf/selectObservationDate
```

### 패키지
```
kobc.ngs.sf.cs.cmsf
```

### API 목록
| API | 설명 |
|-----|------|
| `selectRepricingTrgt` | 리프라이싱 대상 조회 |
| `saveRepricingAplcn` | 리프라이싱 적용 저장 |
| `selectObservationDate` | Observation Date 조회 |

### Controller 신규 생성 판단 근거
cmsf 패키지에 Controller가 이미 여럿 존재 → 기존 Controller에 메서드 추가 X  
기능 단위로 분리된 구조이므로 `RepricingController.java` 신규 생성이 적절

### 파일 구조
```
kobc.ngs.sf.cs.cmsf
├── web
│   └── RepricingController.java
├── service
│   ├── RepricingService.java
│   └── impl/RepricingServiceImpl.java
└── vo
    ├── RepricingTrgtReqVO.java
    ├── RepricingTrgtListVO.java
    └── RepricingAplcnVO.java

mapper
└── RepricingMapper.xml

nga.ui
└── CWMI_SFCM***M.xml
```

### 명칭 확정 (복사 기준: InrtInfr → Repricing 치환)
| 구분 | 기존(참고) | 신규 |
|------|-----------|------|
| Controller | `InrtInfrController` | `RepricingController` |
| Service | `InrtInfrService` | `RepricingService` |
| ServiceImpl | `InrtInfrServiceImpl` | `RepricingServiceImpl` |
| ReqVO | `InrtListReqVO` | `RepricingTrgtReqVO` |
| ListVO | `InrtListVO` | `RepricingTrgtListVO` |
| AplcnVO | — | `RepricingAplcnVO` |
| 메서드 | `selectInrtInfrList` | `selectRepricingTrgt` |

### DataCollection / Submission 명칭
```
dma_rprcInrtReqVO     ← 단건 조회 조건 (XML에서 실제 사용 확인)
dlt_dataList1         ← 목록

sbm_selectRprcInrtList
sbm_saveRprcInrt
```

---

## Eclipse / WebSquare 단축키 정리

| 단축키 | 기능 | 비고 |
|--------|------|------|
| `Ctrl + Shift + O` | import 자동 정리 | Eclipse Java |
| `Alt + Shift + R` | Rename 리팩토링 | Eclipse Java — WebSquare 미지원 |
| `Ctrl + F` | 현재 파일 내 찾기/치환 | WebSquare Studio 포함 |

---

## TIL

* **WebSquare 캘린더 ref 필수**: `ref` 없으면 DataCollection에 값 안 들어감 → 유효성 검사 항상 실패
* **focus() 오타 주의**: 긴 변수명 한 글자 틀리면 ReferenceError — 콘솔 에러 먼저 확인
* **신규 페이지 개발**: 동일 도메인 기존 파일(`금리 등록`) 복사 후 치환이 컨벤션 + 속도 모두 유리
* **cmsf Controller 전략**: 패키지에 Controller 여럿 있으면 기능명 단위로 신규 생성
* **Repricing 발음**: 리프라이싱 (Re + pricing)
* **Interest Rate**: 금리, 약어 `Inrt`
* **Git merge commit**: 팀원 작업 pull 시 자동 생성 — 정상 현상, 내 코드 손실 아님
