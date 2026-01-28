# RULES_SPEC.md - 비즈니스 로직 규칙 명세서

> **문서 버전**: v1.0  
> **최종 수정**: 2026-01-26  
> **작성 목적**: MOQ 고도화, 인증 매칭, 성공사례 보너스 규칙 확정

---

## 1. MOQ (최소주문수량) 고도화 규칙

### 1.1 Hard Gate 조건 (거래 불가)

MOQ 불일치가 **거래 불가 수준**일 때 해당 후보를 결과에서 **즉시 탈락** 시킨다.

| 조건 | 기준 | reason 코드 |
|------|------|-------------|
| 바이어 MOQ < 셀러 MOQ × 0.3 | 바이어가 셀러 최소수량의 30% 미만만 원함 | `MOQ_BUYER_TOO_SMALL` |
| 셀러 MOQ > 바이어 MOQ × 3.0 | 셀러 최소수량이 바이어 요청의 3배 초과 | `MOQ_SELLER_TOO_LARGE` |

**출력 필드**:
```json
{
  "moq_gate_passed": false,
  "moq_gate_reason": "MOQ_BUYER_TOO_SMALL",
  "seller_moq": 5000,
  "buyer_moq": 500,
  "moq_ratio": 0.1
}
```

**예시**:
- 셀러 MOQ = 5000, 바이어 MOQ = 500 → ratio = 0.1 < 0.3 → **탈락** (`MOQ_BUYER_TOO_SMALL`)
- 셀러 MOQ = 10000, 바이어 MOQ = 2000 → ratio = 5.0 > 3.0 → **탈락** (`MOQ_SELLER_TOO_LARGE`)

---

### 1.2 Soft MOQ Score (0 ~ 1)

Hard Gate를 통과한 경우, **범위 내 적합도**에 따라 연속 점수를 부여한다.

#### 수식

```
moq_ratio = buyer_moq / seller_moq

if moq_ratio >= 1.0:
    # 바이어가 셀러 MOQ 이상 원함 → 완벽
    moq_score = 1.0
elif 0.8 <= moq_ratio < 1.0:
    # 약간 부족 → 선형 감쇠
    moq_score = 0.8 + (moq_ratio - 0.8) * 1.0
elif 0.5 <= moq_ratio < 0.8:
    # 중간 불일치 → 더 큰 감쇠
    moq_score = 0.4 + (moq_ratio - 0.5) * (0.4 / 0.3)
elif 0.3 <= moq_ratio < 0.5:
    # 경계 근처 → 급격한 감쇠
    moq_score = (moq_ratio - 0.3) * (0.4 / 0.2)
else:
    # Hard Gate에서 걸러져야 하지만 안전장치
    moq_score = 0.0
```

#### 점수 변환 테이블

| moq_ratio 범위 | moq_score 범위 | 설명 |
|----------------|----------------|------|
| ≥ 1.0 | 1.0 | 완벽 일치 (바이어 수량 ≥ 셀러 최소) |
| 0.8 ~ 0.99 | 0.8 ~ 0.99 | 약간 부족 |
| 0.5 ~ 0.79 | 0.4 ~ 0.79 | 협상 필요 |
| 0.3 ~ 0.49 | 0.0 ~ 0.39 | 위험 구간 |
| < 0.3 | - | Hard Gate 탈락 |

---

### 1.3 MOV (최소주문금액) 반영 규칙

MOV는 `seller_moq × unit_price_min`으로 계산하며, 바이어의 **예산 범위**와 비교한다.

#### 수식

```python
mov_usd = seller_moq * seller_price_min
buyer_budget_min = buyer_moq * buyer_price_min
buyer_budget_max = buyer_moq * buyer_price_max

if mov_usd <= buyer_budget_min:
    mov_score = 1.0  # 바이어 최소예산으로도 충족
elif mov_usd <= buyer_budget_max:
    # 예산 범위 내 → 비례 점수
    mov_score = 1.0 - 0.3 * ((mov_usd - buyer_budget_min) / (buyer_budget_max - buyer_budget_min))
else:
    mov_score = 0.0  # 예산 초과 → 거래 불가
    mov_gate_passed = False
```

#### 예시 계산 (숫자 포함)

**Case A: 예산 범위 내**
```
셀러: moq=1000, price_range=[5.0, 8.0]
바이어: moq=1200, price_range=[6.0, 9.0]

mov_usd = 1000 × 5.0 = $5,000
buyer_budget_min = 1200 × 6.0 = $7,200
buyer_budget_max = 1200 × 9.0 = $10,800

mov_usd($5,000) <= buyer_budget_min($7,200)
→ mov_score = 1.0
→ mov_gate_passed = true
```

**Case B: 예산 범위 경계**
```
셀러: moq=3000, price_range=[3.0, 5.0]
바이어: moq=2000, price_range=[4.0, 6.0]

mov_usd = 3000 × 3.0 = $9,000
buyer_budget_min = 2000 × 4.0 = $8,000
buyer_budget_max = 2000 × 6.0 = $12,000

$8,000 < mov_usd($9,000) <= $12,000
→ mov_score = 1.0 - 0.3 × ((9000 - 8000) / (12000 - 8000))
→ mov_score = 1.0 - 0.3 × 0.25 = 0.925
→ mov_gate_passed = true
```

**Case C: 예산 초과**
```
셀러: moq=5000, price_range=[4.0, 6.0]
바이어: moq=1000, price_range=[3.0, 5.0]

mov_usd = 5000 × 4.0 = $20,000
buyer_budget_max = 1000 × 5.0 = $5,000

mov_usd($20,000) > buyer_budget_max($5,000)
→ mov_score = 0.0
→ mov_gate_passed = false
→ mov_gate_reason = "MOV_EXCEEDS_BUDGET"
```

---

### 1.4 MOQ/MOV 통합 점수

```python
# Final MOQ-related score (0~10 points contribution to fit_score)
moq_final_score = (moq_score * 0.6 + mov_score * 0.4) * 10
```

#### explanation 필드 구조

```json
{
  "moq_gate_passed": true,
  "moq_score": 0.85,
  "mov_gate_passed": true,
  "mov_score": 0.925,
  "moq_final_score": 8.8,
  "order_value_usd": 9000,
  "seller_moq": 3000,
  "buyer_moq": 2000,
  "moq_ratio": 0.667,
  "mov_usd": 9000,
  "buyer_budget_range": [8000, 12000]
}
```

---

## 2. 인증 매칭 규칙

### 2.1 인증 분류

| 분류 | 설명 | 예시 |
|------|------|------|
| `required_certs` | 해당 국가/산업에서 **법적 필수** 인증 | FDA(미국 식품/화장품), NMPA(중국), CE(유럽) |
| `preferred_certs` | 경쟁력을 높이는 **선호** 인증 | ISO, HALAL, KOSHER, GMP |

### 2.2 Required 인증 미충족 시 처리

**기본 규칙**: 필수 인증 1개라도 없으면 **즉시 탈락**

```python
missing_required = set(buyer_required_certs) - set(seller_certs)

if len(missing_required) > 0:
    return {
        "cert_gate_passed": False,
        "cert_gate_reason": "MISSING_REQUIRED_CERTS",
        "missing_required_certs": list(missing_required),
        "cert_score": 0.0
    }
```

### 2.3 점수 산식

#### 필수 충족률 (가중치 70%)

```python
required_match_rate = len(matched_required) / len(buyer_required_certs) if buyer_required_certs else 1.0
required_score = required_match_rate * 0.7
```

#### 선호 인증 가중치 (가중치 30%)

```python
preferred_match_count = len(set(seller_certs) & set(buyer_preferred_certs))
max_preferred_score = min(preferred_match_count * 0.1, 0.3)  # 최대 0.3
```

#### 최종 인증 점수

```python
cert_score = (required_score + max_preferred_score)  # 0 ~ 1.0

# fit_score 기여도 (최대 15점)
cert_contribution = cert_score * 15
```

### 2.4 예시 계산

**Case: FDA 필수, ISO/HALAL 선호**
```
바이어 required_certs: ["FDA"]
바이어 preferred_certs: ["ISO", "HALAL", "GMP"]
셀러 certifications: ["FDA", "ISO", "CE"]

# 필수 체크
matched_required = {"FDA"}  # 1개 일치
required_score = 1.0 * 0.7 = 0.7

# 선호 체크
matched_preferred = {"ISO"}  # 1개 일치
preferred_score = min(1 * 0.1, 0.3) = 0.1

# 최종
cert_score = 0.7 + 0.1 = 0.8
cert_contribution = 0.8 * 15 = 12.0점
```

### 2.5 explanation 필드 구조

```json
{
  "cert_gate_passed": true,
  "cert_score": 0.8,
  "cert_contribution": 12.0,
  "matched_required_certs": ["FDA"],
  "matched_preferred_certs": ["ISO"],
  "missing_required_certs": [],
  "missing_preferred_certs": ["HALAL", "GMP"],
  "seller_certs": ["FDA", "ISO", "CE"],
  "buyer_required_certs": ["FDA"],
  "buyer_preferred_certs": ["ISO", "HALAL", "GMP"]
}
```

---

## 3. 성공사례 보너스 규칙

### 3.1 보너스 수식

```python
success_bonus = 10 * country_match * hs_similarity * recency
```

| 요소 | 값 | 설명 |
|------|-----|------|
| `country_match` | 0 또는 1 | 성공사례 국가 == 대상 바이어 국가 |
| `hs_similarity` | 0.6 / 0.8 / 1.0 | HS코드 일치 수준 |
| `recency` | 1.0 / 0.6 / 0.3 | 성공사례 최신도 |

### 3.2 HS 유사도 기준

| 조건 | hs_similarity |
|------|---------------|
| HS 6자리 완전 일치 | 1.0 |
| HS 4자리 일치 | 0.8 |
| 같은 산업군 (매핑 테이블 기반) | 0.6 |
| 불일치 | 0.0 |

```python
def calc_hs_similarity(case_hs: str, target_hs: str) -> float:
    if case_hs[:6] == target_hs[:6]:
        return 1.0
    elif case_hs[:4] == target_hs[:4]:
        return 0.8
    elif same_industry_group(case_hs, target_hs):
        return 0.6
    else:
        return 0.0
```

### 3.3 최신도 (Recency) 기준

| 조건 | recency |
|------|---------|
| 2년 이내 (≤ 730일) | 1.0 |
| 2~4년 (731~1460일) | 0.6 |
| 4년 초과 | 0.3 |

```python
def calc_recency(case_date: date, today: date) -> float:
    days_ago = (today - case_date).days
    if days_ago <= 730:
        return 1.0
    elif days_ago <= 1460:
        return 0.6
    else:
        return 0.3
```

### 3.4 키워드만 맞는 사례 처리

**규칙**: 국가 불일치인 경우, 보너스는 **0점에 가깝게** 제한

```python
if country_match == 0:
    # 국가 불일치 → 최대 1점만 부여 (참고용)
    success_bonus = min(1.0, 10 * 0 * hs_similarity * recency)  # = 0
    # 대신 reference_only 플래그 설정
    is_reference_only = True
```

### 3.5 다중 성공사례 집계

```python
total_bonus = 0
matched_cases = []

for case in success_cases:
    bonus = calc_single_case_bonus(case, target_country, target_hs)
    if bonus > 0:
        matched_cases.append({
            "case_id": case["id"],
            "bonus": bonus,
            "country_match": ...,
            "hs_similarity": ...,
            "recency": ...
        })
        total_bonus += bonus

# 최대 보너스 캡: 20점
final_success_bonus = min(total_bonus, 20)
best_case_id = matched_cases[0]["case_id"] if matched_cases else None
```

### 3.6 예시 계산

**Case A: 완벽 일치**
```
성공사례: 국가=US, HS=330499, 날짜=2025-06-01
대상 바이어: 국가=US, HS=330499

country_match = 1
hs_similarity = 1.0  (6자리 완전 일치)
recency = 1.0  (1년 미만)

success_bonus = 10 * 1 * 1.0 * 1.0 = 10.0점
```

**Case B: HS 4자리만 일치, 오래됨**
```
성공사례: 국가=US, HS=330410, 날짜=2022-01-01
대상 바이어: 국가=US, HS=330499

country_match = 1
hs_similarity = 0.8  (4자리 일치)
recency = 0.3  (4년 초과)

success_bonus = 10 * 1 * 0.8 * 0.3 = 2.4점
```

**Case C: 국가 불일치 (키워드만 일치)**
```
성공사례: 국가=DE, HS=330499, 날짜=2025-01-01
대상 바이어: 국가=US, HS=330499

country_match = 0
→ success_bonus = 0점
→ is_reference_only = true
```

### 3.7 explanation 필드 구조

```json
{
  "success_bonus": 12.4,
  "matched_cases_count": 2,
  "best_case_id": "case_001",
  "cases_detail": [
    {
      "case_id": "case_001",
      "bonus": 10.0,
      "country_match": true,
      "hs_similarity": 1.0,
      "recency": 1.0,
      "case_country": "US",
      "case_hs": "330499",
      "case_date": "2025-06-01"
    },
    {
      "case_id": "case_002",
      "bonus": 2.4,
      "country_match": true,
      "hs_similarity": 0.8,
      "recency": 0.3,
      "case_country": "US",
      "case_hs": "330410",
      "case_date": "2022-01-01"
    }
  ],
  "reference_only_cases": [
    {
      "case_id": "case_003",
      "reason": "COUNTRY_MISMATCH",
      "case_country": "DE"
    }
  ]
}
```

---

## 4. 통합 FitScore 계산식

최종 fit_score 계산에 모든 규칙을 반영:

```python
def calculate_fit_score(seller, buyer, fraud_risk, success_cases):
    # 1. Gate 체크 (하나라도 실패 시 즉시 탈락)
    if not moq_gate_passed or not mov_gate_passed:
        return None, {"gate_failed": "MOQ/MOV"}
    if not cert_gate_passed:
        return None, {"gate_failed": "CERT"}
    
    # 2. 점수 구성요소
    base_score = 50
    hs_match_bonus = 20 if hs_codes_match else 0
    price_bonus = 15 if price_ranges_overlap else 0
    moq_contribution = moq_final_score  # 0~10
    cert_contribution = cert_score * 15  # 0~15
    fraud_penalty = calculate_fraud_penalty(fraud_risk)  # -25~0
    success_bonus = calculate_success_bonus(success_cases)  # 0~20
    
    # 3. 최종 점수
    total = (base_score + hs_match_bonus + price_bonus + 
             moq_contribution + cert_contribution + 
             fraud_penalty + success_bonus)
    
    return max(0, min(100, total)), breakdown
```

---

## 부록: 필드명 요약

| 카테고리 | 필드명 | 타입 | 설명 |
|----------|--------|------|------|
| MOQ | `moq_gate_passed` | bool | MOQ Hard Gate 통과 여부 |
| MOQ | `moq_gate_reason` | string | 탈락 사유 코드 |
| MOQ | `moq_score` | float | Soft MOQ 점수 (0~1) |
| MOQ | `mov_gate_passed` | bool | MOV Gate 통과 여부 |
| MOQ | `mov_score` | float | MOV 점수 (0~1) |
| MOQ | `moq_final_score` | float | 최종 MOQ 기여 점수 (0~10) |
| MOQ | `order_value_usd` | float | 예상 주문 금액 |
| 인증 | `cert_gate_passed` | bool | 필수 인증 Gate 통과 |
| 인증 | `cert_score` | float | 인증 점수 (0~1) |
| 인증 | `cert_contribution` | float | fit_score 기여 (0~15) |
| 인증 | `missing_required_certs` | list | 미충족 필수 인증 |
| 인증 | `matched_preferred_certs` | list | 일치 선호 인증 |
| 성공사례 | `success_bonus` | float | 성공사례 보너스 (0~20) |
| 성공사례 | `matched_cases_count` | int | 유효 사례 수 |
| 성공사례 | `best_case_id` | string | 최고 점수 사례 ID |
