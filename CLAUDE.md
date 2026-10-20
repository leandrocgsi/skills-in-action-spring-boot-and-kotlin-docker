# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Spring Boot 4.1 / Kotlin / JDK 25 REST API (Maven, base package `br.com.erudio`) from the Erudio course "Formação Spring Boot com Kotlin". It is the Kotlin port of the Java project `rest-with-spring-boot-and-java-erudio-2026`: same endpoints, payloads, Flyway migrations, database and Postman collection (`Collections/`). It manages People and Books, with JWT auth, file upload/download (S3), CSV/XLSX/PDF import/export and email sending. Keep behavior identical to the Java twin unless a change is explicitly requested.

Source is under `src/main/kotlin` and `src/test/kotlin` (set via `sourceDirectory` in `pom.xml`). The `erudio-*` directories at the root are Claude Code plugin packs (enabled in `.claude/settings.json`), not part of the application.

## Commands

No Maven wrapper; use system `mvn` with `JAVA_HOME` on a JDK 25.

```bash
mvn spring-boot:run                  # needs MySQL on :3306 and LocalStack S3 on :4566
mvn clean package -DskipTests        # build the jar
mvn test                             # all tests (needs Docker)
mvn test -Dtest=PersonServicesTest   # one test class
mvn flyway:migrate                   # pom hardcodes localhost DB, root/admin123
```

Swagger UI is served at the root path (`springdoc.swagger-ui.use-root-path: true`), seeded login is `leandro` / `admin123`.

## Tests

- `unittests/**` use Mockito (with `mockito-kotlin` matchers, since plain `any()` returns null and Kotlin rejects it) and need nothing external.
- `integrationtests/**` and `repository/PersonRepositoryTest` extend `AbstractIntegrationTest`, which starts Testcontainers `mysql:9.1.0` and `localstack` (S3 only, bucket `erudio-files-test`) plus an in-process GreenMail SMTP server, and injects their addresses as the highest-priority property source. Docker must be running. The app runs on port 8888 (`src/test/resources/application.yml`).
- `src/test/resources/application.yml` is a full copy of the main one, not an overlay: keep config changes in sync in both files.
- Integration tests are order-dependent (`@TestMethodOrder`, state in `lateinit` fields filled by a first `signin` test), so run the whole class, not a single method. New ones extend `AuthenticatedIntegrationTest`.
- The `withjson/`, `withxml/`, `withyaml/` test packages mirror each other; an endpoint change usually needs all three.
- The surefire `argLine` loads Mockito as a `-javaagent` (via `maven-dependency-plugin` `properties` goal) and passes `--enable-native-access=ALL-UNNAMED`. It has no `@{argLine}` token, so add one if JaCoCo is introduced.

## Architecture

Layering: `controllers` → `services` → `repository` (Spring Data JPA) → `model`.

- **Entity ↔ DTO**: services never return entities; they convert with `DozerMapper.parseObject(...)` (`mapper/`). DTOs extend HATEOAS `RepresentationModel`, and services add links via `linkTo(methodOn(XController::class.java)...)`; changing a controller method means updating those links. Paged results use `PagedResourcesAssembler`.
- **Kotlin and `methodOn`**: `methodOn` builds CGLIB proxies, which need non-final classes. Controllers are opened by the `spring` all-open preset; DTOs returned from controllers are opened through the `all-open:annotation=...hateoas.server.core.Relation` option in `pom.xml`, so a new DTO used with `methodOn` needs `@Relation`.
- **Nullability**: entity/DTO properties are nullable with defaults so each class keeps a no-arg constructor for Dozer, Jackson and Hibernate; don't make them non-null.
- **OpenAPI docs** live in `controllers/docs/*ControllerDocs` interfaces implemented by each controller; put springdoc annotations there. `springdoc.enable-kotlin: false` keeps the schema free of `nullable: true`.
- **Content negotiation**: endpoints produce JSON, XML and YAML (`WebConfig`, Jackson 3 `tools.jackson.*`). Boot configures only JSON/XML mappers, so `JacksonConfig` builds the YAML converter. `jackson-module-kotlin` is the `tools.jackson.module` artifact.
- **Import/export**: `FileExporterFactory` picks `CsvExporter`/`XlsxExporter`/`PdfExporter` from the `Accept` header (`file/exporter/MediaTypes`); `FileImporterFactory` picks `CsvImporter`/`XlsxImporter` from the file extension. To add a format, add a `@Component` implementation and a factory branch. PDF uses JasperReports templates in `src/main/resources/templates/`, compiled at request time (with `jasperreports-jdt`, so it works from the fat jar) and cached per template; `jasperreports.properties` must stay on the classpath (it ignores the missing `Arial` font on Linux).
- **File storage**: `FileStorageService` reads/writes S3 through the AWS SDK v2 `S3Client` (`config/AwsS3ClientConfig`) and creates the bucket at startup if missing. A set `aws.s3.endpoint` (LocalStack, fixed `test`/`test` credentials) uses path-style addressing; a blank endpoint uses real AWS with the default credentials chain.
- **Email**: `EmailController` → `EmailService` → `EmailSender` (SMTP via `spring.mail.*`, credentials from env `EMAIL_USERNAME` / `EMAIL_PASSWORD`). `EmailSender` is a stateful singleton, not safe for concurrent requests.
- **Security**: stateless JWT (auth0 `java-jwt`) in `security/jwt/*`, with `JwtTokenFilter` before `UsernamePasswordAuthenticationFilter`. `/auth/signin`, `/auth/refresh/**`, `/auth/createUser`, Swagger and `/v3/api-docs` are public; `/api/**` requires auth. The filter must never throw because of a bad token (it is treated as anonymous), since exceptions from it bypass `@ControllerAdvice`. Passwords are PBKDF2 via `DelegatingPasswordEncoder`. `model/User` is the `UserDetails`.
- **Errors**: `exception/handler/CustomEntityResponseHandler` maps custom exceptions to statuses; unmapped ones become a 500 `ExceptionResponse`.
- **DI and logging**: classes use field `@Autowired private lateinit var` (the Mockito `@InjectMocks` tests and `ReflectionTestUtils.setField` rely on the field names), and `java.util.logging.Logger`.

## Database and config

- MySQL `localhost:3306`, schema `rest_with_spring_boot_erudio`, `root`/`admin123`. Flyway migrations `V1..V18` in `src/main/resources/db/migration` run at startup; `ddl-auto: none`. Schema changes go in a new `V<n>__*.sql`. The migrations are shared with the Java twin, so editing an applied one causes a checksum mismatch and must be mirrored in both projects.
- `spring.jpa.properties.hibernate.check_nullability: true` is required: springdoc pulls in Bean Validation, and Hibernate otherwise silently disables its not-null check.
- Boot 4 split auto-configuration into modules: keep `spring-boot-starter-flyway`, `-hateoas` and `-webmvc` rather than the raw libraries.
- `Book.launchDate` is a `LocalDate` (column still `datetime(6)`); the XML integration-test `BookDTO` needs `LocalDateXmlAdapter` because JAXB can't marshal `LocalDate`.
- CORS origins come from `cors.originPatterns`.
