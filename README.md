# peritia-plugin

한국 국가법령(법률·대통령령·부령 등, 세법 포함) 조문과 판례·결정례를 **기준일 기준**으로 찾아 근거를 돌려주는
페리티아(Peritia) 원격 MCP 서버(`https://peritia.kr/mcp`)의 에이전트 플러그인 마켓플레이스다.
답변 문장은 만들지 않는다. 호출한 모델이 근거를 받아 쓴다.

이 저장소에는 MCP 연결 설정과 도구 사용 절차(스킬·슬래시 커맨드)만 있다. 검색 엔진과 데이터는 공개하지 않는다.

## 설치

| 클라이언트 | 방법 |
|---|---|
| Claude Code | `claude plugin marketplace add risingdream/peritia-plugin` 후 `claude plugin install peritia@peritia-plugin` |
| Claude Desktop (Code·Cowork) | 설정 → 플러그인 → 추가 → 저장소에서 추가 → `risingdream/peritia-plugin` |
| claude.ai 웹 채팅 | 플러그인 대신 커넥터: 사용자 지정 → 커넥터 → 추가 → 이름 `페리티아`, URL `https://peritia.kr/mcp` |

설치 뒤 첫 연결 때 브라우저 로그인·허용 화면이 한 번 뜬다. Claude Code 는 `/mcp` → `plugin:peritia:peritia` → 인증.
키를 복사해 붙여넣는 단계는 없다.

**허용 목록에 등록된 계정만 연결된다.** 로그인은 되는데 허용 단계에서 막히면 운영자에게
로그인한 이메일로 등록을 요청한다.

## 업데이트

2.0.0 부터 이름에서 legal-kb 를 뺐습니다. 마켓플레이스는 `peritia-plugin`, MCP 서버는 `peritia` 입니다.
1.x 를 설치했다면 한 번 지우고 다시 설치한 뒤 `/mcp` → `plugin:peritia:peritia` 에서 인증합니다.

```bash
claude plugin uninstall peritia@legal-kb-plugin
claude plugin marketplace remove legal-kb-plugin
claude plugin marketplace add risingdream/peritia-plugin
claude plugin install peritia@peritia-plugin
```

그 뒤 새 판은 이렇게 받습니다.

```bash
claude plugin marketplace update peritia-plugin
claude plugin update peritia@peritia-plugin
```

## 들어 있는 것

- 도구 6종 — `research` · `search_legal` · `get_article` · `get_decision` · `neighbors` · `web_search`
- 스킬 `legal-research` — 도구 호출 순서, 웹 검색을 쓸 때와 안 쓸 때, 「법령명」 제N조 + 시행일 인용 형식
- 스킬 `contract-review` — 근로·주택 임대차·용역/하도급 계약서를 `contract_review` 도구로 조항별 판정(green·yellow·red)하고 근거 조문·수정 문구·빠진 조항을 정리한다. 전문가 검토용 초안이며 법률 자문이 아니다
- 커맨드 `/peritia:법령` · `/peritia:조문` · `/peritia:판례` (Claude Code·Desktop 전용)

자세한 내용은 [`plugins/peritia/README.md`](plugins/peritia/README.md).

> 이 저장소는 비공개 서비스 저장소에서 동기화된다. 여기에 직접 커밋하지 않는다.
