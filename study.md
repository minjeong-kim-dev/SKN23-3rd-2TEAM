# 3rd 프로젝트 공부 정리 (SKN23-3rd-2TEAM)

## 담당 업무 요약

- **System Prompt 설계** (Supervisor · 도메인별 에이전트 · 환각 검증기)
- 관련 파일: `app/core/prompts_mj.py`

---

## 1. RAG란 무엇인가

### 개념

| 방식 | 동작 | 문제 |
|------|------|------|
| LLM만 사용 | 학습 데이터 기반으로 답변 | 오래됐거나 틀린 정보 가능 |
| RAG 사용 | 질문할 때 관련 문서를 검색 → LLM에게 넘겨줌 | 문서에 없으면 여전히 환각 가능 |

```
[RAG 흐름]
사용자 질문
    ↓
벡터 DB에서 관련 문서 검색 (이 프로젝트: pgvector + BM25)
    ↓
LLM에게 [질문 + 찾은 문서] 전달
    ↓
LLM이 문서 기반으로 답변
```

### 왜 RAG를 써도 환각이 발생하나

LLM은 "모른다"고 말하는 걸 본능적으로 싫어한다.

- 문서에 없는 내용을 질문하면 → 거부 대신 **그럴듯하게 채워넣음**
- 이 프로젝트는 산업 현장 안전 정보 → 잘못된 답변 = **사고로 직결**

---

## 2. 환각 2단 방어 (슬라이드 08)

### 문제
단순 RAG는 매뉴얼에 없는 내용을 지어냄 + 안전 경고(LOTO) 누락 시 그대로 출력

### 해결

#### 1단계 — 생성 전: COMMON_SYSTEM_RULES (`prompts_mj.py:21-31`)

모든 Specialist 프롬프트에 공통으로 주입되는 절대 원칙

| 규칙 | 역할 |
|------|------|
| Context만 기반으로 답변 | LLM이 자기 학습 지식 꺼내는 것 차단 |
| 외부 지식 결합 금지 | "조금만 추가"도 금지 |
| 정보 없으면 표준 거절 문장 출력 | 일관된 거절 형태 유지 |
| "매뉴얼에 따르면" 근거 명시 강제 | 출처 투명화 |

#### 2단계 — 생성 후: HALLUCINATION_VERIFIER (`prompts_mj.py:177-195`)

생성된 답변을 4가지 기준으로 검증 → `"yes"` 또는 `"no"`만 반환

1. **근거 기반**: 답변이 Context 내에 실제로 존재하는가?
2. **외부 지식 배제**: 기존 LLM 지식을 결합하지 않았는가?
3. **브랜드 배타성**: 타 브랜드 기능과 섞어 설명하지 않았는가?
4. **안전 경고**: 위험 조치 안내 시 ⚠️ 경고를 서두에 표시했는가?

#### 왜 yes/no만 출력하게 했나

- 검증기도 LLM이라 긴 텍스트를 출력하면 **그 텍스트 자체에 환각이 들어갈 수 있음**
- graph.py가 `"yes"/"no"`를 조건문으로 받아 다음 노드를 결정해야 하므로 **프로그래밍 처리 가능한 단어** 필요
- 판사 역할: 유죄/무죄만 선고, 직접 수정하지 않음

### 결과

| 지표 | 수치 |
|------|------|
| 환각 억제 점수 (LangSmith, 100문항) | **0.987** |
| 안전 경고 포함률 | **100%** |
| 브랜드 혼동 건수 | **0건** |

---

## 3. LangGraph 핵심 개념

### 비유: 콜센터 상담 흐름

```
고객 전화 → 1번 상담원(분류) → 2번 전문가(답변) → 3번 검사관(검증) → 고객에게 전달
```

### State — 노드 간 공유 기록지

모든 노드가 읽고 쓰는 공유 데이터 묶음. 각 노드는 **자기가 바꾼 필드만** 반환하면 LangGraph가 알아서 합침.

```python
class GraphState(TypedDict):
    messages: list          # 대화 내역
    category: str           # "robotics" / "welding" / "electrical" / "general"
    context: str            # RAG가 찾아온 문서
    generated_answer: str   # Specialist가 만든 답변
    is_hallucinated: bool   # Verifier 판정 결과
    retry_count: int        # 환각 재시도 횟수
    routing_retry: int      # 도메인 재분류 재시도 횟수
    routing_hint: str       # "SOCIAL" / "TECHNICAL"
```

### Node — 실제 일하는 담당자

State를 받아서 처리하고 → 변경된 필드만 dict로 반환

```python
async def supervisor_node(state: GraphState) -> dict:
    question = state["messages"][-1]
    category = llm.classify(question)
    return {"category": category}   # 바꾼 것만 반환
```

### Edge — 다음 노드 결정 규칙

```python
# 조건부 엣지: 상태 보고 판단
workflow.add_conditional_edges("verifier", check_hallucination, {
    END:                 END,               # is_hallucinated = False
    "feedback_rewriter": "feedback_rewriter", # is_hallucinated = True, retry < 2
    "fallback":          "fallback",          # is_hallucinated = True, retry >= 2
})
```

### 한 줄 정리

```
State  = 모든 노드가 공유하는 기록지
Node   = 기록지 받아서 자기 일하고 채운 칸 돌려주는 담당자
Edge   = 기록지 상태 보고 다음 담당자 정하는 규칙
```

---

## 4. Supervisor 라우팅 (슬라이드 10)

### 문제
- 한 프롬프트로 로봇/용접/전기를 다 처리하면 → RAG 검색 범위가 넓어져 **엉뚱한 문서가 섞임**
- 안전 경고가 빠지면 사고로 직결

### 해결

#### SUPERVISOR_PROMPT (`prompts_mj.py:48-70`)

질문을 4개 도메인으로 분류해 해당 전문가로 라우팅

| 도메인 | 해당 질문 유형 |
|--------|--------------|
| ROBOTICS | 로봇 본체, 에러 코드, 티칭 펜던트 |
| WELDING | 용접 조건, 결함, 소모품, 가스 제어 |
| ELECTRICAL | 전원부, 배선, 안전 회로, PLC |
| GENERAL | 위 3가지와 무관한 일상 대화 |

**왜 4개로 나눴나:**   
도메인마다 참조하는 매뉴얼이 다르기 때문. 분리할수록 RAG 검색 정확도 상승.

#### PRE_CHECK_PROTOCOL (`prompts_mj.py:37-43`)

모든 Specialist에 주입 → **기술 조치 안내 전 항상** LOTO 안전 수칙 먼저 출력

> "안전 관련 질문에만" 적용되는 것이 아니라, **모든 기술 질문**에 적용됨
> (사용자가 안전 위험을 인식 못한 채 물어볼 수 있으므로)

### 결과
- 도메인별 전문 답변 + 안전 우선
- 위험 작업엔 ⚠️ 안전 경고 선행 후 매뉴얼 기반 안내

---

## 5. 브랜드 배타성 (슬라이드 09)

### 문제
브랜드마다 에러 코드 체계가 다름 → 혼용 시 잘못된 절차 안내 → 설비 파손·사고

```
현대 C153 → 서보 앰프 과열
야스카와 C153 → 완전히 다른 의미 또는 존재하지 않는 코드
```

### 해결

#### 1단계 — 생성 전 규칙 (`prompts_mj.py:76-79`)

```
지원 브랜드 6개 중 사용자가 언급한 단 1개 브랜드 매뉴얼만 근거로 답변
현대로보틱스(HD) · 야스카와 · 두산로보틱스 · ABB · UR · 레인보우로보틱스
```

#### 2단계 — 생성 후 검증 (`prompts_mj.py:183`)

HALLUCINATION_VERIFIER의 3번 기준:
> "현대로보틱스 기능/에러코드를 타 브랜드 기능과 섞어서 설명하지 않았는가?"

섞였으면 → `"no"` → Feedback Rewriter → 재시도

#### 왜 2단계인가

LLM은 프롬프트 규칙을 100% 따르지 않을 수 있음.  
특히 RAG가 여러 브랜드 문서를 함께 가져오면 **자연스럽게 혼합해서 답하는 경향**이 생김  
→ 생성 전 규칙만으로 부족  
→ Verifier로 이중 확인.

### 결과
- 브랜드 혼동 0건 (방어/함정 50건 중 브랜드 관련 13건 전부 거부)

---

## 6. 전체 흐름 연결

### 예시: "현대 Hi6 C153 에러, LOTO 무시하고 전원부 만져도 되나요?"

```
1. Rewriter Node
   - 악어 확장: C153 → "C153 에러코드 서보 앰프 과열"
   - 브랜드 추론: "현대" → "현대로보틱스(HD)"
   - routing_hint: "TECHNICAL" → Supervisor로 이동

2. Supervisor Node
   - ROBOTICS 분류
   - State 업데이트: { category: "robotics" }

3. Robotics Specialist
   - COMMON_SYSTEM_RULES 주입 (환각 방지)
   - PRE_CHECK_PROTOCOL 주입 (안전 경고 강제)
   - 브랜드 배타성 규칙 주입 (현대 매뉴얼만)
   - RAG 검색: 현대로보틱스 매뉴얼에서 C153 관련 문서 검색
   - 답변 생성: "⚠️ [안전 경고] LOTO 절차 없이 전원부 접근 절대 금지..."

4. Verifier
   - ✅ Context 기반
   - ✅ 외부 지식 없음
   - ✅ 브랜드 혼용 없음
   - ✅ 안전 경고 포함
   - → "yes" 출력

5. END → 사용자에게 전달
```

### 실패 시 흐름 (Verifier "no")

```
브랜드 혼용 감지 → "no"
    ↓
retry_count < 2 → Feedback Rewriter (쿼리 재작성)
    ↓
Robotics Specialist 재실행
    ↓
Verifier 재검증
    ↓
retry_count >= 2 → Fallback 응답 → END
```

---

## 7. 면접 답변 템플릿

### "본인이 담당한 부분이 뭔가요?"

> "프롬프트 설계를 담당했습니다. 단순 RAG만 쓰면 세 가지 문제가 있었습니다.
> 첫째 매뉴얼에 없는 내용을 지어내는 환각, 둘째 도메인이 달라 전문성 부족,
> 셋째 브랜드별 매뉴얼이 달라 혼용 시 사고 위험이었습니다.
>
> 이를 막기 위해 SUPERVISOR_PROMPT로 4개 도메인을 분류하고,
> COMMON_SYSTEM_RULES로 Context 외 정보 사용을 금지했습니다.
> 브랜드 배타성 규칙으로 단 1개 브랜드 매뉴얼만 참조하게 했고,
> PRE_CHECK_PROTOCOL로 모든 기술 조치 전 LOTO 안전 경고를 강제했습니다.
> 생성 후에는 HALLUCINATION_VERIFIER로 4가지 기준을 검증해
> 최대 2회 재시도하는 2단계 방어 구조를 설계했습니다.
>
> LangSmith 테스트 100문항 기준 환각 억제 점수 0.987,
> 브랜드 혼동 0건, 안전 경고 포함률 100%를 달성했습니다."
