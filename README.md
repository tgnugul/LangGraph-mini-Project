# 제주 맛집 추천 챗봇 (LangGraph + Gemini + SQL)

신한카드 가맹점 이용 데이터와 제주 관광지 조회수 데이터를 결합해, 자연어 질문에 맞는 제주 맛집·카페를 추천하는 대화형 에이전트입니다.
LangGraph로 질문 유형을 분기하고, LLM이 사용자 의도를 SQL로 변환해 SQLite에서 조회한 뒤 결과를 자연어로 설명합니다.

> 2025년 진행 프로젝트 · 개인 포트폴리오용 정리본

## 무엇을 하나요

| 질문 예시 | 처리 경로 |
|---|---|
| "공항 근처에 중식당 있나요?" | **검색형** — 조건에 맞는 가맹점 존재 여부/위치를 SQL로 조회 |
| "애월에서 5월에 뜨는 카페 추천해줘" | **추천형 · 인기** — 이용건수·이용금액 상위 그룹 + 관광지 계절 수요로 스코어링 |
| "협재 해수욕장 근처 저녁에 갈 만한 곳" | **추천형 · 동선/시간대** — 시간대별 이용 비율 + 지역 필터 |
| "OO식당이랑 비슷한 곳 추천해줘" | **추천형 · 유사** — 같은 업종·유사 객단가 그룹에서 탐색 |
| "오늘 날씨 어때?" | **범위 밖** — 맛집 관련 질문으로 유도 |

추천 시 성별·나이·여행 시기가 빠져 있으면 한 번 되물어 개인화에 반영합니다.

## 아키텍처

```
사용자 질문
   │
   ▼
classify_question ──── unknown ──▶ question_unknown ──▶ END
   │
   ├─ search ──▶ extract_mct_type ──▶ db_chain (LLM → SQL → 답변) ──▶ END
   │
   └─ recommend ──▶ ensure_required_info (성별/나이/시기 확인)
                       │
                       ▼
                 classification_concept
                       ├─ most_hot   (인기 스코어링 SQL)
                       ├─ time_route (시간대·동선 스코어링 SQL)
                       └─ similar    (유사 가맹점 SQL)
                       │
                       ▼
                 generate_sql ──▶ execute_sql ──▶ 자연어 답변 ──▶ END
```

- **LangGraph** `StateGraph` 로 노드·조건부 엣지 구성
- **Gemini 2.5 Flash-Lite** 로 질문 분류, 업종 추론, SQL 생성, 답변 생성
- **SQLite + SQLAlchemy** 에 가맹점·관광지 데이터를 테이블로 적재
- 업종 동의어 사전(`SYNONYM_DICT`) + LLM 추론의 2단계로 "빵집 → 베이커리", "해장 → 가정식/단품요리" 매핑
- 스코어링 규칙(인기, 시간대, 성별·연령, 지역, 월)은 JSON 파일로 분리해 프롬프트에 주입

## 데이터

| 파일 | 내용 | 출처 |
|---|---|---|
| `JEJU_MCT_DATA_v2.csv` | 2023년 제주 요식업 가맹점 월별 이용 지표 (매출 상위 30%). 이용건수/금액 구간, 시간대·요일별 이용 비율, 성별·연령대 고객 비율, 현지인 비율 등 | 신한카드 (2024 빅콘테스트) |
| `JT_MT_ACCTO_TRRSRT_SCCNT_LIST_2023MM.csv` | 관광지별 월 조회수 (12개 파일) | 제주관광공사 (비짓제주) |
| `JT_WKDAY_ACCTO_TRRSRT_SCCNT_LIST_2023MM.csv` | 관광지별 요일 조회수 (12개 파일) | 제주관광공사 (비짓제주) |
| `json/*.json` | 스코어링 규칙·컬럼 매핑 (인기, 시간대, 성별·연령, 지역, 월, 유사도) | 직접 작성 |

> 데이터 파일은 라이선스상 저장소에 포함하지 않았습니다. 위 출처에서 내려받아 `DATA_DIR` 경로에 두세요.

## 실행 방법

```bash
pip install langgraph langchain langchain-community langchain-experimental pydantic==2.9.2 sqlalchemy google-generativeai pandas
```

1. `제주_챗봇_agent_최종.ipynb` 를 엽니다 (Colab 기준으로 작성됨, 로컬 실행 시 데이터 경로만 수정).
2. Google AI Studio에서 발급한 API 키를 `GOOGLE_API_KEY` 셀에 입력합니다.
3. 데이터 로딩 → SQLite 적재 셀을 순서대로 실행합니다.
4. 마지막 셀에서 대화를 시작합니다. `exit` 입력 시 종료.

## 기술 스택

`Python` `LangGraph` `LangChain` `Gemini API` `SQLAlchemy` `SQLite` `pandas`

## 한계와 개선 방향

- **동일 결과 반복**: 스코어링이 6단계 구간 컬럼 위주라 동점이 많고, 성별·연령·시간대 정보가 SQL에 충분히 반영되지 않아 사용자가 달라도 같은 가게가 나오는 경우가 있습니다. → 연속형 지표 기반 가중합 스코어링, MMR 다양화, 세션 내 기추천 제외로 개선 예정
- **관광지 데이터 결합**: 가맹점명–관광지명 직접 조인은 매칭률이 낮아, 읍·면·동 단위 집계로 결합하는 방식으로 전환 필요
- **LLM 의존도**: 턴당 LLM 호출이 5회 이상이라 무료 티어에서 429 에러가 잦음 → 의도 추출·답변 생성 2회로 축소하고 랭킹은 코드로 이전
- **평가 지표 부재**: Precision@K, 다양성(ILD) 등 오프라인 평가 추가 예정

## 프로젝트 정보

- 기간: 2025년
- 역할: 데이터 전처리, LangGraph 파이프라인 설계, 프롬프트·스코어링 규칙 작성
