# 2026-05-08 이차보전 금액검증 탭 추가 및 다중 그리드 조회 구현 + Repricing 디버깅

## 오늘 한 일
- 이차보전 금액검증 화면에 새 탭 추가 (dlt_IdsAmtCmptC 그리드)
- 기존 submission 재사용하여 두 개 쿼리 동시 조회 로직 완성
- Repricing saveRepricingAplcn selectKey 에러 2건 해결

---

## 1. WebSquare 탭 + 그리드 추가

### 작업 순서
1. 탭 컴포넌트에 새 탭 항목 추가
2. 탭 내부에 gridView 추가 (`dlt_IdsAmtCmptC`)
3. 컬럼 정의 (`w2:column`) 추가
4. DataCollection에 `dlt_IdsAmtCmptC` 바인딩

### 핵심 교훈
- colgroup의 col 수와 tr 내 th+td 쌍 수 반드시 일치해야 레이아웃 안 깨짐
- 그리드 footer 없는 컬럼에 `<w2:footer></w2:footer>` 빈 태그 명시 안 하면 `.gridFooterTableDefanlt` 스타일 자동 적용됨 → 빈 태그라도 명시할 것

---

## 2. 기존 submission 재사용 (단일 엔드포인트 → 두 그리드 바인딩)

### 핵심 패턴
```xml
target='data:json,[{"id":"dlt_IdsAmtCmptB","key":"idstAmtCmptListBVO"},{"id":"dlt_IdsAmtCmptC","key":"idstAmtCmptListCVO"}]'
```
- submission 1개로 두 그리드에 동시 바인딩 가능
- key 값은 ResVO 필드명과 **대소문자까지 정확히 일치** 필수

---

## 3. 백엔드 구조

### ResVO에 필드 추가
```java
private List<IdstAmtCmptListCVO> idstAmtCmptListCVO;
// getter/setter 필수
```

### ServiceImpl — 쿼리 2개 호출
```java
idstAmtCmptResVO.setIdstAmtCmptListBVO(idstAmtCmptMapper.selectIdstAmtCmptList(vo));
idstAmtCmptResVO.setIdstAmtCmptListCVO(idstAmtCmptMapper.selectIdstAmtCmptListC(vo));
```

### Mapper XML
```xml
<!-- 쿼리 ID는 반드시 다르게 -->
<select id="selectIdstAmtCmptListC" resultType="...IdstAmtCmptListCVO">
    -- C 그리드용 쿼리
</select>
```

---

## 4. 트러블슈팅 - 이차보전

| 증상 | 원인 | 해결 |
|------|------|------|
| C 그리드 `[]` 빈 배열 반환 | ResVO에 C 리스트 필드 미추가 | `idstAmtCmptListCVO` 필드 + getter/setter 추가 |
| 바인딩 안 됨 | target key 대소문자 불일치 | ResVO 필드명과 1:1 대조 |
| 쿼리 집계함수 매핑 안 됨 | `SUM(컬럼)` alias 없으면 MyBatis 매핑 불가 | `SUM(컬럼) AS alias` 명시 |

---

## 5. 트러블슈팅 - Repricing saveRepricingAplcn selectKey

| 에러 | 원인 | 해결 |
|------|------|------|
| `JDBC-8026: Invalid identifier` (APLCN_YMD) | `selectKey` FROM 테이블이 `TIV_RPMT_SCHD_D` — APLCN_YMD 컬럼 없음 | `TIV_FLCTN_INRT_APLCN_B`로 수정 |
| `JDBC-8026: Invalid identifier` (APLCN_YMD = ?) | 필수값 APLCN_YMD에 불필요한 `NVL(..., ' ')` 적용 → 타입 불일치 | `#{aplcnYmd}` 단순 바인딩으로 수정 |

### 핵심 교훈
- `selectKey` FROM 테이블은 **INSERT 대상 테이블과 동일**해야 함
- `NVL(#{param, jdbcType=VARCHAR}, ' ')` 패턴은 null 가능한 PK에만 적용
- 필수값(NOT NULL, 날짜 등)에 NVL 붙이면 타입 불일치로 `Invalid identifier` 발생

---

## 핵심 요약
- **프론트 submission 1개 + target 배열** → 여러 그리드 동시 바인딩 가능
- **백엔드는 쿼리 ID 별도, ResVO 필드 추가, 서비스에서 각각 호출**
- 데이터 안 나올 때 확인 순서: Network Response → ResVO 필드 → resultType → alias
- selectKey FROM 테이블 틀리면 엉뚱한 Invalid identifier 발생 → INSERT 대상 테이블 재확인
