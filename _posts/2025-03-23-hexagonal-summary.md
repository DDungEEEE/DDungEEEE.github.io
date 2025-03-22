---
title: 헥사고날 아키텍처와 DDD에 대한 개념 정리
date: 2025-03-23 12:39:14 +09:00
categories: [백엔드, 아키텍처]
tags:
  [DDD, 도메인 주도 개발, 헥사고날 아키텍처, 소프트웨어 아키텍처, 클린코드]
---

# DDD + 헥사고날 아키텍처 구조 정리

DDD와 헥사고날 아키텍처를 공부하면서 내가 이해한 구조와 개념들을 정리한 글이다. 복잡하게 느껴질 수 있는 아키텍처 개념들을 실제 예제와 함께 정리하며, 어떤 역할을 하는지 중심으로 풀어보았다.

### 🔸 Adapter
- **in**: 사용자의 요청을 받고 응답을 반환하는 역할
  - 예: `Controller`, `RequestDto`, `ResponseDto`
- **out**: 외부 시스템(DB, API 등)과 상호작용
  - 예: `Entity`, `JpaRepository`, 외부 API 연동

> ✅ 도메인은 DB와 무관하게 설계되며, DB에 저장되기 위한 객체는 `Entity`로 별도 구성. `Mapper`를 통해 도메인 ↔ 엔티티 변환을 수행.

### 🔸 Application
- **port**
  - **in**: 유즈케이스 정의 (`ProductUseCase`) → Controller가 호출함
  - **out**: 외부 시스템을 사용하는 인터페이스 (`ProductRepositoryPort`)
- **service**: 실제 유즈케이스 흐름을 구현하는 클래스 (UseCase의 구현체)
- **command**: 도메인을 만들기 위한 내부 요청 DTO
  - `RequestDto → toCommand()` 메서드를 통해 변환
  - 또는 Mapper를 통해 변환 가능

> ✅ `Command`는 도메인에서 사용할 데이터 구조이고, 외부 요청을 그대로 도메인에 넘기지 않기 위한 중간 계층 개념이라고 이해했음.

### 🔸 Domain
- 핵심 비즈니스 규칙과 상태를 정의하는 순수 자바 클래스
  - 예: `Product`, `Category`
  - DB 저장을 전혀 신경 쓰지 않으며, 단지 "업무 의미"에만 집중


## 🔁 흐름 예시: 상품 등록

### 1. Request → Command
```java
public class CreateProductRequest {
    private String name;
    private int price;

    public CreateProductCommand toCommand() {
        return new CreateProductCommand(name, price);
    }
}
```

### 2. Command → UseCase (interface)
```java
public interface ProductUseCase {
    Product createProduct(CreateProductCommand command);
}
```

### 3. UseCase 구현체 (Service)
```java
@RequiredArgsConstructor
public class ProductService implements ProductUseCase {
    private final ProductRepositoryPort productRepository;
    private final ProductMapper productMapper;

    public Product createProduct(CreateProductCommand command) {
        Product product = new Product(command.getName(), command.getPrice());
        product.register();

        ProductEntity entity = productMapper.toEntity(product);
        productRepository.save(entity);

        return product;
    }
}
```

### 4. Domain
```java
public class Product {
    private String name;
    private int price;
    private ProductStatus status;

    public void register() {
        this.status = ProductStatus.REGISTERED;
    }
}
```

### 5. Entity
```java
@Entity
@Table(name = "products")
public class ProductEntity {
    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private int price;

    @Enumerated(EnumType.STRING)
    private ProductStatus status;
}
```

### 6. Repository 구조
```java
// Port (interface)
public interface ProductRepositoryPort {
    void save(ProductEntity entity);
}

// Adapter (구현체)
@Repository
@RequiredArgsConstructor
public class ProductRepositoryAdapter implements ProductRepositoryPort {
    private final SpringDataProductRepository jpaRepo;

    public void save(ProductEntity entity) {
        jpaRepo.save(entity);
    }
}

// 실제 JPA Repository
public interface SpringDataProductRepository extends JpaRepository<ProductEntity, Long> {}
```

### 7. 응답 DTO
```java
public class ProductResponse {
    private Long productId;
    private String name;
    private String status;

    public static ProductResponse from(Product product) {
        return new ProductResponse(product.getId(), product.getName(), product.getStatus().name());
    }
}
```


## 🔄 전체 흐름 요약
```
[Controller]
 ↓
Request DTO (CreateProductRequest)
 ↓
Command (CreateProductCommand)
 ↓
ProductService (UseCase 구현체)
 ↓
도메인 Product 객체 생성 및 상태 변경
 ↓
Entity로 매핑 후 저장
 ↓
도메인 → 응답 DTO로 변환 → 사용자에게 반환
```

___

## 🧠 내가 느낀 핵심 개념 정리

| 구성 요소 | 내가 이해한 역할 정리 |
|------------|------------------|
| Domain     | 비즈니스 규칙을 순수하게 표현하는 클래스. DB를 모름 |
| Entity     | DB에 맞춰 저장하기 위한 객체. 도메인과 분리되어야 함 |
| Command    | 외부 요청을 도메인에서 사용할 수 있게 만든 중간 DTO |
| UseCase    | 유즈케이스를 인터페이스로 정의. Controller가 의존함 |
| Service    | UseCase의 구현체. 외부와 도메인을 조립해서 흐름 구현 |
| Port       | 외부 시스템을 추상화한 인터페이스 (예: RepositoryPort) |
| Adapter    | 실제 포트 구현체. 외부 시스템과 직접 연결됨 |
| Mapper     | 도메인 ↔ Entity, Request ↔ Command 등을 매핑함 |



## ✅ 마무리

DDD + 헥사고날 아키텍처는 구조적으로는 복잡해 보이지만,  
흐름과 구조를 이해하면 실제 프로젝트에서의 유지보수성과 테스트성이 향상될 것이라고 생각된다.

하지만 **무작정 이 구조가 좋다**고 해서 모든 프로젝트에 적용하는 것은 오히려 독이 될 수 있다.  
요구사항이 단순하거나 비즈니스 로직이 거의 없다면 오히려 **오버 엔지니어링**이 될 수 있기 때문에,

> 🔍 "요구사항 분석 → 비즈니스 복잡도 → 시스템 규모"  
를 충분히 고려한 뒤, **필요한 만큼만 유연하게 적용**하는 것이 가장 바람직하다고 느꼈다.

내가 정리한 내용은 아직 부족할 수 있지만, 실제로 개발하며 느낀 걸 기준으로 내 언어로 풀어본 것이다.  
DDD와 헥사고날 아키텍처에 대해 아직 완전히 깊게 이해한 것은 아니지만 이 아키텍처가 왜 생겨났는지, 그리고 **Adapter & Port 구조의 장점**을 이해하는 것만으로도 소프트웨어 아키텍처에 대한 시야가 훨씬 넓어졌다고 느꼈다.
