# AGENTS.md

VetHub is a pet clinic management app built as a **workshop project** (`dev.ilionx.workshop`). It has a Spring Boot REST API in `server/` and a SvelteKit single-page app in `client/`. There is no root build; run commands from inside each directory.

## Tech stack

| Area | Server (`server/`) | Client (`client/`) |
|---|---|---|
| Language | Java 25 | TypeScript |
| Framework | Spring Boot 4.1 (milestone) | SvelteKit 2 + Svelte 5 (runes) |
| Build | Gradle (Kotlin DSL), `./gradlew` | Vite, Bun |
| Data | Spring Data JPA / Hibernate, H2 in-memory | n/a |
| Schema | Liquibase (SQL changesets) | n/a |
| Mapping | MapStruct, Lombok | n/a |
| API docs | springdoc-openapi | `openapi-typescript` (generated types) |
| HTTP | Spring Web MVC, Spring Security | `openapi-fetch` |
| UI | n/a | Tailwind 4, shadcn-svelte, bits-ui, lucide |
| Shared lib | `io.github.jframe` starter (validation, exceptions, mapper config) | n/a |
| Quality | Spotless, Checkstyle, PMD, SpotBugs | `svelte-check` |

Tool versions (Java, Bun, Node) are pinned in `mise.toml`, which also sets `SPRING_PROFILES_ACTIVE=dev` and `SERVER_PORT=8080`.

## Project structure

```
server/src/main/java/dev/ilionx/workshop/
  Application.java
  api/Paths.java                 # all route constants (/v1/...)
  api/<feature>/                 # owner, pet, vet, visit
    controller/  service/  repository/
    model/  model/{request,response,mapper,validator}
  common/{config,exception,security}
server/src/main/resources/
  application.yml, application-dev.yml
  db/changelog/changesets/       # Liquibase SQL
server/src/test/.../support/     # UnitTest, IntegrationTest base classes

client/src/
  routes/                        # pages: owners, owners/[id]/pets/..., vets, visits
  lib/api/client.ts              # shared openapi-fetch client
  lib/api/<resource>/<Resource>Controller.ts
  lib/api/models.ts              # friendly aliases for generated types
  lib/types/api.d.ts             # GENERATED, do not edit
  lib/components/{owners,pets,vets,layout,ui}/
scripts/openapi-sync.sh          # spec -> TS types
```

## How the layers connect

```
Svelte page (+page.svelte, fetches in $effect)
  -> lib/api/<Resource>Controller.ts       thin, throws on error
  -> openapi-fetch client                  base URL: VITE_SERVER_BASE_URL + /api
  == HTTP/JSON, CORS, Basic Auth header ==
  -> Spring Security filters               permitAll
  -> @RestController                       calls validator (owners only), then service
  -> @Service (@Transactional)             builds/updates entities by hand
  -> JpaRepository interface               derived queries (findByLastName, ...)
  -> Hibernate -> H2 (schema from Liquibase)
  <- MapStruct mapper: entity -> response DTO -> JSON
```

How the API contract stays in sync: springdoc serves the spec at `/api/v1/public/docs` (Swagger UI at `/api/v1/public/docs/openapi.html`). Then `bun run sync:api` downloads it to `server/openapi.json` and regenerates `client/src/lib/types/api.d.ts`. The client is typed as `createClient<paths>`, so paths, parameters and bodies are checked when you compile.

## Key patterns and conventions

**Server**
- The code is organized by feature first, then by layer. Controllers stay thin: validate, call the service, map the result.
- Routes are constants in `api/Paths.java`. The context path `/api` is added by `application.yml`.
- Dependency injection uses constructors only: `@RequiredArgsConstructor` with `private final` fields.
- Mapping: entity → response goes through MapStruct abstract mappers (`@Mapper(config = SharedMapperConfig.class)`), using `@Mapping` to flatten (for example `owner.id` → `ownerId`). Request → entity is written by hand in services, because it needs repository lookups.
- Entities are JPA with Lombok `@Getter/@Setter`. DTOs use `@Data` + `@Accessors(chain = true)` and `@Schema` for documentation.
- Validation uses explicit jframe `Validator` components called from the controller (`OwnerValidator`). `@Schema(requiredMode = REQUIRED)` only affects the docs and TypeScript types. Pets have **no** validator.
- Errors: throw jframe `DataNotFoundException(ApiErrorCode.X)` (→ 404) or `ValidationException` (→ 400).
- Schema: change it only through new Liquibase SQL changesets in `db/changelog/changesets/`. They are loaded alphabetically, so name them `yyyyMMddHHmm-<PRD|TST>-<name>.xml`. Hibernate never creates tables. Never edit an applied changeset. Seed data uses `context="tst"`, which the `dev` profile enables together with `drop-first: true`.
- Security: CORS is enabled and Basic Auth is configured (`user`/`password`), but `anyRequest().permitAll()` means auth is not enforced.

**Client**
- Pages call `lib/api/<Resource>Controller.ts` functions and never call `client` directly. Import types from `lib/api/models.ts`.
- openapi-fetch returns `{ data, error }`, and controller functions convert `error` into a thrown error. Pages catch it and show a `svelte-sonner` toast.
- There are no SvelteKit `load` functions; data loads in the browser inside `$effect`. State uses runes (`$state`, `$derived`).


## Commands and gotchas

- Server: `./gradlew bootRun`, `./gradlew test`, single test: `./gradlew test --tests 'dev.ilionx.workshop.api.owner.service.OwnerServiceTest'`, full checks: `./gradlew check`.
- Every Java compile runs `spotlessApply` first, which **rewrites source files**. The compiler uses `-Xlint:all -Werror`, so any warning fails the build.
- The test profile (`src/test/resources/application-test.yml`) sets the context path to `""`, so tests use `/v1/...` without `/api`.
- Client: use Bun (`bun install`, `bun run dev`), even though `package-lock.json` exists. The only verification step is `bun run check`; there are no tests or lint script.
- After any backend API/DTO change, run `bun run sync:api` and then `bun run check`. The sync script starts the backend itself (port 8080 must be free) and logs to `/tmp/petclinic-backend.log`.
- `// BUG:` comments in `Owner`, `Pet` and `Visit` mark intentional workshop exercises. Don't fix them unless asked.
- H2 runs in memory, so all data resets when the server restarts.
- That Liquibase migrations go in server/src/main/resources/db/changelog/
- That the frontend uses openapi-fetch with auto-generated types
- The test base classes (@UnitTest, @IntegrationTest) and when to use each
- The DTO pattern (separate request/response objects, MapStruct mappers)
- refinded stories .opencode/refined that are executed should move completed stories .opencode/completed and update the changelog .opencode/CHANGELOG.md after a successful build