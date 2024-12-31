---
title: "MongoDB를 왜 메인 데이터베이스로 사용하고있나요?"
seoDescription: "SaaS팀에서 Multi-Tenancy 환경을 구축하면서 MognoDB를 메인 데이터베이스로 사용하고있는 이유에 대해서 다뤄봅니다."
datePublished: Wed Dec 18 2024 15:47:45 GMT+0000 (Coordinated Universal Time)
cuid: cm4u2gr5n000u08l0gryd4vz9
slug: why-use-mongodb
tags: mongodb, database, multi-tenancy

---

## 배경

필자는 현재 B2B SaaS 신사업팀에서 백엔드 엔지니어로 일 하고있다.

메인 데이터베이스로 MongoDB를 사용하고있는데, 그 이유와 사용 방법에 대해 소개해보려한다. (실제로 질문도 많이 받았었다)

## 왜 MongoDB를 메인 데이터베이스로 채택했나요?

MongoDB는 세부적인 보안 설정을 컬렉션 별로 지원하기때문이다.

왜 세부적인 보안 설정이 필요하고, 이를 어떻게 활용한 계획인지 설명하기 위해선 우선 Multi-Tenancy에 대해 간략히 설명이 필요할것같다.

> ### Multi-Tenancy가 뭔가요?

Multi-Tenancy는 단일 인스턴스에서 여러 테넌트(고객 또는 조직)를 독립적으로 지원하는 아키텍처이다.

![](https://mblogthumb-phinf.pstatic.net/MjAyMDAxMjBfODgg/MDAxNTc5NDk0ODQ2OTM3.ojXj6Fg4AyZhoXT2uy8tXlhkETmqoz479v_WPyj7cugg.zdmOSUhnn9kXIFEtBWVdAOoW11RrkSoIrCCFe6q-ZLwg.PNG.ki630808/multitenancy2.PNG?type=w800 align="left")

위 이미지처럼 Multi-Tenancy는 소프트웨어 개발과 유지보수 비용을 공유하기 때문에 경제적이다. 따라서, 공급자는 업데이트를 한 번만 하면 다중 테넌트에게 업데이트 된 서비스를 제공할 수 있다.

(Single-Tenant라면, 공급자는 여러 소프트웨어의 인스턴스에 모두 업데이트가 필요하다.)

![](https://velog.velcdn.com/images/jongil512/post/3cbc5ce4-b0b3-479b-bdb8-6975e50e39b1/image.png align="left")

그런데 각 테넌트마다 독립적인 Database Server를 사용하면, 고객사가 10000개라면, 10000개의 DB Server를 관리해야하는 비용이 발생한다. 그래서 서비스 목적성에 맞추어 Multi-Tenancy Model을 채택하여 설계한다.

내가 속한 팀의 Multi-Tenancy 구조는 3번이다. 하나의 인스턴스에서 논리적으로 테넌트를 분리하고 Database Server는 한대만 기동하고있다.

물론, 엔터프라이즈 고객의 요구에 따라 Database Server와 어플리케이션 인스턴스를 독립적으로 분리할 수 있는 인프라 세팅도 준비되어있다.

> ### 어떻게 Tenant 별로 데이터를 관리하고있나요?

위에서 말했듯이, 어플리케이션에서 논리적으로 테넌트들을 분리하고있다. 논리적으로 분리된 테넌트는 각 독립된 MongoDB Collection에 저장이된다.

이를 코드로 나타내면 아래와 같다.

```kotlin
class ReservationRepositoryImpl(
    database: MongodbConfiguration,
) : ReservationRepository {
    private val entityType: Class<ReservationDataModel> = ReservationDataModel::class.java
    private val collection: MongoTemplate = MongoTemplate(database.mongoDatabaseFactory)

    fun create(tenantId: String, entity: ReservationDataModel) {
        val collectionName = "reservations_$tenantId"
        collection.insert(entity, collectionName)
    }

    fun findById(tenantId: String, reservationId: String): ReservationDataModel? {
        val collectionName = "reservations_$tenantId"
        return collection.findById(reservationId, entityType, collectionName)
    }
}
```

최초 고객이 로그인할 때, 테넌트를 설정할 수 있고 접속하면 URL에서 tenantId를 관리하고있고, 도메인 서비스에게 보내는 요청의 Request Param에 tenantId가 모두 포함되어있는 형태이다.

따라서, Domain Service들의 API Endpoint에는 `.../tenants/{tenant}/...`가 붙어있다. 이러한 요청을 통해 구분된 TenantId는 MongoDB의 CollectionName에 Suffix에 붙여 사용한다.

e.g. collection name

* reservations\_{tenantId}
    
* products\_{tenantId}
    

따라서 각 테넌트별로 독립적인 컬렉션을 갖고있는 형태가된다. 독립적이기때문에 얻을 수 있는 이점은 컬렉션 별 보안 설정을 할 수 있다. 물론, 특정 테넌트에 잘못된 데이터가 Hard Updating 되었을 때 다른 테넌트들은 장애 전파가 안되기도하지만 이는 매우 특별한 케이스이긴하다.

> ### 그래서 중요한 보안은 어떻게 설정을 하고있나요?

MongoDB는 컬렉션 수준 보안 설정 기능을 제공한다.

> **역할 기반 액세스 제어 (RBAC) 기능**

* 역할 기반 액세스 제어를 통해 DB의 특정 컬렉션에 대한 권한을 세밀하게 부여할 수 있다.
    
    * 관리자는 사용자 정의 역할(User Defined Role)을 생성하여 각 테넌트의 컬렉션에 대한 읽기, 쓰기, 업데이트 등의 권한을 개별적으로 설정할 수 있다.
        
    * 권한의 수준 범위는 아래와 같다.
        
        * 데이터 베이스 수준
            
            * 특정 테넌트 전용의 DB에만 접근 권한만 부여할 수 있다 (하나의 테넌트가 N개의 DB에 접근할 수도 있다)
                
        * 컬렉션 수준
            
            * 테넌트의 컬렉션만 접근 가능하도록 제한이 가능하다
                
        * 필드 수준
            
            * 특정 필드에도 read, write의 제한이 가능하다
                

> 사용자 정의 역할(User Defined Role)

* MongoDB 관리자(제품팀)는 특정 컬렉션이나 DB에 대해 사용자 정의 역할을 생성할 수 있다.
    
* 각 역할 별 read, write, update와 같은 세부 권한을 커스텀 할 수 있고, 특정 테넌트의 컬렉션에만 적용되도록 제한할 수도 있다.
    
    * e.g.
        
        * reservation\_{tenantA}에서 `병원 IT 관리자1` 에게 해당 컬렉션의 read 권한을 줄 수 있다.
            
        * 제품팀 개발자들의 책임에 따라 관리자가 책임별로 세부 권한을 나누어 관리할 수 있다.
            
            * SaaS 제품 특성 상 병원의 데이터를 제품 개발자라고 모두 볼 수 있는것은 법적으로 안된다.
                

위의 MongoDB RBAC 특성을 통해, 잘못된 쿼리나 운영 실수로 인해 다른 테넌트에 데이터가 노출되거나, 수정되는것을 방지할 수 있게되어 데이터 격리 수준을 한층 강화할 수 있다.

```javascript
db.createRole({
  role: "tenant_admin_reader",
  privileges: [
    {
      resource: { db: "myDatabase", collection: "reservations_tenantA" },
      actions: ["find"]
    }
  ],
  roles: []
});

// 2. 사용자에게 역할 할당
db.createUser({
  user: "tenantAUser",
  pwd: "pw1234",
  roles: ["tenant_admin_reader"]
});
```

위 스크립트처럼 RBAC 설정들을 모두 DB Query를 제공하고 있어서, 백오피스를 구축하면 쉽게 권한 분리도 가능하다.

## 마치며

속한 SaaS팀에서 Multi-Tenancy 환경을 구축하면서 MognoDB를 메인 데이터베이스로 사용하고있는 이유에 대해서 다뤄보았다. 포스트에서 유의할점은 RDBMS에서도 RBAC에 준하는 권한 관리가 가능하다.

그러나 MongoDB는 동적 데이터 구조 관리(Dynamic Data Structure Management) + 스키마 리스(Schema-less)하여 보다 현재 속한 신사업 팀이 보다 빠르고 간단하게 Multi-Tenancy를 구축하는데 도움이 되었던 내용을 공유한것이니 `MongoDB 라서 가능했다!는 아니라는것을 유의바란다.`

(실제로, Mongo Atlas를 사용하고 있었기때문에 Atlas Search 기능을 통해 보다 쉽게 검색 엔진을 도입할 수 있었던 사례도 있는데 이것도 추후 기회가 되면 작성해보도록 하겠다.)