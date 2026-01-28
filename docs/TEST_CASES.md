# TEST_CASES.md - 테스트 케이스 명세서

> **문서 버전**: v1.0  
> **최종 수정**: 2026-01-26  
> **총 케이스 수**: 18개 (추천 6, 매칭 6, 시뮬레이션 6)

---

## 테스트 케이스 구조

각 케이스는 다음 형식으로 작성:
- **ID**: 고유 식별자
- **카테고리**: recommendation / matching / simulation
- **시나리오**: 테스트 목적
- **입력**: JSON 요청 데이터
- **조건**: 외부 의존성 상태 (KOTRA API, 캐시 등)
- **기대 결과**: 예상 응답

---

## 1. 추천 (Recommendation) 테스트 케이스

### REC-001: 정상 추천 - KOTRA API 성공

**시나리오**: KOTRA 수출유망추천정보 API가 정상 응답할 때 추천 결과 반환

**입력**:
```json
{
  "hs_code": "330499",
  "current_export_countries": [],
  "goal": "new_market",
  "top_n": 5
}
```

**조건**:
- KOTRA 수출유망추천정보 API: ✅ 성공 (10개 국가 반환)
- KOTRA 국가정보 API: ✅ 성공
- 캐시: ❌ miss

**기대 결과**:
```json
{
  "hs_code": "330499",
  "goal": "new_market",
  "total_countries_analyzed": 10,
  "recommendations": [
    {
      "rank": 1,
      "country_code": "US",
      "country_name": "미국",
      "success_score": 18.5,
      "success_probability": 0.72
    }
    // ... 총 5개
  ],
  "data_sources": ["KOTRA 수출유망추천정보", "KOTRA 국가정보"]
}
```

---

### REC-002: Fallback - KOTRA API 실패 시 캐시 사용

**시나리오**: KOTRA API 실패 시 캐시된 데이터로 추천 제공

**입력**:
```json
{
  "hs_code": "330499",
  "current_export_countries": [],
  "goal": "new_market",
  "top_n": 5
}
```

**조건**:
- KOTRA 수출유망추천정보 API: ❌ 실패 (timeout)
- 캐시: ✅ hit (30분 전 데이터)

**기대 결과**:
```json
{
  "hs_code": "330499",
  "recommendations": [/* 캐시된 5개 국가 */],
  "data_sources": ["캐시 데이터 (30분 전)"],
  "cache_info": {
    "hit": true,
    "cached_at": "2026-01-26T10:00:00Z",
    "ttl_remaining_seconds": 1800
  }
}
```

---

### REC-003: Fallback - API 실패 + 캐시 미스 → 대체 스코어

**시나리오**: KOTRA API 실패, 캐시도 없을 때 기본 국가 목록 반환

**입력**:
```json
{
  "hs_code": "999999",
  "current_export_countries": [],
  "goal": "new_market",
  "top_n": 5
}
```

**조건**:
- KOTRA 수출유망추천정보 API: ❌ 실패
- 캐시: ❌ miss
- 대체 스코어 로직: ✅ 활성화

**기대 결과**:
```json
{
  "hs_code": "999999",
  "recommendations": [
    {"rank": 1, "country_code": "US", "country_name": "미국", "success_score": 3.5},
    {"rank": 2, "country_code": "CN", "country_name": "중국", "success_score": 3.2},
    {"rank": 3, "country_code": "VN", "country_name": "베트남", "success_score": 3.0},
    {"rank": 4, "country_code": "JP", "country_name": "일본", "success_score": 2.8},
    {"rank": 5, "country_code": "DE", "country_name": "독일", "success_score": 2.6}
  ],
  "data_sources": ["기본 데이터 (대체 스코어)"],
  "fallback_used": true,
  "fallback_reason": "KOTRA_API_FAILED_AND_CACHE_MISS"
}
```

---

### REC-004: Blocked 국가 필터링

**시나리오**: 추천 결과에 제재 대상국이 포함되면 안 됨

**입력**:
```json
{
  "hs_code": "300490",
  "current_export_countries": [],
  "goal": "new_market",
  "top_n": 10
}
```

**조건**:
- KOTRA API: ✅ 성공 (KP, IR, US, CN, JP 포함 응답)
- Blocked 국가: KP, IR

**기대 결과**:
```json
{
  "recommendations": [
    {"country_code": "US"},
    {"country_code": "CN"},
    {"country_code": "JP"}
    // KP, IR은 포함되지 않음
  ],
  "filtered_countries": {
    "blocked": ["KP", "IR"],
    "reason": "수출 제재 대상국"
  }
}
```

**검증**:
- `recommendations`에 `KP`, `IR` country_code가 없어야 함
- `filtered_countries.blocked` 필드에 필터링된 국가 기록

---

### REC-005: Restricted 국가 경고 + 감점

**시나리오**: 제한국(RU)은 결과에 포함하되 경고와 감점 적용

**입력**:
```json
{
  "hs_code": "330499",
  "current_export_countries": [],
  "goal": "market_expansion",
  "top_n": 10
}
```

**조건**:
- KOTRA API: ✅ 성공 (RU 포함 응답)
- RU 원래 success_score: 15.0

**기대 결과**:
```json
{
  "recommendations": [
    {
      "country_code": "RU",
      "country_name": "러시아",
      "success_score": 15.0,
      "success_probability": 0.38,
      "compliance": {
        "status": "restricted",
        "warning": "수출 제한 대상국입니다. 수출 허가가 필요할 수 있습니다.",
        "penalty_applied": -10
      }
    }
  ]
}
```

**검증**:
- `compliance.status` == "restricted"
- `compliance.warning` 메시지 존재
- 감점으로 인해 success_probability가 낮아짐

---

### REC-006: 기존 수출국 제외 (NEW_MARKET 목표)

**시나리오**: new_market 목표 시 이미 수출 중인 국가 제외

**입력**:
```json
{
  "hs_code": "330499",
  "current_export_countries": ["US", "CN", "JP"],
  "goal": "new_market",
  "top_n": 5
}
```

**조건**:
- KOTRA API: ✅ 성공 (US, CN, JP, VN, DE, SG, TH 반환)

**기대 결과**:
```json
{
  "goal": "new_market",
  "recommendations": [
    {"country_code": "VN"},
    {"country_code": "DE"},
    {"country_code": "SG"},
    {"country_code": "TH"}
    // US, CN, JP는 제외됨
  ],
  "excluded_countries": ["US", "CN", "JP"],
  "exclusion_reason": "현재 수출 중인 국가 (new_market 목표)"
}
```

---

## 2. 매칭 (Matching) 테스트 케이스

### MAT-001: 정상 매칭 - FitScore 계산

**시나리오**: 셀러가 바이어를 찾을 때 FitScore 정상 계산

**입력**:
```json
{
  "profile_type": "seller",
  "profile": {
    "hs_code": "330499",
    "country": "KR",
    "price_range": [5.0, 8.0],
    "moq": 1000,
    "certifications": ["FDA", "ISO"]
  },
  "top_n": 10,
  "include_risk_analysis": true
}
```

**조건**:
- 바이어 DB: 50개 바이어 존재
- 무역사기 API: ✅ 성공

**기대 결과**:
```json
{
  "profile_type": "seller",
  "total_candidates": 50,
  "matches": [
    {
      "partner_id": "buyer_001",
      "fit_score": 92.5,
      "score_breakdown": {
        "base": 50,
        "hs_code_match": 20,
        "price_compatible": 15,
        "moq_compatible": 5,
        "certification_match": 10,
        "fraud_risk_penalty": 0,
        "success_case_bonus": 2.5
      }
    }
  ]
}
```

---

### MAT-002: MOQ Gate 실패 - 거래 불가

**시나리오**: 바이어 MOQ가 셀러 MOQ의 30% 미만일 때 매칭 탈락

**입력**:
```json
{
  "profile_type": "seller",
  "profile": {
    "hs_code": "330499",
    "country": "KR",
    "price_range": [5.0, 8.0],
    "moq": 5000,
    "certifications": ["FDA"]
  },
  "top_n": 10
}
```

**조건**:
- 바이어 buyer_035: moq=500 (셀러의 10%)

**기대 결과**:
- buyer_035는 matches에 **포함되지 않음**
- 또는 포함되더라도:
```json
{
  "partner_id": "buyer_035",
  "moq_gate_passed": false,
  "moq_gate_reason": "MOQ_BUYER_TOO_SMALL",
  "moq_ratio": 0.1,
  "excluded": true
}
```

---

### MAT-003: Required 인증 미충족 - 매칭 탈락

**시나리오**: 바이어가 FDA 필수 요구하나 셀러가 없을 때

**입력**:
```json
{
  "profile_type": "buyer",
  "profile": {
    "hs_code": "330499",
    "country": "US",
    "price_range": [6.0, 9.0],
    "moq": 1000,
    "required_certs": ["FDA"],
    "preferred_certs": ["ISO", "GMP"]
  },
  "top_n": 10
}
```

**조건**:
- seller_003: certifications=[] (FDA 없음)

**기대 결과**:
- seller_003는 matches에 **포함되지 않음**
- 탈락 기록:
```json
{
  "excluded_partners": [
    {
      "partner_id": "seller_003",
      "cert_gate_passed": false,
      "cert_gate_reason": "MISSING_REQUIRED_CERTS",
      "missing_required_certs": ["FDA"]
    }
  ]
}
```

---

### MAT-004: 성공사례 보너스 - 국가 불일치 (0점)

**시나리오**: 성공사례 국가가 바이어 국가와 다르면 보너스 0점

**입력**:
```json
{
  "profile_type": "seller",
  "profile": {
    "hs_code": "330499",
    "country": "KR",
    "price_range": [5.0, 8.0],
    "moq": 1000,
    "certifications": ["FDA", "ISO"]
  },
  "top_n": 5
}
```

**조건**:
- 바이어 buyer_001: country=US, hs_code=330499
- 성공사례: country=DE, hs_code=330499, date=2025-06-01

**기대 결과**:
```json
{
  "matches": [
    {
      "partner_id": "buyer_001",
      "country": "US",
      "score_breakdown": {
        "success_case_bonus": 0
      },
      "success_cases_detail": {
        "matched_cases_count": 0,
        "reference_only_cases": [
          {
            "case_id": "case_de_001",
            "reason": "COUNTRY_MISMATCH",
            "case_country": "DE",
            "target_country": "US"
          }
        ]
      }
    }
  ]
}
```

**검증**: 
- `success_case_bonus` == 0
- 국가 불일치 사례는 `reference_only_cases`에 기록

---

### MAT-005: Blocked 국가 바이어 제외

**시나리오**: 북한(KP) 바이어가 DB에 있어도 매칭 결과에서 제외

**입력**:
```json
{
  "profile_type": "seller",
  "profile": {
    "hs_code": "330499",
    "country": "KR",
    "price_range": [5.0, 8.0],
    "moq": 1000
  },
  "top_n": 20
}
```

**조건**:
- 바이어 DB에 country=KP인 바이어 존재 (테스트용)

**기대 결과**:
```json
{
  "matches": [
    // KP 국가 바이어 없음
  ],
  "filtered_partners": {
    "blocked_countries": ["KP"],
    "count": 1
  }
}
```

**검증**:
- matches 배열에 `country: "KP"` 객체가 없어야 함

---

### MAT-006: 무역사기 HIGH 리스크 국가 감점

**시나리오**: 사기 사례 20건 이상 국가는 -20점 감점

**입력**:
```json
{
  "profile_type": "seller",
  "profile": {
    "hs_code": "330499",
    "country": "KR",
    "price_range": [3.0, 6.0],
    "moq": 3000
  },
  "top_n": 10,
  "include_risk_analysis": true
}
```

**조건**:
- 바이어 buyer_011: country=CN
- CN 무역사기 API 응답: case_count=25

**기대 결과**:
```json
{
  "matches": [
    {
      "partner_id": "buyer_011",
      "country": "CN",
      "fit_score": 72.0,
      "score_breakdown": {
        "base": 50,
        "hs_code_match": 20,
        "price_compatible": 15,
        "moq_compatible": 5,
        "certification_match": 2,
        "fraud_risk_penalty": -20
      },
      "fraud_risk": {
        "risk_level": "HIGH",
        "case_count": 25,
        "score_penalty": -20
      }
    }
  ],
  "high_risk_countries": ["CN"]
}
```

---

## 3. 시뮬레이션 (Simulation) 테스트 케이스

### SIM-001: 정상 시뮬레이션 - High Confidence

**시나리오**: 모든 데이터 확보 시 높은 신뢰도로 결과 반환

**입력**:
```json
{
  "hs_code": "330499",
  "target_country": "US",
  "price_per_unit": 10.0,
  "moq": 1000,
  "annual_capacity": 50000,
  "include_news_risk": true
}
```

**조건**:
- KOTRA 수출유망추천정보: ✅ 성공
- KOTRA 국가정보: ✅ 성공 (GDP, growth_rate 포함)
- KOTRA 해외시장뉴스: ✅ 성공

**기대 결과**:
```json
{
  "target_country": "US",
  "success_probability": 0.72,
  "compliance": {
    "status": "normal"
  },
  "data_coverage": {
    "confidence": 1.0,
    "confidence_level": "high",
    "missing_rate": 0.0,
    "missing_fields": []
  }
}
```

---

### SIM-002: Blocked 국가 - 에러 응답

**시나리오**: 북한(KP) 시뮬레이션 요청 시 에러 반환

**입력**:
```json
{
  "hs_code": "330499",
  "target_country": "KP",
  "price_per_unit": 10.0,
  "moq": 1000,
  "annual_capacity": 50000
}
```

**조건**:
- KP는 BLOCKED_COUNTRIES 목록에 포함

**기대 결과**: HTTP 400 Bad Request
```json
{
  "error": true,
  "error_code": "BLOCKED_COUNTRY",
  "error_message": "시뮬레이션 불가: 대상 국가(KP)는 수출 제재 대상국입니다.",
  "target_country": "KP",
  "country_name": "북한",
  "compliance_status": "blocked"
}
```

---

### SIM-003: Restricted 국가 - 경고 + 감점

**시나리오**: 러시아(RU) 시뮬레이션 시 경고와 확률 감점

**입력**:
```json
{
  "hs_code": "300490",
  "target_country": "RU",
  "price_per_unit": 5.0,
  "moq": 2000,
  "annual_capacity": 100000
}
```

**조건**:
- RU는 RESTRICTED_COUNTRIES 목록에 포함
- 기본 success_probability: 0.45

**기대 결과**:
```json
{
  "target_country": "RU",
  "success_probability": 0.35,
  "compliance": {
    "status": "restricted",
    "warning": "러시아는 현재 수출 제한 대상국입니다.",
    "requires_export_license": true,
    "penalty_applied": -10
  },
  "warnings": [
    {
      "code": "RESTRICTED_COUNTRY",
      "severity": "high",
      "message": "대상 국가는 수출 제한 대상입니다."
    }
  ]
}
```

**검증**:
- `success_probability`가 감점 전보다 낮아야 함
- `compliance.status` == "restricted"

---

### SIM-004: Low Confidence - 데이터 결측 많음

**시나리오**: 3개 필드 결측 시 confidence=0.4로 경고

**입력**:
```json
{
  "hs_code": "300490",
  "target_country": "MM",
  "price_per_unit": 3.0,
  "moq": 5000,
  "annual_capacity": 200000,
  "include_news_risk": true
}
```

**조건**:
- KOTRA 수출유망추천정보: ✅ 성공
- KOTRA 국가정보: ⚠️ 부분 (GDP만, growth_rate 없음)
- KOTRA 해외시장뉴스: ❌ 실패 (timeout)
- 시장규모: ❌ 데이터 없음

**기대 결과**:
```json
{
  "target_country": "MM",
  "data_coverage": {
    "confidence": 0.4,
    "confidence_level": "low",
    "missing_rate": 0.6,
    "missing_fields": ["growth_rate", "news_risk", "market_size"],
    "available_data": {
      "export_score": true,
      "gdp": true,
      "growth_rate": false,
      "market_size": false,
      "news_risk": false
    },
    "confidence_warning": "데이터 결측률이 높아 시뮬레이션 결과의 신뢰도가 낮습니다."
  },
  "warnings": [
    {
      "code": "LOW_CONFIDENCE",
      "severity": "medium",
      "message": "데이터 결측으로 결과 신뢰도가 낮습니다."
    }
  ]
}
```

---

### SIM-005: Fallback - API 실패 + 캐시 사용

**시나리오**: KOTRA API 실패 시 캐시된 경제지표 사용

**입력**:
```json
{
  "hs_code": "330499",
  "target_country": "JP",
  "price_per_unit": 8.0,
  "moq": 800,
  "annual_capacity": 30000
}
```

**조건**:
- KOTRA 수출유망추천정보: ❌ 실패
- KOTRA 국가정보: ❌ 실패
- 캐시: ✅ hit (JP 경제지표 1시간 전 데이터)

**기대 결과**:
```json
{
  "target_country": "JP",
  "economic_indicators": {
    "gdp": 4230000000000,
    "growth_rate": 1.8,
    "risk_grade": "A",
    "source": "cache",
    "cached_at": "2026-01-26T09:30:00Z"
  },
  "data_coverage": {
    "confidence": 0.6,
    "confidence_level": "medium"
  },
  "data_sources": [
    "캐시 데이터 (수출유망추천정보)",
    "캐시 데이터 (국가정보)"
  ],
  "cache_info": {
    "used": true,
    "reason": "KOTRA_API_UNAVAILABLE"
  }
}
```

---

### SIM-006: Very Low Confidence - 대부분 결측

**시나리오**: 거의 모든 데이터 결측 시 very_low confidence 경고

**입력**:
```json
{
  "hs_code": "999999",
  "target_country": "XX",
  "price_per_unit": 10.0,
  "moq": 1000,
  "annual_capacity": 10000
}
```

**조건**:
- 존재하지 않는 HS코드
- 존재하지 않는 국가코드
- 모든 API: ❌ 실패
- 캐시: ❌ miss

**기대 결과**:
```json
{
  "target_country": "XX",
  "success_probability": 0.15,
  "data_coverage": {
    "confidence": 0.0,
    "confidence_level": "very_low",
    "missing_rate": 1.0,
    "missing_fields": ["export_score", "gdp", "growth_rate", "market_size", "news_risk"],
    "confidence_warning": "모든 데이터가 결측되어 결과를 신뢰할 수 없습니다."
  },
  "calculation_breakdown": {
    "note": "기본값만 사용됨"
  },
  "warnings": [
    {
      "code": "VERY_LOW_CONFIDENCE",
      "severity": "high",
      "message": "데이터가 거의 없어 결과를 신뢰할 수 없습니다. 다른 국가를 선택하세요."
    },
    {
      "code": "UNKNOWN_COUNTRY",
      "severity": "medium",
      "message": "알 수 없는 국가 코드입니다."
    }
  ],
  "data_sources": ["기본값 (모든 데이터)"]
}
```

---

## 테스트 케이스 요약표

| ID | 카테고리 | 시나리오 | 핵심 검증 포인트 |
|----|----------|----------|------------------|
| REC-001 | 추천 | 정상 흐름 | 추천 결과 반환 |
| REC-002 | 추천 | API 실패 → 캐시 | cache_info.hit == true |
| REC-003 | 추천 | API+캐시 실패 → 대체 | fallback_used == true |
| REC-004 | 추천 | Blocked 국가 필터 | KP, IR 미포함 |
| REC-005 | 추천 | Restricted 경고 | compliance.status == "restricted" |
| REC-006 | 추천 | 기존 수출국 제외 | current_export_countries 제외 |
| MAT-001 | 매칭 | 정상 FitScore | score_breakdown 포함 |
| MAT-002 | 매칭 | MOQ Gate 실패 | moq_gate_passed == false |
| MAT-003 | 매칭 | 필수 인증 미충족 | cert_gate_passed == false |
| MAT-004 | 매칭 | 성공사례 국가불일치 | success_case_bonus == 0 |
| MAT-005 | 매칭 | Blocked 국가 제외 | KP 바이어 미포함 |
| MAT-006 | 매칭 | 사기 HIGH 감점 | fraud_risk_penalty == -20 |
| SIM-001 | 시뮬 | 정상 High Confidence | confidence == 1.0 |
| SIM-002 | 시뮬 | Blocked 국가 에러 | HTTP 400, error_code |
| SIM-003 | 시뮬 | Restricted 감점 | penalty_applied == -10 |
| SIM-004 | 시뮬 | Low Confidence | confidence_level == "low" |
| SIM-005 | 시뮬 | API 실패 → 캐시 | cache_info.used == true |
| SIM-006 | 시뮬 | Very Low Confidence | confidence_level == "very_low" |

---

## 테스트 실행 권장 순서

1. **게이트 테스트 우선**: MAT-002, MAT-003, SIM-002 (Hard fail 케이스)
2. **핵심 필터링**: REC-004, MAT-005 (Blocked 국가)
3. **경고/감점**: REC-005, SIM-003, MAT-006 (Restricted, HIGH 리스크)
4. **정상 흐름**: REC-001, MAT-001, SIM-001
5. **Fallback**: REC-002, REC-003, SIM-005
6. **Confidence**: SIM-004, SIM-006
7. **세부 규칙**: MAT-004, REC-006
