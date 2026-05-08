# 2026-05-06_Repricing 검색조건 추가 및 saveRepricingAplcn 전체 구현

## 오늘 한 것
- Repricing 금리관리 검색 영역 필수선대 셀렉트박스 추가 (퍼블 레이아웃 수정)
- 조회 쿼리 BETWEEN 날짜 검색 + bizClssCd 조건 추가 (UNION ALL 5개 브랜치)
- saveRepricingAplcn Controller → Service → Mapper → XML 전체 구현 및 디버깅

---

## 1. WebSquare - colgroup 레이아웃 수정

### 문제
기존 tr 안에 필수선대 th+td 쌍 추가 시 사업년도 영역 레이아웃 깨짐

### 원인
colgroup의 `<col>` 개수와 tr 내 th+td 쌍 수가 불일치

### 해결
th+td 추가 시 colgroup에도 col 2개 추가
```xml
<!-- 기존: 60px, 350px, 75px, 250px -->
<!-- 추가 후: 60px, 350px, 75px, 120px, 75px, 250px -->
<xf:group tagname="col" style="width:75px;"/>   <!-- 필수선대 th용 -->
<xf:group tagname="col" style="width:120px;"/>  <!-- 필수선대 td용 -->
```

### 핵심
WebSquare table에서 **colgroup col 수 = tr 내 th+td 쌍 수** 반드시 일치

---

## 2. MyBatis - UNION ALL 구조에서 검색조건 추가

### 문제
UNION ALL 브랜치 중 하나에만 조건 추가 → 나머지 브랜치 데이터는 필터 없이 전부 나옴

### 해결
**5개 브랜치 각각** WHERE 끝에 동일 조건 추가

**날짜 BETWEEN 검색**
```xml
<if test='rpcgAplcnBgngYmd != null and rpcgAplcnBgngYmd != "" and rpcgAplcnEndYmd != null and rpcgAplcnEndYmd != ""'>
AND A.APLCN_YMD BETWEEN #{rpcgAplcnBgngYmd} AND #{rpcgAplcnEndYmd}
</if>
```

**사업구분 필터**
```xml
<if test='bizClssCd != null and bizClssCd != ""'>
AND A.BIZ_CLSS_CD = #{bizClssCd}
</if>
```

### 핵심
UNION ALL 구조에서 특정 브랜치에만 조건 추가하면 절대 안 됨. **모든 브랜치에 동일 조건** 추가 필수

---

## 3. saveRepricingAplcn 구현

### 대상 테이블
`TIV_FLCTN_INRT_APLCN_B`  
PK: `BIZ_CLSS_CD, RSNG_NO, WRKG_NO, CTRT_NO, ASSR_STPL_NO, VLBN_SCTS_NO, SN, APLCN_YMD`

### Controller
```java
@PostMapping("/saveRepricingAplcn")
public RclcInrtMngResVO saveRepricingAplcn(@RequestBody RclcInrtMngReqVO vo) throws Exception {
    RclcInrtMngResVO rclcInrtMngResVO = new RclcInrtMngResVO();
    rclcInrtMngService.saveRepricingAplcn(vo);
    return rclcInrtMngResVO;
}
```

### Service 인터페이스
```java
void saveRepricingAplcn(RclcInrtMngReqVO vo) throws Exception;
```

### ServiceImpl
```java
@Override
public void saveRepricingAplcn(RclcInrtMngReqVO rclcInrtMngReqVO) throws Exception {
    if (rclcInrtMngReqVO == null) {
        throw processException("EZZ0009", "변동금리적용내역저장 정보");
    }
    rclcInrtMngMapper.saveRepricingAplcn(rclcInrtMngReqVO);
}
```

### Mapper 인터페이스
```java
void saveRepricingAplcn(RclcInrtMngReqVO vo);
```
- MyBatis `<insert>` Mapper 반환타입은 반드시 `void` 또는 `int`
- VO 타입 반환하면 `unsupported return type` 에러 발생

### Mapper XML
```xml
<insert id="saveRepricingAplcn" parameterType="kobc.ngs.sf.cs.cmsf.vo.RclcInrtMngListVO">
    <selectKey resultType="int" keyProperty="maxSn" order="BEFORE">
        SELECT NVL(MAX(SN), 0) + 1 AS SN   -- 일련번호
        FROM TIV_FLCTN_INRT_APLCN_B
        WHERE BIZ_CLSS_CD  = #{bizClssCd}
        AND RSNG_NO        = NVL(#{rsngNo,     jdbcType=VARCHAR}, ' ')
        AND WRKG_NO        = NVL(#{wrkgNo,     jdbcType=VARCHAR}, ' ')
        AND CTRT_NO        = NVL(#{ctrtNo,     jdbcType=VARCHAR}, ' ')
        AND ASSR_STPL_NO   = NVL(#{assrStplNo, jdbcType=VARCHAR}, ' ')
        AND VLBN_SCTS_NO   = NVL(#{vlbnSctsNo, jdbcType=VARCHAR}, ' ')
        AND APLCN_YMD      = #{aplcnYmd}
    </selectKey>
    INSERT INTO TIV_FLCTN_INRT_APLCN_B (
          BIZ_CLSS_CD
        , RSNG_NO              -- 조달번호
        , WRKG_NO              -- 운용번호
        , CTRT_NO              -- 계약번호
        , ASSR_STPL_NO         -- 보증약정번호
        , VLBN_SCTS_NO         -- 유가증권번호
        , SN
        , APLCN_YMD
        , INRT_CMPT_YMD
        , FLCTN_INRT_RCLC_YMD_CLSS_CD
        , CMPT_INRT_CLSS_CD
        , INRT_SPRD_ITRT
        , MDFCN_RSN_CN
        , CHRG_DEPT_CD
        , FRST_REG_USER_ID
        , FRST_REG_DT
        , LAST_CHG_USER_ID
        , LAST_CHG_DT
    ) VALUES (
          #{bizClssCd}
        , NVL(#{rsngNo,     jdbcType=VARCHAR}, ' ')
        , NVL(#{wrkgNo,     jdbcType=VARCHAR}, ' ')
        , NVL(#{ctrtNo,     jdbcType=VARCHAR}, ' ')
        , NVL(#{assrStplNo, jdbcType=VARCHAR}, ' ')
        , NVL(#{vlbnSctsNo, jdbcType=VARCHAR}, ' ')
        , #{maxSn}
        , #{aplcnYmd}
        , #{inrtCmptYmd}
        , #{flctnInrtRclcYmdClssCd}
        , #{cmptInrtClssCd}
        , #{inrtSprdItrt}
        , #{mdfcnRsnCn}
        , #{chrgDeptCd}
        , #{frstRegUserId}
        , SYSDATE
        , #{lastChgUserId}
        , SYSDATE
    )
</insert>
```

### RclcInrtMngReqVO 추가 필드
```java
/** Observation Date (금리적용일자) */
private String inrtCmptYmd;

/** 조정금리 */
private BigDecimal inrtSprdItrt;

/** 수정사유 */
private String mdfcnRsn;

/** 일련번호 */
private Integer sn;

/** 최대 일련번호 (selectKey 채번용) */
private Integer maxSn;
```

### 프론트 JS (grd_list_onrightbuttonclick 저장 부분)
```javascript
com.win.message("QZZ0001", null, function(info) {
    if(info.click) {
        dma_rclcInrtMngReqVO.set("bizClssCd",              dl.getCellData(row, "bizClssCd"));
        dma_rclcInrtMngReqVO.set("ctrtNo",                 dl.getCellData(row, "ctrtNo"));
        dma_rclcInrtMngReqVO.set("aplcnYmd",               dl.getCellData(row, "aplcnYmd"));
        dma_rclcInrtMngReqVO.set("inrtCmptYmd",            inrtCmptYmd);
        dma_rclcInrtMngReqVO.set("inrtSprdItrt",           inrtSprdItrt);
        dma_rclcInrtMngReqVO.set("mdfcnRsnCn",             mdfcnRsn);
        dma_rclcInrtMngReqVO.set("flctnInrtRclcYmdClssCd", "89050001");
        dma_rclcInrtMngReqVO.set("cmptInrtClssCd",         dl.getCellData(row, "cmptInrtClssCd"));
        dma_rclcInrtMngReqVO.set("chrgDeptCd",             dl.getCellData(row, "deptCd")); // 화면 컬럼명 다를 때 이렇게 매핑
        dma_rclcInrtMngReqVO.set("frstRegUserId",          /* 세션 userId */);
        dma_rclcInrtMngReqVO.set("lastChgUserId",          /* 세션 userId */);
        com.sbm.execute("sbm_saveRepricingAplcn");
    }
});
```

---

## 4. 오늘 발생한 에러 전체 이력

| 에러 | 원인 | 해결 |
|------|------|------|
| `Failed to instantiate java.util.List` | Controller 파라미터에 `@RequestBody` 누락 | `@RequestBody` 추가 |
| `JSON parse error` | 프론트에서 List로 보내는데 서버 단건 VO 수신 불일치 | 단건 VO로 통일 |
| `Duplicate field RclcInrtMngReqVO.inrt` | ReqVO에 동일 필드 중복 선언 | 중복 필드 제거 |
| `JDBC-8026: Invalid identifier` | VALUES에 `NVL(RSNG_NO, ' ')` — `#{}` 없이 컬럼명 직접 사용 | `NVL(#{rsngNo, jdbcType=VARCHAR}, ' ')` 로 수정 |
| `JDBC-5074: Given string does not represent a number` | INSERT 컬럼 순서와 VALUES 순서 불일치 → 문자열이 숫자 컬럼 자리에 삽입 | 컬럼-VALUES 순서 1:1 정렬 |
| `NOT NULL constraint violation FLCTN_INRT_RCLC_YMD_CLSS_CD` | JS에서 해당 필드 세팅 누락 | `dma.set("flctnInrtRclcYmdClssCd", "89050001")` 추가 |
| `unsupported return type: RclcInrtMngListVO` | Mapper 인터페이스 반환타입 오류 | `void`로 변경 |
| `JDBC-10007: UNIQUE constraint violation PK_TIV_FLCTN_INRT_APLCN_B` | `selectKey` WHERE에 `BIZ_CLSS_CD`만 있어 다른 행도 같은 SN 채번 | `selectKey` WHERE에 PK 전체 조건 + `APLCN_YMD` 추가 |
| `java.lang.Integer cannot be cast to java.lang.String` | `selectKey` null 파라미터에 `jdbcType` 미지정 | `jdbcType=VARCHAR` 명시 |

---

## 5. 핵심 학습 포인트

### MyBatis
- `<insert>` Mapper 반환타입은 `void` 또는 `int`만 가능 (`<select>`만 VO 반환)
- `<selectKey>` + `<foreach>` 조합 지원 안 됨 → 단건 INSERT + 서비스 for문 패턴 사용
- null 가능한 VARCHAR 파라미터는 반드시 `jdbcType=VARCHAR` 명시
- `selectKey` WHERE 조건에 **PK 전체** 포함해야 SN 중복 채번 방지
- INSERT 컬럼 순서와 VALUES 순서 반드시 1:1 일치 확인

### WebSquare
- colgroup col 수 = tr 내 th+td 쌍 수 반드시 일치
- `dma.set("저장용필드명", dl.getCellData(row, "화면컬럼명"))` 으로 이름 다른 컬럼 매핑 가능
- `@JsonProperty`로 VO 필드명과 화면 전송 키 매핑 가능

### Spring
- `@RequestBody` 없으면 Spring이 JSON을 폼 파라미터로 바인딩 시도 → `Failed to instantiate` 에러

---

## 6. 미해결 / 다음 세션

- **조정금리(inrtSprdItrt) 초기화 이슈** — 저장 후 그리드에서 값 사라짐. DBeaver에서 실제 저장값 확인 필요
- **Observation Date 바인딩** — `sbm_selectObservationDateInrt` submitdone에서 `e.responseJSON.inrt` → `dataList1[selectedIndex].crtrInrt` 바인딩 마무리
