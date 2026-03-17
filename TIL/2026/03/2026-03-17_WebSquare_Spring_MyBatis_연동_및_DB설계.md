# TIL - 2026.03.17

## 개요
WebSquare5 + Spring MVC + MyBatis + MariaDB 기반 프로젝트 개발 중 발생한 이슈 및 해결 내용 정리

---

## 1. WebSquare `ref` 속성 (data: 바인딩)

### 개념
WebSquare XML에서 UI 컴포넌트를 데이터 모델에 연결할 때 `ref="data:..."` 형식 사용

### 구조
```xml
<!-- dataMap 선언 -->
<w2:dataMap id="dma_search">
    <w2:keyInfo>
        <w2:key id="KEYWORD" dataType="text" />
        <w2:key id="IS_USE"  dataType="text" />
    </w2:keyInfo>
</w2:dataMap>

<!-- UI 컴포넌트 바인딩 -->
<xf:input   ref="data:dma_search.KEYWORD" />
<xf:select1 ref="data:dma_search.IS_USE"  />

<!-- GridView 바인딩 -->
<w2:gridView dataList="data:dlt_sample" />

<!-- Submission 바인딩 -->
<xf:submission ref="data:json,dma_search" target="data:json,dlt_sample" />
```

### 주의사항
- 각 input의 `ref`는 서로 다른 key를 가리켜야 함
- 동일한 `ref`를 사용하면 두 컴포넌트가 같은 값을 공유하게 됨

---

## 2. WebSquare Studio 자동완성 설정

### 위치
```
Window → Preferences → WebSquare
```

### 설정 항목
| 항목 | 설명 |
|------|------|
| Enable Content Assist | 자동완성 on/off |
| Default JavaScript Proposals | JS 기본 자동완성 |
| WebSquare API Proposals | WebSquare API 자동완성 |

### 단축키
- `Ctrl + Space` : 자동완성 수동 호출

---

## 3. Spring Bean 등록 오류 해결

### 에러
```
NoSuchBeanDefinitionException: No bean named 'UserDao' available
```

### 원인
- MyBatis Mapper 인터페이스에 `@Mapper` 어노테이션 누락
- `applicationContext.xml`에 MapperScan 설정 누락

### 해결

#### ① UserDao.java에 @Mapper 추가
```java
@Mapper
@Repository("UserDao")
public interface UserDao {
    ...
}
```

#### ② applicationContext.xml에 mybatis:scan 추가
```xml
<!-- xmlns 추가 -->
xmlns:mybatis="http://mybatis.org/schema/mybatis-spring"

<!-- MapperScan 설정 추가 -->
<mybatis:scan base-package="com.inswave.wrm.with.sample.dao" />
```

---

## 4. ConflictingBeanDefinitionException 해결

### 에러
```
ConflictingBeanDefinitionException: Annotation-specified bean name 'eduDao'
conflicts with existing, non-compatible bean definition
```

### 원인
- 같은 이름의 Dao가 서로 다른 패키지에 존재
  - `com.inswave.wrm.with.sample.dao.EduDao`
  - `com.inswave.wrm.sample.dao.EduDao`

### 해결
`mybatis:scan`에 두 패키지 모두 명시
```xml
<mybatis:scan base-package="com.inswave.wrm.with.sample.dao;com.inswave.wrm.sample.dao" />
```

---

## 5. Unresolved compilation 에러 해결

### 에러
```
The import com.inswave.wrm.sample.dao cannot be resolved
EduDao cannot be resolved to a type
```

### 원인
`EduServiceImpl.java`의 import 경로가 실제 파일 위치와 불일치

### 해결
```java
// 잘못된 import
import com.inswave.wrm.sample.dao.EduDao;

// 올바른 import
import com.inswave.wrm.with.sample.dao.EduDao;
```

---

## 6. WebSquare 파일에 화면명 표시

### 방법
각 xml 파일 `<head>` 태그에 `meta_programName` 속성 추가

```xml
<head meta_programId="HM004M01"
      meta_programName="부서 관리"
      meta_programDesc="부서 목록 조회 및 관리 화면">
```

→ Project Explorer에서 파일명 옆에 자동 표시됨
```
HM004M01.xml - 부서 관리
```

---

## 7. DB 테이블 설계 및 생성

### 테이블 관계도
```
EMPLOYEE
USER_INFO ─────┐
ITEM_INFO ─────┼──→ LOAN_APPLY ──→ LOAN_EVAL ──→ LOAN_APPROVE
               └───────────────────────────────────────────────┘
```

### 업무 흐름
```
1단계. 고객 등록    → USER_INFO 저장 (USER_ID 자동생성: U + 날짜 + 순번)
2단계. 보증 신청    → LOAN_APPLY 저장 (CUR_STATE = '01. 심사중')
3단계. 보증 심사    → LOAN_EVAL 저장 (서류 상태, 보증 계산, 심사 결과)
4단계. 최종 승인    → LOAN_APPROVE 저장 (최종 결과)
```

### 상태값 흐름
```
01. 심사중 → 02. 심사완료 → 03. 승인완료 / 반려
```

### 최종 DDL (실행 순서)
```sql
-- 1. EMPLOYEE
CREATE TABLE EMPLOYEE (
    EMP_NO   VARCHAR(10) PRIMARY KEY COMMENT '직원코드(PK)',
    EMP_NM   VARCHAR(10) NOT NULL    COMMENT '직원이름'
);
INSERT INTO EMPLOYEE VALUES ('E001', '임무성');
INSERT INTO EMPLOYEE VALUES ('E002', '신지수');

-- 2. USER_INFO
CREATE TABLE USER_INFO (
    USER_ID    VARCHAR(20)  PRIMARY KEY COMMENT '고객 고유번호(PK)',
    USER_NM    VARCHAR(50)  NOT NULL    COMMENT '이름',
    RRN_NO     VARCHAR(20)  NOT NULL    COMMENT '주민등록번호',
    BIRTH_DT   VARCHAR(10)             COMMENT '생년월일',
    USER_TYPE  VARCHAR(20)             COMMENT '고객구분(일반/신혼부부/다자녀)',
    MARRY_DT   VARCHAR(10)             COMMENT '결혼날짜',
    CHILD_CNT  INT          DEFAULT 0  COMMENT '자녀수',
    TEL_NO     VARCHAR(20)             COMMENT '전화번호',
    ZIP_CD     VARCHAR(20)             COMMENT '우편번호',
    ADDRESS1   VARCHAR(100)            COMMENT '주소1',
    ADDRESS2   VARCHAR(100)            COMMENT '주소2'
);

-- 3. ITEM_INFO
CREATE TABLE ITEM_INFO (
    ITEM_CODE  VARCHAR(10)  PRIMARY KEY COMMENT '상품코드(PK)',
    ITEM_NM    VARCHAR(100) NOT NULL    COMMENT '상품명',
    LIMIT_AMT  BIGINT       DEFAULT 0  COMMENT '보증한도',
    ITEM_RATIO DECIMAL(5,2)            COMMENT '보증비율(%)',
    ITEM_RATE  DECIMAL(5,3)            COMMENT '보증료율(%)'
);
INSERT INTO ITEM_INFO (ITEM_CODE, ITEM_NM, LIMIT_AMT, ITEM_RATIO, ITEM_RATE)
VALUES ('P001', '일반전세자금보증', 400000000, 90.00, 0.200);
INSERT INTO ITEM_INFO (ITEM_CODE, ITEM_NM, LIMIT_AMT, ITEM_RATIO, ITEM_RATE)
VALUES ('P002', '신혼부부·다자녀 협약보증', 200000000, 100.00, 0.100);

-- 4. LOAN_APPLY
CREATE TABLE LOAN_APPLY (
    APPLY_NO       VARCHAR(30) PRIMARY KEY             COMMENT '신청번호(PK)',
    USER_ID        VARCHAR(20) NOT NULL                COMMENT '고객 고유번호(FK)',
    ITEM_CODE      VARCHAR(10) NOT NULL                COMMENT '상품코드(FK)',
    DEPOSIT_AMT    BIGINT      NOT NULL                COMMENT '임차보증금(전세금)',
    APPLY_AMT      BIGINT      NOT NULL                COMMENT '보증신청금액',
    HOUSE_CNT      INT         DEFAULT 0               COMMENT '주택보유수',
    IS_SPECULATIVE CHAR(1)     DEFAULT 'N'             COMMENT '규제지역 아파트 보유여부(Y/N)',
    APPLY_DT       VARCHAR(10) NOT NULL                COMMENT '신청일자',
    CUR_STATE      VARCHAR(20) DEFAULT '01. 심사중'   COMMENT '현재 진행 상태',
    FOREIGN KEY (USER_ID)   REFERENCES USER_INFO(USER_ID),
    FOREIGN KEY (ITEM_CODE) REFERENCES ITEM_INFO(ITEM_CODE)
);
ALTER TABLE LOAN_APPLY ADD CONSTRAINT UK_LOAN_APPLY UNIQUE (USER_ID, ITEM_CODE);

-- 5. LOAN_EVAL
CREATE TABLE LOAN_EVAL (
    EVAL_NO        VARCHAR(30)  PRIMARY KEY             COMMENT '심사번호(PK)',
    APPLY_NO       VARCHAR(30)  NOT NULL                COMMENT '신청번호(FK)',
    EMP_NO         VARCHAR(20)                          COMMENT '심사담당자(FK)',
    CALC_LIMIT_AMT BIGINT       DEFAULT 0               COMMENT '계산된 보증한도',
    CALC_RATIO     DECIMAL(5,2)                         COMMENT '계산된 보증비율(%)',
    CALC_RATE      DECIMAL(5,3)                         COMMENT '계산된 보증료율(%)',
    DOC_RESIDENT   VARCHAR(10)                          COMMENT '주민등록등본 상태',
    DOC_MARRIAGE   VARCHAR(10)                          COMMENT '혼인관계증명서 상태',
    DOC_FAMILY     VARCHAR(10)                          COMMENT '가족관계증명서 상태',
    DOC_INCOME     VARCHAR(10)                          COMMENT '소득금액증명서 상태',
    EVAL_RESULT    VARCHAR(10)                          COMMENT '심사결과(승인/반려/보완요청)',
    EVAL_COMMENT   VARCHAR(500)                         COMMENT '심사 의견',
    APPROVE_STATE  VARCHAR(20)  DEFAULT '03. 승인대기' COMMENT '승인상태',
    EVAL_DT        VARCHAR(10)                          COMMENT '심사일자',
    FOREIGN KEY (APPLY_NO) REFERENCES LOAN_APPLY(APPLY_NO),
    FOREIGN KEY (EMP_NO)   REFERENCES EMPLOYEE(EMP_NO)
);

-- 6. LOAN_APPROVE
CREATE TABLE LOAN_APPROVE (
    APPROVE_NO    VARCHAR(30)  PRIMARY KEY COMMENT '승인번호(PK)',
    EVAL_NO       VARCHAR(30)  NOT NULL    COMMENT '심사번호(FK)',
    APPLY_NO      VARCHAR(30)  NOT NULL    COMMENT '신청번호(FK)',
    EMP_NO        VARCHAR(20)              COMMENT '승인담당자(FK)',
    FINAL_RESULT  VARCHAR(20)             COMMENT '최종 결과 상태',
    FINAL_COMMENT VARCHAR(1000)           COMMENT '최종 심사 코멘트',
    APPROVE_DT    VARCHAR(10)  NOT NULL    COMMENT '최종 승인 일자',
    FOREIGN KEY (EVAL_NO)  REFERENCES LOAN_EVAL(EVAL_NO),
    FOREIGN KEY (APPLY_NO) REFERENCES LOAN_APPLY(APPLY_NO),
    FOREIGN KEY (EMP_NO)   REFERENCES EMPLOYEE(EMP_NO)
);
```

---

## 8. 오늘의 핵심 정리

| 항목 | 내용 |
|------|------|
| MyBatis Mapper Bean 등록 | `@Mapper` + `mybatis:scan` 둘 다 필요 |
| Bean 이름 충돌 | 패키지가 달라도 같은 이름이면 충돌 발생 |
| import 경로 | 실제 파일 위치와 정확히 일치해야 함 |
| WebSquare ref | dataMap id + key id 철자 정확히 일치 필요 |
| DB 생성 순서 | FK 참조 방향에 맞게 순서 지켜야 함 |
