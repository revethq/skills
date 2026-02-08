---
name: revet-quarkus
description: Project structure and configuration guide for Revet Quarkus/Kotlin multi-module projects
---

# Revet Quarkus Project Guide

Standards and patterns for Revet Quarkus/Kotlin multi-module projects. Use this skill to create new projects, add modules, and audit existing projects for conformance.

## Tech Stack Versions

| Technology | Version | Notes |
|------------|---------|-------|
| Quarkus | 3.31.1 | LTS release |
| Kotlin | 2.3.10 | K2 compiler |
| JVM | 25 | Target runtime |
| Gradle | 9.3.1 | Build tool |
| ktlint | 1.5.0 | Code formatting via ktlint-gradle plugin |

### Gradle Version Catalog

```toml
# gradle/libs.versions.toml
[versions]
quarkus = "3.31.1"
kotlin = "2.3.10"
ktlint = "1.5.0"

[plugins]
quarkus = { id = "io.quarkus", version.ref = "quarkus" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-allopen = { id = "org.jetbrains.kotlin.plugin.allopen", version.ref = "kotlin" }
ktlint = { id = "org.jlleitschuh.gradle.ktlint", version.ref = "ktlint" }
```

## Multi-Module Project Structure

Standard layout for Revet libraries:

```
{project-name}/
├── {project-name}-core/           # Domain models, interfaces
├── {project-name}-web/            # JAX-RS resources, DTOs
├── {project-name}-persistence-runtime/  # Hibernate Panache entities
├── gradle/
│   └── libs.versions.toml
├── build.gradle.kts               # Root build file
├── settings.gradle.kts
└── gradle.properties
```

### Module Naming Convention

| Module Type | Suffix | Example |
|-------------|--------|---------|
| Core domain | `-core` | `revet-iam-core` |
| Web/REST | `-web` | `revet-iam-web` |
| Persistence | `-persistence-runtime` | `revet-iam-persistence-runtime` |

### Module Dependencies

```
persistence-runtime → core
web → core
```

The `web` and `persistence-runtime` modules depend on `core`. They do not depend on each other.

## Module Configuration

### Core Module (Library)

Core modules are Kotlin libraries without Quarkus runtime dependencies.

```kotlin
// {project}-core/build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.ktlint)
    `java-library`
}

dependencies {
    api(libs.jakarta.validation.api)
    // Domain-only dependencies
}

java {
    withSourcesJar()
    withJavadocJar()
}
```

### Web Module (Quarkus Application)

Web modules include JAX-RS resources and Quarkus extensions.

```kotlin
// {project}-web/build.gradle.kts
plugins {
    alias(libs.plugins.quarkus)
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.kotlin.allopen)
    alias(libs.plugins.ktlint)
}

dependencies {
    implementation(project(":{project}-core"))

    implementation(enforcedPlatform(libs.quarkus.bom))
    implementation(libs.quarkus.kotlin)
    implementation(libs.quarkus.rest)
    implementation(libs.quarkus.rest.jackson)
}

allOpen {
    annotation("jakarta.ws.rs.Path")
    annotation("jakarta.enterprise.context.ApplicationScoped")
    annotation("jakarta.enterprise.context.RequestScoped")
}
```

### Persistence Module (Quarkus Extension)

Persistence modules use Hibernate Panache with Kotlin.

```kotlin
// {project}-persistence-runtime/build.gradle.kts
plugins {
    alias(libs.plugins.quarkus)
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.kotlin.allopen)
    alias(libs.plugins.kotlin.noarg)
    alias(libs.plugins.ktlint)
}

dependencies {
    implementation(project(":{project}-core"))

    implementation(enforcedPlatform(libs.quarkus.bom))
    implementation(libs.quarkus.hibernate.orm.panache.kotlin)
    implementation(libs.quarkus.jdbc.postgresql)
}

allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}

noArg {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

## Known Workarounds

### Quarkus Bug #36506 - Dev Mode Classloading

**Issue:** In Quarkus dev mode, multi-module projects may fail with `ClassNotFoundException` when extension modules reference core library classes.

**Workaround:** Add core module to the runtime classpath explicitly:

```kotlin
// {project}-persistence-runtime/build.gradle.kts
quarkus {
    // Force core module classes to be visible in dev mode
    quarkusBuildProperties.put(
        "quarkus.bootstrap.incubating-model-resolver",
        "true"
    )
}
```

**Alternative:** Run dev mode from the web module with explicit module dependencies:

```bash
./gradlew :project-web:quarkusDev
```

### Gradle 9 Convention API Migration

**Issue:** Gradle 9 removed the deprecated Convention API. Plugins using `project.convention` fail.

**Affected Plugins:**
- Older versions of kotlin-allopen
- Older versions of kotlin-noarg
- Some Quarkus extension plugins

**Solution:** Update to Kotlin 2.3.10+ and Quarkus 3.31.1+ which use the new `extensions` API:

```kotlin
// Old (fails in Gradle 9)
project.convention.getPlugin(AllOpenExtension::class.java)

// New (Gradle 9 compatible)
project.extensions.getByType(AllOpenExtension::class.java)
```

**Verification:**

```bash
./gradlew build --warning-mode all 2>&1 | grep -i convention
```

## Conformance Checklist

Use this checklist when auditing existing projects:

### Project Structure
- [ ] Uses standard module suffixes (`-core`, `-web`, `-persistence-runtime`)
- [ ] Has `gradle/libs.versions.toml` version catalog
- [ ] Root `build.gradle.kts` applies common plugins to subprojects

### Version Requirements
- [ ] Quarkus version >= 3.31.1
- [ ] Kotlin version >= 2.3.10
- [ ] Gradle version >= 9.3.1
- [ ] JVM target = 25

### Core Module
- [ ] Uses `java-library` plugin
- [ ] Does NOT depend on Quarkus BOM
- [ ] Publishes sources and javadoc jars

### Web Module
- [ ] Uses `io.quarkus` plugin
- [ ] Configures `allOpen` for JAX-RS annotations
- [ ] Depends on core module

### Persistence Module
- [ ] Uses `io.quarkus` plugin
- [ ] Configures `allOpen` for JPA annotations
- [ ] Configures `noArg` for JPA annotations
- [ ] Depends on core module (not web module)

### Code Quality
- [ ] ktlint plugin applied to all modules
- [ ] No ktlint violations (`./gradlew ktlintCheck`)

## Remediation Patterns

### Missing Version Catalog

Create `gradle/libs.versions.toml`:

```bash
mkdir -p gradle
cat > gradle/libs.versions.toml << 'EOF'
[versions]
quarkus = "3.31.1"
kotlin = "2.3.10"
ktlint = "1.5.0"

[libraries]
quarkus-bom = { module = "io.quarkus.platform:quarkus-bom", version.ref = "quarkus" }
quarkus-kotlin = { module = "io.quarkus:quarkus-kotlin" }
quarkus-rest = { module = "io.quarkus:quarkus-rest" }
quarkus-rest-jackson = { module = "io.quarkus:quarkus-rest-jackson" }
quarkus-hibernate-orm-panache-kotlin = { module = "io.quarkus:quarkus-hibernate-orm-panache-kotlin" }
quarkus-jdbc-postgresql = { module = "io.quarkus:quarkus-jdbc-postgresql" }
jakarta-validation-api = { module = "jakarta.validation:jakarta.validation-api", version = "3.1.0" }

[plugins]
quarkus = { id = "io.quarkus", version.ref = "quarkus" }
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-allopen = { id = "org.jetbrains.kotlin.plugin.allopen", version.ref = "kotlin" }
kotlin-noarg = { id = "org.jetbrains.kotlin.plugin.noarg", version.ref = "kotlin" }
ktlint = { id = "org.jlleitschuh.gradle.ktlint", version.ref = "ktlint" }
EOF
```

### Missing allOpen Configuration

Add to web or persistence module:

```kotlin
allOpen {
    // For web modules
    annotation("jakarta.ws.rs.Path")
    annotation("jakarta.enterprise.context.ApplicationScoped")
    annotation("jakarta.enterprise.context.RequestScoped")

    // For persistence modules
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

### Outdated Quarkus Version

Update in `gradle/libs.versions.toml`:

```toml
[versions]
quarkus = "3.31.1"
```

Then refresh dependencies:

```bash
./gradlew dependencies --refresh-dependencies
```

## New Project Scaffold

To create a new Revet library project:

```bash
PROJECT=revet-example

# Create directories
mkdir -p $PROJECT/{$PROJECT-core,$PROJECT-web,$PROJECT-persistence-runtime}/src/main/kotlin
mkdir -p $PROJECT/gradle

# Create settings.gradle.kts
cat > $PROJECT/settings.gradle.kts << EOF
rootProject.name = "$PROJECT"

include("$PROJECT-core")
include("$PROJECT-web")
include("$PROJECT-persistence-runtime")
EOF

# Create gradle.properties
cat > $PROJECT/gradle.properties << EOF
kotlin.code.style=official
org.gradle.jvmargs=-Xmx2g
EOF
```

Then add `build.gradle.kts` files for each module following the patterns above.
