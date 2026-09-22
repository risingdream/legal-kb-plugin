# legal-kb-plugin

한국 국가법령(법률·대통령령·부령 등, 세법 포함) 조문과 판례·결정례를 **기준일 기준**으로 찾아 근거를 돌려주는
원격 MCP 서버(`https://law.tax-insight.kr/mcp`)의 에이전트 플러그인 마켓플레이스다.
답변 문장은 만들지 않는다. 호출한 모델이 근거를 받아 쓴다.

이 저장소에는 MCP 연결 설정과 도구 사용 절차(스킬·슬래시 커맨드)만 있다. 검색 엔진과 데이터는 공개하지 않는다.

## 설치

| 클라이언트 | 방법 |
|---|---|
| Claude Code | `claude plugin marketplace add risingdream/legal-kb-plugin` 후 `claude plugin install legal-kb@legal-kb-plugin` |
| Claude Desktop (Code·Cowork) | 설정 → 플러그인 → 추가 → 저장소에서 추가 → `risingdream/legal-kb-plugin` |
| claude.ai 웹 채팅 | 플러그인 대신 커넥터: 사용자 지정 → 커넥터 → 추가 → 이름 `legal-kb`, URL `https://law.tax-insight.kr/mcp` |

설치 뒤 첫 연결 때 브라우저 로그인·허용 화면이 한 번 뜬다. Claude Code 는 `/mcp` → `legal-kb` → 인증.
키를 복사해 붙여넣는 단계는 없다.

**허용 목록에 등록된 계정만 연결된다.** 로그인은 되는데 허용 단계에서 막히면 운영자에게
로그인한 이메일로 등록을 요청한다.

## 업데이트

```bash
claude plugin marketplace update legal-kb-plugin
claude plugin update legal-kb@legal-kb-plugin
```

## 들어 있는 것

- 도구 6종 — `research` · `search_legal` · `get_article` · `get_decision` · `neighbors` · `web_search`
- 스킬 `legal-research` — 도구 호출 순서, 웹 검색을 쓸 때와 안 쓸 때, 「법령명」 제N조 + 시행일 인용 형식
- 커맨드 `/legal-kb:법령` · `/legal-kb:조문` · `/legal-kb:판례` (Claude Code·Desktop 전용)

자세한 내용은 [`plugins/legal-kb/README.md`](plugins/legal-kb/README.md).

> 이 저장소는 비공개 서비스 저장소의 `plugins/legal-kb/` 에서 동기화된다. 여기에 직접 커밋하지 않는다.
