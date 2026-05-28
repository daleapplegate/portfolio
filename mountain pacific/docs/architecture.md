# OPAS Architecture

OPAS (Oracle Provider Application System, artifact `pas-presentation`) is a
classic three-tier Java web application that captures provider Pre-Admission
Screening (PAS) and Profile Prior Authorization (PA) workflows for a state
Medicaid contractor. It is deployed either as a runnable JAR with an embedded
Tomcat or as a WAR on an external Tomcat 9.

## At a glance

| Aspect           | Value                                                              |
| ---------------- | ------------------------------------------------------------------ |
| Language         | Java 8 source (built on JDK 11+, classfile 52.0)                   |
| Build            | Maven 3.6+ multi-module reactor (`opas-parent` → 3 modules)        |
| Framework        | Spring (core/aop/tx/jdbc), Spring Security (LDAP)                  |
| UI               | JSF 2.1 + ICEfaces 3.3 on JSP/JSPX pages                           |
| Persistence      | Hibernate ORM + Spring `JdbcTemplate` for PL/SQL & native SQL      |
| Database         | Oracle 10g+ (heavy use of PL/SQL packages and DB triggers)         |
| Reports          | BIRT report runtime                                                |
| Runtime          | Tomcat 9 (embedded or external)                                    |
| Entry point      | `com.mpqh.opas.tiger.pas.presentation.OPASMain` (embedded mode)    |
| CI/CD            | Azure DevOps Pipelines (`azure-pipelines.yml`)                     |
| Auth             | LDAP bind via Spring Security; roles `ROLE_PAS_USER`, `ROLE_PAS_VIEW_ONLY` |

## Module layout

The reactor (`pom.xml`) declares three modules built in strict dependency order:

```
opas-parent (pom)
├── PasIntegration   ── pas-integration.jar
├── PasBusiness      ── pas-business.jar         (depends on pas-integration)
└── PasPresentation  ── pas-presentation.{war,jar} (depends on the other two)
```

Each module corresponds to one tier of the application.

### `PasIntegration` — data and infrastructure

Owns everything that touches the database.

- `com.mpqh.pas.integration.hibernate` — Hibernate entity classes (one per
  Oracle table/view, e.g. `PasProfile`, `PasProfilePa`, `VPasProviderInfo`),
  DAOs, and the Profile PA lifecycle enum (`ProfilePaStatus`).
- `com.mpqh.pas.integration.persistence` — repository/DAO helpers and JDBC-based
  staging utilities.
- Spring configs: `applicationContext.xml`, `integrationContext.xml`,
  `hibernate.cfg.xml`.
- SQL/PLSQL deployment scripts live under
  `PasIntegration/src/main/resources/db/...` (versioned by ticket/task).

The shared Spring contexts here are loaded into both the test JVMs and the
running web app.

### `PasBusiness` — domain services

Stateless `@Service`-style classes that implement business rules and
orchestrate persistence calls.

- `com.mpqh.pas.business.services` — top-level service interfaces & impls.
- `com.mpqh.pas.business.tiger.service.impl` — the "tiger" (modernized) service
  layer:
  - `StandardProfileService` — Profile PA orchestration (`createProfilePA`,
    `updateProfilePAStatus`, `endProfilePA`, transmit staging).
  - `ProfilePaTransitionEngine` + `ProfilePaLifecycleTransitionRule` — encodes
    the six-state Profile PA lifecycle
    (`NEW-PND → NEW-STAGED → NEW-TRN → DC-PND → DC-STAGED → DC-TRN`) and
    rejects illegal transitions.
  - `ProfilePaNumberStrategyDelegate` — pluggable PA number generators
    (Tiger, TigerPlus, etc.).
- `com.mpqh.pas.business.helpers` — `SystemParamsHelper` and similar
  configuration shims.
- Spring config: `applicationContext.xml` declares the service beans and the
  `@Transactional` PlatformTransactionManager.

### `PasPresentation` — web UI

The deployable artifact. Packages JSF/ICEfaces views, JSP pages, REST/JSON
endpoints, and the embedded launcher.

- `com.mpqh.pas.presentation.beans` — JSF backing beans (e.g. `ProfileBean`,
  `DischargeBean`, `IntakeBean`). Beans wire user input into business services
  and translate domain results into ICEfaces UI state.
- `com.mpqh.pas.presentation.security` — Spring Security filter chain,
  authentication entry points, role-based view gating.
- `com.mpqh.pas.presentation.navigation`, `validators`, `converters`,
  `bundle`, `treehandler`, `utils`, `web`, `exceptions` — the usual JSF
  scaffolding (menus, validators, converters, message bundles, breadcrumb
  trees, web filters/listeners, exception handlers).
- `com.mpqh.pas.presentation.services` (legacy package) — JSON download
  endpoints for non-Profile-PA processes (PERS provider feed, eval download).
- Web assets under `PasPresentation/WebRoot/`:
  - `*.jsp` / `*.jspx` page templates (e.g. `Profile_inc.jspx`).
  - `css/tiger-style.css` — the modernized "tiger" theme.
  - `WEB-INF/web.xml`, `faces-config.xml`, etc.
- Embedded runner: `OPASMain` (root package
  `com.mpqh.opas.tiger.pas.presentation`) boots Tomcat 9 programmatically,
  serving the unpacked WebRoot.

## Dependency direction

```
Browser
   │   HTTP / JSF postbacks (ICEfaces partial-page renders)
   ▼
PasPresentation (JSF beans, Spring Security)
   │   Spring DI (interface calls)
   ▼
PasBusiness (services + transition engine)
   │   DAO / repository / JdbcTemplate
   ▼
PasIntegration (Hibernate entities, JDBC, PL/SQL calls)
   │
   ▼
Oracle DB (tables, views, PL/SQL packages, triggers)
```

`PasIntegration` knows nothing about Spring beans defined in business or
presentation, `PasBusiness` knows nothing about JSF, and `PasPresentation`
never talks to Hibernate directly — UI code goes through a service interface.

## Cross-cutting concerns

- **Transactions** — `@Transactional` declared on service methods in
  `PasBusiness`; the manager is wired in `applicationContext.xml`.
- **Security** — Spring Security with LDAP authentication; method- and
  view-level role checks via `ROLE_PAS_USER` / `ROLE_PAS_VIEW_ONLY`. JSF
  components use ICEfaces' `renderedOnUserRole` attribute for in-template
  authorization.
- **Audit** — every service that mutates Oracle state takes an `auditName`
  argument and writes it through to `UPDATED_BY`/`CREATED_BY` columns and
  to lifecycle log helpers (e.g. `ProfilePaTransitionLogHelper`).
- **Configuration** — runtime config comes from
  `application.properties` (loaded by Spring) + JVM system properties
  (`opas.port`, `opas.buildVersion`, etc.) + environment variables
  (`OPAS_DB_URL`, `OPAS_DB_USERNAME`, `OPAS_DB_PASSWORD`, `OPAS_PORT`, …).
- **Build stamp** — `-Dopas.buildVersion=...` is injected by the CI pipeline
  and rendered on the login page for traceability.
- **Logging** — log4j; rolling files under `logs/` when running the
  embedded jar.

## Two coexisting code generations

The Maven setup includes a `<tiger>` profile and the package tree carries a
parallel `tiger/` namespace inside each module (e.g.
`com.mpqh.pas.business.tiger.service.impl`). This is the modernized code path
that newer features target; the older (non-tiger) classes are kept in place to
support legacy screens until they are migrated. The build assembles both at
once — there is no separate "legacy" artifact.

## Two parallel PA processes (important)

OPAS runs two distinct PA flows that should not be confused:

1. **Profile PA** — the modern flow owned by `StandardProfileService`,
   `ProfilePaTransitionEngine`, and the `ProfilePaStatus` enum. State lives in
   `PAS_PROFILE_PA.PROFILE_PA_STS_CD` with codes
   `NEW-PND / NEW-STAGED / NEW-TRN / DC-PND / DC-STAGED / DC-TRN`.
2. **PERS PA** — the older flow that uses the `PERSPROV*` constants in
   `PasCodeTable` and lives in services such as `ProfileDownload`,
   `JSONProfileObject`, and `EvalDownload`. Despite the similar names this is
   a separate state machine on different tables and must be left alone when
   editing Profile PA code.

## Database integration

- **Schema deployment** — PL/SQL bootstrap and migration scripts ship inside
  `PasIntegration/src/main/resources/db/pas/…`. They are applied to the
  target Oracle instance by the DBA outside the Maven build.
- **Batch processes** — long-running staging/transmit work (e.g. moving
  `*-STAGED` rows to `*-TRN`) is performed by Oracle PL/SQL packages
  (`PKG_PAS_PERS_TRANSMIT`, etc.) on a scheduler, not by the Java tier.
- **Reports** — BIRT (`*.rptdesign` in `PasReports/`) consumes the same Oracle
  schema directly.

## Build, test, package

```bash
# fast iteration on backend layers
mvn -Ptiger clean install -pl PasIntegration,PasBusiness -am

# full build incl. UI + tests
mvn -Ptiger clean install

# runnable jar (embedded Tomcat)
mvn -Ptiger clean package
java -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

Tests use JUnit 4 + Mockito 1.10.19. The bulk of the lifecycle / transition
logic is covered in `PasBusiness/test/.../ProfilePaTransitionEngineTest.java`
and `StandardProfileServiceTest.java`; UI bean wiring is covered in
`PasPresentation/test/.../ProfileBeanTest.java` and `DischargeBeanTest.java`.

## Deployment topology

```
                 ┌──────────────────────────────────────┐
                 │            Tomcat 9 host             │
                 │  (embedded JAR  or  external WAR)    │
                 │                                      │
   Browser ─────►│  pas-presentation.war / runner.jar   │
                 │   ├── JSF / ICEfaces views           │
                 │   ├── Spring Security (LDAP)         │
                 │   └── Spring beans → services        │
                 └──────────────┬───────────────────────┘
                                │ JDBC (Oracle thin)
                                ▼
                         ┌────────────────┐
                         │   Oracle DB    │
                         │  (PAS schema)  │
                         │  + PL/SQL pkgs │
                         └────────────────┘
                                ▲
                                │
                         BIRT / Reports
                         (PasReports)
```

For end-to-end operational instructions (env vars, datasource setup, JVM
flags, troubleshooting), see [OPAS_DEPLOYMENT_GUIDE.md](OPAS_DEPLOYMENT_GUIDE.md)
and [OPAS_BUILD_REFERENCE.md](OPAS_BUILD_REFERENCE.md).
