# legal-kb 플러그인

한국 법령 조문·판례·결정례를 **기준일 기준**으로 찾아 근거 묶음을 돌려주는 원격 MCP 서버를
Claude Code 에 붙인다. 답변 문장은 만들지 않는다 — 호출한 모델이 쓴다.

## 설치

```bash
claude plugin marketplace add risingdream/legal-kb-plugin
claude plugin install legal-kb@legal-kb-plugin
```

Claude Desktop 은 설정 → 플러그인 → 추가 → 저장소에서 추가 → `risingdream/legal-kb-plugin`.

설치 직후 `/mcp` 에서 `legal-kb` 를 고르면 브라우저가 열린다. 로그인하고 허용하면 끝이다.
키를 복사해 붙여넣는 단계는 없다.

**허용 목록에 등록된 계정만** 연결된다. 로그인은 되는데 허용 단계에서 막히면
운영자에게 로그인한 이메일로 등록을 요청한다.

## 도구

| 도구 | 쓰임 |
|---|---|
| `research` | 질문 하나로 조문+판례를 한 번에. 처음 부를 도구 |
| `search_legal` | 하이브리드 검색. 조문·판례 목록을 따로 받는다 |
| `get_article` | 법령명+조문번호+기준일로 그 개정판 본문 |
| `get_decision` | 사건번호로 판례·결정례 전문 |
| `neighbors` | 본법↔시행령, 조문→판례 연결선 |
| `web_search` | legal-kb 에 없는 최신 사실(올해 세율·고시·보도자료)만 |

## 스킬 — 도구 사용법까지 같이 온다 (v1.1.0)

도구만 붙이면 "어떤 순서로 몇 번 부를지"는 모델이 알아서 판단한다. 그 판단이 답의 품질을 좌우한다 —
같은 모델·같은 31문항에서 검색을 한 번만 받아 답한 경우보다 **도구를 골라 평균 3.3회 부른 경우가
judge overall +0.26** 높았다(RAG-8083).

그래서 `legal-research` 스킬이 절차를 싣는다.

- 호출 순서 — `research` 먼저, 부족한 것만 `get_article` · `get_decision` · `search_legal` 로 채운다
- `web_search` 를 **쓸 때와 안 쓸 때** — 판례·결정례 법리 질문에서 웹은 인용할 조문·사건번호를 밀어낸다(사건번호 −0.064)
- 인용 형식 — 「법령명」 제N조 + 시행일, 판례는 사건번호 + 선고일
- 기준일(`as_of`)과 현행성 — 구법 `[경고]` · 시행예정 `[대조]`
- 수록 범위 — **조세·관세·지방세 법령 79종.** 민법·상법 조문은 없다

스킬은 켜지기 전까지 이름과 한 줄 설명(30~50 토큰)만 쓴다. 법률 질문이 들어올 때만 본문이 실린다.

## 슬래시 커맨드

모델 판단을 건너뛰고 바로 부른다.

| 커맨드 | 하는 일 |
|---|---|
| `/legal-kb:법령 <질의>` | `research` 로 조문·판례 근거를 모아 정리 |
| `/legal-kb:조문 <법령명> <조문번호>` | `get_article` 로 그 개정판 본문과 시행일. 약칭은 정식 명칭으로 펴서 부른다 |
| `/legal-kb:판례 <사건번호>` | `get_decision` 으로 전문. 예: `조심 2020서1582` |

## 업데이트

```bash
claude plugin marketplace update legal-kb-plugin
claude plugin update legal-kb@legal-kb-plugin
```

## claude.ai 웹 채팅

claude.ai 는 플러그인을 받지 않는다. 사용자 지정 → 커넥터 → 추가에서
이름 `legal-kb`, URL `https://law.tax-insight.kr/mcp` 로 등록한다.
도구 6종은 같고, 스킬·슬래시 커맨드는 붙지 않는다.
