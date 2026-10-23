# WSO2 Identity Server - Project Structure & Architecture

## Overview

**WSO2 Identity Server (IS)** is an enterprise-grade, open-source Identity and Access Management (IAM) solution built on the WSO2 Carbon Framework. It supports authentication, authorization, identity federation, and user provisioning through standards like SAML 2.0, OAuth 2.0, OpenID Connect, SCIM 2.0, and WS-Federation.

- **Version:** 7.3.0-rc1-SNAPSHOT
- **Group ID:** `org.wso2.is`
- **Artifact ID:** `identity-server-parent`
- **License:** Apache License 2.0

---

## Technology Stack

| Category | Technology |
|----------|-----------|
| Language | Java 21 |
| Build System | Apache Maven (multi-module POM) |
| Runtime Framework | WSO2 Carbon Kernel 4.12.25 (OSGi/Eclipse Equinox) |
| Servlet Container | Apache Tomcat (embedded via Carbon) |
| Default Database | H2 (development); MySQL, PostgreSQL, Oracle, MSSQL (production) |
| Security | Bouncycastle, Apache Rampart (WS-Security), PKCS12 keystores |
| Protocols | SAML 2.0, OAuth 2.0, OpenID Connect, SCIM 2.0, WS-Federation, FAPI |
| Configuration | TOML (`deployment.toml`), JSON (`default.json`), Jinja2 templates |
| OSGi Plugins | Eclipse Equinox, P2 provisioning |
| Web Framework | Jaggery (JavaScript-based) |
| Testing | TestNG, Selenium, Cypress, WSO2 Carbon Automation Framework, JMeter |
| Code Coverage | JaCoCo |

---

## Root Directory Structure

```
product-is/
├── pom.xml                         # Root Maven POM (parent for all modules)
├── modules/                        # Core source modules (14 Maven modules + others)
├── docs/                           # Code style (wso2_codestyle.xml), contribution guidelines
├── etc/                            # Javadoc CSS, copyright templates
├── jmeter-tests/                   # Performance test plans (SAML, Passive-STS)
├── oidc-conformance-tests/         # OpenID Connect conformance test suite (Python)
├── oidc-fapi-conformance-tests/    # FAPI conformance tests
├── product-scenarios/              # End-to-end product scenario tests
├── usecases/                       # Use case documentation
├── legal/                          # Legal notices
├── README.md                       # Project overview and getting started
├── SECURITY.md                     # Security vulnerability reporting
├── LICENSE                         # Apache License 2.0
└── release-notes.html              # Release notes
```

---

## Module Descriptions

The project is organized into **18 modules** under `modules/`. The root `pom.xml` declares 14 of them as Maven sub-modules.

### Core Distribution

| Module | Description |
|--------|-------------|
| **distribution** | Assembles the final deployable server distribution (ZIP). Unpacks Carbon Core, merges JSON configs, includes REST API WAR, sets file permissions, and creates the production artifact. |
| **p2-profile-gen** | Generates the OSGi P2 repository profile from `carbon.product`. Defines which OSGi features/plugins the server includes. Uses Eclipse Equinox. |
| **features** | Aggregator for OSGi feature definitions: `org.wso2.identity.styles.feature`, `org.wso2.identity.ui.feature`, `org.wso2.identity.utils.feature`, `org.wso2.identity.jaggery.apps.feature`. |

### Authentication

| Module | Description |
|--------|-------------|
| **authenticators** | Aggregator for outbound/federated authenticators (SAML SSO, STS, passive federation). |
| **local-authenticators** | Aggregator for local authentication plugins (IWA, FIDO2, push notification, etc.). |
| **social-authenticators** | Aggregator for social login providers (Google, Facebook, GitHub, LinkedIn, Microsoft, etc.). |
| **connectors** | Aggregator for connector-based authenticators (Twitter, X.509, SMS OTP, etc.). |

### OAuth2 & Provisioning

| Module | Description |
|--------|-------------|
| **oauth2-grant-types** | Custom OAuth2 grant type extensions (JWT Bearer, SAML grants, etc.). |
| **provisioning-connectors** | Outbound provisioning connectors (Google, Salesforce, Workday, etc.). |

### REST API & UI

| Module | Description |
|--------|-------------|
| **api-resources** | REST API web application (`api.war`) providing management and user endpoints. |
| **integration-ui-templates** | UI template configurations for integration flows. |
| **styles** | Product and service UI styling/theming components. |

### Testing

| Module | Description |
|--------|-------------|
| **integration** | Comprehensive test infrastructure with sub-modules: `tests-common`, `tests-integration`, `tests-cypress-integration`, `tests-ui-integration`. |
| **tests-utils** | Reusable test utilities and SOAP admin stubs. |

### Other

| Module | Description |
|--------|-------------|
| **agents** | Mobile proxy IDP agent implementation (iOS). |
| **samples** | Sample applications and demonstrations. |
| **migration** | Migration utilities (references external `identity-migration-resources` repo). |

---

## Key Configuration Files

All located under `modules/distribution/src/repository/resources/conf/`:

| File | Purpose |
|------|---------|
| `deployment.toml` | **Primary server configuration** - hostname, ports, databases, keystores, admin credentials, user stores |
| `default.json` | Default application properties - UI settings, cipher config, authorization, menu items |
| `log4j2.properties` | Logging configuration |
| `catalina-server.xml` | Embedded Tomcat servlet container configuration |
| `carbon.properties` | JVM cipher transformation settings |
| `secret-conf.properties` | Secure/encrypted property storage |
| `key-mappings.json` | Configuration key mapping definitions |
| `infer.json` | Type inference configuration for config resolution |

### Distribution-specific Configs (`modules/distribution/conf/`):

- `bps/` - Business Process Server configuration
- `policies/` - Authorization policy templates (role-based, group-based, time-based, scope-based)

### Security Resources (`modules/distribution/src/repository/resources/security/`):

- `wso2carbon.p12` - Primary PKCS12 keystore
- `client-truststore.p12` - Client trust store

---

## Entry Point & Server Startup Flow

WSO2 Identity Server does **not** use a traditional Java `main()` method. It is an **OSGi-based application** launched through the WSO2 Carbon kernel:

### Startup Flow

```
1. User runs: ./bin/wso2server.sh (generated in distribution target)
   │
2. Shell script sets environment:
   ├── CARBON_HOME, JAVA_HOME, classpath
   ├── Calls adaptive.sh (loads Nashorn/ASM for adaptive auth)
   ├── Calls fips.sh (if FIPS mode enabled)
   └── Calls openssl-tls.sh (TLS configuration)
   │
3. JVM launches Eclipse Equinox OSGi framework
   │
4. Carbon Kernel bootstrap:
   ├── Reads carbon.product (P2 profile)
   ├── Parses deployment.toml → generates XML configs
   ├── Merges default.json with feature-specific configs
   └── Initializes internal components
   │
5. OSGi bundle activation:
   ├── Loads plugins from repository/components/plugins/
   ├── Activates features from P2 profile
   ├── Starts embedded Tomcat (catalina-server.xml)
   └── Deploys web applications (api.war, etc.)
   │
6. Server ready:
   ├── Management Console: https://localhost:9443/carbon
   ├── REST APIs: https://localhost:9443/api/*
   └── Authentication endpoints: https://localhost:9443/oauth2, /samlsso, etc.
```

### Key Startup Scripts (`modules/distribution/src/bin/`)

| Script | Purpose |
|--------|---------|
| `adaptive.sh` / `adaptive.bat` | Configures Nashorn engine and ASM libraries for adaptive authentication scripting |
| `fips.sh` / `fips.bat` | Enables FIPS 140-2 compliant mode with Bouncycastle FIPS provider |
| `openssl-tls.sh` | Configures OpenSSL-based TLS settings |

Note: The main `wso2server.sh` script is provided by the Carbon Kernel dependency (`wso2carbon-core`) and is unpacked during the distribution build, not stored directly in this repository.

---

## Build & Packaging Flow

```
mvn clean install
   │
   ├── 1. features/           → Builds OSGi feature definitions
   ├── 2. p2-profile-gen/     → Generates P2 repository from carbon.product
   ├── 3. connectors/         → Builds connector plugins
   ├── 4. api-resources/      → Builds REST API WAR (api.war)
   ├── 5. authenticators/     → Builds authenticator bundles
   ├── 6. social-auth/        → Builds social authenticators
   ├── 7. provisioning/       → Builds provisioning connectors
   ├── 8. local-auth/         → Builds local authenticators
   ├── 9. oauth2-grant-types/ → Builds OAuth2 grant types
   ├── 10. integration-ui/    → Builds UI templates
   ├── 11. distribution/      → Final assembly:
   │       ├── Unpacks wso2carbon-core (base server)
   │       ├── Copies P2 profile (OSGi bundles)
   │       ├── Copies api.war to webapps/
   │       ├── Merges JSON config files
   │       └── Creates wso2is-7.3.0-rc1-SNAPSHOT.zip
   ├── 12. styles/            → Builds UI styles
   ├── 13. tests-utils/       → Builds test utilities
   └── 14. integration/       → Runs integration tests against the distribution
```

---

## Integration Test Structure

Located under `modules/integration/`:

```
integration/
├── tests-common/                    # Shared test infrastructure
│   ├── admin-clients/              # SOAP admin service clients
│   ├── admin-stubs/                # Generated SOAP stubs
│   ├── integration-test-utils/     # Base test utilities
│   ├── jacoco-report-generator/    # Code coverage reports
│   ├── extensions/                 # Test framework extensions
│   └── ui-pages/                   # Selenium page objects
├── tests-integration/
│   └── tests-backend/              # Backend integration tests (871 test files)
│       └── src/test/java/org/wso2/identity/integration/test/
│           ├── oauth2/             # OAuth2 flow tests
│           ├── saml/               # SAML SSO tests
│           ├── scim2/              # SCIM 2.0 API tests
│           ├── oidc/               # OpenID Connect tests
│           ├── application/        # App management tests
│           ├── user/               # User management tests
│           ├── idp/                # Identity provider tests
│           ├── consent/            # Consent management tests
│           ├── rest/               # REST API tests
│           ├── provisioning/       # Provisioning tests
│           └── ...                 # 40+ test packages
├── tests-cypress-integration/      # Cypress E2E UI tests
└── tests-ui-integration/           # Additional UI tests
```

### External Test Suites

| Directory | Purpose |
|-----------|---------|
| `jmeter-tests/` | Performance testing with Apache JMeter (SAML, Passive-STS flows) |
| `oidc-conformance-tests/` | OpenID Connect certification conformance tests (Python) |
| `oidc-fapi-conformance-tests/` | Financial-grade API (FAPI) conformance tests |
| `product-scenarios/` | End-to-end product scenario validation |

---

## Default Server Configuration (`deployment.toml`)

```toml
[server]
hostname = "localhost"
node_ip = "127.0.0.1"
base_path = "https://$ref{server.hostname}:${carbon.management.port}"

[super_admin]
username = "admin"
password = "admin"

[user_store]
type = "database_unique_id"

[database.identity_db]
type = "h2"
url = "jdbc:h2:./repository/database/WSO2IDENTITY_DB"

[database.shared_db]
type = "h2"
url = "jdbc:h2:./repository/database/WSO2SHARED_DB"

[keystore.primary]
file_name = "wso2carbon.p12"
type = "PKCS12"
```

Production deployments typically replace H2 with MySQL, PostgreSQL, Oracle, or MSSQL.
