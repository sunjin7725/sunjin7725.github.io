---
title: LLM 구조화된 출력 이해하기 — JSON 형식부터 값 검증까지
description: 문서에서 날짜와 금액을 추출하는 예제로 JSON 출력, JSON Schema, 업무 규칙 검증과 재시도 설계를 단계별로 정리한다.
categories: [LLM]
tags: [llm, structured-output, json-schema, ollama, validation]
comments: true
toc: true
mermaid: true
---

## JSON으로 받았다고 바로 저장해도 될까?

LLM을 업무 프로그램에 연결하면 자연어 답변보다 구조화된 값이 필요한 경우가 많다. 예를 들어 문서에서 거래처명, 작성일, 합계 금액을 추출한 뒤 데이터베이스에 저장하려면 다음과 같은 결과가 편리하다.

```json
{
  "vendor_name": "가나다상사",
  "document_date": "2026-09-17",
  "total_amount": 1250000
}
```

하지만 JSON 파싱에 성공했다는 사실은 값이 정확하다는 뜻이 아니다. 모델이 원문에 없는 금액을 만들었더라도 문법은 완벽한 JSON일 수 있다.

앞선 [Workflow와 Agent 글](/posts/workflow-and-agent/)에서는 문서 조회, 정보 추출, 검증, 저장을 분리했다. 이번 글에서는 그중 **구조화된 출력과 검증**을 자세히 살펴본다. 예제는 구조를 설명하기 위한 것이며 특정 문서나 모델의 실측 결과는 아니다.

## 1. 검증해야 할 세 개의 층

LLM 출력은 최소한 세 단계로 나눠 확인하는 편이 좋다.

| 검증 단계 | 확인하는 질문 | 잡을 수 있는 문제 |
|---|---|---|
| JSON 문법 | 문자열을 JSON으로 파싱할 수 있는가? | 작은따옴표, 쉼표 누락, 코드 블록 혼입 |
| 스키마 | 필요한 필드와 자료형을 지켰는가? | 필드 누락, 문자열로 반환된 금액, 예상하지 않은 필드 |
| 업무 규칙과 원문 | 값이 업무 조건과 근거에 맞는가? | 미래 날짜, 음수 금액, 원문에 없는 값 |

예를 들어 아래 응답은 JSON 문법과 간단한 자료형 검사를 통과할 수 있다.

```json
{
  "vendor_name": "가나다상사",
  "document_date": "2026-09-17",
  "total_amount": 9800000
}
```

원문 금액이 `1,250,000원`이었다면 업무 결과로는 오답이다. **올바른 JSON과 올바른 데이터는 별개의 문제**다.

## 2. 프롬프트만으로 JSON을 요청할 때

가장 간단한 방법은 출력 형식을 프롬프트에 적는 것이다.

```text
다음 문서에서 거래처명, 작성일, 합계 금액을 추출하세요.
설명이나 Markdown 없이 아래 키를 가진 JSON 객체만 반환하세요.

- vendor_name: 문자열 또는 null
- document_date: YYYY-MM-DD 문자열 또는 null
- total_amount: 정수 또는 null

원문에 값이 없거나 확정할 수 없으면 추측하지 말고 null을 반환하세요.
```

작은 실험이나 구조화된 출력을 지원하지 않는 환경에서는 이 방식으로 시작할 수 있다. 다만 프롬프트는 요청이지 프로그램 수준의 제약은 아니다. 모델이나 입력에 따라 설명 문장, Markdown 코드 블록, 다른 필드가 섞일 수 있다.

응답에서 중괄호 부분만 정규식으로 잘라내는 처리는 오류를 감출 수 있다. 앞뒤에 불필요한 설명이 붙은 것인지, 문자열 안에 중괄호가 들어간 것인지 구분하기 어렵기 때문이다. 먼저 JSON 파싱을 시도하고 실패를 명시적으로 처리하는 편이 원인을 추적하기 쉽다.

## 3. JSON Schema로 출력 계약 만들기

구조화된 출력을 지원하는 추론 API에서는 JSON Schema를 전달해 응답 구조를 제한할 수 있다. Ollama의 Structured Outputs도 `format` 필드에 JSON Schema를 받을 수 있다. [Ollama Structured Outputs 문서](https://docs.ollama.com/capabilities/structured-outputs)

문서 추출 결과의 스키마를 다음처럼 정의해보자.

```json
{
  "type": "object",
  "properties": {
    "vendor_name": {
      "type": ["string", "null"]
    },
    "document_date": {
      "type": ["string", "null"],
      "format": "date"
    },
    "total_amount": {
      "type": ["integer", "null"],
      "minimum": 0
    }
  },
  "required": ["vendor_name", "document_date", "total_amount"],
  "additionalProperties": false
}
```

각 키워드가 맡는 역할은 다음과 같다.

| 키워드 | 역할 |
|---|---|
| `properties` | 허용할 필드와 각 필드의 스키마 정의 |
| `required` | 반드시 존재해야 하는 필드 지정 |
| `additionalProperties: false` | 정의하지 않은 필드 거부 |
| `type: ["string", "null"]` | 값이 없을 때 `null` 허용 |
| `format: "date"` | 날짜 형식에 대한 의미 부여 |

JSON Schema에서는 `properties`에 필드를 적는 것만으로 필수 항목이 되지 않는다. `required`를 별도로 지정해야 한다. 정의하지 않은 필드도 기본적으로 허용되므로 필요하면 `additionalProperties` 정책을 명시한다. [JSON Schema 객체 문서](https://json-schema.org/understanding-json-schema/reference/object)

`format` 검증 방식은 사용하는 검증기와 설정에 따라 다를 수 있다. `format: "date"`를 선언했다고 가정하지 말고 실제 검증 라이브러리가 이를 강제하는지 확인한다.

## 4. Ollama에서 스키마를 전달하는 예제

아래 예제는 Pydantic 모델에서 JSON Schema를 만들고, 같은 모델로 응답을 다시 검증한다. Ollama Python 라이브러리와 Pydantic이 설치되어 있고 지정한 모델이 로컬에 준비된 환경을 가정한다.

```python
from datetime import date
from typing import Optional

from ollama import chat
from pydantic import BaseModel, ConfigDict, Field


class DocumentFields(BaseModel):
    model_config = ConfigDict(extra="forbid")

    vendor_name: Optional[str]
    document_date: Optional[date]
    total_amount: Optional[int] = Field(ge=0)


source_text = """
거래명세서
공급자: 가나다상사
작성일: 2026년 9월 17일
합계: 1,250,000원
"""

response = chat(
    model="gpt-oss",
    messages=[{
        "role": "user",
        "content": (
            "문서에서 거래처명, 작성일, 합계 금액을 추출하세요. "
            "값을 확인할 수 없으면 추측하지 말고 null을 반환하세요.\n\n"
            f"{source_text}"
        ),
    }],
    format=DocumentFields.model_json_schema(),
    options={"temperature": 0},
)

result = DocumentFields.model_validate_json(response.message.content)
print(result)
```

Pydantic 모델 하나를 스키마 생성과 응답 검증에 같이 사용하면 두 정의가 어긋날 가능성을 줄일 수 있다. Ollama 공식 예제도 `model_json_schema()`와 `model_validate_json()`을 사용하는 방식을 안내한다.

`temperature=0`은 출력을 더 일관되게 만들기 위한 설정이다. 동일한 응답을 항상 보장하거나 값의 정확성을 검증하는 기능은 아니다. 또한 구조화된 출력의 지원 범위는 사용하는 Ollama 배포 방식과 버전, 모델을 확인해야 한다.

## 5. `null`, 빈 문자열, 기본값을 구분하자

원문에 값이 없을 때 모델이 임의로 채우지 않도록 출력 규칙을 정해야 한다.

```json
{
  "vendor_name": null,
  "document_date": null,
  "total_amount": null
}
```

`null`은 확인하지 못한 값으로 사용할 수 있다. 빈 문자열 `""`, 숫자 `0`, 오늘 날짜를 기본값으로 넣으면 실제 값과 미확인 값을 구분하기 어려워진다.

필드가 응답에 반드시 있어야 한다는 것과 값이 반드시 존재해야 한다는 것도 다르다. `required`에 필드를 넣고 자료형에 `null`을 허용하면, 모델은 키를 생략하지 않으면서도 미확인 상태를 표현할 수 있다.

업무에 따라 상태를 더 구체적으로 나눌 수도 있다.

```json
{
  "total_amount": null,
  "total_amount_status": "not_found"
}
```

다만 상태 종류를 필요 이상으로 늘리면 모델이 비슷한 상태를 구분하기 어려워진다. 후속 처리에 실제로 필요한 구분만 둔다.

## 6. 값의 근거를 함께 받기

추출값을 원문과 비교하려면 근거 문자열을 함께 반환하도록 설계할 수 있다.

```json
{
  "total_amount": 1250000,
  "total_amount_evidence": "합계: 1,250,000원"
}
```

근거 필드는 검토와 오류 분석에 도움이 된다. 그러나 모델이 근거 문자열까지 만들어낼 수 있으므로, 근거가 실제 원문에 존재하는지도 프로그램에서 확인해야 한다.

문서의 좌표나 페이지 번호가 필요하다면 OCR·문서 파서가 제공한 식별자를 유지하는 방식이 더 안정적일 수 있다. 모델에게 페이지 번호를 추측하게 하지 않고, 입력 청크에 부여한 ID를 선택하도록 할 수 있다.

## 7. 스키마 뒤에 업무 검증을 추가하기

스키마를 통과한 결과에도 업무 검증이 필요하다.

```python
from datetime import date


def validate_business_rules(result: DocumentFields, source_text: str) -> list[str]:
    errors = []

    if result.document_date and result.document_date > date.today():
        errors.append("document_date가 오늘보다 미래입니다.")

    if result.total_amount is not None:
        amount_text = f"{result.total_amount:,}"
        if amount_text not in source_text:
            errors.append("total_amount의 원문 근거를 확인할 수 없습니다.")

    return errors
```

이 코드는 검증 층을 보여주기 위한 단순 예제다. 원문이 `1,250,000`, `1250000`, `1 250 000`처럼 여러 형식을 사용할 수 있고 OCR 오인식도 생길 수 있으므로 실제 업무에서는 금액 정규화 규칙을 별도로 설계해야 한다.

대표적인 업무 검증 항목은 다음과 같다.

- 날짜가 허용된 기간에 속하는가?
- 금액의 부호와 범위가 업무 규칙에 맞는가?
- 합계가 세부 항목의 계산 결과와 일치하는가?
- 문서 식별자가 이미 처리된 값은 아닌가?
- 추출값 또는 근거가 실제 원문에서 확인되는가?

정확성 검증이 어려운 필드는 자동 저장 대신 검토 대상으로 보낼 수 있다. 모든 오류를 모델 재호출로 해결하려고 하면 같은 오답을 반복하거나 근거 없는 값을 새로 만들 수 있다.

## 8. 실패 종류에 따라 재시도 방법을 바꾸기

재시도는 실패 원인에 맞춰야 한다.

| 실패 유형 | 처리 방향 |
|---|---|
| 일시적인 통신 오류 | 제한된 횟수로 지수 백오프 후 재시도 |
| JSON 파싱·스키마 오류 | 검증 오류와 스키마를 전달해 형식 수정 요청 |
| 필수 정보가 원문에 없음 | 재시도보다 `null` 또는 검토 상태로 종료 |
| 업무 규칙 위반 | 원문 근거를 다시 확인하거나 사람 검토로 전환 |
| 같은 오류 반복 | 호출 한도에서 중단하고 실패 원인 기록 |

모델에 오류를 돌려줄 때는 전체 내부 로그보다 수정에 필요한 항목을 명확히 전달한다.

```text
total_amount는 0 이상의 정수 또는 null이어야 합니다.
원문에 금액이 없다면 null을 반환하세요.
다른 필드는 이전 결과를 유지하세요.
```

재시도 전후의 입력, 모델 버전, 검증 오류, 최종 상태를 기록하면 어떤 문서에서 반복적으로 실패하는지 분석할 수 있다. 개인정보나 민감한 원문은 로그 보관 정책에 맞춰 마스킹한다.

## 9. 실제 평가에서는 형식과 정확도를 따로 측정한다

구조화된 출력 기능을 비교할 때 하나의 성공률로 합치면 원인을 파악하기 어렵다.

| 평가 항목 | 계산 예시 |
|---|---|
| JSON 파싱 성공률 | 파싱 성공 건수 ÷ 전체 건수 |
| 스키마 준수율 | 스키마 통과 건수 ÷ 전체 건수 |
| 필드 정확도 | 정답과 일치한 필드 수 ÷ 평가 대상 필드 수 |
| 문서 완전 일치율 | 모든 필드가 맞은 문서 수 ÷ 전체 문서 수 |
| 미확인 판단 정확도 | 근거 없는 값을 만들지 않고 `null` 처리한 비율 |

스키마 준수율이 높고 필드 정확도가 낮다면 출력 형식보다 추출 자체를 개선해야 한다. OCR 품질, 입력 문맥, 필드 정의, 모델 선택을 살펴볼 수 있다.

평가 데이터에는 정상 문서뿐 아니라 값이 누락된 문서, 여러 금액이 있는 문서, 정정 문서, OCR 오류가 있는 문서를 포함한다. 정상 예제만으로는 추측하지 않고 `null`을 반환하는지 확인하기 어렵다.

## 저장 전에 검증 경계를 세우기

구조화된 출력은 LLM의 답변을 프로그램이 다루기 쉬운 데이터로 바꾸는 데 유용하다. 하지만 JSON Schema가 보장하는 것은 주로 **형태**다. 원문과 값의 일치, 업무 규칙, 저장 권한까지 대신 보장하지는 않는다.

실무에서는 `모델 출력 → 파싱 → 스키마 검증 → 업무 검증 → 저장 또는 검토`의 경계를 분명히 두는 것이 좋다. 모델이 JSON을 잘 반환하게 만드는 것보다 중요한 목표는, **검증하지 않은 값을 다음 시스템으로 넘기지 않는 것**이다.
