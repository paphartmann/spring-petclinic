# Copilot instructions for Spring PetClinic

## Build, test, and run

- Use Java 17 or newer. The repository supports both wrapper-based build tools:
  - Maven (the CI build): `./mvnw -B verify`
  - Gradle: `./gradlew build`
- Run the complete test suite directly with `./mvnw test` or `./gradlew test`.
- Run one JUnit class with Maven, for example:
  `./mvnw -Dtest=OwnerControllerTests test`
  Run one method with `./mvnw -Dtest=OwnerControllerTests#processFindFormByLastName test`.
  With Gradle, use `./gradlew test --tests 'org.springframework.samples.petclinic.owner.OwnerControllerTests'`
  or append `--tests '...OwnerControllerTests.processFindFormByLastName'`.
- Formatting and static checks are part of the build. Maven runs Spring Java Format and nohttp/checkstyle during `validate`; use `./mvnw validate` for those checks. Gradle equivalents include `./gradlew check`, `./gradlew checkstyleMain checkstyleTest`, and `./gradlew checkstyleNohttp`.
- Start the application with `./mvnw spring-boot:run` or `./gradlew bootRun`, then open `http://localhost:8080/`.
- The Maven `css` profile is the supported way to regenerate CSS after changing SCSS or Bootstrap:
  `./mvnw package -P css`. The generated `src/main/resources/static/resources/css/petclinic.css` is the committed asset; Gradle does not compile SCSS.
- The default database is in-memory H2. For local MySQL/PostgreSQL development, start the matching service with `docker compose up mysql` or `docker compose up postgres`, then run with `--spring.profiles.active=mysql` or `--spring.profiles.active=postgres`.
- MySQL integration tests use Testcontainers and PostgreSQL integration tests use Spring Boot Docker Compose, so Docker is required for those tests. The test application `main()` methods in `PetClinicIntegrationTests` and the database integration test classes are useful IDE run configurations.

## Architecture

- `PetClinicApplication` is the Spring Boot entry point. The application is a server-rendered Spring MVC application: controllers return Thymeleaf view names under `src/main/resources/templates`, with shared fragments in `templates/fragments`.
- Code is organized primarily by feature: `owner` contains the owner/pet/visit domain, repositories, validation, and controllers; `vet` contains vet/specialty domain and both HTML and JSON endpoints; `system` contains cross-cutting web, cache, welcome, and error configuration; `model` contains shared entity base classes.
- Domain classes are JPA entities backed by Spring Data repository interfaces. SQL schemas and seed data are kept per database under `src/main/resources/db/{h2,mysql,postgres}`. `application.properties` selects H2 by default and resolves schema/data locations from the `database` property; the MySQL and PostgreSQL property files provide the profile-specific connection settings.
- The web layer uses conventional Spring MVC form binding and Jakarta Bean Validation. Owner and pet forms use model attributes, `BindingResult`, redirect attributes, and Thymeleaf templates; owner searches are paginated with five results per page. `/vets.html` is the HTML view and `/vets` returns a JSON wrapper (`Vets`).
- Caching is enabled through JCache/Caffeine for the `vets` repository data. Internationalization is session-based, defaults to English, and changes through the `lang` request parameter; message bundles live under `src/main/resources/messages`.
- Tests mirror the feature packages. Use MVC slice tests with mocked repositories for controller behavior, entity/validator tests for domain rules, and `@SpringBootTest` tests for full-context HTTP behavior. Database-specific tests are explicitly profile/container based and may be disabled when Docker is unavailable.
- Kubernetes manifests are in `k8s/`; `docker-compose.yml` is for local database services and test infrastructure, not application source configuration. There is no project Dockerfile; use Spring Boot's image build (`./mvnw spring-boot:build-image`) when an image is needed.

## Repository-specific conventions

- Follow the existing Spring Java Format and EditorConfig settings: Java/XML use tabs with width 4; Gradle files use two spaces. Keep the existing license header on new Java/XML source files.
- Prefer constructor injection and package-private controllers where the existing feature uses that pattern. Keep repository query methods expressed as Spring Data derived-query method names when possible.
- Preserve the form-binding safeguards in controllers: disallow identifier fields with `WebDataBinder`, validate with `@Valid` and `BindingResult`, and use redirect-after-success with flash messages. Maintain the existing view-name and route conventions when adding pages.
- Keep persistence changes synchronized across all supported databases: update the relevant schema and seed files under `db/h2`, `db/mysql`, and `db/postgres`, plus profile properties when connection behavior changes.
- Keep translations synchronized across message bundles. `I18nPropertiesSyncTest` checks the properties files, so a new message key should be added consistently to the supported locales.
- Use the existing test style and Spring Boot test slices. Controller tests should exercise route, model, validation, redirect, and view behavior with repository mocks; integration tests should use the existing H2, Testcontainers MySQL, or Docker Compose PostgreSQL setup rather than inventing another database harness.
- Run `pre-commit` when changing repository-wide files; `.pre-commit-config.yaml` currently runs the Gitleaks hook. All commits also require a `Signed-off-by` trailer per the repository contribution policy.
