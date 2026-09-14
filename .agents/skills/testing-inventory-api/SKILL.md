---
name: testing-inventory-api
description: Run the inventory Spring Boot REST API locally with disposable H2 data and verify account authentication, CRUD, and CORS.
---

# Local API runtime testing

Repo: online-inventory-management-system. This checkout is backend-only; use curl for API workflows, not a fabricated frontend.

## Environment

Set `JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64`; default Java may be older. If Maven Central is rate-limited, configure a central mirror at `https://maven-central.storage-download.googleapis.com/maven2` in `~/.m2/settings.xml`.

When MySQL is unavailable, H2 is test-scoped and requires the test classpath. Run from repository root:

```sh
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 mvn spring-boot:run \
  -Dspring-boot.run.useTestClasspath=true \
  '-Dspring-boot.run.arguments=--spring.datasource.url=jdbc:h2:mem:api_e2e;MODE=MySQL;DATABASE_TO_LOWER=TRUE;DB_CLOSE_DELAY=-1;NON_KEYWORDS=USER --spring.datasource.driver-class-name=org.h2.Driver --spring.datasource.username=sa --spring.datasource.password= --spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.H2Dialect --spring.jpa.hibernate.ddl-auto=update --spring.jpa.show-sql=true'
```

The dialect override is necessary because main resources configure MySQLDialect. `NON_KEYWORDS=USER` permits the mapped user table. Explicit H2Dialect may log a harmless redundant-dialect warning. Wait for `Started DevaInventoryApplication` and inspect DDL logs; a running server alone does not prove every table was created. H2 does not validate MySQL-specific driver/dialect or existing database migration.

## Account-to-token flow

1. POST JSON `/api/roles` with `roleName` and `roleDescription`; expect 202, capture `roleId`.
2. POST JSON `/api/users/{roleId}` with `firstName`, `lastName`, `email`, `userName`, `password`; expect 202 and generated `userId`.
3. POST URL-encoded `/api/login` with **userName** and password, not username or JSON. Expect JSON `access_token` and `refresh_token` and matching response headers.
4. POST `/api/checktoken/{access_token}` verifies JWT and returns `userName` and `roles`. Wrong credentials should return 401.

Use disposable local credentials. Endpoints currently permit anonymous requests; do not mistake successful CRUD for authorization enforcement.

## CRUD and CORS

Categories: POST/GET `/api/categories`; GET/PUT/DELETE `/api/categories/{id}`. Fields: `categoryName`, `categoryDescription`. POST expects 202.
Brands: POST/GET `/api/brands`; PUT/DELETE `/api/brands/{id}`. Fields: `brandName`, `brandDescription`. POST expects 200.
Verify GET after each mutation: some service methods swallow errors and delete controllers report `deleted=true` regardless. User update/delete may be unimplemented; check before choosing them for complete CRUD.

Send preflight OPTIONS with arbitrary `Origin`, `Access-Control-Request-Method: POST` or PUT, and `Access-Control-Request-Headers: content-type,authorization`; expect reflected allowed origin, credentials=true, and requested methods/headers allowed.

Shell API evidence belongs in captured response logs; record only actual browser interaction, e.g. viewing live GET JSON before/after mutation.

## Devin Secrets Needed

None for isolated H2 testing. Production database credentials should be supplied separately if production-like MySQL testing is required.
