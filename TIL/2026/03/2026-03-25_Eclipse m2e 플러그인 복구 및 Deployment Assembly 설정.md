# 2026-03-25_Eclipse m2e 플러그인 복구 및 Deployment Assembly 설정

## 📌 카테고리
- 트러블슈팅 — Eclipse / Maven / WebSquare Studio

---

## 🔥 문제 상황

GitHub에 파일 커밋 후 Tomcat 서버 구동 시 아래 에러 발생:

```
ClassNotFoundException: websquare.http.controller.JavascriptInitializer
ClassNotFoundException: org.springframework.web.context.ContextLoaderListener
Context [] startup failed due to previous errors
```

→ Maven 의존성 JAR가 배포 경로에 포함되지 않아 서버가 클래스를 찾지 못하는 상태

---

## 🔍 원인 분석

### 1단계 — git과 무관함 확인
- `git diff HEAD~1 HEAD --name-only` 결과: `Common_Popup01.xml` 하나만 변경
- git이 원인이 아님

### 2단계 — Eclipse Build Path 확인
- `Properties → Java Build Path → Libraries` 탭 확인
- `org.eclipse.m2e.MAVEN2_CLASSPATH_CONTAINER` 가 **Persisted container** 상태로 표시
- Maven 메뉴가 프로젝트 우클릭에 **존재하지 않음**

### 3단계 — m2e 플러그인 누락 확인
```
C:\WEBSQUARE_DEV_PACK_SP5\bin\websquare\plugins\
```
→ `org.eclipse.m2e.*` 파일이 **하나도 없음**

동료 PC와 비교 시 아래 10개 파일 누락 확인:
- `org.eclipse.m2e.archetype.common_1.15.0.20200310-1832/`
- `org.eclipse.m2e.maven.indexer_1.15.0.20200310-1832/`
- `org.eclipse.m2e.maven.runtime_1.15.0.20200310-1832/`
- `org.eclipse.m2e.maven.runtime.slf4j.simple_1.15.0.20200310-1832/`
- `org.eclipse.m2e.core_1.15.0.20200305-1308.jar`
- `org.eclipse.m2e.core.ui_1.15.0.20200305-1308.jar`
- `org.eclipse.m2e.jdt_1.15.0.20200226-1722.jar`
- `org.eclipse.m2e.launching_1.15.0.20200225-1921.jar`
- `org.eclipse.m2e.model.edit_1.15.0.20200225-1921.jar`
- `org.eclipse.m2e.workspace.cli_0.4.0.jar`

---

## 🛠 해결 과정

### STEP 1 — m2e 플러그인 파일 복사
동료 PC의 `plugins` 폴더에서 위 10개 파일/폴더 복사 →  
`C:\WEBSQUARE_DEV_PACK_SP5\bin\websquare\plugins\` 에 붙여넣기

### STEP 2 — bundles.info 등록
파일만 넣어도 Eclipse가 인식 못함. 직접 등록 필요:

```
C:\WEBSQUARE_DEV_PACK_SP5\bin\websquare\configuration\org.eclipse.equinox.simpleconfigurator\bundles.info
```

아래 6줄 추가 (기존 `org.eclipse.m2e.archetype.common` 줄 아래):

```
org.eclipse.m2e.core,1.15.0.20200305-1308,plugins/org.eclipse.m2e.core_1.15.0.20200305-1308.jar,4,false
org.eclipse.m2e.core.ui,1.15.0.20200305-1308,plugins/org.eclipse.m2e.core.ui_1.15.0.20200305-1308.jar,4,false
org.eclipse.m2e.jdt,1.15.0.20200226-1722,plugins/org.eclipse.m2e.jdt_1.15.0.20200226-1722.jar,4,false
org.eclipse.m2e.launching,1.15.0.20200225-1921,plugins/org.eclipse.m2e.launching_1.15.0.20200225-1921.jar,4,false
```

### STEP 3 — guava 의존성 추가
Error Log 확인 시 추가 에러 발견:

```
Unresolved requirement: Require-Bundle: com.google.guava; bundle-version="[27.1.0,28.0.0)"
```

- 동료 PC에서 `com.google.guava_27.1.0.v20190517-1946.jar` 복사 → plugins에 추가
- bundles.info에도 등록:

```
com.google.guava,27.1.0.v20190517-1946,plugins/com.google.guava_27.1.0.v20190517-1946.jar,4,false
```

### STEP 4 — Maven 메뉴 확인 및 Update Project
Studio 재시작 후 프로젝트 우클릭 → **Maven 메뉴 복구 확인**  
→ `Maven → Update Project` 실행

### STEP 5 — Deployment Assembly 수정
Maven Update 후 다시 에러 발생:
```
Cannot find entry: "/target/m2e-wtp/web-resources"
ClassNotFoundException: org.springframework.web.context.ContextLoaderListener
```

`Properties → Deployment Assembly` 에서:
1. `/target/m2e-wtp/web-resources` → **Remove**
2. **Add → Java Build Path Entries → Maven Dependencies**
3. Deploy Path: `WEB-INF/lib` 입력
4. **Apply and Close** → 서버 재시작

---

## ✅ 결과
- Maven 메뉴 정상 복구
- Tomcat 서버 정상 구동
- Maven 의존성 JAR가 `WEB-INF/lib` 에 정상 배포됨

---

## 💡 핵심 교훈

| 항목 | 내용 |
|------|------|
| m2e 플러그인 위치 | `{WebSquare설치경로}/bin/websquare/plugins/` |
| 플러그인 등록 파일 | `configuration/org.eclipse.equinox.simpleconfigurator/bundles.info` |
| 파일만 넣으면 안 됨 | bundles.info에 반드시 수동 등록 필요 |
| Maven 저장소 위치 | `C:\WEBSQUARE_DEV_PACK_SP5\maven\repository\` (기본 `.m2` 아님) |
| Maven Update 후 | Deployment Assembly 재확인 필요 |

---

## 🔗 관련 파일
- `bundles.info` — `C:\WEBSQUARE_DEV_PACK_SP5\bin\websquare\configuration\org.eclipse.equinox.simpleconfigurator\`
- `websquare.ini` — `C:\WEBSQUARE_DEV_PACK_SP5\bin\websquare\`
