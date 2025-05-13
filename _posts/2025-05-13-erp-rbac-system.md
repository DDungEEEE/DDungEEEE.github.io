---
published: true
title: 
date: 2025-05-13 14:20:00 +09:00
categories: [백엔드, RBAC, ERP]
tags:
  [ERP, RBAC, 권한설계, Spring Security]
---

# 유동적인 권한 관리 시스템 설계하기 – ERP 프로젝트에서의 RBAC 고민

## 배경

ERP 프로젝트의 요구사항을 분석하던 중, 루트 권한을 가진 `ADMIN` 사용자가 시스템 내에서 역할(Role)을 생성하고, 역할에 따른 권한(Permission)을 부여할 수 있도록 하는 기능이 필요하다는 요구사항이 도출되었다.

이전까지는 단순히 `ADMIN`, `USER`와 같은 구분만으로 역할을 관리해왔던 나에게는, 이러한 요구사항이 다소 낯설게 느껴졌다.  
처음에는 다음과 같은 의문이 들었다:

- "굳이 SystemRole을 별도의 엔티티로 분리해야 하나?"  
- "그때그때 필요한 권한을 enum으로 선언해서 관리하면 되는 거 아닌가?"  
- "Enum은 불변성도 있고, 다른 개발자에게도 명확하지 않나?"

하지만, 도메인 요구사항을 정확히 이해하지 못한 채 구현을 진행하는 것만큼 비효율적인 개발은 없다. 그래서 요구사항부터 다시 정리하고자 했다.

## 도출된 권한 요구사항

1. 사용자의 직무에 따른 시스템 권한 조정이 가능해야 한다.
2. 권한은 고정된 Enum이 아니라, 동적으로 생성 및 부여될 수 있어야 한다.
3. 권한 부여자는 ADMIN이거나 특정한 시스템 권한을 가진 사용자여야 한다.

## 권한 설계 전 이해해야 할 도메인 요소

- **직무(Position)**: 사용자가 수행하는 업무의 성격 (예: 개발자, 디자이너, 인사담당자 등)
- **직급(Rank)**: 조직 내에서의 계급 (예: 사원, 주임, 대리, 과장, 이사 등)

직무와 직급의 조합에 따라 업무 권한 범위가 세분화된다.

예를 들어:

- 백엔드 개발자 - 과장: 코드 전체 관리 및 릴리즈 제어
- 백엔드 개발자 - 대리: 코드 수정 및 PR 생성
- 백엔드 개발자 - 사원: 코드 조회만 가능

## 역할 및 권한 구성 전략

### 1. SystemRole 기반 권한 관리

`SystemRole`이라는 별도 테이블을 생성하여 사용자에게 부여하고, Spring Security에서 이 Role을 기반으로 인가 처리를 수행한다.

#### 장점

- Spring Security와의 연동이 간단하며, `hasRole()`로 빠르게 인가 처리 가능
- 관리자/운영자와 일반 사용자의 권한을 명확히 분리 가능
- UI에서 Role 기반 메뉴 노출 등 제어가 쉬움
- 권한 위임 및 회수 시 SystemRole 컬럼만 바꾸면 돼서 유연함
- 초기 개발 속도가 빠름

#### 단점

1. 직무/직급 변경 시 자동 반영되지 않음
2. 조직 구조와 분리되어 일관성이 떨어짐
3. 권한 중복 및 관리 포인트 증가 (SystemRole과 Position+Rank 양쪽 관리)
4. SystemRole이 사실상 "권한 그룹"처럼 동작하면서 Role 개념이 불명확해짐
5. `"인사_과장"`처럼 구체적인 이름은 추상적인 역할로 사용하기 어려움

### 2. 직무 + 직급 → Permission 기반 권한 관리

직무와 직급을 복합키로 사용하여, Permission과 직접 매핑한다.

#### 장점

- 조직 구조와 일치하는 직관적인 권한 모델링 가능
- 직무/직급 변경 시 권한 자동 변경 → 유지보수 용이
- Role 정의 없이도 직무/직급 기반으로 일관된 권한 정책 수립 가능
- UI 설계도 조직 구조 기준으로 명확하게 가능

#### 단점
- 시스템에 대한 권한과 업무에 대한 권한을 분리하기 어렵다
- 시스템 권한을 특정 사용자 Ex) 개발자 에게 부여해야할 수도 있는데, 이 때 사용자 이름으로 하드코딩을 해야한다. 즉, 일관성이 사라진다.

## 유동적인 RBAC를 위한 설계 방식

시스템이 확장 가능해야 하므로, Permission을 정적인 Enum이 아닌 DB에 저장하고, Action(CREATE, READ, UPDATE, DELETE 등) 단위로 Permission을 설계한다.

```java
@Getter
public enum Action {
    CREATE("생성"), EDIT("수정"), DELETE("삭제"), VIEW("조회");
    private final String description;

    Action(String description) {
        this.description = description;
    }
}
```

Permission 엔티티는 다음과 같이 구성한다:

```java
@Entity
@Table(name = "permission")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
public class Permission extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String domain;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Action action;

    @Column(length = 200)
    private String description;

    @Column(nullable = false, unique = true)
    private String code;

    @PrePersist
    @PreUpdate
    protected void setPermissionCode() {
        if (this.domain != null && this.action != null) {
            this.code = this.domain.toUpperCase() + "_" + this.action.name();
        }
    }
}
```

## 직무 + 직급 + 권한 매핑을 위한 설계

업무 권한을 직무(Position)와 직급(Rank) 기준으로 세분화해 관리하기 위해,  
다음과 같은 매핑 테이블 구조를 설계했다.

### PositionRankPermission 엔티티

```java
@Entity
public class PositionRankPermission {

    @EmbeddedId
    private PositionRankPermissionId id;

    @ManyToOne
    @MapsId("permissionId")
    @JoinColumn(name = "permission_id")
    private Permission permission;
}
```

### PositionRankPermissionId (복합 키 클래스)

```java
@Embeddable
public class PositionRankPermissionId implements Serializable {

    private Long positionId;

    @Enumerated(EnumType.STRING)
    private Rank rank;

    private Long permissionId;
}
```

직무 + 직급 → 복수의 권한을 부여해야 하기 때문에 이런 복합키를 설정하였다.
또한 @EmbeddedId는 복합키를 **하나의 Value Object(VO)**처럼 다룰 수 있고 타입 안정성을 높다.
마지막으로 Permission은 별도의 엔티티이므로 연관관계가 필요하기 떄문에 외래키로 엮어준것이다.

### 장점

| 항목 | 설명 |
|------|------|
| 권한 중복 방지 | `positionId + rank + permissionId`로 유일하게 권한을 정의하므로 중복 등록 불가 |
| 유연한 확장성 | 권한이 추가되거나 역할이 바뀌어도 관계만 추가 가능 |
| JPA 친화적 구조 | `@EmbeddedId + @MapsId`로 엔티티 간 관계와 키를 깔끔하게 매핑 |
| 명확한 권한 모델링 | 직무/직급 기준으로 일관되게 정의 가능 |
| 실시간 권한 변경 대응 | 직무 또는 직급이 바뀌면 권한도 자동 변경되어 유지보수 용이 |

### 권한 조회 코드

```java
public List<Permission> getPermissions(Long positionId, Rank rank) {
    return positionRankPermissionRepository
        .findAllById_PositionIdAndId_Rank(positionId, rank)
        .stream()
        .map(PositionRankPermission::getPermission)
        .toList();
}
```


## 결론

기존에 단순한 Role 기반으로 권한을 관리하던 방식은 작은 규모의 시스템이나 권한이 단순한 서비스에는 충분히 효과적이다.  
그러나 ERP와 같은 기업 시스템에서는 **조직 구조(직무, 직급)와 시스템 제어 권한(운영자, 관리자 등)**이 명확히 분리되어야 하며,  
이 둘을 동일한 방식으로 처리하면 **보안적 리스크와 유지보수 어려움**이 생긴다.

따라서 아래와 같이 **두 축의 권한 관리 체계를 병행**하는 것이 실용적이다:

- **업무 권한(Position + Rank → Permission)**  
  - 실질적으로 회사에서 수행하는 업무 기능을 제어  
  - 직무 변경/승진 시 자동으로 권한도 연동  
  - 근태 승인, 인사 평가, 재고 열람/수정 등의 도메인 행위 처리에 적합

- **시스템 관리자 권한(SystemRole → Permission)**  
  - 시스템을 제어하거나 다른 사용자 계정/권한을 관리하는 기능 전용  
  - 예: 유저 등록, 로그 조회, 권한 수정, 관리자 페이지 접근  
  - 해당 권한은 일반 업무 흐름과 무관하게 일부 사용자에게 명시적으로 부여

이와 같은 이중 구조를 설계하면 다음과 같은 장점이 있다:

- 업무 권한과 시스템 제어 권한의 명확한 **역할 분리**
- **보안적 책임 소재 구분**이 명확해짐
- 향후 조직 구조가 바뀌어도 Role 정책에 영향을 주지 않음
- UI/화면 설계 시에도 Role/Permission 기준으로 **접근 제어가 일관성 있게 가능**

즉, 이 구조는 도메인에 대한 이해와 유연한 시스템 운영을 모두 충족시킬 수 있는 **현실적이고 확장 가능한 RBAC 설계 전략**이다.
---