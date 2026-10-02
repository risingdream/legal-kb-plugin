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
아래 표의 id(`pit.loss_order` · `pit.financial_global` · `inh.spouse_deduction` 포함)는 **terms·steps 로 식을 다시 짜지 않는다.** 조문을 읽은 뒤 직접 짜면 공제 순서·배당가산 순차·배우자공제 5억 하한이 빠진다.

| id | 쓰는 곳 | inputs | 단계 |
|---|---|---|---|
| `acq.house_standard_rate` | 주택 유상취득 취득세 표준세율(지방세법 제11조①8). 2020-01-01 이후 6~9억 산식((가액×2/3억원−3)/100 → **비율** 소수 넷째자리 반올림, 7억 → 0.0167), 2013-12-26~2019-12-31 판은 1·2·3%. 세액 10원 미만 절사 | `price` 취득당시가액(지분 취득이면 전체 주택 가액) | `rate` · `acq_tax` · `edu_tax`(지방교육세, 제151조①1: 취득세율×50%×20%) · `total`(감면 전 취득세+지방교육세, 농특세 제외) |
| `acq.house_heavy` | 주택 유상취득 다주택·법인 중과(지방세법 제13조의2①). 법인 12%, 조정대상지역 2주택·비조정 3주택 8%, 조정 3주택 이상·비조정 4주택 이상 12%(= 1천분의 40 + 중과기준세율 1천분의 20 × 100분의 200·400). 중과가 아니면 제11조①8 표준세율. 지방교육세는 중과면 (4%−2%)×20% = 0.4%, 농특세는 85㎡ 초과만 0.6%·1.0%(표준세율이면 0.2%). 위택스 15건 대조 | `price` · `houses`(취득 후 1세대 주택 수, 이번 주택 포함) · `regulated`(조정대상지역 1/0) · `corporate`(법인 1/0, 생략 0) · `exempt`(시가표준액 1억원 이하·일시적 2주택 등 중과 제외 1/0, 생략 0) · `over_85`(85㎡ 초과 1/0, 생략 0) | `heavy_mult`(0 = 중과 아님) · `standard_rate` · `rate` · `acq_tax` · `edu_tax` · `farm_rate` · `farm_tax` · `total`(취득세+지방교육세+농특세) |
| `acq.by_cause` | 상속(①1: 농지 2.3%, 그 밖 2.8%)·상속 외 무상취득/증여(①2: 3.5%, 비영리 2.8%)·원시취득/신축(①3: 2.8%)·그 밖 유상취득(①7: 농지 3%, 그 밖 4% — 토지·상가 등 주택 외). 조정대상지역 시가표준액 3억원 이상 주택 증여 중과(제13조의2② 12%, 시행령 제28조의6② 제외), 상속 1가구 1주택·자경농지 특례(제15조①2: 표준세율 − 2%, 농특세 비과세). 지방교육세 (표준세율−2%)×20%(중과 0.4%), 농특세 0.2%(중과 1.0%, 85㎡ 이하 주택·상속 특례 0) | `price` 과세표준 · `cause`(1·2·3·7) · `farmland` · `nonprofit` · `special` · `house` · `over_85` · `regulated` · `std_value`(생략 price) · `heavy_exempt` — 0/1 은 생략 0 | `std_rate` · `gift_heavy` · `inh_special` · `rate` · `acq_tax` · `edu_tax` · `farm_rate` · `farm_tax` · `total` |
| `acq.movable` | 부동산 외 취득(제12조①): 선박(상속 2.5%·무상 3%·원시/수입 2.02%·매매 3%·소형 2.02%·그 밖 2%), 차량(비영업 승용 7%·경자동차 4%·125cc 이하 이륜 2%·그 밖 비영업 5%·영업용 4%·자동차 외 2%), 기계장비(등록 3%·비등록 2%), 항공기(2%·2.02%·5,700kg 이상 2.01%), 입목·광업/어업/양식업권·회원권 2%. 차량·기계 과표 = max(신고가, 시가표준액)(제10조의5①2), 면세점 50만원(제17조). 지방교육세 (세율−2%)×20% 선박·기계·항공기만(제150조1), 농특세 0.2%(제124조 자동차 비과세). 공식 계산기 미대조(grade C) | `price` · `item`(1~7) · `sub`(목·세목, item 5~7 은 0) · `light` · `auto124` · `std_value`(생략 price) — 0/1 은 생략 0 | `rate` · `auto` · `base` · `exempt` · `acq_tax` · `edu_tax` · `farm_tax` · `total` |
| `acq.relief` | 취득세 감면 엔진(지방세특례제한법 제6①·17·22조의2·29·33②·36조의3·36조의5·66④⑤·67조, 최소납부 제177조의2, 지방교육세 지방세법 제151조①1다, 농특세법 제4·5조). 코드 1 생애최초 1호(300만)·2 생애최초 2호(200만)·3 출산·양육(500만)·4 서민주택(면제)·5 자경농민(50%)·6 경형 승용(75만)·7 경형 승합·화물(면제)·8~11 다자녀·12 장애인·13 국가유공자·14 보훈 상이(50%)·15 전기차(140만, 2026-12-31 일몰)·16 수소차(140만)·17 수소 화물(50%). 면제는 열거 밖이면 산출 200만원 초과 시 85%. 감면분 농특세 20%(85㎡ 이하 비과세). 위택스 감면코드 대조 grade A 5·B 3, 정액형 지방교육세 미확인(`edu_alt` 병기) | `code` · `tax`(감면 전 acq_tax) · `edu` · `farm` · `over_85` · `share` · `prior_tax` | 감면율·한도 → 최소납부 → 감면세액 → 감면 후 `acq_tax` · `edu_tax`(`edu_alt`) · `farm_tax`(`farm_alt`) · `total` |
| `prop.house` | 주택분 재산세(지방세법 제110조·제111조①3나·제111조의2·제112조①2·제113조③·제151조①6). 공정시장가액비율 60%, 1세대 1주택은 43·44·45%(시행령 제109조①2 단서, 2024~2026년도). 특례세율은 시가표준액 9억원 이하 1세대 1주택만. **공동소유는 주택 전체 과세표준에 세율 → 지분 안분**. 도시지역분 1천분의 1.4, 지방교육세 재산세액의 20%. 소유자별 세액 10원 미만 절사. 과세표준상한(제110조③·시행령 제109조의2): 작년 시가표준액×올해 비율 + 올해 과세표준×5% 와 비교해 작은 값 — `prior_price` 생략 시 미반영. 조례 가감·감면은 밖. 토지·건축물분은 `prop.land`·`prop.building` | `price` 주택 전체 시가표준액 · `one_house`(1/0) · `share`(지분, 생략 1) · `urban`(도시지역분 1/0, 생략 1) · `prior_price`(직전 연도 시가표준액, 생략하면 올해 값 = 상한 미반영) | `ratio` · `base_full` · `base_cap` · `base`(상한 적용) · `house_tax`(주택 전체) · `prop_tax` · `urban_tax` · `edu_tax` · `total`(지분 기준) |
| `prop.fire_resource` | 소방분 지역자원시설세(지방세법 제146조③④·제147조②, 시행령 제109조①·제138조). 과세표준 = 건축물 시가표준액×70%(주택 건축물 부분은 건물분 가액×60%), 6구간 누진(1만분의 4~12), 화재위험 100분의 200·대형 100분의 300, 10원 미만 절사, 세부담상한 150%. 서울시 ETAX 건축물 13건 grade A, 주택분·배수는 grade C | `price` · `kind`(1 건축물/2 주택 건축물 부분) · `fire`(0/1/2) · `base_given`(고지서 과세표준) · `prev_tax` | `ratio` · `base` · `std_tax` · `calc_tax` · `total` |
| `prop.relief` | 재산세 감면 엔진(지방세특례제한법 제31조의3④·제35조②·제36조의4②·제58조의3·제75조의5①2·제177조의2·제179조, 지방세법 제113조①·제151조①6, 종합부동산세법 제6조③). 코드 1 민간임대(40㎡ 이하 면제·60㎡ 이하 75%·85㎡ 이하 50%)·2 주택연금(25%, 5억 초과는 5억 기준 세액×25%)·3 전세사기피해(60㎡ 이하 50%·초과 25%)·4 창업중소기업(3년 면제+2년 50%)·5 인구감소지역 창업(5년 면제+3년 50%). 토지는 과세표준 경감, 면제는 산출 50만원 초과 시 85%, 감면 → 세부담상한 순(시행령 제118조3호 도출). 공식 계산기에 감면 칸이 없어 grade C | `object`(1 주택·2 토지·3 건축물) · `code` · `price` · 주택·건축물 `tax`·`urban_tax` / 토지 `base`·`kind`·`urban_tax`·`relief_base` · `area` · `year` · `share` · `prev_tax`·`prev_urban` | 감면율 → 최소납부 → 감면 후 재산세·도시지역분·지방교육세 · `total` · `cpt_price_cut` |
| `cpt.house` | 주택분 종합부동산세(종합부동산세법 제8조①·제9조①②③⑤~⑨·제10조·제10조의2, 시행령 제4조의3①·제5조의2, 농어촌특별세법 제5조①). 공제 12억/9억/법인 0원 × 60%, 세율 2주택 이하·3주택 이상·법인 2.7%/5%. 재산세 공제 = 부과세액 × (과세표준 × 재산세 비율 × 1천분의 4) ÷ 표준세율 상당액(1주택은 prop.house 식으로 안에서 재계산). 세액공제 연령+보유 80% 한도(재산세 공제 후). 세부담상한 prev_total×150%. 농특세 20%. 분자 비율 45%/60%·분모 상한은 홈택스 미대조(grade C) | `price_sum` · `owner` · `corp` · `houses` · `prop_levied`·`prop_std`(다주택) · `prop_one_house` · `share` · `joint` · `prior_price` · `relief_cut` · `age` · `years_held` · `excluded_price` · `prev_total` | 과세표준 → 세율 → 재산세 공제 → 세액공제 → 세부담상한 → 종부세 · 농특세 · `total`(joint 면 `sep_total` 병기) |
| `cpt.land` | 토지분 종합부동산세(종합부동산세법 제12조①·제13조·제14조①③④⑥·제15조, 시행령 제2조의4②·제5조의3·제6조·제7조, 농어촌특별세법 제5조①). 종합합산 공제 5억·세율 1%/2%/3%, 별도합산 공제 80억·0.5%/0.6%/0.7%, 공정시장가액비율 100%. 재산세 공제 = 부과세액 × (과세표준 × 70% × 종합 0.5%/별도 0.4%) ÷ 표준세율 상당액(시·군·구별 합계 입력, 생략하면 한 시·군·구로 재계산). 세부담상한 prev_total×150%(법인 예외 없음). 농특세 20%. 분자 단일 세율·분모 합산 단위는 홈택스 미대조(grade C) | `kind`(1 종합합산·2 별도합산) · `price_sum` · `prop_levied`·`prop_std` · `relief_cut` · `prev_total` | 과세표준 → 세율 → 재산세 공제 → 세부담상한 → 종부세 · 농특세 · `total` |
| `cg.one_house` | 1세대 1주택 양도소득세(소득세법 제89조①3·제95조②·시행령 제160조). 비과세 요건 충족이면 양도차익 × (양도가액 − 기준금액)/양도가액만 과세(기준금액 **2021-12-08 양도분부터 12억원**(부칙 선시행), 2021-01-01~12-07 9억원). 장특공은 보유 3년 미만 0 · **거주 2년 이상 표2(보유+거주)** · 그 밖 표1. 기본공제 250만원·기본세율까지. 2021-01-01 이후 양도분 | `transfer_price` · `acquisition_price` · `expenses` · `acquired_on` · `transferred_on` · `residence_years`(보유기간 중 거주 만 연수) · `one_home_exempt`(비과세 요건 충족 1 / 미충족 0) · `residence_waived`(상생임대 거주요건 면제, 2022-08-02 양도분~) | `gain` · `years` · `taxable_gain` · `ltd_rate` · `ltd` · `income` · `base` · `tax` |
| `cg.relief` | 양도세 세액감면 엔진(소득세법 제90조①, 조특법 제69조~제70조·제77조~제77조의3·제97조의5·제133조, 농특세법 제5조①·시행령 제4조). 코드 1 자경농지·2 축사용지·3 어업용·4 자경산지(경영 연수 10~50%)·5 농지대토·6 공익사업(현금·채권·3·5년 특약, 2025.3.14. 판 분리)·7 대토보상 40%·8 개발제한구역 40·25%·9 장기일반민간임대 100%. 편입일·임대기간 기준시가 또는 보상액 비율로 감면소득 안분, 제133조 그룹 한도, 조문별 농특세. 공식 계산기 미대조(grade C) | `code` · `income` · `tax`·`base` · `deduction` · `std_acq`·`std_mid`·`std_transfer` 또는 `part_comp`·`total_comp` · `form` · `prior_same_year`·`prior_4y` | 감면율 → 안분 → 감면세액 → 한도 → 농특세 → `total` |
| `cg.relief_deduct` | 소득공제형 감면(소득세법 제90조②, 조특법 제99조의2·제98조의3, 시행령 제40조①). 취득 후 5년 이내 양도는 세액감면, 그 뒤는 5년간 소득(기준시가 안분)을 공제하고 기본세율로 재계산, 농특세 = 공제 전후 세액 차 × 20%. grade C | `code` · `overcrowded` · `acquired_on` · `transferred_on` · `income` · `std_acq`·`std_mid`·`std_transfer` · `tax`·`base`(5년 이내) | 5년 판정 → 공제 소득 → 재계산 세액 → 농특세 → `total` |
| `cg.rental_ltd` | 임대주택 장기보유특별공제 특례(조특법 제97조의3 70%·제97조의4 추가공제율 2~10%p, 소득세법 제95조②). 70% 는 임대기간 양도차익에만, 나머지는 표1(도출). grade C | `mode` · `gain` · `acquired_on` · `transferred_on` · `rent_years`(mode 2) · `std_*`(mode 1, 2026.2.27.~) 또는 `rent_gain` | 공제율 → 장기보유특별공제 → 양도소득금액 → 산출세액 |
| `pit.loss_order` | 종합소득 결손금·이월결손금 공제 순서(소득세법 제45조). 당해 사업결손은 근로→연금→기타→이자→배당, 이월은 사업→근로→연금→기타→이자→배당. **종합과세 원천징수세율분은 `withholding_fin`**(⑤ 전단, 공제하지 않고 종합에만 합산). **분리과세(2천만원 이하) 금융은 넣지 않음.** 기본세율분 중 공제하지 않기로 한 금액(⑤ 후단)은 `withholding_fin` 에 합산. 부동산임대(주거용 제외) 결손은 다른 소득에서 공제하지 않고, 그 이월은 임대소득에서만(②·③2). 당해 임대결손 이월액은 `rental_loss`. 당해 결손 먼저(⑥). 2021-01-01 이후(법률 제17758호) | `business` · `rental` · `wage` · `pension` · `other` · `interest`(기본세율분) · `dividend`(기본세율분) · `withholding_fin` · `carryforward`(기간 내면 금액, 지났으면 0) · `rental_cf` | `cur_loss` · `rental_loss` · `leftover_current` · `leftover_cf` · `total`(결손 공제 후 종합소득금액) |
| `pit.financial_global` | 금융소득 종합과세(제14조③6·④·제17조③·제56조·제62조). 이자+가산 제외 배당+가산 대상 배당(가산액 제외) **2천만원 이하**면 합산하지 않음. 초과면 이자(비영업대금 포함)→가산 제외 배당→가산 대상 배당 순으로 기준금액을 채운 뒤 남은 가산 대상 배당에만 **100분의 10** 가산. **제1호** =(과세표준−2천만원)×기본세율 + 2천만원×14%. **제2호** = 금융×제129조 세율(비영업대금 25%·그 밖 14%) + 다른 종합소득 산출세액. 큰 금액이 산출세액. 배당세액공제 = min(가산액, 산출세액−제2호). 끝수 `floor_won`. 2024-01-01~2026-12-31 | `interest`(14%) · `interest_nonbiz`(비영업대금 25%) · `dividend_ex`(가산 제외) · `dividend`(가산 대상, 가산 전) · `other_income` · `income_deduction` | `financial` · `global_yn` · `gross_up` · `tax_1` · `tax_2` · `tax_calc` · `div_credit` · `tax`(배당세액공제 후) |
| `inh.spouse_deduction` | 상속 배우자공제·일괄공제(상속세 및 증여세법 제19조·제21조). 공제 = **max(5억원, min(실제액, 산식 한도, 30억원))**. 일괄공제는 기초+인적(`personal`)과 5억원 중 큰 금액, **배우자 단독상속이면 일괄 5억원 불가**. `estate`(산식 A)는 채무·공과금·비과세·불산입 차감 후(상증령 제17조①). 분할기한 미이행·무신고는 실제액 0→하한 5억. 제24조 종합한도는 레시피 밖. 2025-10-01 이후 | `spouse_actual` · `estate` · `bequest` · `added_gift` · `spouse_share` · `prior_gift` · `personal`(제18조 2억+제20조) · `spouse_only`(1/0) | `formula` · `cap` · `spouse_deduction` · `chosen_personal` · `total` |

- 다주택·법인 중과(제13조의2①)는 `acq.house_heavy`. 중과 세율은 제11조①7나 1천분의 40 + 중과기준세율 × 배수라, 표준세율 레시피의 `rate` 에 가산하면 틀린다(경고 `recipe_scope` 가 `acq.house_heavy` 를 안내).
- 상속·증여·원시취득(신축)은 제11조①8(유상거래 주택)이 아니므로 `acq.house_standard_rate` 가 아니라 `acq.by_cause`. 표준세율 레시피에 상속·증여 단계를 붙이면 경고 `recipe_scope` 가 `acq.by_cause` 를 안내한다.
- 레시피 밖: 무상취득 중과(제13조의2②)·고급주택 겹침(③)·생애최초 감면·상속·증여·원시취득 → terms·steps 로 직접.
- `cg.one_house` 밖: 1세대 1주택이 아닌 주택·보유 2년 미만(단기세율)·미등기는 `cg.rate_select`, 다주택 중과는 `cg.house_surcharge`(아래 "양도소득세 복합 사례"). 지분·부수토지 보유기간 상이·겸용주택은 `cg.one_house` 선택 입력(house_area·land_area 등). 그 밖은 terms·steps 로 직접.
  1세대 1주택 해당·비과세 요건·거주기간은 사실 판정이라 되묻고 inputs 로 준다. 다른 양도와 합산하면 `income` 뒤에 steps 를 붙인다(기본공제는 한 번만).
- `pit.loss_order` 밖: 총수입−필요경비·근로소득공제·추계신고(제45조④). 기간이 지난 이월은 0으로 준다. `total` 뒤에 종합소득공제·산출세액(`floor_won`)을 붙인다. 일반 사업 결손을 같은 해 부동산임대 소득과 먼저 통산하는지는 쟁점(미확정) — 레시피는 제45조① 문언대로 근로부터 공제한다.
- `pit.financial_global` 밖: 조세특례제한법 제104조의27 고배당 특례배당·출자공동사업자 배당 25%. 지방소득세는 뒤에 steps. 가산 제외 배당은 `dividend_ex`, 비영업대금은 `interest_nonbiz`.
- `inh.spouse_deduction` 밖: 제20조 인적공제 해당 여부·감정평가 등 그 밖의 공제(제23조 이하)·제24조 종합한도. 법정상속분과 `personal`(기초+인적)은 사실 입력이다.
- 지방교육세·합계는 레시피 단계(`edu_tax`·`total`)를 옮긴다. 같은 id 로 다시 붙이면 `invalid_args` 다. 감면·지분 안분을 steps 로 붙였으면 감면 후 납부 합계도 단계로 만든다.
  국민주택규모(85㎡, 수도권 밖 읍·면 100㎡) 초과 주택의 농어촌특별세(농어촌특별세법 제5조①6, 이하는 제4조11 비과세)는 같은 호출에 붙인다(면적을 모르면 `total` 을 "85㎡ 이하 기준"으로 답한다):
  `{"recipe":"acq.house_standard_rate","inputs":{"price":800000000},"terms":{"farm_base":{"value":"100분의 2",…,"cite":{"law":"농어촌특별세법","article":"제5조","paragraph":1}},"farm_ratio":{"value":"100분의 10",…}},"steps":[{"id":"farm_tax","label":"농어촌특별세","expr":"floor_10won(price * farm_base * farm_ratio)"},{"id":"grand_total","label":"납부 합계(농특세 포함)","expr":"total + farm_tax"}]}`
- 거부: `recipe_not_found`(후보 `candidates`) · `missing_input` · `recipe_out_of_range`(정의 기간 밖 `as_of`).

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
| `prorate(amount, part, whole)` | 안분 |

합성 세율(예: 표준세율 + 중과기준세율 × 배수)은 합친 값을 term 으로 주지 말고 **구성 상수를 각각 cite 와 함께** 준 뒤 식에서 합친다.
결과가 정수가 아니면 `result_not_rounded` 경고가 난다 — 마지막 단계에 `floor_won` 을 잊지 않는다.

## 응답

| 필드 | 뜻 |
|---|---|
| `status` | `ok` · `ok_with_warnings` |
| `result` | 마지막 단계 `{id, label, value}` |
| `steps[]` | 단계별 `value` · `uses` · `detail`(걸린 구간 `row`·`over`·`upto`·세율 `rate_text`·누진공제액 `progressive_deduction`·구간 초과액 `excess`, 보유기간 기산일 `start`·만료일 `expiry`) |
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
