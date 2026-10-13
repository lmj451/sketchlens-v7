# lmj451-sketchlens-frontend-ui-v7

진단 구조를 **MobileNetV3(분류) → LLM(채점·개선 문장)** 2단계로 바꾼 버전. 파일 구조는 v5·v6과 같음 (html / css / js 분리).

바로 열기: `SketchLens_v7.html` (GitHub Pages를 켜면 `https://<아이디>.github.io/<저장소>/SketchLens_v7.html`)

## 진단 흐름
1. **1단계 · MobileNetV3 (분류)**
   전이학습한 분류 모델. 클래스 = 의류 카테고리 N개 + "기타"(비패션 이미지)
   이미지 → 클래스별 확률
2. **분기 (서버, `decideRelevance()`)**
   - 가장 높은 클래스가 "기타" → 거절 (LLM 호출 안 함)
   - 확률 ≥ 0.8 → 통과
   - 0.4 ≤ 확률 < 0.8 → 사용자 재확인 → "스케치가 맞아요"를 누르면 2단계 진행
   - 확률 < 0.4 → 거절
3. **2단계 · 멀티모달 LLM (채점 + 개선 문장)**
   이미지 + 1단계 카테고리 + 채점 기준표(고정 프롬프트) + 고민 글·focus → 세부 기준별 rating·observation + 항목별 advice
4. **서버 (점수 계산, `merge()`)**
   항목 점수 = 유효한 세부 기준 점수 합계 ÷ (유효한 세부 기준 개수 × 2) × 100 → 최종 응답으로 합쳐 화면에 전달

## v6 → v7 변경 사항
- DeepSeek 2단계 구조 → MobileNetV3 분류 + LLM 채점 구조로 변경
- 로딩 문구: "1단계: MobileNetV3로 … 분류하는 중" → "2단계: LLM이 … 채점하는 중"
- 진단 결과 상단에 분류 결과 표시 (예: 분류 티셔츠 92%)
- 입력·출력 탭: 1단계 분류 응답 / 2단계 LLM 응답 / 최종 응답(서버 점수 계산 후)을 각각 표시
  - 1단계에서 거절되거나 재확인 대기 중이면 1단계 응답만 표시
- 기준 가이드: "진단 과정" 카드 추가, 패션 스케치 확인 기준을 분류 결과 기준으로 변경
- 코드·문서에서 "Django" → "백엔드 서버" (서버 언어 미정)

## 지금 상태
서버 연동 전이라 `buildDemoClassify()` / `buildDemoScore()`가 이미지 선을 간단히 계산해
**실제 응답과 같은 모양의 JSON**을 만든다. 분류 결과와 점수 값 자체는 의미 없고, 화면 흐름과 입출력 형식 확인용.

## 백엔드 연동 시 바꿀 곳 (js/sketchlens.js에서 `TODO(백엔드 연동 지점)` 검색)
서버 언어와 상관없이 주고받는 값은 같다.

1. `requestClassify()` → `POST /api/classify` FormData(image)
2. `requestScore()` → `POST /api/score` {diagnosis_id, worry, focus}
3. `merge()`의 점수 계산은 실제로는 서버가 한다. 서버가 최종 응답을 주면 프론트는 `render()`만 호출하면 됨

### 1단계 응답 예시 (`/api/classify`)
```json
{
  "diagnosis_id": "…",
  "model": "mobilenetv3_small",
  "top": { "label": "티셔츠", "prob": 0.92 },
  "predictions": [
    { "label": "티셔츠", "prob": 0.92 },
    { "label": "셔츠", "prob": 0.05 },
    { "label": "원피스", "prob": 0.03 }
  ]
}
```

### 최종 응답 예시 (`/api/score`, 서버가 점수 계산까지 마친 것)
```json
{
  "criteria_version": "v1_draft",
  "relevance": { "category": "티셔츠", "confidence": 0.92 },
  "focus": ["proportion", "balance"],
  "scores": { "line_quality": 83, "proportion": 67, "silhouette": 100, "balance": 83, "detail": 100 },
  "evidence": {
    "proportion": {
      "status": "analyzed",
      "subratings": [
        { "criterion": "대응 부위의 길이 차이", "status": "analyzed", "rating": 0, "observation": "오른쪽 선 양이 반대쪽의 1.43배" },
        { "criterion": "대응 부위의 너비 차이", "status": "analyzed", "rating": 2, "observation": "중심선 기준 좌우 폭 차이가 전체 너비의 2%" },
        { "criterion": "부위 간 크기 관계", "status": "analyzed", "rating": 2, "observation": "전체 세로/가로 비율 0.90" }
      ],
      "earned_points": 4, "max_points": 6,
      "advice": "어깨너비와 총장 기준을 먼저 정하고, 좌우 소매처럼 짝이 되는 부위의 길이를 같은 기준선에서 다시 재보세요."
    }
  }
}
```

## 확인 필요
- 분류 클래스 목록(`CATEGORIES`)은 데모용. 실제로는 학습 데이터의 카테고리에 맞춰 정할 것.
- 선 품질 외 4개 항목의 세부 기준은 초안 (`AXES` 배열). 확정하면 LLM 고정 프롬프트의 채점 기준표와 함께 바꿀 것.
- 고민 글 → focus 해석은 지금 키워드 규칙(`RULES`). LLM에 맡길지는 테스트 후 결정.
