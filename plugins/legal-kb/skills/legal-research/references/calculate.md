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
아래 표의 id(`pit.comprehensive` · `pit.loss_order` · `pit.financial_global` · `inh.spouse_deduction` 포함)는 **terms·steps 로 식을 다시 짜지 않는다.** 조문을 읽은 뒤 직접 짜면 공제 순서·배당가산 순차·배우자공제 5억 하한이 빠진다.

| id | 쓰는 곳 | inputs | 단계 |
|---|---|---|---|
| `acq.house_standard_rate` | 주택 유상취득 취득세 표준세율(지방세법 제11조①8). 2020-01-01 이후 6~9억 산식((가액×2/3억원−3)/100 → **비율** 소수 넷째자리 반올림, 7억 → 0.0167), 2013-12-26~2019-12-31 판은 1·2·3%. 세액 10원 미만 절사 | `price` 취득당시가액(지분 취득이면 전체 주택 가액) | `rate` · `acq_tax` · `edu_tax`(지방교육세, 제151조①1: 취득세율×50%×20%) · `total`(감면 전 취득세+지방교육세, 농특세 제외) |
| `acq.house_heavy` | 주택 유상취득 다주택·법인 중과(지방세법 제13조의2①). 법인 12%, 조정대상지역 2주택·비조정 3주택 8%, 조정 3주택 이상·비조정 4주택 이상 12%(= 1천분의 40 + 중과기준세율 1천분의 20 × 100분의 200·400). 중과가 아니면 제11조①8 표준세율. 지방교육세는 중과면 (4%−2%)×20% = 0.4%, 농특세는 85㎡ 초과만 0.6%·1.0%(표준세율이면 0.2%). 위택스 15건 대조 | `price` · `houses`(취득 후 1세대 주택 수, 이번 주택 포함) · `regulated`(조정대상지역 1/0) · `corporate`(법인 1/0, 생략 0) · `exempt`(시가표준액 1억원 이하·일시적 2주택 등 중과 제외 1/0, 생략 0) · `over_85`(85㎡ 초과 1/0, 생략 0) | `heavy_mult`(0 = 중과 아님) · `standard_rate` · `rate` · `acq_tax` · `edu_tax` · `farm_rate` · `farm_tax` · `total`(취득세+지방교육세+농특세) |
| `acq.by_cause` | 상속(①1: 농지 2.3%, 그 밖 2.8%)·상속 외 무상취득/증여(①2: 3.5%, 비영리 2.8%)·원시취득/신축(①3: 2.8%)·그 밖 유상취득(①7: 농지 3%, 그 밖 4% — 토지·상가 등 주택 외). 조정대상지역 시가표준액 3억원 이상 주택 증여 중과(제13조의2② 12%, 시행령 제28조의6② 제외), 상속 1가구 1주택·자경농지 특례(제15조①2: 표준세율 − 2%, 농특세 비과세). 지방교육세 (표준세율−2%)×20%(중과 0.4%), 농특세 0.2%(중과 1.0%, 85㎡ 이하 주택·상속 특례 0) | `price` 과세표준 · `cause`(1·2·3·7) · `farmland` · `nonprofit` · `special` · `house` · `over_85` · `regulated` · `std_value`(생략 price) · `heavy_exempt` — 0/1 은 생략 0 | `std_rate` · `gift_heavy` · `inh_special` · `rate` · `acq_tax` · `edu_tax` · `farm_rate` · `farm_tax` · `total` |
| `acq.heavy_13` | 지방세법 제13조 중과: ① 과밀억제권역 본점·주사무소 신축·증축·공장 신설·증설 std+4%, ② 대도시 법인 설립·지점·전입(5년 내)·대도시 공장 std×3−4%(매매 8%·신축 4.4%·농지 5%, 주택은 제13조의2①1 12%), ⑤ 골프장·고급주택·고급오락장 std+8%(매매 12%·고급주택 10억 11%), ⑥ ①+② std×3, ⑦ ②+⑤ std×3+4%, 제13조의2③ 고급주택+다주택·증여 중과 16%·20%. 지방교육세 본문((std−2%)×20%, 주택 std×50%×20%), ②⑥⑦ ×3(제151조①1가), 제13조의2 해당 0.4%(나목). 농특세 std 자리 2% 대입×10%(85㎡ 이하 주택 0). 위택스는 ③ 겹침만 대조(grade A 7), 나머지 조문 손계산 | `price` · `cause`(1·2·3·7·8) · `farmland` · `kind`(1·2·5·6·7) · `house_heavy`(0·8·12) · `small_house` — 0/1 은 생략 0 | `std_rate` · `hh_mult` · `lux_overlap` · `metro_house` · `rate` · `acq_tax` · `edu_base_rate` · `edu_tax` · `farm_rate` · `farm_tax` · `total` |
| `acq.formal` | 지방세법 제15조①1·3·4 — 환매·적격합병·공유물/합유물 분할 세율 특례(표준세율 − 2%, 주택 ×50%, 제13조② ×300%), 지방교육세(특례 전 표준세율 기준, 제13조② ×300%), 농특세 비과세(농특세법 제4조 제10호의4·시행령 제4조③). 재산분할(①6)·건축물 이전(①5)은 기본 세율 미확인이라 밖. 위택스·ETAX 에 제15조 칸이 없어 grade C | `price` · `kind`(1·3·4) · `house` · `house_value` · `farmland` · `metro` | 특례 세율 → `acq_tax` · `edu_tax` · `farm_tax` · `total` |
| `acq.fixed_2pct` | 지방세법 제15조② — 개수·지목변경/종류변경·과점주주 간주취득 등 중과기준세율 2%(제13조① 6%·⑤ 10%), 지방교육세 없음(제151조①1 괄호), 농특세 = 취득세×10%(농특세법 제5조⑤, 제124조 자동차 비과세). 과점주주 과세표준 = 부동산등 총가액 × (취득 후 지분율 − 비교 지분율)(50% 초과만), 지목변경 = 사실상취득가격 또는 변경 후−전 시가표준액. grade C | `kind`(1~8) · `price` 또는 `value_after`·`value_before` 또는 `corp_assets`·`ratio_after`·`ratio_prev` · `heavy`(0·1·5) · `car` | 과세표준 → `rate` → `acq_tax` · `farm_tax` · `total` |
| `acq.movable` | 부동산 외 취득(제12조①): 선박(상속 2.5%·무상 3%·원시/수입 2.02%·매매 3%·소형 2.02%·그 밖 2%), 차량(비영업 승용 7%·경자동차 4%·125cc 이하 이륜 2%·그 밖 비영업 5%·영업용 4%·자동차 외 2%), 기계장비(등록 3%·비등록 2%), 항공기(2%·2.02%·5,700kg 이상 2.01%), 입목·광업/어업/양식업권·회원권 2%. 차량·기계 과표 = max(신고가, 시가표준액)(제10조의5①2), 면세점 50만원(제17조). 지방교육세 (세율−2%)×20% 선박·기계·항공기만(제150조1), 농특세 0.2%(제124조 자동차 비과세). 공식 계산기 미대조(grade C) | `price` · `item`(1~7) · `sub`(목·세목, item 5~7 은 0) · `light` · `auto124` · `std_value`(생략 price) — 0/1 은 생략 0 | `rate` · `auto` · `base` · `exempt` · `acq_tax` · `edu_tax` · `farm_tax` · `total` |
| `acq.relief` | 취득세 감면 엔진(지방세특례제한법 제6①·17·22조의2·29·33②·36조의3·36조의5·66④⑤·67조, 최소납부 제177조의2, 지방교육세 지방세법 제151조①1다, 농특세법 제4·5조). 코드 1 생애최초 1호(300만)·2 생애최초 2호(200만)·3 출산·양육(500만)·4 서민주택(면제)·5 자경농민(50%)·6 경형 승용(75만)·7 경형 승합·화물(면제)·8~11 다자녀·12 장애인·13 국가유공자·14 보훈 상이(50%)·15 전기차(140만, 2026-12-31 일몰)·16 수소차(140만)·17 수소 화물(50%). 면제는 열거 밖이면 산출 200만원 초과 시 85%. 감면분 농특세 20%(85㎡ 이하 비과세). 위택스 감면코드 대조 grade A 5·B 3, 정액형 지방교육세 미확인(`edu_alt` 병기) | `code` · `tax`(감면 전 acq_tax) · `edu` · `farm` · `over_85` · `share` · `prior_tax` | 감면율·한도 → 최소납부 → 감면세액 → 감면 후 `acq_tax` · `edu_tax`(`edu_alt`) · `farm_tax`(`farm_alt`) · `total` |
| `prop.house` | 주택분 재산세(지방세법 제110조·제111조①3나·제111조의2·제112조①2·제113조③·제151조①6). 공정시장가액비율 60%, 1세대 1주택은 43·44·45%(시행령 제109조①2 단서, 2024~2026년도). 특례세율은 시가표준액 9억원 이하 1세대 1주택만. **공동소유는 주택 전체 과세표준에 세율 → 지분 안분**. 도시지역분 1천분의 1.4, 지방교육세 재산세액의 20%. 소유자별 세액 10원 미만 절사. 과세표준상한(제110조③·시행령 제109조의2): 작년 시가표준액×올해 비율 + 올해 과세표준×5% 와 비교해 작은 값 — `prior_price` 생략 시 미반영. 조례 가감·감면은 밖. 토지·건축물분은 `prop.land`·`prop.building` | `price` 주택 전체 시가표준액 · `one_house`(1/0) · `share`(지분, 생략 1) · `urban`(도시지역분 1/0, 생략 1) · `prior_price`(직전 연도 시가표준액, 생략하면 올해 값 = 상한 미반영) | `ratio` · `base_full` · `base_cap` · `base`(상한 적용) · `house_tax`(주택 전체) · `prop_tax` · `urban_tax` · `edu_tax` · `total`(지분 기준) |
| `prop.fire_resource` | 소방분 지역자원시설세(지방세법 제146조③④·제147조②, 시행령 제109조①·제138조). 과세표준 = 건축물 시가표준액×70%(주택 건축물 부분은 건물분 가액×60%), 6구간 누진(1만분의 4~12), 화재위험 100분의 200·대형 100분의 300, 10원 미만 절사, 세부담상한 150%. 서울시 ETAX 건축물 13건 grade A, 주택분·배수는 grade C | `price` · `kind`(1 건축물/2 주택 건축물 부분) · `fire`(0/1/2) · `base_given`(고지서 과세표준) · `prev_tax` | `ratio` · `base` · `std_tax` · `calc_tax` · `total` |
| `prop.relief` | 재산세 감면 엔진(지방세특례제한법 제31조의3④·제35조②·제36조의4②·제58조의3·제75조의5①2·제177조의2·제179조, 지방세법 제113조①·제151조①6, 종합부동산세법 제6조③). 코드 1 민간임대(40㎡ 이하 면제·60㎡ 이하 75%·85㎡ 이하 50%)·2 주택연금(25%, 5억 초과는 5억 기준 세액×25%)·3 전세사기피해(60㎡ 이하 50%·초과 25%)·4 창업중소기업(3년 면제+2년 50%)·5 인구감소지역 창업(5년 면제+3년 50%). 토지는 과세표준 경감, 면제는 산출 50만원 초과 시 85%, 감면 → 세부담상한 순(시행령 제118조3호 도출). 공식 계산기에 감면 칸이 없어 grade C | `object`(1 주택·2 토지·3 건축물) · `code` · `price` · 주택·건축물 `tax`·`urban_tax` / 토지 `base`·`kind`·`urban_tax`·`relief_base` · `area` · `year` · `share` · `prev_tax`·`prev_urban` | 감면율 → 최소납부 → 감면 후 재산세·도시지역분·지방교육세 · `total` · `cpt_price_cut` |
| `acq.relief_biz` | 법인·사업용 취득세 감면(지방세특례제한법 제79·80·80조의2·57조의2①③④·58조의3·58조의2·78조①⑤, 시행규칙 제7조②·제8조③④, 최소납부 제177조의2, 농특세법 시행령 제4조⑦5). 코드 1 본점 지방 이전·2 공장 지방 이전(면제, 산출 200만원 초과 85%, 종전 가액 초과액 과세)·3~5 기회발전특구(50% + 조례, 2026-12-31 일몰)·6 적격합병(제15조① 산출액 60%)·7 분할·8 현물출자·9 통합·10 법인전환(50%, 임대·공급업 제외)·11 창업중소기업 75%·12·13 지식산업센터 35%(수도권 15%)·14 산업단지 입주자 35/50/75·15 대수선 25·16 사업시행자 15/35/50. 감면분 농특세 20%(제79·80·58조의2·57조의2①·③2 비과세). 위택스 감면코드 대조 grade A 15·B 1 | `code` · `tax` · `edu` · `farm` · `region` · `ordinance` · `rental` · `new_value`·`prev_value`·`base` · `prior_tax` | 감면율(+조례) → 초과액 과세분 → 최소납부 → 감면세액 → 감면 후 `acq_tax` · `edu_tax` · `farm_tax` · `farm_rel` · `total` |
| `prop.relief_biz` | 법인·사업용 재산세 감면(지방세특례제한법 제79①·80①·80조의2①②③·58조의2①2·②2·78조⑤2나, 제177조의2·제179조, 지방세법 제113조①·제122조·제151조①6, 종부세법 제6조③). 코드 1 본점 이전·2 공장 이전(5년 면제+3년 50%)·3 특구 창업(5년 면제, 수도권 3년+2년 50%, 조례 5년 연장 ≤50%)·4 특구 이전(5년 면제+조례 연장)·5 특구 공장 신증설(5년 75%, 수도권 35%)·6 지식산업센터 설립자(5년 35%, 수도권 15%)·7 최초 분양(5년 35%)·8 산업단지 입주자(5년 35/60/75). 토지 과세표준 경감, 면제는 산출 50만원 초과 85%, 도시지역분 감면 없음. grade C | `object`(2 토지·3 건축물) · `code` · `year` · `region` · `ordinance` · `price` · 건축물 `tax`·`urban_tax` / 토지 `base`·`kind`·`urban_tax`·`relief_base` · `prev_tax`·`prev_urban` | 감면율 → 최소납부 → 감면 후 재산세·도시지역분·지방교육세 · `total` · `cpt_price_cut` |
| `cpt.house` | 주택분 종합부동산세(종합부동산세법 제8조①·제9조①②③⑤~⑨·제10조·제10조의2, 시행령 제4조의3①·제5조의2, 농어촌특별세법 제5조①). 공제 12억/9억/법인 0원 × 60%, 세율 2주택 이하·3주택 이상·법인 2.7%/5%. 재산세 공제 = 부과세액 × trunc10(과세표준 × 재산세 비율 × 1천분의 4 ÷ 합산 재산세 과세표준의 누진 재산세), 비율은 재산세상 1세대 1주택 43~45%·아니면 60%. 세액공제는 연령·보유 각각 원 미만 절사 후 80% 한도(재산세 공제 후). 세부담상한 prev_total×150%. 농특세 20%. 홈택스 간이세액계산 17건 일치(grade A, 법인 단일세율·재산세 감면 포함). 제10조의2 특례·과세표준상한 걸린 분모는 미대조 | `price_sum` · `owner` · `corp` · `houses` · `prop_levied`(다주택) · `prop_base_sum`(45%·60% 혼재) · `prop_one_house` · `share` · `joint` · `prior_price` · `relief_cut` · `age` · `years_held` · `excluded_price` · `prev_total` | 과세표준 → 세율 → 재산세 공제 → 세액공제 → 세부담상한 → 종부세 · 농특세 · `total`(joint 면 `sep_total` 병기) |
| `cpt.land` | 토지분 종합부동산세(종합부동산세법 제12조①·제13조·제14조①③④⑥·제15조, 시행령 제2조의4②·제5조의3·제6조·제7조, 농어촌특별세법 제5조①). 종합합산 공제 5억·세율 1%/2%/3%, 별도합산 공제 80억·0.5%/0.6%/0.7%, 공정시장가액비율 100%. 재산세 공제 = 부과세액 × (과세표준 × 70% × 종합 0.5%/별도 0.4%) ÷ 국내 그 구분 토지를 합산해 표준세율로 계산한 재산세 상당액(여러 시·군·구면 공제가 분자보다 작다), 공제액 10원 미만 절사. 세부담상한 prev_total×150%(법인 예외 없음). 농특세 20%. 홈택스 간이세액계산 13건 대조(grade A, 농특세·납부세액 절사는 조문) | `kind`(1 종합합산·2 별도합산) · `price_sum` · `prop_levied`(시·군·구별 본세 합계) · `relief_cut` · `prev_total` | 과세표준 → 세율 → 재산세 공제 → 세부담상한 → 종부세 · 농특세 · `total` |
| `cg.one_house` | 1세대 1주택 양도소득세(소득세법 제89조①3·제95조②·시행령 제160조). 비과세 요건 충족이면 양도차익 × (양도가액 − 기준금액)/양도가액만 과세(기준금액 **2021-12-08 양도분부터 12억원**(부칙 선시행), 2021-01-01~12-07 9억원). 장특공은 보유 3년 미만 0 · **거주 2년 이상 표2(보유+거주)** · 그 밖 표1. 기본공제 250만원·기본세율까지. 2021-01-01 이후 양도분 | `transfer_price` · `acquisition_price` · `expenses` · `acquired_on` · `transferred_on` · `residence_years`(보유기간 중 거주 만 연수) · `one_home_exempt`(비과세 요건 충족 1 / 미충족 0) · `residence_waived`(상생임대 거주요건 면제, 2022-08-02 양도분~) | `gain` · `years` · `taxable_gain` · `ltd_rate` · `ltd` · `income` · `base` · `tax` |
| `cg.relief` | 양도세 세액감면 엔진(소득세법 제90조①, 조특법 제69조~제70조·제77조~제77조의3·제97조의5·제133조, 농특세법 제5조①·시행령 제4조). 코드 1 자경농지·2 축사용지·3 어업용·4 자경산지(경영 연수 10~50%)·5 농지대토·6 공익사업(현금·채권·3·5년 특약, 2025.3.14. 판 분리)·7 대토보상 40%·8 개발제한구역 40·25%·9 장기일반민간임대 100%. 편입일·임대기간 기준시가 또는 보상액 비율로 감면소득 안분, 제133조 그룹 한도, 조문별 농특세. 공식 계산기 미대조(grade C) | `code` · `income` · `tax`·`base` · `deduction` · `std_acq`·`std_mid`·`std_transfer` 또는 `part_comp`·`total_comp` · `form` · `prior_same_year`·`prior_4y` | 감면율 → 안분 → 감면세액 → 한도 → 농특세 → `total` |
| `cg.relief_deduct` | 소득공제형 감면(소득세법 제90조②, 조특법 제99조의2·제98조의3, 시행령 제40조①). 취득 후 5년 이내 양도는 세액감면, 그 뒤는 5년간 소득(기준시가 안분)을 공제하고 기본세율로 재계산, 농특세 = 공제 전후 세액 차 × 20%. grade C | `code` · `overcrowded` · `acquired_on` · `transferred_on` · `income` · `std_acq`·`std_mid`·`std_transfer` · `tax`·`base`(5년 이내) | 5년 판정 → 공제 소득 → 재계산 세액 → 농특세 → `total` |
| `cg.rental_ltd` | 임대주택 장기보유특별공제 특례(조특법 제97조의3 70%·제97조의4 추가공제율 2~10%p, 소득세법 제95조②). 70% 는 임대기간 양도차익에만, 나머지는 표1(도출). grade C | `mode` · `gain` · `acquired_on` · `transferred_on` · `rent_years`(mode 2) · `std_*`(mode 1, 2026.2.27.~) 또는 `rent_gain` | 공제율 → 장기보유특별공제 → 양도소득금액 → 산출세액 |
| `cg.incorporation_carryover` | 법인전환·중소기업 통합 양도소득세 이월과세(조특법 제32조·제31조·제2조①6, 시행령 제28조·제29조, 지특법 제57조의2③5·④). 이월과세액 = 다른 양도자산이 없다고 보아 계산한 산출세액(기본공제·기본세율, 다른 세율 자산은 `cg.rate_select` 결과를 `tax`로). 요건: 자본금 ≥ 순자산가액·사업양수도 3개월·소비성서비스업 제외, 주택 제외. 사후관리 5년: 자산 2분의 1 처분·미사용 또는 주식 50% 처분 → 거주자 납부(법인 기납부분 제외). 취득세 2025~2027 50%·2022~2024 75%(임대·공급업 제외). grade C | `kind` · `method` · `transferred_on` · `est_on` · `capital` · `net_assets` · `consumptive` · `income` 또는 `tax` · `disposed_ratio` · `shares_disposed_ratio` · `exception` · `event_on` · `corp_paid` · `rental` · `acq_tax`·`edu`·`farm` | `eligible` → `tax_now` → `recapture`·`recapture_tax` → `acq_relief`·`acq_total` → `carryover_tax` |
| `cit.relocation_gain` | 공장·본사 이전 양도차익 익금불산입(조특법 제60조②·제61조③, 시행령 제56조③·제57조④). (양도가액 − 장부가액 − 이월결손금) × min(1, 재투자액 ÷ 양도가액). 양도 기한 2028-12-31(2025년까지 판 2025-12-31). 양도 사업연도 + 5부터 5개 사업연도 균분 이상 익금산입. 최저한세 대상. 법인세 세액은 계산하지 않음. grade C | `kind`(60·61) · `transferred_on` · `sale_price` · `book_value` · `loss` · `new_site_cost` · `direct_use_ratio` · `other_assets` · `moving_cost` · `fy_year` · `claimed` | `ratio` → `deferred` · `taxable_now` · `start_year`~`end_year` · `annual_min`·`last_year` · `excess` · `deferred_total` |
| `cit.tax` | 법인세 산출세액(법인세법 제55조①②, 시행령 제92조). as_of 는 **사업연도 개시일**(세율 개정은 시행 후 개시 사업연도부터). 2018~2022 개시 10·20·22·25%, 2023~2025 개시 9·19·21·24%, 2026 개시부터 10·20·22·25%. 2025 개시부터 성실신고확인 대상 소규모 법인(제60조의2①1)은 2억 이하 구간 없는 ①2 표(2025 19%·2026 20%~). 1년 미만 사업연도는 base × 12 ÷ 월수에 세율 → × 월수 ÷ 12, 원 미만 절사. 토지등 양도소득·세액공제·최저한세·지방소득세 밖. 국세청 신고안내 작성사례 grade B 1 + 손계산 C | `base`(과세표준) · `months`(1~12, 1개월 미만 1개월) · `small_rental`(0·1) | `annual_base` · `annual_tax` → `tax` |
| `cit.entertainment` | 기업업무추진비(2023 이전 접대비) 손금 한도(법인세법 제25조②④⑤, 조특법 제136조②③④⑤⑥) — 서식 제23호(갑) 순서. 기본 1,200만원(중소 3,600만원, 2019 개시 2,400만원) × 월수/12 + 수입금액 한도(0.3·0.2·0.03% 누진, 2019 개시 0.2·0.1·0.03%, 2020 개시 0.35·0.25·0.06% 특례 일수 안분) + 특수관계 수입은 (총수입 기준 − 일반 기준) × 10%. 부동산임대 주업 50%·정부출자 70%. 문화 min(지출, 일반한도 × 20%), 전통시장 min(지출, × 10%, 2026 개시 20%). as_of 사업연도 개시일. 국세청 신고안내 조정명세서 작성사례 grade B 2 + 손계산 C | `spent` · `no_card`(3만원 초과 증빙 없음) · `sme` · `months` · `rev_general` · `rev_related` · `rental_main` · `gov` · `culture` · `market` · `fy_days`·`days_2020`(2020 만) | `base_limit` · `rev_total`·`rev_limit`·`rel_limit` → `general_limit` · `culture_limit` · `market_limit` → `limit` · `excess`(한도초과) · `allowed` · `disallowed` |
| `cit.incorporation_compare` | 법인전환 세 부담 비교(소득세법 제14조③6·제47조①·제55조①·제59조·제129조①2나, 법인세법 제55조①). 같은 사업 이익을 **개인 종합소득세**(사업소득금액 − 종합소득공제 → 기본세율)와 **법인 쪽 합계**(이익 − 대표 급여 − 법인 부담 사회보험료 → 법인세 + 대표 근로소득세(근로소득공제·근로소득세액공제) + 배당 14% 원천징수)로 나란히 내고 `diff`(양수면 법인 쪽 세금이 적음)·`retained`(법인에 남아 아직 개인 미과세인 이익)를 준다. as_of 는 **과세연도 말일**(법인 사업연도는 같은 해 역년으로 봄). 급여·배당 배분을 바꿔 여러 번 불러 손익분기를 찾는다. 매출만 주어지면 계산하지 말고 사업소득금액·소득공제를 묻는다. 대표 급여 쪽 홈택스 연말정산 자동계산 grade A 5 + 국세청 법인세 작성사례 B 1 + 손계산 C | `profit`(개인 사업소득금액) · `deduction`(개인 종합소득공제 합계) · `salary` · `employer_si`(`pay.social_insurance` 사업주 부담) · `ceo_deduction`(생략 `deduction`) · `dividend`(2천만원 이하) · `small_rental`(0·1) | `indiv_base` → `indiv_tax` / `corp_base` → `corp_tax`, `wage_ded` → `ceo_base` → `ceo_calc_tax` − `ceo_wage_credit` = `ceo_tax`, `div_tax` → `corp_route_tax` · `diff` · `retained` |
| `pit.comprehensive` | 종합소득세 기본 계산(소득세법 제45조·제47조·제47조의2·제50조·제51조·제51조의3·제55조). 사업소득금액 = `book_profit` + `adj_add`(필요경비불산입·총수입금액산입 합계) − `adj_sub`(필요경비산입 합계) + 단순경비율 추계(`est_revenue`·`est_rate_pct`) + 주택임대 `rental_income`. **총급여액 `wage_total`·총연금액 `pension_total` 을 그대로** 넣으면 근로소득공제(한도 2천만원)·연금소득공제(한도 900만원)는 레시피가 계산 — 근로소득금액만 주어지면 `wage_amount`. 기타소득금액 `other_income`(300만원 초과는 종합과세). 이월결손금 `carried_loss` 는 공제 기간 안 금액만(2020년 이후 발생 15년, 2009~2019년 10년). 기본공제 150만원 × (본인 + `spouse` + `dependents`), 추가공제 `elderly`(70세 이상) 100만·`disabled` 200만·`woman` 50만(종합소득금액 3천만원 이하)·`single_parent` 100만(부녀자와 겹치면 한부모), `pension_premium`, `other_deduction`. 과세표준만 주어지면 `taxable_base`. 판은 2020-01-01부터, 기본세율표는 as_of 판. 끝수 `floor_won`. |
| `pit.loss_order` | 종합소득 결손금·이월결손금 공제 순서(소득세법 제45조). 당해 사업결손은 근로→연금→기타→이자→배당, 이월은 사업→근로→연금→기타→이자→배당. **종합과세 원천징수세율분은 `withholding_fin`**(⑤ 전단, 공제하지 않고 종합에만 합산). **분리과세(2천만원 이하) 금융은 넣지 않음.** 기본세율분 중 공제하지 않기로 한 금액(⑤ 후단)은 `withholding_fin` 에 합산. 부동산임대(주거용 제외) 결손은 다른 소득에서 공제하지 않고, 그 이월은 임대소득에서만(②·③2). 당해 임대결손 이월액은 `rental_loss`. 당해 결손 먼저(⑥). 2021-01-01 이후(법률 제17758호) | `business` · `rental` · `wage` · `pension` · `other` · `interest`(기본세율분) · `dividend`(기본세율분) · `withholding_fin` · `carryforward`(기간 내면 금액, 지났으면 0) · `rental_cf` | `cur_loss` · `rental_loss` · `leftover_current` · `leftover_cf` · `total`(결손 공제 후 종합소득금액) |
| `pit.financial_global` | 금융소득 종합과세(제14조③6·④·제17조③·제56조·제62조). 이자+가산 제외 배당+가산 대상 배당(가산액 제외) **2천만원 이하**면 합산하지 않음. 초과면 이자(비영업대금 포함)→가산 제외 배당→가산 대상 배당 순으로 기준금액을 채운 뒤 남은 가산 대상 배당에만 **100분의 10** 가산. **제1호** =(과세표준−2천만원)×기본세율 + 2천만원×14%. **제2호** = 금융×제129조 세율(비영업대금 25%·그 밖 14%) + 다른 종합소득 산출세액. 큰 금액이 산출세액. 배당세액공제 = min(가산액, 산출세액−제2호). 끝수 `floor_won`. 2024-01-01~2026-12-31 | `interest`(14%) · `interest_nonbiz`(비영업대금 25%) · `dividend_ex`(가산 제외) · `dividend`(가산 대상, 가산 전) · `other_income` · `income_deduction` | `financial` · `global_yn` · `gross_up` · `tax_1` · `tax_2` · `tax_calc` · `div_credit` · `tax`(배당세액공제 후) |
| `inh.spouse_deduction` | 상속 배우자공제·일괄공제(상속세 및 증여세법 제19조·제21조). 공제 = **max(5억원, min(실제액, 산식 한도, 30억원))**. 일괄공제는 기초+인적(`personal`)과 5억원 중 큰 금액, **배우자 단독상속이면 일괄 5억원 불가**. `estate`(산식 A)는 채무·공과금·비과세·불산입 차감 후(상증령 제17조①). 분할기한 미이행·무신고는 실제액 0→하한 5억. 제24조 종합한도는 레시피 밖. 2025-10-01 이후 | `spouse_actual` · `estate` · `bequest` · `added_gift` · `co_heirs`(함께 상속하는 자녀 수, 법정상속분은 레시피 계산) 또는 `spouse_share` · `prior_gift` · `personal`(제18조 2억+제20조) · `spouse_only`(1/0) | `formula` · `cap` · `spouse_deduction` · `chosen_personal` · `total` |
| `inh.presumed` | 추정상속재산(상속세 및 증여세법 제15조·시행령 제11조). 재산 종류별(현금·예금·유가증권 / 부동산·권리 / 그 밖) 처분·인출액이 1년 2억·2년 5억 **이상**이면 미입증액 − **min(처분·인출액×20%, 2억원)** 산입(미달이면 0). 채무 부담은 합계로 같은 방식, 비금융기관 변제의무 없는 채무는 전액. 1년·2년 기준이 모두 걸릴 때 선택은 미확인 → period 를 바꿔 두 번. 공식 계산기 미대조(grade C) | `period`(1/2) · `cash_out`·`cash_proven` · `realty_out`·`realty_proven` · `other_out`·`other_proven` · `debt_in`·`debt_proven` · `debt_no_repay` | `cash_presumed` · `realty_presumed` · `other_presumed` · `debt_presumed` · `presumed_total`(→ inh.tax `presumed`) |
| `inh.corp_bequest` | 영리법인 수유 지분상당액(제3조의2②·시행령 제3조②) = {상속세 상당액 − 받은 재산×10/100} × 지분율(음수면 0). 납부의무자 2026.1.1.~ 상속인·상속인의 배우자·직계비속·직계비속의 배우자(그 전 상속인과 그 직계비속). 상속세 상당액 계산법은 시행령에 없음(미확인). grade C | `corp_received` · `corp_tax_equiv` · `holder_ratio` | `corp_floor` · `corp_excess` · `holder_share` |
| `gift.excluded` | 합산배제·의제 증여세(상증법 제55조①1~3). path 1 명의신탁 전액·2 특수관계법인 의제이익 전액·3 그 밖의 합산배제 − 3천만원. 재산 취득자금 추정(제45조, 시행령 제34조①): 미입증액 ≥ min(취득가액 20%, 2억원)이면 미입증액 전액. 관계·혼인공제·10년 합산 없음, 세대생략 할증·신고세액공제 3% 있음. 제45조 합산배제는 2022-01-01~. grade C | `path`(1·2·3, 생략 3) · `gift_value` 또는 `acquisition_value`+`proven` · `appraisal_fee` · `generation_skip` · `minor` · `filed` | `unproven` · `presume_floor` · `presumed_gift` · `excluded_value` · `excl_ded` · `base` · `tax` · `surtax` · `filing_credit` · `payable` |
| `gift.startup` | 창업자금 증여세 과세특례(조특법 제30조의5). (min(이전+이번, 한도) − 5억원)⁺ × 10% − 이전분. 한도 30억(~2015-12-21)·30/50억(~2022)·50/100억(2023~, 10명 이상 고용). 신고세액공제·10년 합산 없음, 초과분은 gift.tax. grade C | `fund_value` · `prior_fund` · `hire10` | `cap` · `eligible` · `excess` · `base` · `tax` · `payable` |
| `gift.succession` | 가업승계 증여세 과세특례(조특법 제30조의6). 가업자산상당액 = 주식 × (총자산 − 사업무관자산)/총자산 → 한도(2015~2022 100억 / 2023~ 경영 10·20·30년 300·400·600억) − 공제(5억/10억) → 10%, 30억(~2022)·60억(2023)·120억(2024~) 초과 20%. 수증자 2명 이상은 가액 비례 안분. 납부유예(제30조의7) = 일반 납부세액 × 가업자산상당액/총 증여재산가액. grade C | `share_value` · `total_assets` · `non_business_assets` · `years_managed` · `my_share_value` · `general_payable` · `total_gift_value` | `biz_value` · `cap` · `eligible` · `general_part` · `base` · `tax` · `my_tax` · `payable` · `deferral` |
| `gift.farm_relief` | 영농자녀 증여세 감면(조특법 제71조① 100%, 제133조 5년간 감면세액 1억원 한도). relief 를 gift.tax credits_before_filing 으로 2단 호출. grade C | `farm_tax`(gift.tax tax+surtax) · `used_relief_5y` | `relief_full` · `relief_room` · `relief` · `unrelieved` |
| `civ.inheritance_share` | 민법 법정상속분·유류분 부족액(제1000조·제1003조·제1009조·제1112조~제1118조). 같은 순위 균분, 배우자는 직계비속·직계존속 상속분의 5할 가산. 유류분액 = (상속개시 재산 + 산입 증여 − 상속채무) × 법정상속분 × 유류분율(직계비속·배우자 2분의 1, 직계존속 3분의 1), 부족액 = 유류분액 − 특별수익 − 순상속분액(구체적 상속분 − 채무 × 법정상속분, 한정승인이면 음수를 0으로 — 대법원 2020다247428). 형제자매 유류분은 2024-04-25 위헌결정 뒤 상속개시분부터 0(2025-01-31 제4호 삭제). 판례 대조 grade B 3·조문 C 9. 1991-01-01 이후 | `claimant`(1 배우자·2 직계비속·3 직계존속·4 형제자매) · `spouse` · `children` · `ascendants` · `siblings` · `estate`(유증 포함) · `gifts` · `debts` · `special` · `concrete_share` · `limited` | `spouse_share` · `member_share` · `heir_share` · `reserve_rate` · `base` · `reserve` · `debt_share` · `net_share` · `shortage` |
| `civ.attorney_fee_cost` | 변호사보수의 소송비용 산입액(변호사보수의 소송비용 산입에 관한 규칙 제3조·제5조·별표, 민사소송법 제109조①). 2020-12-28 판 300만원까지 30만원, 2,000만원까지 10%, 5,000만원까지 8%, 1억원까지 6%, 1.5억까지 4%, 2억까지 2%, 5억까지 1%, 초과 0.5%(누진). 지급보수액 한도, 보전처분·자백·무변론·이행권고결정 ½, 청구 감축은 별표 금액 비율 안분(대법원 2024스672). 2008·2013·2018 판(하한 10만·30만원) 있음. as_of 는 심급 사건 접수일. 결정 수치 grade B 3·조문 C 20 | `value`(감축 후 소가) · `kind`(1 본안·2 보전처분·3 자백·무변론·이행권고) · `paid` · `claimed` | `cost` · `cost_before` · `reduced` |
| `civ.brokerage_fee` | 부동산 중개보수 상한(공인중개사법 제32조④, 시행규칙 제20조④⑤·별표 1·별표 2). 주택은 별표 1 구간 요율·한도액(작은 쪽), 주거용 오피스텔 매매 1천분의 5·임대 1천분의 4, 그 밖 1천분의 9. 임대차 거래금액 = 보증금 + 월 차임 × 100(합 5천만원 미만이면 × 70). 요구 보수 초과분 `excess`. 원 미만 절사, 일방 상한. 2021-10-19 이후(그 전 주택 요율은 시·도 조례). 서울부동산정보광장 계산기 grade A 20 | `kind`(1 주택·2 주거용 오피스텔·3 그 밖) · `deal`(1 매매·교환·2 임대차) · `amount` · `rent` · `fee` | `base` · `rate` · `by_rate` · `limit` · `fee_cap` · `excess` |
| `civ.interest` | 금전채무 지연손해금(민법 제379조·제397조·제479조, 상법 제54조, 소송촉진법 제3조·법정이율 규정, 이자제한법 제2조·최고이자율 규정). 이율 = max(법정이율 5%·상사 6%, 약정이율), 금전대차(`loan` 1)는 약정 당시 최고이자율(2007.6.30. 30%·2014.7.15. 25%·2018.2.8. 24%·2021.7.7. 20%)로 자름(약정 지연손해금도 — 대법원 2018다22350). 소장 부본 송달 다음 날부터 max(소송촉진법 이율, 약정이율): 2015.9.30.까지 연 2할·2015.10.1. 15%·2019.6.1. 12%(시행일 이후 기간분, 그 전 1심 변론종결이면 종전 이율). 일부 변제 2회는 지연손해금 먼저 충당. 일수는 법원 업무용 계산프로그램 기간식(만 1년 + 나머지 일수/365, 그 1년 사이에 2월 29일이 있으면 366), 구간마다 원 미만 버림. 판결 계산식·설명서 예시 grade B 4·조문 C 14. 2003-06-01 이후 | `principal` · `kind`(1 민사·2 상사) · `from`(첫날) · `to`(마지막 날 = as_of) · `rate` · `loan` · `contracted_on` · `served_on`(소장 부본 송달일 — 항쟁 타당이면 판결 선고일) · `closed_on` · `pay1`·`pay1_on`·`pay2`·`pay2_on` | `pre_rate` · `ltp_rate_20`·`ltp_rate_15`·`ltp_rate_12` · `interest_1`·`interest_2`·`interest_3` · `principal_after` · `unpaid_interest` · `total` |
| `reg.filing_fee` | 등기신청수수료(등기사항증명서 등 수수료규칙 제5조의2·제5조의3·제5조의5). 단가 × 부동산 개수(상업등기는 건수·목적 수). 2025-08-01 판: 소유권이전·근저당 설정 등 서면 18,000·이폼 15,000·전자 10,000, 상속 20,000·이폼 17,000·전자 20,000(특례 없음), 신탁 8,000·6,000·4,000, 그 밖(말소·변경) 4,000·3,000·1,000, 회사 설립·관할 밖 본점 이전 35,000·28,000·20,000, 그 밖 상업 7,000·5,000·2,000. 개정 전 판은 15,000·3,000·30,000·6,000원, 신탁 면제. grade B(등기예규 예시)·C | `kind`(1~6) · `method`(0 서면·1 전자신청·2 이폼) · `count` · `exempt`(kind 4·6 면제 사유) | `paper`·`efile`·`eform` → `unit` → `total` |
| `reg.license_tax` | 등기 등록면허세·지방교육세(지방세법 제28조①1·6, 제151조①2). 소유권 보존 1천분의 8, 지상권·저당권(근저당)·지역권·전세권·임차권·경매·가압류·가처분·가등기 1천분의 2(최저 6천원), 말소·변경 등 그 밖의 등기 건당 6천원, 영리법인 설립·증자 1천분의 4·비영리 1천분의 2·재평가 전입 1천분의 1(최저 11만2천5백원), 본점 이전 11만2천5백원, 지점 설치·그 밖 법인 등기 4만2백원. 대도시 법인등기 중과 × 100분의 300(최저세액 적용 뒤, 그 밖 법인 등기·부동산 등기 제외). 지방교육세 = 등록면허세 × 20%. 위택스 미리계산 grade A | `kind`(1~11 부동산·21~28 법인) · `base` · `count` · `metro` · `ord_ratio` · `ord_edu_ratio` | `rate` → `min_tax` → `std_tax` → `heavy` → `reg_tax` · `edu_tax` · `total` |
| `reg.housing_bond` | 부동산등기 제1종 국민주택채권 매입금액·즉시매도 본인부담(주택도시기금법 시행령 별표1 부표 15호·4호). 주택 유상·토지·그 밖 건물·상속증여 등 무상은 시가표준액(공유 지분, 공동주택 세대당) × 구간 요율(천분의, 특별시·광역시 구분), 저당권은 설정금액 × 1천분의 10(10억 상한). 하한 미만 0, 1만원 단수(5천원 이상 올림), 본인부담 = 채권 × 즉시매도 할인율(사용자 확인 값). 주택도시기금 매입대상금액조회 grade A 22 | `kind`(1~5) · `metro` · `base` · `discount` | `rate` · `amount` · `bond` · `burden` |
| `pay.severance` | 평균임금·퇴직금(근로기준법 제2조①6·②, 근로자퇴직급여 보장법 제4조①·제8조①). 1일 평균임금 = 3개월 임금총액(상여금·연차수당 3/12 가산) ÷ 산입 일수(1전 미만 올림), 1일 통상임금보다 낮으면 통상임금. 퇴직금 = 평균임금 × 30 × 재직일수/365(원 미만 반올림), 재직 1년·주 15시간 미만 0. 고용노동부 퇴직금 계산기 대조 | `tenure_days`(퇴직일 − 입사일, 퇴직일은 마지막 근무일 다음 날) · `period_days`(퇴직일 전 3개월 달력 일수) · `wages_3m` · `annual_bonus` · `annual_leave_pay` · `excluded_days` · `ordinary_daily` 또는 `ordinary_monthly` · `weekly_hours` · `daily_avg`(1일 평균임금을 알면) | `base_daily`(평균임금) · `severance` |
| `pay.retirement_tax` | 퇴직소득세(소득세법 제48조·제55조·제146조②, 지방세법 제103조의13①). 근속연수공제(2020~2022 옛 표·2023~ 새 표, as_of 판) → 환산급여 = (퇴직소득 − 근속연수공제) × 12/근속연수 → 환산급여공제 → 환산산출세액 × 근속연수/12. IRP 이연, 지방소득세 10%, 차감원천징수 10원 절사. 홈택스 퇴직소득 모의계산 대조 | `retire_income`(비과세 뺀 퇴직소득금액 — 퇴직금이면 `pay.severance` 의 `severance`) · `service_years`(1년 미만 올림) · `deferred_amount`(연금계좌 이체액) | `tax` · `local_tax` · `withhold` |
| `pay.social_insurance` | 직장가입자 월 4대보험료 근로자·사업주 몫(국민연금법 제88조, 국민건강보험법 제76조·시행령 제44조, 노인장기요양보험법 시행령 제4조, 고용산재보험료징수법 제13조·시행령 제12조). 2020-01 ~ 2026-12 판: 연금 기준소득월액 상하한(7월 판)·연도별 요율, 건강·장기요양은 사람별 10원 절사, 고용 사업주분은 실업급여·고용안정을 따로 절사. 4대사회보험 정보연계센터 모의계산 대조 | `wage`(비과세 뺀 월 보수) · `pension_wage` · `health_wage` · `emp_size`(1 150명 미만 / 2 150명 이상 우선지원대상기업 / 3 150명 이상 1천명 미만 / 4 1천명 이상·국가·지자체) · `ind_rate`(산재 천분율) · `pension_on` · `ue_on` | `worker_total` · `employer_total` |
| `pay.labor_allowance` | 근로기준법 소계산 묶음(제2조·제18조③·제26조·제55조·제60조, 시행령 제6조·제30조, 최저임금 고시 2022~2027). 연차 15일 + 근속 2년마다 1일(25일 한도), 1년 미만·80% 미만은 개근 월 1일(최대 11일) · 미사용 연차수당 · 해고예고수당(30일분 통상임금) · 주휴(min(주 소정근로시간, 40)/40 × 8시간) · 최저임금 일급·월 환산(주 40시간 209시간). 노동포털 연차 계산기 대조 | `ordinary_hourly` 또는 `ordinary_monthly` · `weekly_hours` · `daily_hours` · `service_years` · `full_months` · `attendance_low` · `small_workplace` · `unused_days` · `notice_days` · `notice_exempt` · `week_attended` | `annual_days` · `annual_pay` · `notice_pay` · `weekly_holiday_pay` · `min_monthly` |
| `pay.yearend` | 근로소득 연말정산 결정세액(소득세법 제47조·제50조·제51조·제55조·제59조·제59조의2~4, 지방세법 제103조의13), 귀속 2023년부터. 근로소득세액공제 한도(원 미만 절사)·자녀·출산·연금계좌(600/900만원)·보험·의료비(총급여 3% 문턱을 일반 의료비에서 먼저)·교육·기부금 → 표준세액공제 13만원 경로와 항목 경로 중 결정세액이 낮은 쪽(`use_standard`) → 지방소득세·차감징수 10원 절사. 홈택스 연말정산 자동계산 대조 | `wage_total` · `spouse` · `dependents` · `elderly` · `disabled` · `children` · `pension_premium` · `health_ins` · `pension_saving` · `pension_irp` · `ins_general` · `med_general` · `med_special` · `education` · `donation` · `housing_ded` · `other_deduction` · `rent_credit` · `prepaid_tax` · `prepaid_local` 등 | `determined`(결정세액) · `local_tax` · `use_standard` · `due_tax`(차감징수, result) |
| `pay.withholding` | 매월 근로소득 원천징수(소득세법 시행령 제194조·별표2, 지방세법 제103조의13). 간이세액표 가족 수 칸(1~11) → 1천만원 초과 산식 → 8~20세 자녀 공제(2026.3.1. 이후 20,830/45,830/+33,330) → 80·120% 선택 → 지방소득세 10%, 10원 절사. 홈택스 간이세액표 조회 grade A 24 | `monthly_pay` · `family` · `children` · `ratio` | `tax` · `tax_full` · `local_tax` · `total` |
| `vat.payable` | 부가가치세 일반과세자 납부(환급)세액(부가가치세법 제30조·제37조·제39조·제40조·제45조, 시행령 제81조). 매출세액 = 과세 공급가액 × 10%(원 미만 절사), 대손세액 = 대손금액(부가세 포함) × 110분의 10 을 **매출세액에서** 빼고 회수분은 더함, 매입세액 = 수취 매입세액 − 제39조 불공제 − 공통매입세액 × 면세공급가액 ÷ 총공급가액(면세비율 5% 미만이면서 공통매입세액 5백만원 미만, 또는 5만원 미만이면 전액 공제). 음수면 환급. 경감·공제세액·기납부세액·가산세 차감·가산. 국세청 전자신고 책자 홈택스 신고서 grade B 3 | `taxable_supply` · `zero_rate_supply` · `bad_debt` · `bad_debt_recovered` · `input_tax` · `nondeductible_tax` · `common_input_tax` · `exempt_supply` · `credits` · `prepaid` · `penalty` | `output_tax` · `bad_debt_tax` · `sales_tax` · `common_exempt` · `deductible_input` · `payable` · `final_tax` |
| `vat.simplified` | 간이과세자 납부세액(제63조·제69조, 시행령 제111조②, 2021-07-01~). 공급대가 × 업종별 부가가치율(1 소매·음식 15%·2 제조·농림어업 20%·3 숙박 25%·4 건설·운수·정보통신 30%·5 금융·전문과학·부동산임대 등 40%·6 그 밖 서비스 30%) × 10%, 세금계산서등 수취 매입 공급대가 × 0.5% 공제, 공제 합계는 납부세액 한도(환급 없음), 연 4,800만원 미만 납부면제. 국세청 책자 grade B 2 | `biz` · `supply` · `purchase` · `credits` · `other_credits` · `prepaid` | `rate` · `tax` · `purchase_credit` · `credit_allowed` · `exempt` · `final_tax` |
| `vat.deemed_supply` | 간주공급 공급가액(제10조①②④⑤⑥·제29조, 시행령 제66조·제19조의2, 2020-10-07~). 1 일반 재화 시가, 2 건물·구축물 취득가액 × (1 − 5/100 × 경과된 과세기간 수, 20 한도), 3 그 밖 감가상각자산 × (1 − 25/100 × 경과 과세기간, 4 한도), 면세사업 일부 사용은 × 면세비율(5% 미만 0), 4 사용인 경조사·설날·추석·창립기념일·생일 선물은 목별 1명당 연 10만원 초과분. 매입세액 공제받은 재화만. grade C | `kind` · `market_price` · `acquisition` · `periods` · `exempt_supply` · `total_supply` · `per_person` · `employees` | `capped_periods` · `dep_value` · `welfare_excess` · `supply_value` · `output_tax` |
| `basic.penalty` | 국세 신고·납부 가산세(국세기본법 제47조의2·제47조의3·제47조의4·제48조②, 시행령 제27조의4, 2020-01-01~). 무신고 20%(부정 40%·역외 60%, 복식부기의무자·법인은 수입금액 × 1만분의 7·14 와 큰 쪽, 영세율과세표준 1천분의 5 가산), 과소신고 10%(부정행위분만 40%·60%), 기한후신고 1·3·6개월 50·30·20%·수정신고 1·3·6개월·1년·1년 6개월·2년 90~10% 감면(납부지연은 감면 없음), 납부지연 미납세액 × 일수 × 1일 이율(기간분 1만분의 3 → 10만분의 25(2019.2.12.) → 10만분의 22(2022.2.15.)), 지정납부기한까지 미납 3%, 150만원(2021년까지 100만원) 미만 고지분은 고지 뒤 가산세 없음, 종전 5년 한도. 2026.7.1. 개정: 일 이자는 납부고지일 전날까지, 지정납부기한 뒤 월 1만분의 67 × 경과 개월(60개월 한도), 2026.6.30. 전 지정납부기한 경과분은 종전 규정(부칙 제3조). 공개 계산기 없음 grade C | `kind`(0 납부지연만·1 무신고·2 과소신고) · `tax_base` · `evasion` · `fraud_amount` · `revenue` · `zero_rated` · `self_corrected`·`known_in_advance`·`prelim`·`due_on`·`filed_on` · `unpaid`·`pay_due_on`·`paid_on`·`notice_on`·`notice_due_on`·`notice_tax` | `pen_filing_raw` → `cut_rate` → `pen_filing` · `pen_daily_pre` · `pen_daily_post` · `pen_monthly` · `pen_notice` · `pen_delay` · `penalty_total` |
| `vat.invoice_penalty` | 부가가치세법 제60조 가산세(2020-01-01~). 미등록 1%·타인 명의 1%(2025~ 2%), 지연발급 1%·미발급 2%(전자 의무자 종이 발급 등 1%)·지연전송 0.3%·미전송 0.5%·기재 부실 1%, 가공 3%(2026~ 4%)·위장·과다 기재 2%, 신용카드 0.5%, 매출·매입처 합계표 0.5%(예정분 확정 제출 0.3%), 현금매출명세서 1%. 국세기본법 제49조 한도(과세기간·의무 종류별 중소기업 5천만원·그 밖 1억원, 미발급·가공·위장·고의는 없음), 1개월 이내 등록·합계표 제출 50%(제48조②3나). grade C | 위반 종류별 공급가액 `unregistered`·`borrowed_name`·`late_issue`·`no_issue`·`no_issue_reduced`·`late_send`·`no_send`·`wrong_entry`·`fake`·`disguised`·`overstated`·`card`·`sales_summary`·`sales_summary_late`·`purchase_summary`·`cash_stmt` · `sme` · `intentional` | 항목별 `pen_*` → `limited`·`capped_amount` → `penalty_total` |
| `stc.headcount` | 상시근로자 수(조특법 시행령 제23조⑪⑫, 2026.2.27.~ 제11조의2⑧ 준용) — 통합고용·사회보험료·근로소득증대 세액공제 공통. (매월 말 상시근로자 수 합 + 월 60시간 이상 단시간근로자 × 0.5 + 요건 충족 단시간 × 0.75) ÷ 과세연도 개월 수, **100분의 1 미만 절사**. 청년등 수는 청년등만 넣어 따로 부른다. 국세청 컨설팅 사례1 grade B | `full_sum` · `part_sum` · `part_qual_sum` · `months` | `headcount` |
| `stc.employment` | 통합고용세액공제(조특법 제29조의8, 시행령 제26조의8). **2025.12.31. 이 속하는 과세연도까지**: 청년등·청년등 외 증가 인원(각각 전체 증가 한도) × 1인당(중소 수도권 1,450·850만원, 밖 1,550·950만원, 중견 800·450만원, 그 외 400·0원), 같은 금액을 다음 1년(중소·중견 2년) 공제. **2026.12.31. 이 속하는 과세연도부터**: 직전·전전·전전전 대비 증가 A·B·C × 1·2·3년차 금액(중소 청년등 700/1,600/1,700만원 등, 2024 이전 개시 연도 제외), 중견 5명·그 외 10명 최소고용증가인원수 초과분만. 육아휴직 복귀자·정규직 전환(2024년 전환분) 1,300·900만원, 농특세 20%. 국세청 사례1 grade B | `size` · `region` · `total_cur` · `total_prev` · `youth_cur` · `youth_prev` · (2026~) `total_prev2`·`total_prev3`·`youth_prev2`·`youth_prev3`·`years` · `returnees` · `conversions` | `base_credit` · `extra_credit` · `farm_tax` · `credit` |
| `stc.employment_recapture` | 통합고용세액공제 사후관리 추징(종전 규정, 시행령 제26조의8④). 청년등 감소 ≥ 전체 감소: (청년등 감소 − 전체 감소) × (청년등 금액 − 청년등 외 금액) + 전체 감소 × 청년등 금액, 그 밖: 청년등 감소 × 청년등 금액 + 청년등 외 감소 × 청년등 외 금액, 청년등만 감소: × 차액. 3년차는 × 공제 횟수 − 2년차 납부액, 한도 공제받은 세액. **as_of 는 최초 공제연도 종료일.** grade C | `size` · `region` · `first_total` · `first_youth` · `first_youth_inc` · `now_total` · `now_youth` · `period` · `times` · `credited` · `paid_first` | `recapture` |
| `stc.social_insurance` | 중소기업 사회보험료 세액공제(조특법 제30조의4①, 시행령 제27조의4, 2024.12.31. 이 속하는 과세연도까지). 청년등(15~29세·경력단절 여성) 증가(전체 증가 한도) × 청년등 1인당 사회보험료 × 100% + (전체 증가 − 청년등 증가) × 청년등 외 1인당 × 50%(신성장 서비스업 75%). 1인당 = (총급여 × 사회보험료율 − 보조금) ÷ 인원. 사회보험료율은 입력. grade C | `total_cur` · `total_prev` · `youth_cur` · `youth_prev` · `youth_wage` · `other_wage` · `ins_rate` · `new_growth` · `youth_subsidy` · `other_subsidy` | `credit` |
| `stc.wage_increase` | 근로소득증대 세액공제(조특법 제29조의4, 시행령 제26조의4, 시행규칙 제14조의2). 평균임금(1천원 이하 절사) → 증가율(1만분의 1 미만 절사) → 직전 3년 평균(직전 2·3년 음수 0) → [당해 평균 − 직전 평균 × (1 + 3년 평균)] × 직전 인원 × 중소 20%·중견 10%. 중소기업은 1천분의 32 기준 특례(제5항)와 큰 쪽, 직전 증가율 음수·30% 미만이면 시행규칙 식, 정규직 전환 임금증가분 + . 상시근로자 수 감소 시 0. grade C | `size` · `wage_cur`·`count_cur` · `wage_p1`~`wage_p4` · `count_p1`~`count_p4` · `conv_inc` | `credit_base` · `credit_sme` · `credit_conv` · `credit` |

- 다주택·법인 중과(제13조의2①)는 `acq.house_heavy`. 중과 세율은 제11조①7나 1천분의 40 + 중과기준세율 × 배수라, 표준세율 레시피의 `rate` 에 가산하면 틀린다(경고 `recipe_scope` 가 `acq.house_heavy` 를 안내).
- 상속·증여·원시취득(신축)은 제11조①8(유상거래 주택)이 아니므로 `acq.house_standard_rate` 가 아니라 `acq.by_cause`. 표준세율 레시피에 상속·증여 단계를 붙이면 경고 `recipe_scope` 가 `acq.by_cause` 를 안내한다.
- 레시피 밖: 무상취득 중과(제13조의2②)·고급주택 겹침(③)·생애최초 감면·상속·증여·원시취득 → terms·steps 로 직접.
- `cg.one_house` 밖: 1세대 1주택이 아닌 주택·보유 2년 미만(단기세율)·미등기는 `cg.rate_select`, 다주택 중과는 `cg.house_surcharge`(아래 "양도소득세 복합 사례"). 지분·부수토지 보유기간 상이·겸용주택은 `cg.one_house` 선택 입력(house_area·land_area 등). 그 밖은 terms·steps 로 직접.
  1세대 1주택 해당·비과세 요건·거주기간은 사실 판정이라 되묻고 inputs 로 준다. 다른 양도와 합산하면 `income` 뒤에 steps 를 붙인다(기본공제는 한 번만).
- `pit.comprehensive` 밖: 이자·배당 종합과세(`pit.financial_global`), 비주거용 부동산임대업 결손·금융소득이 섞인 결손 공제 순서(`pit.loss_order`), 기준경비율 추계, 세액공제·지방소득세(뒤에 steps), 퇴직·양도소득.
- `pit.loss_order` 밖: 총수입−필요경비·근로소득공제·추계신고(제45조④). 기간이 지난 이월은 0으로 준다. `total` 뒤에 종합소득공제·산출세액(`floor_won`)을 붙인다. 일반 사업 결손을 같은 해 부동산임대 소득과 먼저 통산하는지는 쟁점(미확정) — 레시피는 제45조① 문언대로 근로부터 공제한다.
- `pit.financial_global` 밖: 조세특례제한법 제104조의27 고배당 특례배당·출자공동사업자 배당 25%. 지방소득세는 뒤에 steps. 가산 제외 배당은 `dividend_ex`, 비영업대금은 `interest_nonbiz`.
- `inh.spouse_deduction` 밖: 제20조 인적공제 해당 여부·감정평가 등 그 밖의 공제(제23조 이하)·제24조 종합한도. `personal`(기초+인적)은 사실 입력이다. 법정상속분은 `co_heirs` 로 레시피가 계산한다(민법 제1009조②).
- `civ.inheritance_share` 밖: 증여의 산입 여부(제1114조·특별수익)·재산 평가·구체적 상속분(초과특별수익 재분배)은 사실 입력이다. 반환 상대방별 분담(유증 먼저 제1116조, 수인이면 초과액 비율)·반환 방식(2026-03-17 개정 제1115조 가액 지급+이자)·소멸시효는 계산하지 않는다. 상속세 배우자공제는 `inh.spouse_deduction`.
- `civ.interest` 밖: 변제기 전 약정이자만 따로 내는 계산(같은 식이지만 충당·소송촉진법 표지는 무의미), 지연손해금에 대한 지연손해금(이행청구 다음 날부터), 비용·지정충당·여러 채무의 법정충당(민법 제476조~제478조), 선이자 공제(이자제한법 제3조), 대부업법 상한, 감액(이자제한법 제6조·민법 제398조②) 판단, 복리·연체 가산금리, 2003-05-31 전 소송촉진법 연 25%. 장래이행의 소(민사소송법 제251조)는 `served_on` 을 비운다.
- `civ.attorney_fee_cost` 밖: 제6조 재량 감액·증액, 소송비용 부담 비율(일부 승소), 공동소송인 분담, 소가 산정 자체. 심급마다 따로 불러 더한다.
- `civ.brokerage_fee` 밖: 2021-10-18 이전 거래(시·도 조례 요율), 실비, 조례 개정 전(서울 2021-12-29까지) 거래는 해당 시·도 조례 판 확인, 쌍방 합계는 각각 계산해 더한다.
- `cit.tax`·`cit.entertainment` 밖: 토지등 양도소득 법인세(제55조의2)·미환류소득·세액공제·감면·최저한세·법인지방소득세, 개인사업자 기업업무추진비(소득세법 제35조), 적격증빙·문화비·전통시장 해당 여부 판정(사실 입력). 2019 개시 사업연도가 2020 에 걸치는 접대비 특례 안분, 한도초과액의 자산 감액 세무조정도 밖이다.
- `cit.incorporation_compare` 밖(RAG-9989): 지방소득세(양쪽 모두 국세 산출세액의 10% 수준), 중소기업 특별세액감면(법인도 적용)·기장세액공제(개인 전용)·창업중소기업 감면(법인전환은 창업 아님, 조특법 제6조⑩2)·자녀·특별·표준세액공제, 지역가입자 건강보험료·국민연금, 대표 퇴직금, 가지급금 인정이자, 2천만원 초과 배당의 종합과세(`pit.financial_global`). 전환할 때 사업용 고정자산 양도소득세 이월과세·취득세 감면은 `cg.incorporation_carryover`. 법인에 남긴 이익(`retained`)은 나중에 배당할 때 다시 과세되므로 `diff` 만으로 법인이 유리하다고 단정하지 않는다.
- `reg.filing_fee` 밖: 등록면허세·지방교육세·취득세·국민주택채권·열람·발급수수료·법무사 보수·국가 면제(수수료규칙 제7조③). 소유권이전과 근저당 설정처럼 등기 목적이 여럿이면 목적마다 불러 더한다.
- `reg.license_tax` 밖: 매매·상속·증여·신축 등 취득을 원인으로 하는 소유권 이전·보존 등기(취득세 — `acq.*`), 선박·차량·기계장비·항공기·지식재산권 등록, 면허분 등록면허세, 지방세특례제한법 감면(감면세액·감면분 농특세), 등기신청수수료(`reg.filing_fee`)·국민주택채권.
- `reg.housing_bond` 밖: 건축허가·면허 등 매입(부표 1~14·16~20호), 부담부증여 부담·증여 부분 안분, 매입 면제(별표1 2·3호). 할인율은 그날 공시값을 사용자에게 확인한다. 여러 부동산의 1만원 단수는 은행 매입 화면을 따른다.
- `vat.*` 고르는 법·밖(RAG-9987): 개인적공급·사업상증여·폐업 잔존재화는 `vat.deemed_supply` 의 `supply_value` 를 `vat.payable` 의 `taxable_supply` 에 더한다. 매입세액 불공제 여부(비영업용 소형승용차·기업업무추진비·면세 관련·사업 무관·등록 전)는 사실 판정이라 금액을 `nondeductible_tax` 로. 밖: 의제매입세액·재활용폐자원 공제율, 재고매입세액·재고납부세액, 감가상각자산 납부세액 재계산(제41조), 공급가액 없는 과세기간의 공통매입 안분(시행령 제81조④), 부동산 임대 간주임대료(시행령 제65조 — 계산한 금액을 공급가액에), 가산세(가산세 레시피), 2021-06-30 이전 간이과세.
- 가산세 고르는 법·밖(RAG-9986): 신고·납부 가산세는 세목과 관계없이 `basic.penalty`(양도세 정산 안에서 함께 계산하면 `cg.settle`), 세금계산서·합계표·등록 가산세는 `vat.invoice_penalty` 를 따로 불러 더한다. `as_of` 는 `basic.penalty` 납부일, `vat.invoice_penalty` 위반일. 밖: 원천징수 등 납부지연가산세(제47조의5), 인지세 가산세, 독촉 비용(2026.7.1.~ 등기우편 요금), 정당한 사유 면제(제48조①), 개별 세법 가산세(소득세법 제81조 이하·법인세법 제75조 이하·간이과세자 제68조의2), 지방세 가산세. 2020.1.1. 전 납부·수정신고는 판이 없다.
- 고용 세액공제 고르는 법·밖(RAG-9985): 인원은 먼저 `stc.headcount`(전체·청년등 따로), 그 값을 `stc.employment`·`stc.social_insurance`·`stc.wage_increase` 에 넣는다. 2026년 중에 끝나는 사업연도라도 2025.12.31. 이 속하면 종전 규정이라 `as_of` 2025-12-31. 2024·2025년 증가분의 2·3년차 공제·추징은 개편 뒤에도 종전 규정(`stc.employment_recapture`, as_of 최초 공제연도 종료일). 밖: 고용증대세액공제(제29조의7) 자체 계산, 정규직 전환·육아휴직 복귀자 2년 내 퇴사 추징, 사회보험료 세액공제 추징, 최저한세·이월공제, 청년등·상시근로자 해당 여부 판정(사실 입력).
- `pay.*` 고르는 법(RAG-9507): 퇴직금은 `pay.severance`, 그 세금은 `severance` 를 `pay.retirement_tax` 의 `retire_income` 으로(한 호출에 레시피 하나). 연차·해고예고·주휴·최저임금 월급은 `pay.labor_allowance` 한 번. 월급의 4대보험료는 `pay.social_insurance`. 근로소득만 있는 연말정산은 `pay.yearend`, 사업·금융소득과 합산하면 `pit.comprehensive`. 매월 급여 원천징수는 `pay.withholding`.
- `pay.*` 밖: 상여 원천징수(제136조)·일용근로 원천징수, 임원 퇴직소득 한도초과·중간정산 합산(제148조)·2016~2019 퇴직소득 경과규정, 지역가입자·두루누리 지원, 중소기업 취업자 감면·외국인 단일세율·기부금 이월, 주택자금·신용카드 소득공제와 월세 세액공제 계산(계산한 금액을 입력). 연차는 그 1년을 채운 다음 날에도 근로관계가 있어야 생긴다(1년만 일하고 퇴직하면 15일 없음).
- 등기 비용 합계(RAG-9518): 합계 레시피는 없다. 등기 목적마다 따로 불러 total 을 더한다 — 소유권 이전은 `acq.*`, 설정·말소·보존·법인 등기는 `reg.license_tax`, 신청수수료는 목적마다 `reg.filing_fee`(매매 + 근저당 설정이면 수수료 2번). 국민주택채권 즉시매도 본인부담은 `reg.housing_bond` 의 `burden` 을 더한다. 법무사 보수는 별도라고 밝힌다.
- 지방교육세·합계는 레시피 단계(`edu_tax`·`total`)를 옮긴다. 같은 id 로 다시 붙이면 `invalid_args` 다. 감면·지분 안분을 steps 로 붙였으면 감면 후 납부 합계도 단계로 만든다.
  국민주택규모(85㎡, 수도권 밖 읍·면 100㎡) 초과 주택의 농어촌특별세(농어촌특별세법 제5조①6, 이하는 제4조11 비과세)는 같은 호출에 붙인다(면적을 모르면 `total` 을 "85㎡ 이하 기준"으로 답한다):
  `{"recipe":"acq.house_standard_rate","inputs":{"price":800000000},"terms":{"farm_base":{"value":"100분의 2",…,"cite":{"law":"농어촌특별세법","article":"제5조","paragraph":1}},"farm_ratio":{"value":"100분의 10",…}},"steps":[{"id":"farm_tax","label":"농어촌특별세","expr":"floor_10won(price * farm_base * farm_ratio)"},{"id":"grand_total","label":"납부 합계(농특세 포함)","expr":"total + farm_tax"}]}`
- 거부: `recipe_not_found`(후보 `candidates`) · `missing_input` · `recipe_out_of_range`(정의 기간 밖 `as_of`).
- `missing_input` 되묻기 카드(RAG-9555): 응답 `ask` 에 빠진 필수 입력 **전부**가 온다 — `{recipe, recipe_label, as_of, given, questions:[{name, label, hint, kind: choice|word|date|amount|number, options:[{value,label}], free, or}]}`.
  질문에 있던 사실을 빠뜨렸으면 고쳐 다시 부르고, 없는 사실이면 값을 지어내지 말고 사용자에게 `questions` 를 한 번에 묻는다(`choice` 는 `options` 값만 받는다). 답을 `given` 에 더해 같은 `recipe`·`as_of` 로 다시 부른다.

### 양도소득세 복합 사례 — 레시피 고르는 순서 (RAG-8836)

레시피끼리는 한 호출에 하나씩 쓰고, 앞 호출의 단계 값을 다음 호출 `inputs` 로 옮긴다.

| 순서 | 상황 | 레시피 | 넘기는 값 |
|---|---|---|---|
| 1 | 취득 당시 실지거래가액을 모름 | `cg.converted_acquisition` | `expenses` 단계(환산취득가액 포함 필요경비 전체)를 다음 레시피에 `acquisition_price` 0·`expenses`=그 값. 신축·증축 5년 내면 `surcharge` 단계(가산세)를 따로 더한다 |
| 2a | 1세대 1주택 비과세·고가주택 | `cg.one_house` | 겸용·부수토지·보유기간 상이는 선택 입력 |
| 2b | 그 밖의 양도(단기·미등기·비사업용·분양권) | `cg.rate_select` | 조합원입주권은 `is_union_right`=1 + 인가 전 토지·건물분 양도차익 `union_pre_gain`(장특공은 이 금액에만 표1 율, 0이면 공제 0). 지상권·전세권·부동산임차권·취득권·이용권은 `is_other_right`=1(장특공 0). 다주택이면 `cg.house_surcharge` 결과 `surcharge`(0·0.2·0.3, 2018-04-01~2021-05-31 양도는 0.1·0.2)를 입력으로 — 2단 호출. 중과 유예는 양도일 판(2019-12-17~2020-06-30 보유 10년 이상, 2022-05-10~2026-05-09, 2026-05-10 이후는 `contract_date`). 2018-01-01~2021-05-31 조정대상지역 분양권은 `presale_regulated`=1(50%). 2018-01-01 전 양도는 판이 없다 |
| 3 | 증여받은 자산을 5년(2023-01-01 전 증여분)·10년 안에 양도 | `cg.carryover` | 이월과세·일반 두 경로를 모두 계산, 이월과세 세액이 더 적으면 일반 경로(`applied`). 증여세 상당액은 필요경비 |
| 3 | 부담부증여 | `cg.burdened_gift` | 채무 인수분만 양도. 증여세 몫도 단계로 나온다 |
| 3 | 주식 | `cg.stock` | 두 묶음 A·B, 대주주·중소기업·국외주식·단기 플래그. 묶음별 가장 이른 양도일 `transferred_a`·`transferred_b` 를 준다. 레시피가 먼저 양도한 묶음부터 기본공제를 뺀다(제103조②, 같은 날이면 세율이 높은 묶음 먼저). 공제 순서를 따로 넣지 않는다 |
| 3 | 국외 부동산 | `cg.foreign` | 원화 환산 금액, `resident_5y`(국외 5년 거주) |
| 3 | 파생상품 | `cg.derivative` | 판 2018-04-01~. `gain` = 과세기간 계약별 손익 합계(직접 비용 반영, 순손실이면 음수). 탄력세율 10%(시행령 제167조의9, 법 본문 20% 아님)·기본공제 250만원은 주식과 별도 |
| 4 | 자경농지·대토·공익사업 감면 | `cg.reduction` | `art69`·`art70`·`art77_form`(0 현금·1 채권·3·5 특약)·`art77` 플래그, 앞 단계의 `tax`·`base`·감면소득 |
| 5 | 같은 해 여러 건 | `cg.annual` | 판 2018-01-01~. 슬롯 a~f(6건, b~f 생략 가능), **한 호출 = 한 소득 그룹**(부동산 등/주식), 자산별 소득금액은 앞 레시피 `income`. 기본공제 250만원은 그룹당 1회. 부동산 등 그룹에서 둘 이상 양도하면 제104조⑤ 비교과세(합산 기본세율 `tax_cmp` vs 자산별 합계 `tax_sep` 중 큰 쪽)를 `tax` 가 반영 — 주식·파생·신탁은 `compare: 0`. 중과·비사업용 토지는 rate 0 + `add_x`(0.1·0.2·0.3) |
| 6 | 결정세액·납부할 세액·가산세·지방소득세 | `cg.settle` | `tax`=산출세액(`cg.annual` 의 `tax`), `reduction_tax`=`cg.reduction` 감면세액, `prelim_a`·`b`·`c`=예정신고 기납부세액(넷 이상은 합산), 가산세 사실 `filing_kind`·`late_days`. `as_of` 는 납부일. 단계 `determined`(결정세액)·`payable`(납부할 세액, 기납부가 크면 음수=환급)·`local_tax`·`total_amount`. 기납부가 있으면 `cg.annual` 산출세액에서 빼지 말고 여기서 뺀다 |

사실 판정(1세대 1주택·중과 배제 주택·대주주·중소기업·거주기간·취득 사유)은 도구가 하지 않는다 — 질문에 적힌 사실로 판단해 inputs 로 준다. 적히지 않은 항목(기본공제 사용 여부·다른 양도·국외 5년 거주 등)은 해당 없음으로 가정해 바로 계산하고 가정을 답에 적는다. 계산 전에 되묻지 않는다.

## 식(`expr`)

사칙연산 · 비교 · `&&` `||` · 삼항 `c ? a : b` 와 아래 함수만 쓴다.
**식 안 숫자는 0·1·100·1000 만** 허용한다(`uncited_constant`). 사례 금액·인원(2명)·연수·개월 수(12)도 terms 로 빼서 이름으로 쓴다 — 사실이면 `source:"user"`(cite 없음), 법에 적힌 숫자면 `source:"law"` + cite. 세율표 구간 숫자는 옮겨 적지 말고 `table_from` 으로 꺼낸다.

| 함수 | 뜻 |
|---|---|
| `progressive(base, table)` | 누진세액 = 구간 기본세액(`base_tax`) + 초과분 × 세율. 끝수 처리 안 함. 근로·연금소득공제표(소득세법 제47조·제47조의2)도 이 함수 |
| `bracket_lookup(table, x)` | x 가 속한 구간의 값(공제율·세율·금액). 두 표 함수는 인자 순서를 바꿔 써도 된다 |
| `floor_won(x)` · `floor_10won(x)` · `floor_to(x, unit)` | 원 미만 · 10원 미만 · unit 원 미만 절사 |
| `round_half_up(x, places)` | 소수 places 자리 사사오입 |
| `min(a, b, …)` · `max(a, b, …)` | 한도 적용. '0 미만이면 0' 은 `max(0, a - b)` (삼항으로 우회하지 않는다) |
| `pct(n)` · `permille(n)` | n/100 · n/1000. 이미 비율인 값(`"20%"`·`"20/100"`)에 또 씌우지 않는다 |
| `full_years(from, to, "civil"\|"inclusive")` | 만 연수. 기본 civil(초일 불산입) |
| `days_between(from, to, "civil"\|"inclusive")` | 일수. 기본 civil(초일 불산입, to − from), inclusive 는 +1 |
| `daily_interest(원금, 첫날, 마지막 날, 이율1, 변경일2, 이율2, …, "actual"\|"365"\|"period")` | 일할 이자(끝수 처리 안 함). 기간은 양 끝 포함, 변경일부터 새 이율(최대 4구간). actual(기본)은 윤년에 속한 날을 1/366, "365" 는 모두 1/365, "period" 는 법원 업무용 계산프로그램 기간식(이율 구간마다 만 1년 + 나머지 일수 ÷ 그 1년에 2월 29일이 있으면 366·없으면 365, 같은 이율로 바뀌는 변경일은 끊지 않음). 같은 날 변경일이 겹치면 뒤 이율 |
| `next_day(날짜)` | 다음 날. 날짜 인자 안에서만: "송달된 날의 다음 날부터" → `next_day(served_on)` |
| `prorate(amount, part, whole)` | 안분 |

합성 세율(예: 표준세율 + 중과기준세율 × 배수)은 합친 값을 term 으로 주지 말고 **구성 상수를 각각 cite 와 함께** 준 뒤 식에서 합친다.
결과가 정수가 아니면 `result_not_rounded` 경고가 난다 — 마지막 단계에 `floor_won` 을 잊지 않는다.

## 응답

| 필드 | 뜻 |
|---|---|
| `status` | `ok` · `ok_with_warnings` |
| `result` | 마지막 단계 `{id, label, value}` |
| `steps[]` | 단계별 `expr` · `value` · `detail`(걸린 구간 `row`·`over`·`upto`·세율 `rate_text`·누진공제액 `progressive_deduction`·구간 초과액 `excess`, 보유기간 기산일 `start`·만료일 `expiry`) · 삼항 조건식은 `conditions`. **MCP 기본은 축약 응답**(RAG-9498): 값 0 단계는 id 만 `zero_steps` 에, `uses`·`applied` 와 표 `rows` 는 뺀다 — 필요하면 `detail:"full"` |
| `zero_steps` | 값이 0 인 단계 id(축약 응답) |
| `terms[]` | 입력 항목. `cite` 에 서버가 찾은 개정판 `enforced_from`, `verified`(`paragraph`·`article`·`false`·`null`) |
| `tables[]` | 꺼낸 세율표. `from.enforced_from` 이 단계표 근거의 시행일이다(축약 응답은 `rows` 없음) |
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
| `recipe_scope` | 레시피에 범위 밖 계산(예: 표준세율 레시피 + 중과 가산)을 덧붙였다 | 해당하면 레시피 없이 terms·steps 로 다시 부른다. 해당하지 않으면 무시 |
| `result_not_rounded` | 결과에 원 미만이 남았다 | 끝수 함수를 넣어 다시 부른다 |
| `split_effective` | 인용 조문의 그 항·호가 부칙상 다른 날 시행된다 — `as_of` 판 문구가 아직 시행 전이거나, 이미 시행된 개정 문구가 뒤 판에만 있다 | 메시지의 시행일·근거 부칙을 답에 적는다. 개정 전·후 숫자가 다르면 `get_article` 로 맞는 판을 확인해 다시 부른다 |

### 거부 — 계산하지 않고 `isError`

`invalid_expression`(`at` 에 위치) · `float_literal` · `unknown_name` · `type_error` · `missing_cite` · `missing_label` ·
`missing_source` · `uncited_constant` · `table_not_found`(후보 `candidates`) · `table_unparsed`(표 원문) ·
`div_by_zero` · `date_order` · `limits` · 레시피 `recipe_not_found` · `missing_input` · `recipe_out_of_range`. 코드대로 고쳐 다시 부른다. 오류가 여럿이면 message 에 "입력 오류 N건" 으로 모두 나열되니 한 번에 고친다.

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
