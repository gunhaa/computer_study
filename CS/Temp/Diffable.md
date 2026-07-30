# Diffable

## 1. 배경

Portal은 Kong을 관리하기 위한 내부 Portal 서비스로, Portal에서 관리하는 데이터를 기반으로 Kong의 리소스를 생성·변경하는 구조였습니다.

기존에는 Portal에서 사용자가 리소스를 변경할 때 Kong API 호출이 실패하더라도 Portal에서는 성공으로 처리되는 문제가 존재했습니다.

이로 인해 다음과 같이 두 시스템 간 데이터 정합성이 깨지는 상황이 발생할 수 있었습니다.

```text
Portal
  │
  │ 변경 요청
  ▼
Kong API
  │
  ├── 성공 → Portal / Kong 정합성 일치
  │
  └── 실패 → Portal 성공 처리
              ↓
          정합성 불일치
```

실제 운영 과정에서 Portal과 Kong의 데이터 정합성이 깨지는 문제가 발생했고, 이후 두 시스템의 데이터를 수동으로 비교하고 복구해야 하는 상황이 발생했습니다.

또한, 기능이 수정되며 생기는 버그로 인해 잘못된 데이터가 생성되는 경우도 있었습니다.

이에 따라 **Portal과 Kong의 현재 상태를 비교하여 정합성 여부와 차이점을 확인할 수 있는 API**를 제안하고 구현했습니다.

---

## 2. 목표

초기에는 특정 도메인의 차이점만 확인하는 API를 만드는 것이 목적이었습니다.

하지만 실제 시스템 구조를 확인한 결과, Portal과 Kong 사이에서 비교해야 할 도메인이 많았기 때문에 특정 API에만 적용하는 방식보다 **공통적인 비교 구조를 정의하고 여러 도메인에 적용하는 것이 적절하다고 판단했습니다.**

최종적으로 약 10개 정도의 도메인에 대해 정합성을 확인할 수 있는 API를 구현했습니다.

목표는 다음과 같습니다.

- Portal과 Kong의 정합성 확인
- Portal에만 존재하는 데이터 확인
- Kong에만 존재하는 데이터 확인
- 양쪽에 존재하지만 값이 다른 데이터 확인
- 운영팀이 API를 통해 현재 정합성 상태를 직접 확인
- 이후 ADMIN이나 Batch 등에서 활용할 수 있는 기반 제공
- 도메인이 추가되더라도 동일한 방식으로 비교할 수 있는 구조 구성
- **Portal과 Kong의 데이터 모델 차이와 도메인 간 매핑 관계를 문서화하여, 제3자도 각 데이터의 대응 관계와 비교 기준을 명확하게 이해할 수 있도록 함**

---

# 3. 문제: Portal과 Kong의 데이터 모델 차이

Portal과 Kong은 동일한 데이터를 관리하지만, 데이터 모델과 저장 형태가 동일하지 않았습니다.

예를 들어 Portal의 여러 컬럼을 조합하여 Kong의 리소스 이름을 생성하는 경우처럼, 단순히 두 Entity를 직접 비교할 수 없는 경우가 존재했습니다.

따라서 다음과 같은 변환 계층이 필요했습니다.

```text
Portal Domain
     │
     ▼
ComparisonService(정규화)
     │
     ▼
XXCompareDto.fromPortal(portalDomain)

Kong Domain
     │
     ▼
ComparisonService(정규화)
     │
     ▼
XXCompareDto.fromKong(kongDomain)
```

`CompareDto`는 단순한 API Response DTO가 아니라, **Portal과 Kong의 서로 다른 데이터 모델을 비교하기 위한 중간 모델** 역할을 담당했습니다.

각 비교 대상에 맞는 Factory 생성 메서드를 통해 비교에 필요한 데이터를 구성하고, 필요한 경우 기존 Domain이나 DTO의 메서드를 활용하여 비교 가능한 형태로 변환했습니다.

---

# 4. Diffable 도입

도메인마다 Portal과 Kong의 매핑 방식과 비교 기준이 달랐기 때문에, 각 도메인별 비교 규칙을 하나의 공통 계약으로 묶기 위해 `Diffable` 인터페이스를 정의했습니다.

```text
XXCompareDto
       │
       │ implements
       ▼
    Diffable
```

`Diffable`은 비교 대상이 가져야 하는 최소한의 비교 규칙을 정의합니다.

주요 역할은 다음과 같습니다.

- 비교 대상의 식별 기준 제공
- Portal과 Kong 데이터 간 차이점 비교
- 도메인별 비교 규칙을 각 `CompareDto`에 응집

따라서 각 도메인은 자신의 데이터 구조와 비교 규칙을 직접 관리하면서도, 외부에서는 공통된 `Diffable` 계약을 통해 처리할 수 있도록 구성했습니다.

```text
e.g.
ConsumerCompareDto
       │
       └── Diffable

ServiceCompareDto
       │
       └── Diffable

AclCompareDto
       │
       └── Diffable
```

---

# 5. ComparisonResult.of(portal, kong)

Portal과 Kong의 데이터가 각각 `Diffable`을 구현한 `CompareDto`로 변환되면, `ComparisonResult.of(portal, kong)`를 통해 두 대상을 비교합니다.

전체적인 흐름은 다음과 같습니다.

```text
Portal 원본 데이터
       │
       ▼
ComparisonService(정규화)
       │
       ▼
Portal CompareDto
       │
       │ implements Diffable
       │
       ├──────────────┐
       │              │
       │              ▼
       │       ComparisonResult.of()
       │              ▲
       │              │
       │              │
       │ implements Diffable
       │              │
       ▼              │
Kong CompareDto ◄─────┘
       ▲
       │
ComparisonService(정규화)
       ▲
       │
Kong 원본 데이터
```

이를 통해 Portal과 Kong의 원본 데이터 모델 차이를 `CompareDto` 단계에서 흡수하고, 실제 비교 로직에서는 동일한 비교 추상화를 사용할 수 있도록 했습니다.

---

# 6. ComparisonResult Container

비교 결과는 `ComparisonResult`를 통해 관리했습니다.

최종 결과에는 다음과 같은 정보가 포함됩니다.

```text
ComparisonResult Container
├── API Gateway Info
├── Portal Data extends Diffable
├── Kong Data extends Diffable
└── Mismatches
```

`Mismatches`에는 Portal과 Kong 사이에서 발견된 차이점을 저장합니다.

이를 통해 API 응답에서 단순히 `MATCH`인지 `MISMATCH`인지 확인하는 것뿐만 아니라 실제 Portal 데이터와 Kong 데이터를 함께 확인할 수 있도록 구성했습니다.

---

# 7. 전체 아키텍처

전체적인 비교 흐름은 다음과 같습니다.

```text
         ┌────────────────────────────┐
         │(Portal || Kong) raw Domain │
         └─────────────┬──────────────┘
                       │
          ┌────────────│─────────────┐
          │ ComparisonService(정규화)  │
          └────────────┬─────────────┘
                       │
                ┌──────────────┐
                │ XXCompareDto │
                └──────┬───────┘
                       │
                  implements
                       │
                       ▼
                  Diffable
                 /         \
                /           \
        Portal Data       Kong Data
                \           /
                 \         /
                  \       /
               ComparisonResult.of()
                       │
                       ▼
              ComparisonResult Container
              ├── APIGW
              ├── Portal Data extends Diffable
              ├── Kong Data extends Diffable
              └── Mismatches
                       │
                       ▼
                  API Response
```

새로운 비교 대상이 추가되는 경우에도 기존 비교 구조를 변경하기보다,

```text
새로운 XXCompareDto
        │
        ├── Diffable 구현
        │
        ▼
ComparisonResult Container에 적용
```

하는 방식으로 확장할 수 있도록 구성했습니다.

따라서 도메인별 비교 규칙은 각 `CompareDto`에 응집시키고, 공통적인 비교 흐름은 `Diffable`과 `ComparisonResult`를 통해 처리하도록 역할을 분리했습니다.

---

# 8. 현재 구조의 한계

현재 구조에서 가장 아쉬웠던 부분은 **비교 결과를 표현하는 모델의 확장성**입니다.

당시에는 주로 현재 시점의 정합성을 확인하는 것이 목적이었기 때문에 비교 결과를 `Mismatches` 중심으로 표현했습니다.

하지만 향후 다음과 같은 기능이 추가된다면 별도의 결과 모델이 필요할 수 있습니다.

```text
현재
Diff
 └── Portal / Kong 상태 비교

향후
History
 └── 변경 이력 분석

향후
Inference
 └── 상태 차이의 원인 추론

향후
Reconcile
 └── 정합성 복구
```

당시에는 이러한 기능이 아직 요구사항으로 확정되지 않았기 때문에 현재 구현에서는 **두 시스템의 현재 상태를 비교하는 Diff 기능에 책임을 한정**했습니다.

향후 새로운 책임이 추가될 경우 기존 `Diffable` 인터페이스에 모든 기능을 추가하기보다, ISP에 따라 새로운 역할의 상위 인터페이스를 이용해 같은 컨테이너의 생성 메서드에서 템플릿 메서드 형식으로 기능을 호출하는 형식으로 구현할 수 있습니다

```text
e.g.
Diffable
  └── 현재 상태 비교

Inferable
  └── 변경 원인 추론

Reconcileable
  └── 정합성 복구
```

처럼 각 책임을 분리하는 방식입니다.

---

# 9. 설계 의도

이 구조에서 가장 중요했던 부분은 **미래의 모든 기능을 미리 추상화하는 것이 아니었습니다.**

당시의 핵심 문제는 다음과 같았습니다.

```text
Portal ↔ Kong
      정합성 불일치
          ↓
     수동 비교 필요
          ↓
      비교 대상 증가
          ↓
   도메인마다 비교 규칙 상이
          ↓
   공통 비교 계약 필요
```

따라서 `Diffable`의 핵심 목적은 **약 10개의 서로 다른 도메인을 일관된 방식으로 비교하면서 도메인별 매핑 및 비교 로직을 각 구현체에 응집시키는 것**이었습니다.

향후 확장성은 이러한 구조에서 얻을 수 있는 부가적인 장점으로 판단했습니다.
