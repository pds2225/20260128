# SIMULATION_SPEC.md - 시뮬레이션 서비스 명세서

> **문서 버전**: v1.0  
> **최종 수정**: 2026-01-26  
> **작성 목적**: simulation_service의 compliance + confidence 반영 규칙 확정

---

## 1. 개요

`simulation_service`는 수출 성과 시뮬레이션 결과를 제공하며, 다음 두 가지 핵심 요소를 반영해야 한다:

1. **Compliance (준수)**: 제재국/규제국 필터링
2. **Confidence (신뢰도)**: 데이터 결측률 기반 결과 신뢰도

---

## 2. Compliance (제재국/규제국 처리)

### 2.1 국가 분류

| 분류 | 처리 방식 | 예시 국가 |
|------|-----------|-----------|
| `blocked` | **거래 불가** - 결과 반환 금지 | 북한(KP), 이란(IR), 시리아(SY), 쿠바(CU) |
| `restricted` | **경고 + 감점** - 결과 반환하되 경고 포함 | 러시아(RU), 벨라루스(BY), 미얀마(MM), 베네수엘라(VE) |
| `normal` | **정상 처리** | 그 외 모든 국가 |

### 2.2 Blocked 국가 처리 방안

**선택된 방안**: `error` 응답 반환 (HTTP 400)

#### 결정 근거
- 제재국 거래는 **법적 리스크**가 크므로 명확한 거부 필요
- 빈 결과나 경고보다 **즉각적 피드백**이 사용자 혼란 방지
- API 소비자가 프로그래밍적으로 처리하기 용이

#### 응답 구조

```json
{
  "error": true,
  "error_code": "BLOCKED_COUNTRY",
  "error_message": "시뮬레이션 불가: 대상 국가(KP)는 수출 제재 대상국입니다.",
  "target_country": "KP",
  "country_name": "북한",
  "compliance_status": "blocked",
  "legal_notice": "해당 국가로의 수출은 대한민국 대외무역법 및 국제 제재 규정에 따라 금지됩니다.",
  "reference_url": "https://www.customs.go.kr/kcs/ad/cdm/exportrestriction.do"
}
```

### 2.3 Restricted 국가 처리

**결과는 반환하되** 다음을 포함:
1. `compliance_warning` 메시지
2. `risk_adjustment`에 추가 감점 (-10점)
3. `requires_export_license` 플래그

#### 감점 수식

```python
if compliance_status == "restricted":
    compliance_penalty = -10
    success_probability = max(0.05, success_probability - 0.10)
```

---

## 3. Confidence (결과 신뢰도)

### 3.1 Confidence 산식

```python
def calculate_confidence(data_availability: dict) -> tuple[float, str]:
    """
    결측률 기반 confidence 계산
    
    Returns:
        confidence: 0.0 ~ 1.0
        confidence_level: "high" | "medium" | "low" | "very_low"
    """
    required_fields = [
        "export_score",      # 수출유망추천정보
        "gdp",               # 국가정보
        "growth_rate",       # 국가정보
        "market_size",       # 시장규모
        "news_risk"          # 뉴스 리스크
    ]
    
    available_count = sum(1 for f in required_fields if data_availability.get(f))
    total_count = len(required_fields)
    
    missing_rate = (total_count - available_count) / total_count
    confidence = 1.0 - missing_rate
    
    # Confidence Level 결정
    if confidence >= 0.8:
        level = "high"
    elif confidence >= 0.6:
        level = "medium"
    elif confidence >= 0.4:
        level = "low"
    else:
        level = "very_low"
    
    return round(confidence, 2), level
```

### 3.2 Confidence Level 기준

| confidence 값 | level | 설명 | UI 표시 권장 |
|---------------|-------|------|-------------|
| 0.80 ~ 1.00 | `high` | 모든 주요 데이터 확보 | ✅ 신뢰할 수 있음 |
| 0.60 ~ 0.79 | `medium` | 일부 데이터 결측 | ⚠️ 참고용으로 활용 |
| 0.40 ~ 0.59 | `low` | 상당 부분 결측 | ⚠️ 주의 필요 |
| 0.00 ~ 0.39 | `very_low` | 대부분 결측 | ❌ 결과 신뢰 불가 |

### 3.3 Data Coverage 구조

```json
{
  "data_coverage": {
    "confidence": 0.6,
    "confidence_level": "medium",
    "missing_rate": 0.4,
    "total_fields": 5,
    "available_fields": 3,
    "missing_fields": ["growth_rate", "news_risk"],
    "available_data": {
      "export_score": true,
      "gdp": true,
      "growth_rate": false,
      "market_size": true,
      "news_risk": false
    },
    "data_source_status": {
      "kotra_export_recommend": "success",
      "kotra_country_info": "partial",
      "kotra_overseas_news": "failed"
    }
  }
}
```

---

## 4. 입력/출력 JSON 예시

### 4.1 예시 A: Blocked 국가인 경우

#### 입력

```json
{
  "hs_code": "330499",
  "target_country": "KP",
  "price_per_unit": 10.0,
  "moq": 1000,
  "annual_capacity": 50000,
  "include_news_risk": true
}
```

#### 출력 (HTTP 400 Bad Request)

```json
{
  "error": true,
  "error_code": "BLOCKED_COUNTRY",
  "error_message": "시뮬레이션 불가: 대상 국가(KP)는 수출 제재 대상국입니다.",
  "target_country": "KP",
  "country_name": "북한",
  "compliance_status": "blocked",
  "legal_notice": "해당 국가로의 수출은 대한민국 대외무역법 및 국제 제재 규정에 따라 금지됩니다.",
  "reference_url": "https://www.customs.go.kr/kcs/ad/cdm/exportrestriction.do",
  "blocked_countries_list": ["KP", "IR", "SY", "CU"],
  "timestamp": "2026-01-26T10:30:00Z"
}
```

---

### 4.2 예시 B: 데이터 결측이 큰 경우 (Restricted 국가 + Low Confidence)

#### 입력

```json
{
  "hs_code": "300490",
  "target_country": "RU",
  "price_per_unit": 5.0,
  "moq": 2000,
  "annual_capacity": 100000,
  "include_news_risk": true
}
```

#### 출력 (HTTP 200 OK with warnings)

```json
{
  "target_country": "RU",
  "country_name": "러시아",
  "hs_code": "300490",
  "estimated_revenue_min": 15000.0,
  "estimated_revenue_max": 45000.0,
  "success_probability": 0.25,
  "market_size": 50000000000,
  "market_share_min": 0.00003,
  "market_share_max": 0.00009,
  
  "compliance": {
    "status": "restricted",
    "warning": "러시아는 현재 수출 제한 대상국입니다. 수출 허가가 필요할 수 있습니다.",
    "requires_export_license": true,
    "penalty_applied": -10,
    "restricted_since": "2022-03-01",
    "reference_url": "https://www.motie.go.kr/",
    "recommendations": [
      "수출 전 전략물자관리원(KOSTI) 사전 상담 권장",
      "결제 조건 및 물류 경로 사전 확인 필수"
    ]
  },
  
  "data_coverage": {
    "confidence": 0.4,
    "confidence_level": "low",
    "missing_rate": 0.6,
    "total_fields": 5,
    "available_fields": 2,
    "missing_fields": ["growth_rate", "news_risk", "market_size"],
    "available_data": {
      "export_score": true,
      "gdp": true,
      "growth_rate": false,
      "market_size": false,
      "news_risk": false
    },
    "data_source_status": {
      "kotra_export_recommend": "success",
      "kotra_country_info": "partial",
      "kotra_overseas_news": "timeout"
    },
    "confidence_warning": "데이터 결측률이 높아 시뮬레이션 결과의 신뢰도가 낮습니다. 참고용으로만 활용하세요."
  },
  
  "economic_indicators": {
    "gdp": 1775000000000,
    "growth_rate": null,
    "risk_grade": "D"
  },
  
  "news_risk_adjustment": null,
  
  "calculation_breakdown": {
    "base_probability": 0.30,
    "components": {
      "export_ml": {
        "raw_score": 5.5,
        "normalized": 0.183,
        "contribution": 0.073
      },
      "economic": {
        "factor": 0.3,
        "contribution": 0.075
      },
      "news_sentiment": {
        "factor": 0.5,
        "contribution": 0.1,
        "note": "뉴스 데이터 없음 - 기본값 사용"
      },
      "trends": {
        "factor": 0.3,
        "contribution": 0.045
      }
    },
    "compliance_penalty": -0.10,
    "pre_penalty_probability": 0.35,
    "final_probability": 0.25
  },
  
  "warnings": [
    {
      "code": "RESTRICTED_COUNTRY",
      "severity": "high",
      "message": "대상 국가는 수출 제한 대상입니다."
    },
    {
      "code": "LOW_CONFIDENCE",
      "severity": "medium",
      "message": "데이터 결측으로 결과 신뢰도가 낮습니다."
    },
    {
      "code": "MISSING_NEWS_DATA",
      "severity": "low",
      "message": "뉴스 기반 리스크 분석을 수행할 수 없습니다."
    }
  ],
  
  "data_sources": [
    "KOTRA 수출유망추천정보",
    "KOTRA 국가정보 (부분)",
    "기본값 (시장규모, 뉴스)"
  ],
  
  "generated_at": "2026-01-26T10:35:00Z"
}
```

---

### 4.3 예시 C: 정상 케이스 (High Confidence)

#### 입력

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

#### 출력 (HTTP 200 OK)

```json
{
  "target_country": "US",
  "country_name": "미국",
  "hs_code": "330499",
  "estimated_revenue_min": 120000.0,
  "estimated_revenue_max": 380000.0,
  "success_probability": 0.72,
  "market_size": 203680000000,
  "market_share_min": 0.00006,
  "market_share_max": 0.00019,
  
  "compliance": {
    "status": "normal",
    "warning": null,
    "requires_export_license": false
  },
  
  "data_coverage": {
    "confidence": 1.0,
    "confidence_level": "high",
    "missing_rate": 0.0,
    "total_fields": 5,
    "available_fields": 5,
    "missing_fields": [],
    "available_data": {
      "export_score": true,
      "gdp": true,
      "growth_rate": true,
      "market_size": true,
      "news_risk": true
    },
    "data_source_status": {
      "kotra_export_recommend": "success",
      "kotra_country_info": "success",
      "kotra_overseas_news": "success"
    }
  },
  
  "economic_indicators": {
    "gdp": 25460000000000,
    "growth_rate": 2.5,
    "risk_grade": "A"
  },
  
  "news_risk_adjustment": {
    "risk_adjustment": 5.2,
    "negative_count": 3,
    "positive_count": 12,
    "total_analyzed": 50
  },
  
  "calculation_breakdown": {
    "base_probability": 0.30,
    "components": {
      "export_ml": {
        "raw_score": 18.5,
        "normalized": 0.617,
        "contribution": 0.247
      },
      "economic": {
        "factor": 0.8,
        "contribution": 0.2
      },
      "news_sentiment": {
        "factor": 0.67,
        "contribution": 0.134
      },
      "trends": {
        "factor": 0.7,
        "contribution": 0.105
      }
    },
    "compliance_penalty": 0,
    "final_probability": 0.72
  },
  
  "warnings": [],
  
  "data_sources": [
    "KOTRA 수출유망추천정보",
    "KOTRA 국가정보",
    "KOTRA 해외시장뉴스"
  ],
  
  "generated_at": "2026-01-26T10:40:00Z"
}
```

---

## 5. 구현 참고사항

### 5.1 Blocked 국가 목록 상수

```python
BLOCKED_COUNTRIES = {
    "KP": {"name_kr": "북한", "name_en": "North Korea"},
    "IR": {"name_kr": "이란", "name_en": "Iran"},
    "SY": {"name_kr": "시리아", "name_en": "Syria"},
    "CU": {"name_kr": "쿠바", "name_en": "Cuba"},
}

RESTRICTED_COUNTRIES = {
    "RU": {"name_kr": "러시아", "name_en": "Russia", "since": "2022-03-01"},
    "BY": {"name_kr": "벨라루스", "name_en": "Belarus", "since": "2022-03-01"},
    "MM": {"name_kr": "미얀마", "name_en": "Myanmar", "since": "2021-02-01"},
    "VE": {"name_kr": "베네수엘라", "name_en": "Venezuela", "since": "2019-01-01"},
}
```

### 5.2 Confidence 계산 함수

```python
def build_data_coverage(data_results: dict) -> dict:
    """데이터 커버리지 및 confidence 계산"""
    
    field_checks = {
        "export_score": data_results.get("export_rec") and len(data_results["export_rec"]) > 0,
        "gdp": data_results.get("country_info", {}).get("gdp") is not None,
        "growth_rate": data_results.get("country_info", {}).get("growth_rate") is not None,
        "market_size": data_results.get("market_size") is not None,
        "news_risk": data_results.get("news_risk") is not None,
    }
    
    available_count = sum(field_checks.values())
    total_count = len(field_checks)
    missing_rate = (total_count - available_count) / total_count
    confidence = round(1.0 - missing_rate, 2)
    
    # Level 결정
    if confidence >= 0.8:
        level = "high"
    elif confidence >= 0.6:
        level = "medium"
    elif confidence >= 0.4:
        level = "low"
    else:
        level = "very_low"
    
    return {
        "confidence": confidence,
        "confidence_level": level,
        "missing_rate": round(missing_rate, 2),
        "total_fields": total_count,
        "available_fields": available_count,
        "missing_fields": [k for k, v in field_checks.items() if not v],
        "available_data": field_checks,
    }
```

### 5.3 Compliance 체크 함수

```python
from fastapi import HTTPException

def check_compliance(country_code: str) -> dict:
    """국가 컴플라이언스 체크"""
    
    country_upper = country_code.upper()
    
    if country_upper in BLOCKED_COUNTRIES:
        info = BLOCKED_COUNTRIES[country_upper]
        raise HTTPException(
            status_code=400,
            detail={
                "error": True,
                "error_code": "BLOCKED_COUNTRY",
                "error_message": f"시뮬레이션 불가: 대상 국가({country_upper})는 수출 제재 대상국입니다.",
                "target_country": country_upper,
                "country_name": info["name_kr"],
                "compliance_status": "blocked",
            }
        )
    
    if country_upper in RESTRICTED_COUNTRIES:
        info = RESTRICTED_COUNTRIES[country_upper]
        return {
            "status": "restricted",
            "warning": f"{info['name_kr']}은(는) 현재 수출 제한 대상국입니다.",
            "requires_export_license": True,
            "penalty_applied": -10,
            "restricted_since": info["since"],
        }
    
    return {
        "status": "normal",
        "warning": None,
        "requires_export_license": False,
    }
```

---

## 6. 필드명 요약

| 카테고리 | 필드명 | 타입 | 설명 |
|----------|--------|------|------|
| Compliance | `compliance.status` | string | "blocked" / "restricted" / "normal" |
| Compliance | `compliance.warning` | string | 경고 메시지 |
| Compliance | `compliance.requires_export_license` | bool | 수출 허가 필요 여부 |
| Compliance | `compliance.penalty_applied` | int | 적용된 감점 |
| Confidence | `data_coverage.confidence` | float | 0.0 ~ 1.0 |
| Confidence | `data_coverage.confidence_level` | string | "high" / "medium" / "low" / "very_low" |
| Confidence | `data_coverage.missing_rate` | float | 0.0 ~ 1.0 |
| Confidence | `data_coverage.missing_fields` | list | 결측 필드명 목록 |
| Confidence | `data_coverage.available_data` | object | 필드별 가용 여부 |
| Confidence | `data_coverage.data_source_status` | object | 데이터 소스별 상태 |
| Warning | `warnings[]` | array | 경고 목록 |
| Warning | `warnings[].code` | string | 경고 코드 |
| Warning | `warnings[].severity` | string | "high" / "medium" / "low" |
| Warning | `warnings[].message` | string | 경고 메시지 |
