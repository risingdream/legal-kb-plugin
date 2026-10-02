# legal-kb 도구 — 인자와 응답

`SKILL.md` 가 정한 순서를 따르되, 인자를 정확히 넣어야 할 때 이 파일을 본다.
값은 서버 `toolDefs()` 실측이다(2026-09-23, `property_lookup`·`property_issue`·`search_commentary` 2026-10-02, 문서 도구 2026-10-03).

## `research` — 처음 부를 도구

| 인자 | 필수 | 기본 | 설명 |
|---|---|---|---|
| `query` | ✔ | — | 법률 질의. 문장 그대로 넣는다 |
| `as_of` | | 오늘 | 기준일 `YYYY-MM-DD` |
| `max_items` | | 12 | 근거 건수. 상한 12 |
| `domain` | | (없음) | 직역 기본값. 직역 이름(`세무사`·`회계사`·`노무사`·`변리사`·`법무사`·`변호사`) 또는 코드. 아래 "직역 기본값" 참고 |

응답: `statutes[]`(law_name · article_no · 시행일 · 현행/시행예정) · `cases[]`(case_no · 선고일 · `case_text` 요지).
구법에 `[경고]`, 시행예정에 `[대조]` 고지가 붙는다. **웹 검색은 포함하지 않는다.**

`case_no` 는 저장 정본 표기다(조세심판원은 `조심 2020인8553`, 법원은 `2015두42435`).
**손대지 말고 그대로 `get_decision(case_no)` 에 넣는다.**

## `search_legal` — 키워드를 바꿔 다시 훑을 때

| 인자 | 기본 | 설명 |
|---|---|---|
| `query` (필수) | — | 검색어 |
| `as_of` | 오늘 | 기준일 |
| `list_limit` | 5 | `statutes` · `cases` **각각**의 건수. 상한 10 |
| `rerank` | `off` | `llm` 이면 LLM이 두 목록을 재정렬. +1.5s. `list_limit=3` 과 함께 쓰면 기본값보다 토큰이 적고 재검색률은 비슷하다 |
| `trace_cited` | `true` | 상위 판례가 인용한 현행 조문을 후보에 합산. 보통 그대로 둔다 |
| `hint_articles` | `true` | 질문을 외부 LLM에 보내 근거 조문 후보를 지목받는다. **질의를 외부로 보내면 안 되는 상황이면 `false`** |
| `include_hits` | `false` | 호환용 섞인 목록. 두 목록과 2/3가 겹쳐 응답이 두 배가 된다. 켜지 마라 |
| `include_scheduled` | `false` | 시행예정을 `scheduled` 배열로 |
| `expand` | `false` | 연결선 확장 |
| `limit` | 10 | `include_hits=true` 일 때만 의미 있다 |
| `domain` | (없음) | 직역 기본값. `research` 와 같다 |

읽을 것은 `statutes` 와 `cases` 두 목록뿐이다. `rerank`/`hint` 가 `fallback` 이면 보조 LLM이 실패해
원 순서로 돌아간 것이고 결과 자체는 유효하다.
`cases[].case_no` 는 `research` 와 같은 정본 표기이고, 그대로 `get_decision` 에 넣는다.

## 직역 기본값 `domain` (RAG-8202)

같은 질문도 직역마다 답이 다르다 — "심사청구는 누구에게 제기하나?" 는 세무사면 「국세기본법」 제62조(국세청장),
노무사면 「산업재해보상보험법」 제103조(근로복지공단), 변리사면 「특허법」 제59조(출원심사청구)다.
질문에 법령명·분야 용어가 없어 서버가 분야를 못 가리는 경우에만 `domain` 이 개입한다.

- **사용자 직역을 알면 직역 이름을 넘긴다**: `research(query, domain="노무사")`. 로그인 프로필과 같은 동작이다.
  - 노무사·변리사·법무사: 그 분야 법령·사건에 가중만 준다(범위를 자르지 않는다).
  - 세무사·회계사: 목록은 그대로 두고, 단서 없는 질문에만 세법 범위 조문 상위 3건을 **`related_tax_laws`** 로 따로 싣는다.
    심사청구·이의신청·불복처럼 직역마다 답이 갈리는 질문에서 여기에 「국세기본법」 조문이 온다. 조문 목록에 없으면 이 칸을 본다.
  - 변호사: 전 분야라 기본값이 없다. `none` 과 같이 프로필도 보지 않고 질문 분류만 쓴다.
- 질문에 분명한 분야 단서가 있으면 `domain` 과 무관하게 **항상 질문을 따른다**(세무사가 가족법을 물어도 민법으로 간다).
- 코드 `tax` 는 다르다 — "이 질문은 세법 질문이다" 라고 호출자가 확신할 때만. 세법 범위에서 먼저 찾고 타법은 `related_other_laws` 로 분리한다.
  비세법 질문에 `tax` 를 주면 정답이 잘려 나간다(해고 예고·연차·취소소송 같은 질문 20/51 miss). **프로필 대용으로 쓰지 마라.**
- 안 넘기면 서버가 로그인 사용자의 프로필 직역(웹에서 고른 값)을 쓰고, 그것도 없으면 질문 분류만 쓴다.
- 응답 `domain` · `domain_source`(`arg` | `profile`) 로 무엇이 적용됐는지 알 수 있다. 끄려면 `domain="none"`.

## `get_article` — 조문 전문

| 인자 | 필수 | 설명 |
|---|---|---|
| `law_name` | ✔ | 정식 명칭. `민법` · `근로기준법` · `부가가치세법` · `소득세법 시행령` · `상속세 및 증여세법` |
| `article_no` | ✔ | `제89조` · `89` · `제55조의2` 모두 받는다 |
| `document_id` | | `research`의 문서 ID. 부칙(`document_kind=addendum`)은 반드시 넣는다. 법령·조문과 일치해야 하며 다른 문서로 대체 조회하지 않는다 |
| `as_of` | | 기준일 |
| `include_scheduled` | | `true` 면 시행예정을 `scheduled` 필드로만 반환 |

기준일에 시행 중인 개정판이 없고 시행예정만 있으면 `isError: not_in_force`.
문서 ID로 부칙 시행판을 읽을 때는 `enforced_from`을 `as_of`에 넣는다. 그 판이 없으면 `not_found`다. 이 경로는 `include_scheduled`로 다른 판을 보충하지 않는다.
약칭은 통하지 않는다 — `부가세법` ✗, `조특법` ✗.

### 옮겨진 조문 `moved_to` · `moved_from`

- 구 조문 번호로 물으면 `moved_to` 에 현행 대응 조문이 온다("제N조로 옮겨졌다").
- 현행 조문으로 물으면 `moved_from` 에 이 조문이 이어받은 구 조문이 온다.
- 근거는 판례 참조조문의 "(현행 제N조 참조)" 표기다. 조문 하나만 가리키면 `relation=successor`(대응 조문),
  여럿을 가리키면 `relation=related`(관련 조문)다. `related` 는 같은 조문이라고 단정하지 않는다.
- 판례가 구 조문으로 판단한 사건을 현행 조문으로 옮겨 인용할 때 쓴다.

## `get_decision` — 판례·결정례 전문

| 인자 | 필수 | 기본 | 설명 |
|---|---|---|---|
| `case_no` | ✔ | — | `조심 2020서1582` · `2019두31600` |
| `limit` | | 5 | 상한 20 |

**여러 건을 섹션별로 돌려준다. 1순위에 의존하지 말라고 서버가 명시한다** — 실제로 읽고 질문에 맞는 건을 고른다.

### 표기는 가리지 않는다 (RAG-8097)

`research` · `search_legal` 의 `case_no` 를 **그대로** 넣으면 된다. 서버가 세 단계로 맞춰 본다.

| 넣은 값 | 찾는다 |
|---|---|
| `조심 2020인8553` | ✔ 저장 정본 |
| `2020인8553` | ✔ 접두어 없어도 |
| `조심2020인8553` | ✔ 공백 없어도 |
| `대법원 2015두42435` | ✔ 없는 접두어를 붙여도 |
| `조심 2024광476` | ✔ 끝자리 앞 0(`0476`) 이 빠져도 |

조세심판원만 `조심` 접두어를 요구하던 비대칭은 없어졌다. **접두어를 직접 붙이거나 떼어 재시도하지 마라.**

### 못 찾으면 후보가 온다

```json
{"error":"not_found","message":"…","candidates":["조심 2024광4762","조심 2024광4765"]}
```

`candidates` 는 같은 연도·부호에서 번호가 가까운 **실재** 사건번호다. 그중 질문에 맞는 것이 있으면
다시 부르고, **없으면 그 사건을 인용하지 마라** — 기억으로 메우면 환각이 된다.

## `neighbors` — 연결선 (거의 안 쓴다)

`law_name`+`article_no`, `case_no`, 또는 `document_id` 중 하나로 부른다.
`expand` 기본 `false`라 그대로 부르면 **빈 배열**이 온다. `depth` 는 1 또는 2(기본 2).
현행 개정판만 따라간다. `search_legal` 이 `trace_cited=true` 로 판례→조문을 이미 따라가므로
본법↔시행령을 명시적으로 훑을 때가 아니면 부를 필요가 없다.

## `web_search` — 최신 사실만

| 인자 | 기본 | 설명 |
|---|---|---|
| `query` (필수) | — | 질문 그대로보다 **확인하려는 사실**(법령명·연도·항목)을 넣는다 |
| `max_results` | 4 | 본문을 읽을 문서 수. 상한 6 |
| `max_chars` | 3000 | `web_text` 총 길이. 상한 6000 |

응답은 합쳐진 텍스트 `web_text` 한 덩어리와 `sources[]`.
호출 기준은 `SKILL.md` 의 표를 따른다. 조문 질문에 섞으면 인용할 조문이 밀려난다.

## `search_commentary` — 실무 해설(2차 문헌) (RAG-9049)

국세청 책자·법령 해설 같은 2차 문헌에서 조각을 찾는다. `research`·`search_legal` 에는 해설이 붙지 않는다 — 필요할 때 이 도구로만 부른다.

| 인자 | 기본 | 설명 |
|---|---|---|
| `query` (필수) | — | 실무 질의. 서식 이름·절차·항목을 넣는다(예: `종합소득세 신고서 사업소득명세서 업종코드`) |
| `law_name` · `article_no` | (없음) | 관련 조문을 알면 넣는다. 그 조문(법령)을 인용한 해설이 앞에 온다 |
| `year` | 질의의 연도 → 최신판 | 귀속연도. 해마다 나오는 책자는 이 연도를 덮는 판을 고른다 |
| `limit` | 3 | 상한 5 |

응답: `commentary[]`(title · citation · excerpt · access_level · page · warning) · `commentary_note`.
`excerpt` 는 발췌다 — 밖의 본문을 지어내지 않는다. `warning`(개정 전 해설)이 있으면 그 내용을 현행 법으로 인용하지 않는다.

## `property_lookup` — 부동산 공부 조회 (RAG-9005)

토지대장·건축물대장·공시가격을 공공 API(브이월드·건축HUB)로 조회한다. 계산 레시피에 넣을 공시가격·면적을 사용자가 주지 않았을 때 쓴다.

| 인자 | 기본 | 설명 |
|---|---|---|
| `address` | — | 도로명·지번 주소. 공동주택은 동·호까지. `address`·`pnu` 중 하나 필수 |
| `pnu` | — | 19자리 필지 고유번호 |
| `years` | 작년·올해 | 공시가격 기준연도(YYYY, 최대 10개). 취득·양도·평가 연도를 넣는다 |
| `dong`·`ho` | — | 주소에 동·호가 없을 때. `ho` 가 있으면 전유·공용면적과 공동주택가격을 조회 |
| `sections` | 전부 | `land`(토지대장)·`building`(건축물대장)·`price`(공시가격) 중 일부 |

응답: `notice`(공적 증명 아님 안내) · `pnu` · `land_register.parcels[]`(지목 `land_category`·`area_m2`) · `building_ledger`(`titles[]`·`recap_titles[]` 의 `main_purpose`·`total_floor_area_m2`·`use_approval_date`, 호실 `unit_areas`) · `official_prices.land`(개별공시지가 `unit_price_per_m2`·총액 `amount`)·`official_prices.house`(개별주택가격 또는 공동주택가격). 갈래마다 `source`(기관·조회 시각)가 붙는다.

- 결과는 **공적 증명이 아니다**(발급본 아님). 답에 그 안내와 출처·조회 시각을 함께 쓴다.
- 레시피 입력: `prop.house` 의 `price`·`prior_price`(과세표준상한)는 과세연도·직전 연도 `official_prices.house` — 주택 재산세는 `years` 에 두 해를 함께 넣는다. `cg.converted_acquisition` 의 취득·양도 당시 기준시가, `val.real_estate` 의 `standard_value` 는 해당 연도 `official_prices` 의 `amount`(토지는 `land`, 단독·공동주택은 `house` — 토지를 더하지 않는다). 날짜가 그해 공시일(통상 4~5월) 전이면 직전 연도 값.
- 공시 전 연도는 `missing_years`. 비주거 건물분 기준시가(국세청 고시)·등기부(소유자·권리관계)는 조회하지 않는다.
- 하루 호출 상한이 있다(`limit_exceeded`). 같은 물건·연도는 하루 동안 캐시된다.
- 공동주택 필지의 `official_prices.land` 에 `scope: "complex"` 가 붙으면 단지 토지 전체 총액이다(호실 몫 아님) — 호실은 `house`(공동주택가격)를 쓴다. 동을 주면 `building_ledger.unit_title` 이 그 동 표제부다.

## `property_issue` — 대장 발급본(PDF) 발급 (RAG-9025)

정부24 에서 건축물대장·토지대장 **발급본**(공적 증명)을 회사 계정으로 발급한다. 사용자별·전체 하루 상한이 있고 수십 초~2분 걸린다.
값(면적·지목·용도·공시가격)만 필요하면 `property_lookup` 으로 충분하다 — 그때는 발급하지 않는다. 제출용 원본·대장 원문 확인이 꼭 필요할 때만 부른다.

| 인자 | 기본 | 설명 |
|---|---|---|
| `kind` | (필수) | `building` 건축물대장 · `land` 토지대장 |
| `address` | — | 도로명·지번 주소(동·호 포함 가능). 건축물대장은 주소가 필요하다 |
| `pnu` | — | 19자리 필지 고유번호. 주소 후보가 여럿일 때 고른 필지 |
| `dong`·`ho` | — | 공동주택 호실. `ho` 가 있으면 전유부 |
| `register_kind` | 자동 | `general`·`recap`(총괄표제부)·`title`(표제부)·`unit`(전유부). 비우면 공동주택+호 → 전유부, 공동주택 → 표제부, 그 밖 → 일반 |
| `price_year` | 최근 | 토지대장에 적을 공시지가 연도 |

응답: `status`(`issued`·`reused`) · `ledger` · `aply_no`(정부24 접수번호) · `pages` · `sha256` · `file_name` · `usage`(오늘 남은 수) · `download_url`·`expires_at`(15분 유효 서명 링크). PDF 본문은 싣지 않는다.

- 같은 사용자·같은 물건·같은 대장 종류는 같은 날 다시 발급하지 않고 그날 발급본을 준다(`reused`).
- 상한(`limit_exceeded`)·미설정(`unavailable`)이면 발급하지 않는다. 링크가 만료되면 도구를 다시 부른다(같은 날이면 재발급 없이 새 링크).

## 문서 도구 — 인용 검증·개인 문서 만들기·고치기 (RAG-9072)

연결이 문서 권한을 가질 때만 목록에 보인다. 기존 연결(조회만)에는 없다 — 쓰려면 연결을 해지하고 다시 연결해 동의 화면에서 문서 권한을 받거나, 웹 **내 토큰**에서 권한을 골라 새 토큰을 발급한다.

| 도구 | 권한(scope) | 하는 일 |
|---|---|---|
| `doc_verify_citations` | `docs:verify` 또는 `docs:write` | 내가 쓴 글의 인용 검증. **저장하지 않는다** |
| `doc_create` | `docs:write` | 내 개인 문서함에 새 문서(사건 문서는 못 만든다) |
| `doc_update` | `docs:write` | 내 개인 문서 한 자리를 바로 고침. 직전 본은 판으로 자동 저장 |
| `doc_verify` | `docs:write` | 저장된 내 문서의 인용을 다시 검증해 문서에 기록 |

- 서버는 글을 쓰지 않는다. 마크다운은 **내가(호출자 모델이) 쓴다**. 서버는 블록으로 바꾸고 인용을 검증(§6.3: `ok`·`not_found`·`text_mismatch`·`part_mismatch`·`amended`)해 저장만 한다.
- 인용은 문장 안에 `「법령명」 제N조 제N항`(부칙은 `「법령명」 부칙 제N조`), 판례는 사건번호 그대로 쓴다. 이 꼴이 아니면 인용으로 잡히지 않는다.
- 문서를 쓰기 전에 `doc_verify_citations` 로 초안 인용을 먼저 거른다. `not_found`·`text_mismatch` 는 `get_article`·`get_decision` 으로 확인해 고친 뒤 저장한다. `amended` 는 기준일 판은 맞지만 그 뒤 개정됐다는 경고다.
- 금액 칸은 `calculate` 결과만 `calc` 로 넘긴다(직접 계산한 숫자 금지).
- 본문을 읽어 오는 도구는 없다. 결과의 `link`(`https://chat.taxdesk.kr/doc/<id>`)로 웹 편집기에서 열어 확인·내려받기(DOCX·PDF·HWPX)한다. 경정청구서 같은 제출 서식은 웹에서 칸 출처를 확인한 뒤 제출하라고 사용자에게 알린다.

### `doc_verify_citations`

| 인자 | 설명 |
|---|---|
| `markdown` | 검증할 본문(최대 200KB). `citations` 와 둘 중 하나 |
| `citations` | `[{law_name, article_no, clause?, quote?, addenda?}]` 또는 `[{case_no}]`(최대 200개) |
| `as_of` | 기준일 YYYY-MM-DD. 비면 오늘. 과거면 그 뒤 개정(`amended`)도 본다 |

응답: `as_of` · `total` · `counts`(상태별 개수) · `issues`(ok 아닌 인용만 `label`·`status`·`note`) · `note`.

### `doc_create`

| 인자 | 설명 |
|---|---|
| `doc_type` | `free`(자유 문서, 본문 절 `body`) · `review_opinion`(검토의견서) · `correction_claim`(경정청구서) · `explanation_reply`(해명자료 제출서) |
| `markdown` | `free` 본문 |
| `sections` | 절 있는 서식의 `{절 key: 마크다운}`. 검토의견서 `question`·`facts`·`laws`*·`analysis`*·`conclusion`·`basis`*, 경정청구서 `reason`*, 해명자료 `item_1`*·`evidence` (* 인용 필수 절) |
| `title` · `as_of` | 제목(비면 서식 이름) · 인용 검증 기준일 |
| `values` · `calc` | 서식 칸 `{칸 key: 값}`(사용자가 준 값만) · `[{key, value, ref}]`(calculate 결과) |

응답: `doc_id` · `link` · `sections`(절 key·채움 여부) · `cites` · `issues`(블록 위치 포함) · `warnings`(인용 필수 절에 통과한 인용이 없음 등) · `fields`·`unfilled`(서식 칸).

### `doc_update` · `doc_verify`

| 인자 | 설명 |
|---|---|
| `doc_id` | 고칠 문서 id |
| `markdown` | 그 자리에 넣을 새 내용(자리 전체를 새로 쓴다) |
| `section_key` | 절 하나 전체. 절이 하나뿐인 문서(`free`)는 비워도 된다 |
| `block_ids` | `doc_verify`·`doc_create` 의 `issues[].block_id` — 같은 자리에 붙은 블록을 문서 순서대로 |
| `note` | 무엇을 왜 고쳤는지(판 이름 「MCP 수정 전 · …」에 남는다) |

`doc_update` 응답: `status: applied` · `version_id`·`version_name`(되돌릴 판) · `cites` · `issues` · `warnings`. 서식 칸이 든 블록은 고칠 수 없다(칸 값은 웹 편집기에서).
`doc_verify` 는 `doc_id` 만 받는다. 응답의 `issues` 를 `doc_update` 대상으로 쓴다.

상한: 사용자당 쓰기 시간당 20회(`rate_limited`), 개인 문서 수 상한(`limit_exceeded`), 입력 마크다운 200KB. 사건 문서·남의 문서 id 는 `not_found` 로 거절된다.

## `calculate` — 세액·과세표준·공제액 계산

금액은 이 도구로만 계산한다. 인자·함수·경고 코드·예시는 [`calculate.md`](calculate.md).

## 오류 코드

| 코드 | 뜻 |
|---|---|
| `not_found` | 그 법령명·조문번호·사건번호가 KB에 없다 |
| `not_in_force` | 기준일에 시행 중인 개정판이 없다 |
| `unavailable` | 검색기 연결이 없다(서버 측) |
| `unknown_tool` | 도구 이름 오타 |
| `insufficient_scope` | 이 연결·토큰에 문서 권한(`docs:verify`·`docs:write`)이 없다 — 다시 연결해 동의하거나 새 토큰 |
| `rate_limited` | 문서 쓰기 시간당 상한을 넘었다 |
| `limit_exceeded` | 개인 문서 수 상한(문서 도구)·하루 발급 상한(`property_issue`) |
