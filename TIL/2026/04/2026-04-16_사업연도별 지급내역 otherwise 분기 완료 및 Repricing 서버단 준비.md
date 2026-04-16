# 2026-04-16_사업연도별 지급내역 otherwise 분기 완료 및 Repricing 서버단 준비

## 오늘 한 것

* `selectNtnEsntlBizYr` otherwise 분기 에러 원인 분석 및 수정
* 중복 행 분석 — 차수별 정상 표시 확인
* RMV_YMD 연도 불일치 현상 원인 파악
* Repricing 서버단 파일 구조 파악 및 생성 준비
* 그리드 데이터 미표시 체크포인트 정리

---

## 1. otherwise 분기 JDBC-8026 에러

### 증상
합계여부 체크박스 클릭 후 조회 시 `JDBC-8026: invalid identifier` 발생.  
에러 메시지: `and h.ntn_esntl_grsh_no = b.ntn_esntl_grsh_no`

### 원인 1 — NOT EXISTS 안에서 B 참조
`otherwise` 1번 쿼리 FROM에 B 테이블이 없는데 NOT EXISTS 안에서 `B.NTN_ESNTL_GRSH_NO` 참조.

```sql
-- 잘못된 코드
AND NOT EXISTS (
    SELECT 1 FROM TPS_NTN_ESNTL_GRSH_H H
    WHERE H.NTN_ESNTL_GRSH_NO = B.NTN_ESNTL_GRSH_NO  -- B 없음
)

-- 수정
AND NOT EXISTS (
    SELECT 1 FROM TPS_NTN_ESNTL_GRSH_H H
    WHERE H.NTN_ESNTL_GRSH_NO = A.NTN_ESNTL_GRSH_NO  -- A로 대체
)
```

`when` 분기는 B가 JOIN되어 있어서 `B.` 써도 됐지만,  
`otherwise`는 FROM에 A만 있으므로 `A.`로 변경 필요.

### 원인 2 — 2번 쿼리 FROM에 H 테이블 누락
`otherwise` 2번 쿼리 SELECT/WHERE에서 `H.DSGN_YMD`, `H.RMV_YMD`, `H.PRHS_SN` 전부 참조하는데 FROM에 H 없음.

```sql
-- 수정
FROM TPS_NTN_ESNTL_GRSH_SBSD_GIVE_B A
   , TPS_NTN_ESNTL_GRSH_I B
   , TPS_NTN_ESNTL_GRSH_H H    -- 추가
   , (SELECT ...) C
```

### 적용 범위
`when` / `otherwise` 양쪽 모두 2번 쿼리(이력 있는 선대)에서 H 참조 → 두 분기 모두 FROM에 H 있어야 함.

---

## 2. 중복 행 분석

### 증상
그리드에 같은 선대번호 행이 여러 개 표시됨.

### 결론 — 정상
`sumYn == "0"` (차수별 전체 펼치기 모드)에서 같은 선대가 차수별로 여러 행 나오는 것은 **의도된 동작**.

| 케이스 | 판단 |
|--------|------|
| 같은 선대번호, 차수 다름 | 정상 |
| 같은 선대번호, 차수 동일 | 진짜 중복 — 조사 필요 |

확인 결과:
- `323006D`: 차수 2, 3
- `323001D`: 차수 2, 3
- `322001D`: 차수 2, 5
- `320055D`: 차수 1~9, 9건 → 등록 화면 보조금 지급 내역 9건과 일치 → **쿼리 로직 정상**

---

## 3. RMV_YMD 연도 불일치

### 증상
`BIZ_YR=2023` 행에 `RMV_YMD=20250101` (연도 불일치) 표시.

### 원인
`DSGN_YMD`는 `= A.BIZ_YR` 조건으로 필터했지만 `RMV_YMD`는 연도 조건 없이 SELECT.  
H 테이블에서 나중에 수정된 해지일자(2025년)가 그대로 붙어서 나옴.

### 결론 — 무시
해지일자가 당해년도로 바뀐 **정책 변경 이전 데이터**이므로 현재는 무시.  
신규 등록 시 유효성 검사(지정일자 당해년도) 복원 후 자연스럽게 해결됨.

---

## 4. Repricing 서버단 파일 구조

### 생성 파일 목록

```
kobc.ngs.sf.cs.cmsf
├── web
│   └── RepricingController.java
├── service
│   ├── RepricingService.java
│   └── impl/RepricingServiceImpl.java
└── vo
    ├── RepricingReqVO.java
    ├── RepricingListVO.java
    └── RepricingResVO.java

mapper
└── RepricingMapper.xml
```

### 참고 패턴
기존 `RclcInrtMng` 파일 구조 참고하여 복사 후 치환.

```java
// ServiceImpl 패턴
@Service("repricingService")
public class RepricingServiceImpl extends KobcServiceImpl implements RepricingService {

    @Autowired
    RepricingMapper repricingMapper;

    @Override
    public RepricingListVO selectRepricingTrgt(RepricingReqVO repricingReqVO) throws Exception {
        RepricingListVO repricingListVO = new RepricingListVO();
        repricingListVO.setRepricingListVO(repricingMapper.selectRepricingTrgt(repricingListVO));
        return repricingListVO;
    }
}
```

### VO 구성
- **ReqVO** — 요청 파라미터 (Controller → Service)
- **ListVO** — 목록 조회 결과 한 행 매핑 (`List<ListVO>`)
- **ResVO** — 응답 전체 구조 래핑 (페이징 + 여러 목록 묶기)

---

## 5. 그리드 데이터 미표시 체크포인트

서버에서 값은 정상 반환되는데 그리드에 안 찍힐 때 확인 순서:

1. **`ref` 속성** — 그리드 컬럼 `ref`가 VO 필드명과 일치하는지
2. **DataCollection 이름** — 조회 함수에서 꽂는 DataCollection이 그리드에 바인딩된 것과 동일한지
3. **응답 JSON 키** — 서버 응답 키와 DataCollection 이름 일치하는지

---

## 잔여 작업

| 항목 | 상태 |
|------|------|
| otherwise 분기 검증 (합계 금액 확인) | 미완 |
| 테스트 데이터 정리 (4/10 이후 DELETE) | 미완 |
| 지정일자 당해년도 유효성 검사 복원 | 미완 |
| Repricing 서버단 파일 생성 | 미완 — 구조 파악 완료 |
| Git 커밋/푸시 | 미완 |

---

## TIL

* **`otherwise` vs `when` FROM 절 차이 주의**: 두 분기는 조인 구조가 다를 수 있음 — `when`에서 됐다고 `otherwise`에서도 된다는 보장 없음. 항상 각 분기 FROM 절 독립 확인 필요
* **invalid identifier 1순위 원인**: FROM에 없는 테이블 별칭 참조 — 에러 메시지의 컬럼명으로 역추적
* **디비버에서 되는데 서버에서 안 되는 경우**: 동적 쿼리(`<choose>`, `<if>`) 분기 중 해당 분기만 따로 실행한 것일 수 있음 — 전체 분기 포함 실행 여부 확인
* **RMV_YMD 연도 조건 미적용**: `DSGN_YMD`에만 연도 필터를 걸면 `RMV_YMD`는 무조건 최신값이 붙음 — 비즈니스 로직 확인 후 처리 방향 결정 필요
* **차수별 표시 모드에서 중복 판단 기준**: 선대번호 + 차수 조합이 동일한 경우만 진짜 중복
* **VO 3종 구분**: ReqVO(입력), ListVO(행 단위 출력), ResVO(응답 전체 래핑) — 프로젝트마다 다를 수 있으니 기존 파일 필드 직접 확인
