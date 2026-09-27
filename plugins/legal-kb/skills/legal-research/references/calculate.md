# `calculate` — 세액 계산 도구

`SKILL.md` 의 "세액 계산" 절을 따르되, 입력을 정확히 짜야 할 때 이 파일을 본다.
값은 서버 `toolDefs()` · `internal/legalcalc` 실측이다(2026-09-27). 설계: `docs/research/calculate-tool-design.md`.

도구는 **산수·끝수·근거 확인만** 한다. 어느 세율·공제를 쓸지, 1세대 1주택인지 같은 판정은 호출하는 모델 몫이다.

## 입력

| 인자 | 설명 |
|---|---|
| `as_of` | 기준일 `YYYY-MM-DD`. 기본 오늘. `table_from` 과 상수 검증이 모두 이 날짜의 조문 개정판을 쓴다. **양도일·과세기간 끝 날짜를 넣는다** |
| `terms` | 식 변수. 이름은 영문 소문자·숫자·밑줄(최대 50개). 항목마다 `label` 필수 |
| `steps` | `[{id, label, expr}]` 차례로 계산(최대 30개). 뒤 단계는 앞 `id` 를 쓴다. **마지막 단계가 결과** |
| `expression` | `steps` 대신 식 하나. 둘 다 주면 거부 |
| `recipe` · `inputs` | 저장소 레시피(아래 "레시피"). `inputs` 만 주면 레시피가 terms·steps 를 채운다. 더 준 `terms`·`steps` 는 레시피 단계 뒤에 붙는다 |

### `terms` 항목 세 가지

| 모양 | 쓰는 곳 |
|---|---|
| `{value, label, source:"user"}` | 사용자 사실 — 가액·필요경비·취득일·양도일·면적 |
| `{value, label, source:"law", cite:{law, article, paragraph?, item?, subitem?}}` | 법 숫자 — 기본공제·공제율·한도·기준금액. **cite 없으면 거부** |
| `{label, table_from:{law, article, paragraph?, item?, subitem?, table?, column?}}` | 조문 세율표를 서버가 꺼낸다. 손으로 옮기지 않는다 |

`value` 형식: 정수 `2500000` · 분수 `"10/100"` · `"6%"` · `"100분의 6"` · `"250만원"` · 날짜 `"2024-05-20"`.
실수 `0.1` 은 받지만 되도록 분수 문자열로 준다.

`table_from` 예:

| 표 | 선택자 |
|---|---|
| 종합소득 기본세율(누진) | `{"law":"소득세법","article":"제55조","paragraph":1}` |
| 장기보유특별공제 표 1 | `{"law":"소득세법","article":"제95조","paragraph":2,"table":1}` |
| 1세대 1주택 장특공 표 2 | `{"law":"소득세법","article":"제95조","paragraph":2,"table":2,"column":"보유기간"}` (거주는 `"거주기간"`) |
| 비사업용토지 세율 | `{"law":"소득세법","article":"제104조","paragraph":1,"item":8}` |

파싱이 안 되는 표는 `table_unparsed` 와 원문이 온다. 그때만 수동 표를 준다:
`{label, source:"law", cite, table:{kind:"progressive", rows:[{over, upto, base_tax, rate}]}}` 또는
`kind:"bracket"` 의 `rows:[{from, from_incl, to, to_incl, value}]`. 모든 셀 숫자가 상수 검증 대상이다.

### 레시피 — 자주 틀리는 계산은 저장소 정의로

골든셋에서 반복해 틀린 계산만 저장소 YAML(`go-server/internal/legalcalc/recipes/`)로 고정했다. 법 숫자·cite·끝수는
정의에 있고, `as_of` 로 시행 기간 판을 고른 뒤 같은 엔진(상수 검증 포함)으로 계산한다. 응답 `recipe` 에 id·판 기간·`notes`(적용 범위·주의)가 온다.

| id | 쓰는 곳 | inputs | 단계 |
|---|---|---|---|
| `acq.house_standard_rate` | 주택 유상취득 취득세 표준세율(지방세법 제11조①8). 2020-01-01 이후 6~9억 산식((가액×2/3억원−3)/100 → **비율** 소수 넷째자리 반올림, 7억 → 0.0167), 2013-12-26~2019-12-31 판은 1·2·3%. 세액 10원 미만 절사 | `price` 취득당시가액(지분 취득이면 전체 주택 가액) | `rate` · `acq_tax` |
| `cg.one_house` | 1세대 1주택 양도소득세(소득세법 제89조①3·제95조②·시행령 제160조). 비과세 요건 충족이면 양도차익 × (양도가액 − 기준금액)/양도가액만 과세(기준금액 **2021-12-08 양도분부터 12억원**(부칙 선시행), 2021-01-01~12-07 9억원). 장특공은 보유 3년 미만 0 · **거주 2년 이상 표2(보유+거주)** · 그 밖 표1. 기본공제 250만원·기본세율까지. 2021-01-01 이후 양도분 | `transfer_price` · `acquisition_price` · `expenses` · `acquired_on` · `transferred_on` · `residence_years`(보유기간 중 거주 만 연수) · `one_home_exempt`(비과세 요건 충족 1 / 미충족 0) | `gain` · `years` · `taxable_gain` · `ltd_rate` · `ltd` · `income` · `base` · `tax` |

- 레시피 밖: 다주택·법인 중과(제13조의2)·고급주택·생애최초 감면·상속·증여·원시취득 → terms·steps 로 직접.
- `cg.one_house` 밖: 1세대 1주택이 아닌 주택(다주택·중과)·보유 2년 미만(단기세율)·지분·부수토지 보유기간 상이·미등기 → terms·steps 로 직접.
  1세대 1주택 해당·비과세 요건·거주기간은 사실 판정이라 되묻고 inputs 로 준다. 다른 양도와 합산하면 `income` 뒤에 steps 를 붙인다(기본공제는 한 번만).
- 지방교육세처럼 뒤 계산이 필요하면 같은 호출에 붙인다:
  `{"recipe":"acq.house_standard_rate","inputs":{"price":800000000},"terms":{"half":{"value":"100분의 50",…,"cite":{"law":"지방세법","article":"제151조","paragraph":1}},"edu":{"value":"100분의 20",…}},"steps":[{"id":"edu_tax","label":"지방교육세","expr":"floor_10won(price * rate * half * edu)"}]}`
- 거부: `recipe_not_found`(후보 `candidates`) · `missing_input` · `recipe_out_of_range`(정의 기간 밖 `as_of`).

## 식(`expr`)

사칙연산 · 비교 · `&&` `||` · 삼항 `c ? a : b` 와 아래 함수만 쓴다.
**식 안 숫자는 0·1·100·1000 만** 허용한다. 그 밖의 숫자는 terms 로 빼고 cite 를 단다(`uncited_constant`).

| 함수 | 뜻 |
|---|---|
| `progressive(base, table)` | 누진세액 = 구간 기본세액(`base_tax`) + 초과분 × 세율. 끝수 처리 안 함 |
| `bracket_lookup(table, x)` | x 가 속한 구간의 값(공제율·세율·금액) |
| `floor_won(x)` · `floor_10won(x)` · `floor_to(x, unit)` | 원 미만 · 10원 미만 · unit 원 미만 절사 |
| `round_half_up(x, places)` | 소수 places 자리 사사오입 |
| `min(a, b, …)` · `max(a, b, …)` | 한도 적용 |
| `pct(n)` · `permille(n)` | n/100 · n/1000. 이미 비율인 값(`"20%"`·`"20/100"`)에 또 씌우지 않는다 |
| `full_years(from, to, "civil"\|"inclusive")` | 만 연수. 기본 civil(초일 불산입) |
| `prorate(amount, part, whole)` | 안분 |

합성 세율(예: 표준세율 + 중과기준세율 × 배수)은 합친 값을 term 으로 주지 말고 **구성 상수를 각각 cite 와 함께** 준 뒤 식에서 합친다.
결과가 정수가 아니면 `result_not_rounded` 경고가 난다 — 마지막 단계에 `floor_won` 을 잊지 않는다.

## 응답

| 필드 | 뜻 |
|---|---|
| `status` | `ok` · `ok_with_warnings` |
| `result` | 마지막 단계 `{id, label, value}` |
| `steps[]` | 단계별 `value` · `uses` · `detail`(걸린 구간 `row`·`over`·`upto`·세율 `rate_text`·누진공제액 `progressive_deduction`, 보유기간 기산일 `start`·만료일 `expiry`) |
| `terms[]` | 입력 항목. `cite` 에 서버가 찾은 개정판 `enforced_from`, `verified`(`paragraph`·`article`·`false`·`null`) |
| `tables[]` | 꺼낸 세율표. `from.enforced_from` 이 단계표 근거의 시행일이다 |
| `warnings[]` | `{code, term?, step?, message, found_in?}` |

### 경고 — 답에 옮긴다

| code | 뜻 | 대처 |
|---|---|---|
| `unverified_constant` | 그 숫자가 인용 조문 `as_of` 본문에 없다. `found_in` 이 있으면 **다른 개정판(구법) 숫자** | `get_article` 로 확인해 고쳐 다시 부른다. 합성 세율이면 구성 상수로 나눠 다시 부른다. 못 고치면 답에 적는다 |
| `pct_of_fraction` | `pct()`·`permille()` 인자가 1 미만 — 이미 비율인 값에 또 씌웠을 수 있다(20% → 0.2%) | 인자가 비율이면 `pct()` 를 빼고 다시 부른다 |
| `constant_other_paragraph` | 같은 조 다른 항에서 찾음(정보성) | cite 의 항을 고친다 |
| `cite_not_in_force` | `as_of` 에 그 조문이 없다 | `as_of` 나 조문을 확인 |
| `table_inconsistent` | 누진표 검산 실패(파싱 의심) | 수동 표로 다시 |
| `table_row_condition` | 표 행에 단서(예: 보유 3년 이상 한정)가 붙어 있다 | 조건을 사실과 대조해 답에 적는다 |
| `bracket_miss` | 어느 구간에도 안 걸려 0 | 입력·표 선택 확인 |
| `result_not_rounded` | 결과에 원 미만이 남았다 | 끝수 함수를 넣어 다시 부른다 |

### 거부 — 계산하지 않고 `isError`

`invalid_expression`(`at` 에 위치) · `float_literal` · `unknown_name` · `type_error` · `missing_cite` · `missing_label` ·
`missing_source` · `uncited_constant` · `table_not_found`(후보 `candidates`) · `table_unparsed`(표 원문) ·
`div_by_zero` · `date_order` · `limits` · 레시피 `recipe_not_found` · `missing_input` · `recipe_out_of_range`. 코드대로 고쳐 다시 부른다.

## 예시 1 — 비사업용토지 양도소득 산출세액

```json
{
  "as_of": "2024-05-20",
  "terms": {
    "transfer_price":   {"value": 500000000, "label": "양도가액", "source": "user"},
    "acquisition_cost": {"value": 200000000, "label": "취득가액·필요경비", "source": "user"},
    "acquired_on":      {"value": "2011-03-02", "label": "취득일", "source": "user"},
    "transferred_on":   {"value": "2024-05-20", "label": "양도일", "source": "user"},
    "basic_deduction":  {"value": 2500000, "label": "양도소득 기본공제", "source": "law",
                         "cite": {"law": "소득세법", "article": "제103조", "paragraph": 1}},
    "ltd_table":  {"label": "장기보유특별공제율 표 1",
                   "table_from": {"law": "소득세법", "article": "제95조", "paragraph": 2, "table": 1}},
    "land_rates": {"label": "비사업용토지 세율",
                   "table_from": {"law": "소득세법", "article": "제104조", "paragraph": 1, "item": 8}}
  },
  "steps": [
    {"id": "gain",  "label": "양도차익",         "expr": "transfer_price - acquisition_cost"},
    {"id": "years", "label": "보유기간(년)",     "expr": "full_years(acquired_on, transferred_on)"},
    {"id": "ltd",   "label": "장기보유특별공제", "expr": "floor_won(gain * bracket_lookup(ltd_table, years))"},
    {"id": "base",  "label": "과세표준",         "expr": "gain - ltd - basic_deduction"},
    {"id": "tax",   "label": "산출세액",         "expr": "floor_won(progressive(base, land_rates))"}
  ]
}
```

결과: 양도차익 300,000,000 · 보유 13년 → 표 1 `100분의 26` · 장특공 78,000,000 · 과세표준 219,500,000 ·
제104조①제8호 `1억5천만원 초과 3억원 이하` 구간 → **산출세액 85,420,000원**.

## 예시 2 — 종합소득 산출세액(식 하나)

```json
{
  "as_of": "2024-12-31",
  "terms": {
    "base":  {"value": 100000000, "label": "종합소득과세표준", "source": "user"},
    "rates": {"label": "기본세율", "table_from": {"law": "소득세법", "article": "제55조", "paragraph": 1}}
  },
  "expression": "floor_won(progressive(base, rates))",
  "label": "산출세액"
}
```

`as_of` 를 `2020-06-01` 로 바꾸면 7구간 표(2020년 본)로 다시 계산한다 — 시점 비교는 이렇게 두 번 부른다.
