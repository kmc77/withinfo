# 2026-05-18 TIL — Repricing 서버단 Observation Date 계산 및 저장쿼리 디버깅

> **화면:** CWMI_SFCM201M.xml (Repricing 금리관리)
> **패키지:** `kobc.ngs.sf.cs.cmsf`
> **서비스:** `RclcInrtMngServiceImpl`
> **연관 세션:** 4/23(프론트 수정버튼 활성/편집모드 구현)의 연장선

---

## 0. 작업 한눈 요약

| 항목 | 내용 | 결과 |
|------|------|------|
| 서버단 `selectRepricingTrgt` 기준금리 재계산 | Compounded SOFR 행만 for 루프 분기 | ✅ 구현 |
| `selectCrtrInrt` 비즈니스 로직 오해 | 입출력 관계 잘못 이해 → CrtrInrtResVO에 필드 추가 시도 | ❌ 헛수고 → 정정 |
| NPE 디버깅 | reqVO 필수값 누락 → 기존 `oneditend` 파라미터 그대로 이식 | ✅ 해결 |
| 특수채(SPE_BOND_AAA_3M) NPE | 사수 코드 영역, 임의 우회 위험 | ⏸ 사수 보고 사안 |
| Git stash 발견 (사수 + 본인 작업 섞임) | 1611라인 NPE 방어 / 2122라인 파라미터 순서 변경 | ⏸ 사수 확인 필요 |
| 활성화 로우 기준금리 0 이슈 | 같은 `selectCrtrInrt`인데 서버/프론트 결과 다름 | ⏸ 파라미터 비교 진행 중 |
| 저장 ClassCastException (sn) | `Integer ↔ String` 미스매치 | ✅ `TO_NUMBER(#{sn})`로 해결 |
| 저장 후 수정사유 미반영 | WHERE `SN=0` 매칭 0건 → 프론트 `sn` 세팅 누락 | ✅ 원인 확정 |

---

## 1. 서버단 `selectRepricingTrgt` — Observation Date 기반 기준금리 재계산

### 1.1 배경

- **기존 동작:** `inrtCmptYmd`(Observation Date) 셀 변경시에만 `sbm_selectObservationDateInrt` 호출 → 기준금리 재계산
- **문제:** **최초 조회 시점**에는 재계산 안 됨. 화면 처음 떴을 때 `crtrInrt`가 DB 원본값 그대로 노출
- **해결 방향:** 서버단 `selectRepricingTrgt`에서 조회 결과 루프 돌며 같은 계산 수행

### 1.2 핵심 비즈니스 로직 (헤맨 부분 → 명확화)

```
Observation Date (inrtCmptYmd)  →  [selectCrtrInrt]  →  기준금리 (inrt)
       입력 (input)                    함수              출력 (return)
```

- `selectCrtrInrt`의 리턴 VO(`CrtrInrtResVO`)에서 꺼내야 할 값: **`resVO.getInrt()`**
- **Compounded SOFR(`cmptInrtClssCd`)** 일 때만 분기. 나머지 금리구분코드는 패스
- 서버 루프에서 세팅해야 할 값: **`vo.setCrtrInrt(resVO.getInrt())` 한 줄**

### 1.3 최종 코드

```java
@Override
public RclcInrtMngResVO selectRepricingTrgt(RclcInrtMngReqVO rclcInrtMngReqVO) throws Exception {
    RclcInrtMngResVO rclcInrtMngResVO = new RclcInrtMngResVO();
    rclcInrtMngResVO.setRclcInrtMngListVO(rclcInrtMngMapper.selectRepricingTrgt(rclcInrtMngReqVO));

    // Compounded SOFR인 경우만 기준금리 재계산
    for (RclcInrtMngListVO vo : rclcInrtMngResVO.getRclcInrtMngListVO()) {
        if (SfConstants.COMPOUNDED_SOFR.equals(vo.getCmptInrtClssCd())) {
            CrtrInrtReqVO reqVO = new CrtrInrtReqVO();

            // oneditend에서 세팅하던 파라미터 그대로 이식
            reqVO.setBgngYmd(vo.getInrtCmptYmd());                            // aplcnYmd
            reqVO.setRpmtYmd(vo.getInrtCrtrYmd());                            // inrtCrtrYmd
            reqVO.setInrtClssCd(vo.getCmptInrtClssCd());                      // cmptInrtClssCd
            reqVO.setFlctnInrtRclcYmdClssCd(vo.getFlctnInrtRclcYmdClssCd());
            reqVO.setIntAplcnMaysClssCd("89030003");
            reqVO.setAdltPrcsMethClssCd("11020002");
            reqVO.setNtnAreaCd("KR");

            CrtrInrtResVO resVO = cashFlowCmptService.selectCrtrInrt(reqVO);
            vo.setCrtrInrt(resVO.getInrt());
        }
    }

    return rclcInrtMngResVO;
}
```

### 1.4 시도했다가 폐기한 접근

- 처음엔 `BeanUtils.copyProperties(vo, reqVO)` + 필드명 다른 것 수동 세팅 방식
- 결국 명시적 세팅이 더 명확해서 위 코드로 정리
- `CrtrInrtResVO`에 `inrtCmptYmd` 필드 추가하려던 시도 → **잘못된 진단이라 폐기**

---

## 2. 헤맨 원인 — 셀프 회고

### 2.1 무엇이 잘못됐나

**1) 비즈니스 로직 미숙지**

`selectCrtrInrt`의 입출력 관계를 거꾸로 이해함:
- 실제: Observation Date(`inrtCmptYmd`) **입력** → 기준금리(`inrt`) **출력**
- 착각: Observation Date 자체를 계산해서 리턴하는 함수인 줄

→ `resVO`에서 `inrtCmptYmd`를 꺼내려고 했고
→ `CrtrInrtResVO`에 필드 추가하고
→ `selectCrtrInrt` 내부에서 `result.setInrtCmptYmd()` 세팅하려고 했음

**2) 잘못된 진단**

로직을 모르니까 "리턴 VO에 필드가 없어서 null이다" → "필드 추가하면 된다" 방향으로 흘렀음.
실제 문제는 **애초에 꺼내려는 값이 틀렸던 것**.

### 2.2 교훈

> **기존 동작하는 로직(`oneditend`)을 먼저 끝까지 읽고 흐름 파악 후 이식했어야 함.**

새 기능 만들기 전에 같은 API를 이미 호출하는 기존 로직 흐름을 100% 이해하는 게 우선.

---

## 3. NPE 디버깅

### 3.1 증상

```
NullPointerException
at vo.setCrtrInrt(resVO.getInrt())
```

`resVO.getInrt()`가 null. → `selectCrtrInrt` 내부에서 계산 실패.

### 3.2 진단 방법

```java
System.out.println("bgngYmd === " + reqVO.getBgngYmd());
System.out.println("inrtClssCd === " + reqVO.getInrtClssCd());
System.out.println("flctnInrtRclcYmdClssCd === " + reqVO.getFlctnInrtRclcYmdClssCd());
System.out.println("inrtCmptYmd === " + reqVO.getInrtCmptYmd());
```

→ `reqVO` 필수값 중 빠진 게 있는지 확인

### 3.3 해결

`oneditend`에서 세팅하던 파라미터 7개 그대로 이식 (위 1.3 코드 참조).
`BeanUtils.copyProperties` 빼고 명시적 세팅으로 정리.

---

## 4. 특수채(SPE_BOND_AAA_3M) NPE — 사수 영역

### 4.1 증상

특수채 조회 시 `CashFlowCmptServiceImpl` 내부에서 NPE 발생.
정확한 위치: 2133라인 부근

```java
} else if (SfConstants.SPE_BOND_AAA_3M.equals(inrtClssCd)) {
    crtrInrt = selectIndexInrt(inrtCmptYmd, inrtClssCd,
                               SfConstants.FLCTN_INRT_RCLC_YMD_CLSS_CD1,
                               bgngYmd, endYmd);
    crtrInrt = crtrInrt.multiply(KobcNumberUtil.ONE_HUNDRED);  // ← 여기서 NPE
}
```

→ `selectIndexInrt`가 null 리턴 시 `multiply()` 호출에서 NPE

### 4.2 시도한 우회들과 결과

| 시도 | 위치 | 결과 |
|------|------|------|
| `selectRepricingTrgt` 루프에 try-catch | 내 서비스 코드 | 효과 없음 (다른 경로에서 발생) |
| 프론트 `sbm_selectObservationDateInrt_submitdone`에 에러 분기 | 내 화면 코드 | 서버 500이면 submitdone 안 탐 |
| `submiterror` 콜백 빈 함수로 막기 | 프론트 | **모든 에러를 삼킴**(부작용 큼) — 폐기 |
| 사수 코드 직접 수정 | `CashFlowCmptServiceImpl` 2133 | **금지 영역** — 불가 |

### 4.3 결론

> **사수에게 보고할 사안.**
> "특수채 데이터 없을 때 `selectCrtrInrt` 2133라인에서 NPE 나는데, 프론트/서비스 어디서 어떻게 처리할지" → 사수 판단 필요.

기술적으로 깔끔한 단독 우회 없음. submiterror 차단은 부작용 너무 크고, 서버 처리는 사수 코드 수정이 필요.

---

## 5. Git stash 발견 — 사수 작업과 본인 작업 섞임

GitHub Desktop의 stash 목록에서 `CashFlowCmptServiceImpl` 변경분 발견.

### 5.1 stash 내용 분석

**1) 1611라인 — NPE 방어 코드**

```java
// 기존
return indexInrtResVO.getInrt();

// stash 변경분
return indexInrtResVO == null ? BigDecimal.ZERO : indexInrtResVO.getInrt();
```

→ 위 4번 특수채 NPE 그 지점. **사수가 이미 null 방어 처리해놨음.**
→ 내가 우회 안 해도 되는 상황일 수 있음. 단, restore 여부 사수 확인 필수.

**2) 2122라인 — `selectCmpSofrInrt` 파라미터 순서 변경**

```java
// 기존: rpmtYmd, inrtCmptYmd
// stash: inrtCmptYmd, rpmtYmd
```

→ Compounded SOFR 호출 시 상환일자/이전상환일자 순서가 바뀜.
→ "값이 이상하게 부풀려진" 기존 버그의 진짜 원인일 가능성 있음.

### 5.2 안전 원칙

내 작업 + 사수 작업이 **한 stash에 섞여 있는 상황**이라:

| 옵션 | 안전성 | 메모 |
|------|--------|------|
| **Discard** | ❌ 절대 금지 | 사수 변경 다 날아감. 복구 거의 불가 |
| **Restore** | ⭕ 안전 | 일단 Changes로 꺼냄. 되돌릴 수 있음 |

**액션:** Restore 후 파일별 diff로 사수 변경 vs 내 변경 구분 → 사수에게 stash 출처와 정리 방향 확인.

---

## 6. `inrtClssCd` vs `cmptInrtClssCd` — 잠재 코드체계 버그 (의심)

### 6.1 패턴

서버 for 루프(05-18 추가분):
```java
reqVO.setInrtClssCd(vo.getCmptInrtClssCd());
```

프론트 `oneditend`(기존):
```js
dma_rclcInrtMngReqVO.set("inrtClssCd", dataList1.getCellData(row, "cmptInrtClssCd"));
```

**두 경로 다 `cmptInrtClssCd` 값을 `inrtClssCd`에 넣는 같은 패턴.**

### 6.2 의심

- `selectCrtrInrt`는 `inrtClssCd`로 SOFR / 특수채 / 일반기준금리 **분기**
- Compounded SOFR는 `cmptInrtClssCd == inrtClssCd` 일치라 우연히 정상
- **Term SOFR** 같은 다른 코드는 잘못된 산출식 탈 가능성
- 두 경로 다 같은 잠재 버그를 공유하는 셈

### 6.3 확정 전 필요한 정보

1. `selectCrtrInrt` 매퍼/서비스에서 `inrtClssCd`를 어떻게 사용하는지 (분기 변수인지)
2. `cmptInrtClssCd` vs `inrtClssCd` 코드체계가 실제 다른지 (DB 직접 비교)
3. `dataList1`에 순수 `inrtClssCd` 컬럼이 별도로 있는지

→ 사수 확인 사안. 임의 수정 금지.

---

## 7. 활성화 로우 기준금리 0 디버깅

### 7.1 증상

- 활성화 조건: `today <= aplcnYmd` (미래 적용일자 → 수정 가능)
- 활성화된 로우의 **`crtrInrt`(기준금리)** 값이 **0**으로 찍힘
- F12 Network로 확인: 서버 응답 자체가 0
- 같은 행에서 Observation Date를 **똑같은 날짜로 다시 입력**하면 `oneditend` 거쳐서 **정상값**으로 바뀜

### 7.2 핵심 단서

> **같은 `selectCrtrInrt` 함수, 다른 결과 = 파라미터가 다르다.**

- 서버 `selectRepricingTrgt` for 루프 → `selectCrtrInrt` → 0 리턴
- 프론트 `oneditend` → `sbm_selectObservationDateInrt` → 같은 `selectCrtrInrt` → 정상값 리턴

### 7.3 원인 후보 (가능성 순)

**1) 사수 stash 적용 여부 (1611라인 NPE 방어)**

`selectIndexInrt`가 null 리턴 시 `BigDecimal.ZERO` 반환. 미래 일자는 아직 indexInrt 데이터 없을 가능성 높음 → null → 0.

**2) `selectCmpSofrInrt` 파라미터 순서 (2122라인)**

stash 변경이 적용된 상태면 `inrtCmptYmd, rpmtYmd` 순서로 호출 → 미래 일자에선 SOFR 조회 0건 → 0.

**3) 두 경로의 reqVO 필드 차이**

- `flctnInrtRclcYmdClssCd` 누락 의심 → 분기 못 타고 0 리턴 가능
- `bgngYmdFxtnYn`, `rpmtYmd` 등 부수 필드 차이

### 7.4 진단 방법 (다음 세션 진입점)

`CashFlowCmptServiceImpl.selectCrtrInrt` 진입부에 임시 로그/BP:

```java
log.info("reqVO: " + reqVO.toString());
```

1. 화면 조회(서버 루프) → reqVO 캡쳐
2. 같은 행 `inrtCmptYmd` 편집(oneditend) → reqVO 캡쳐
3. 두 reqVO **모든 필드 값 비교** → 차이나는 필드가 범인

---

## 8. 저장 쿼리 `updateRepricingAplcn` — ClassCastException

### 8.1 증상

```
java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
parameter #15 'sn'
```

| 위치 | 타입 |
|------|------|
| DDL (`TIV_FLCTN_INRT_APLCN_B.SN`) | NUMBER(3,0) |
| `RclcInrtMngListVO.sn` | String |
| 실제 들어오는 값 | Integer |

VO가 String 선언인데 화면에서 Integer로 들어와서 MyBatis 캐스팅 실패.

### 8.2 시도한 해결책

| 시도 | 결과 |
|------|------|
| `#{sn, jdbcType=INTEGER}` | 효과 없음 |
| `#{sn, jdbcType=NUMERIC}` | 효과 없음 |
| `#{sn, javaType=java.lang.String, jdbcType=VARCHAR}` | `Integer cannot be cast to String` 그대로 |
| **`TO_NUMBER(#{sn})`** | ✅ **해결** |

### 8.3 최종 UPDATE 쿼리

```xml
<update id="updateRepricingAplcn" parameterType="kobc.ngs.sf.cs.cmsf.vo.RclcInrtMngListVO">
    UPDATE TIV_FLCTN_INRT_APLCN_B
    SET
        INRT_CRTR_YMD                  = #{inrtCrtrYmd}
        , INRT_CMPT_YMD                = #{inrtCmptYmd}
        , FLCTN_INRT_RCLC_YMD_CLSS_CD  = #{flctnInrtRclcYmdClssCd}
        , CMPT_INRT_CLSS_CD            = #{cmptInrtClssCd}
        , APLCN_INRT                   = #{aplcnInrt}
        , ADJ_INRT                     = #{adjInrt}
        , MDFCN_RSN_CN                 = #{mdfcnRsnCn}
        , CHRG_DEPT_CD                 = #{chrgDeptCd}
        , LAST_CHG_USER_ID             = #{lastChgUserId}
        , LAST_CHG_DT                  = SYSDATE
    WHERE BIZ_CLSS_CD  = #{bizClssCd}
        AND RSNG_NO    = NVL(#{rsngNo}, ' ')
        AND WRKG_NO    = NVL(#{wrkgNo}, ' ')
        AND CTRT_NO    = NVL(#{ctrtNo}, ' ')
        AND APLCN_YMD  = #{aplcnYmd}
        AND SN         = TO_NUMBER(#{sn})
</update>
```

### 8.4 학습 포인트

- **Tibero도 `TO_NUMBER` 지원** (Oracle 호환 DB)
- jdbcType만으론 javaType이 String일 때 캐스팅 못 막음
- DDL이 NUMBER인데 VO가 String이면 쿼리에서 `TO_NUMBER`로 감싸는 게 가장 깔끔

---

## 9. 저장 후 수정사유 미반영 — `SN=0` 매칭 0건

### 9.1 증상

UPDATE 자체는 에러 없이 실행됐는데 DB에 반영 안 됨.
콘솔 SQL 확인:

```
AND SN = 0
```

실제 데이터의 sn이 0이 아니라 **매칭 0건 → 업데이트 0행**. 에러 없이 조용히 실패.

### 9.2 범인 추적

저장 시 `dma_rclcInrtMngReqVO`에 세팅하는 키 목록 점검:

```
ctrtNo, rsngNo, bizClssCd, inrtCmptYmd, adjInrt, mdfcnRsnCn,
cmptInrtClssCd, lastChgUserId, frstRegDt, aplcnYmd,
flctnInrtRclcYmdClssCd, aplcnInrt, chrgDeptCd
```

→ **`sn`을 세팅하는 코드가 없음.** 그래서 WHERE 절 `#{sn}`이 기본값 0으로 들어감.

### 9.3 수정 (다음 세션 적용)

`com.sbm.execute(sbm_saveRepricingAplcn);` 바로 위에 한 줄 추가:

```js
dma_rclcInrtMngReqVO.set("sn", dl.getCellData(row, "sn"));
```

### 9.4 사전 확인 필요

1. **DataCollection Editor**: `dataList1`에 `sn` 컬럼 존재 여부
2. **조회 쿼리** `selectRepricingTrgt`: SELECT 절에 `SN` 내려오는지

응답 JSON에서 `"sn": "1"` 확인됐으니 dataList에도 컬럼 있을 가능성 높음. 없으면 columnInfo에 hidden으로 추가해야 함.

### 9.5 부가 의심 — `INRT_CRTR_YMD = NVL(NULL, TO_CHAR(SYSDATE))`

콘솔에 `inrtCrtrYmd`가 null로 와서 `SYSDATE`로 덮어쓰는 패턴 발견.
**수정 화면에서 기존 금리기준일자를 SYSDATE로 날리는 게 의도된 동작인지 사수 확인 필요.**

---

## 10. 핵심 교훈 5선

| # | 교훈 |
|---|------|
| 1 | 새 기능 만들기 전, **같은 API를 이미 호출하는 기존 로직(예: `oneditend`)을 끝까지 먼저 읽고 흐름 파악** |
| 2 | `selectCrtrInrt`의 입출력 관계 같은 **함수 시그니처(목적)는 코드 추가 전에 100% 명확화** |
| 3 | DDL `NUMBER` ↔ VO `String` 미스매치는 **쿼리에서 `TO_NUMBER` 감싸는 게 가장 깔끔** (jdbcType만으론 부족) |
| 4 | UPDATE WHERE 조건 PK 누락은 **에러 없이 0행 업데이트**로 조용히 실패 → 콘솔 SQL의 `AND ... = 0` 패턴 즉시 의심 |
| 5 | **Git stash에 사수 작업이 섞여 있으면 Discard 절대 금지.** Restore 후 diff 분리 → 사수 확인 |

---

## 11. 다음 세션 진입점 (Action Items)

### 우선순위 1 — 즉시 적용
- [ ] 프론트 저장 직전 `dma_rclcInrtMngReqVO.set("sn", dl.getCellData(row, "sn"));` 한 줄 추가
- [ ] `dataList1`에 `sn` 컬럼 존재 확인 (없으면 columnInfo에 추가)

### 우선순위 2 — 사수 확인 후 진행
- [ ] Git stash 출처 확인 및 Restore 여부 결정
- [ ] 특수채(SPE_BOND_AAA_3M) NPE 처리 위치 협의
- [ ] `INRT_CRTR_YMD = NVL(NULL, SYSDATE)` 의도 여부
- [ ] `inrtClssCd` vs `cmptInrtClssCd` 코드체계 분리 필요성

### 우선순위 3 — 디버깅 진행
- [ ] `selectCrtrInrt` 진입부 로그 → 서버 루프 vs `oneditend` reqVO 필드 비교 → 활성화 로우 기준금리 0 원인 확정

---

## 12. 관련 파일

| 파일 | 변경 내용 |
|------|----------|
| `RclcInrtMngServiceImpl.java` | `selectRepricingTrgt` for 루프 + Compounded SOFR 분기 추가 |
| `RclcInrtMngMapper.xml` | `updateRepricingAplcn` UPDATE 쿼리 (`TO_NUMBER(#{sn})` 적용) |
| `CWMI_SFCM201M.xml` | 저장 직전 `sn` 세팅 추가 예정 |
| `CashFlowCmptServiceImpl.java` | (사수 영역) 1611, 2122라인 stash 변경분 검토 대상 |
