---
name: Stack Auto-Detection
description: Comprehensive auto-detection of language, framework, test runner, coverage, security tools, linters, auth patterns, factories, DB, API style, CI — by reading real files in the target repo.
version: 1.0.0
---

# Stack Auto-Detection

Deep inspection of the target repository. Reads actual dependency files, parses configs, analyzes existing code and test patterns. The goal is to minimize questions to the developer by detecting as much as possible automatically.

## Detection Order

Run detections in this order (each step may inform the next):

1. **Language & Package Manager** — which dependency files exist
2. **Framework** — parse dependency files for framework names
3. **Directory Structure** — map source and test directories
4. **Test Runner & Config** — detect test framework and configuration
5. **Coverage Tool** — detect coverage configuration
6. **Security Tools** — detect SAST and security linters
7. **Linters & Formatters** — detect code style tools
8. **Database** — detect DB type and ORM
9. **Auth Pattern** — detect authentication approach from existing code/tests
10. **Factory / Fixture Pattern** — detect test data creation approach
11. **API Style** — REST, GraphQL, gRPC
12. **CI/CD Platform** — detect existing CI configuration
13. **Container / Infra** — Docker, docker-compose, k8s
14. **Monorepo Detection** — multiple apps/packages

---

## 1. Language & Package Manager

```bash
# Ruby
[ -f "Gemfile" ] && LANG=ruby PM=bundler
[ -f "Gemfile.lock" ] && LOCKFILE=Gemfile.lock

# JavaScript / TypeScript
[ -f "package.json" ] && LANG=javascript PM=npm
[ -f "yarn.lock" ] && PM=yarn
[ -f "pnpm-lock.yaml" ] && PM=pnpm
[ -f "bun.lockb" ] && PM=bun
[ -f "tsconfig.json" ] && LANG=typescript

# Python
[ -f "requirements.txt" ] && LANG=python PM=pip
[ -f "pyproject.toml" ] && LANG=python PM=poetry_or_pip
[ -f "Pipfile" ] && LANG=python PM=pipenv
[ -f "setup.py" ] && LANG=python PM=setuptools
[ -f "uv.lock" ] && PM=uv

# Go
[ -f "go.mod" ] && LANG=go PM=gomod

# Rust
[ -f "Cargo.toml" ] && LANG=rust PM=cargo

# Java / Kotlin
[ -f "pom.xml" ] && LANG=java PM=maven
[ -f "build.gradle" ] && LANG=java PM=gradle
[ -f "build.gradle.kts" ] && LANG=kotlin PM=gradle

# Elixir
[ -f "mix.exs" ] && LANG=elixir PM=mix

# PHP
[ -f "composer.json" ] && LANG=php PM=composer

# .NET / C#
[ -f "*.csproj" ] || [ -f "*.sln" ] && LANG=csharp PM=dotnet

# Swift
[ -f "Package.swift" ] && LANG=swift PM=spm

# Dart / Flutter
[ -f "pubspec.yaml" ] && LANG=dart PM=pub
```

**Version detection** — extract language version:

```bash
# Ruby: .ruby-version, Gemfile (ruby "X.Y.Z"), .tool-versions
[ -f ".ruby-version" ] && RUBY_VERSION=$(cat .ruby-version)
grep -m1 'ruby ' Gemfile 2>/dev/null | grep -oP '"\K[^"]+'

# Node: .nvmrc, .node-version, package.json engines.node, .tool-versions
[ -f ".nvmrc" ] && NODE_VERSION=$(cat .nvmrc)
[ -f ".node-version" ] && NODE_VERSION=$(cat .node-version)

# Python: .python-version, pyproject.toml [project] requires-python, runtime.txt
[ -f ".python-version" ] && PYTHON_VERSION=$(cat .python-version)

# Go: go.mod "go X.Y"
grep -m1 '^go ' go.mod 2>/dev/null | awk '{print $2}'

# Rust: rust-toolchain.toml, rust-toolchain
[ -f "rust-toolchain.toml" ] && grep -oP 'channel\s*=\s*"\K[^"]+' rust-toolchain.toml
```

---

## 2. Framework Detection

### Ruby — parse Gemfile

```bash
grep -E "gem ['\"]rails['\"]" Gemfile && FRAMEWORK=rails
grep -E "gem ['\"]sinatra['\"]" Gemfile && FRAMEWORK=sinatra
grep -E "gem ['\"]hanami['\"]" Gemfile && FRAMEWORK=hanami
grep -E "gem ['\"]grape['\"]" Gemfile && FRAMEWORK=grape
grep -E "gem ['\"]roda['\"]" Gemfile && FRAMEWORK=roda

# Rails version from Gemfile.lock
grep -A1 '^\s*rails ' Gemfile.lock 2>/dev/null | grep -oP '\d+\.\d+\.\d+'
```

### JavaScript/TypeScript — parse package.json

```bash
# Read dependencies and devDependencies
node -e "
  const pkg = require('./package.json');
  const all = {...(pkg.dependencies||{}), ...(pkg.devDependencies||{})};
  console.log(JSON.stringify(Object.keys(all)));
" 2>/dev/null

# Or without node:
grep -oP '"(next|express|fastify|koa|hapi|nestjs|nuxt|vue|react|svelte|angular|remix|astro|gatsby|electron|expo|react-native)"\s*:' package.json
```

**Framework mapping from package.json:**

| Dependency | Framework | Type |
|-----------|-----------|------|
| `next` | Next.js | fullstack |
| `express` | Express | backend |
| `fastify` | Fastify | backend |
| `koa` | Koa | backend |
| `@hapi/hapi` | Hapi | backend |
| `@nestjs/core` | NestJS | backend |
| `nuxt` | Nuxt | fullstack |
| `vue` | Vue.js | frontend |
| `react` | React | frontend |
| `react-native` | React Native | mobile |
| `svelte` / `@sveltejs/kit` | Svelte/SvelteKit | frontend/fullstack |
| `@angular/core` | Angular | frontend |
| `remix` / `@remix-run/node` | Remix | fullstack |
| `astro` | Astro | frontend |
| `gatsby` | Gatsby | frontend |
| `electron` | Electron | desktop |
| `expo` | Expo (React Native) | mobile |
| `@trpc/server` | tRPC | API layer |
| `graphql` / `apollo-server` / `@apollo/server` | GraphQL | API layer |
| `prisma` / `@prisma/client` | Prisma | ORM |
| `drizzle-orm` | Drizzle | ORM |
| `typeorm` | TypeORM | ORM |
| `sequelize` | Sequelize | ORM |
| `mongoose` | Mongoose | ODM (MongoDB) |
| `knex` | Knex | query builder |

### Python — parse requirements.txt / pyproject.toml

```bash
# requirements.txt
grep -iE '^(django|flask|fastapi|starlette|tornado|pyramid|falcon|sanic|litestar|quart)' requirements.txt 2>/dev/null

# pyproject.toml — [project] dependencies or [tool.poetry.dependencies]
grep -iE '(django|flask|fastapi|starlette|tornado|pyramid|falcon|sanic|litestar)' pyproject.toml 2>/dev/null

# Pipfile
grep -iE '(django|flask|fastapi|starlette)' Pipfile 2>/dev/null
```

**Python framework mapping:**

| Dependency | Framework |
|-----------|-----------|
| `django` / `djangorestframework` | Django / DRF |
| `flask` | Flask |
| `fastapi` | FastAPI |
| `starlette` | Starlette |
| `tornado` | Tornado |
| `pyramid` | Pyramid |
| `falcon` | Falcon |
| `sanic` | Sanic |
| `litestar` | Litestar |

**Python ORM detection:**

| Dependency | ORM |
|-----------|-----|
| `sqlalchemy` | SQLAlchemy |
| `django` (built-in) | Django ORM |
| `tortoise-orm` | Tortoise |
| `peewee` | Peewee |
| `mongoengine` | MongoEngine |

### Go — parse go.mod

```bash
grep -E '(gin-gonic/gin|labstack/echo|gofiber/fiber|go-chi/chi|gorilla/mux|julienschmidt/httprouter|beego|buffalo|iris)' go.mod 2>/dev/null
```

### Rust — parse Cargo.toml

```bash
grep -E '(actix-web|axum|rocket|warp|tide|gotham)' Cargo.toml 2>/dev/null
```

### Java/Kotlin — parse pom.xml / build.gradle

```bash
# Maven
grep -E '(spring-boot|quarkus|micronaut|jakarta|javax.servlet|dropwizard|vert.x|play-framework)' pom.xml 2>/dev/null

# Gradle
grep -E '(spring-boot|quarkus|micronaut|ktor)' build.gradle build.gradle.kts 2>/dev/null
```

### Elixir — parse mix.exs

```bash
grep -E '(:phoenix|:plug|:absinthe)' mix.exs 2>/dev/null
```

### PHP — parse composer.json

```bash
grep -E '(laravel/framework|symfony/|slim/slim|cakephp/cakephp|yiisoft/yii2)' composer.json 2>/dev/null
```

---

## 3. Directory Structure

Map actual source and test directories by scanning what exists:

```bash
# Find source directories (exclude common non-source dirs)
find . -maxdepth 3 -type d \
  -not -path './.git/*' \
  -not -path './node_modules/*' \
  -not -path './vendor/*' \
  -not -path './.bundle/*' \
  -not -path './tmp/*' \
  -not -path './log/*' \
  -not -path './coverage/*' \
  -not -path './__pycache__/*' \
  -not -path './.venv/*' \
  -not -path './venv/*' \
  -not -path './target/*' \
  -not -path './dist/*' \
  -not -path './build/*' \
  -not -path './.next/*' \
  -not -path './.nuxt/*' \
  2>/dev/null | head -100
```

### Framework-specific directory patterns

**Rails:** `app/models/`, `app/controllers/`, `app/views/`, `app/services/`, `app/jobs/`, `app/helpers/`, `app/mailers/`, `config/`, `db/migrate/`, `lib/`

**Django:** `*/models.py`, `*/views.py`, `*/serializers.py`, `*/urls.py`, `*/admin.py`, `*/forms.py`, `*/tasks.py`, `*/templates/`, `*/management/commands/`

**FastAPI:** `app/routers/`, `app/models/`, `app/schemas/`, `app/services/`, `app/core/`, `app/api/`

**Express/Fastify:** `src/routes/`, `src/controllers/`, `src/models/`, `src/middleware/`, `src/services/`

**Next.js:** `app/` (App Router) or `pages/` (Pages Router), `components/`, `lib/`, `utils/`, `api/` or `pages/api/`

**NestJS:** `src/modules/`, `src/*.module.ts`, `src/*.controller.ts`, `src/*.service.ts`

**Go:** `cmd/`, `internal/`, `pkg/`, `api/`, `handlers/`, `models/`, `services/`

**Rust:** `src/`, `src/handlers/`, `src/models/`, `src/routes/`

**Spring Boot:** `src/main/java/**/controller/`, `src/main/java/**/service/`, `src/main/java/**/repository/`, `src/main/java/**/model/`

**Phoenix:** `lib/*/controllers/`, `lib/*/views/`, `lib/*/live/`, `lib/*/schemas/`

**Laravel:** `app/Http/Controllers/`, `app/Models/`, `app/Services/`, `routes/`, `resources/views/`

[MUST] Map found directories to pipeline categories:

```yaml
file_patterns:
  model: "<detected model directory>/**/*.<ext>"
  endpoint: "<detected endpoint directory>/**/*.<ext>"
  service: "<detected service directory>/**/*.<ext>"
  # ... etc
source_dirs: "<space-separated detected source dirs>"
```

---

## 4. Test Runner & Config

### Ruby

```bash
grep -E "gem ['\"]rspec['\"]" Gemfile && TEST_RUNNER=rspec
grep -E "gem ['\"]minitest['\"]" Gemfile && TEST_RUNNER=minitest
grep -E "gem ['\"]cucumber['\"]" Gemfile && TEST_RUNNER_E2E=cucumber
grep -E "gem ['\"]capybara['\"]" Gemfile && TEST_RUNNER_E2E=capybara

# Detect test directory
[ -d "spec" ] && TEST_DIR=spec
[ -d "test" ] && TEST_DIR=test

# RSpec config
[ -f ".rspec" ] && RSPEC_CONFIG=$(cat .rspec)
[ -f "spec/spec_helper.rb" ] && SPEC_HELPER=spec/spec_helper.rb
[ -f "spec/rails_helper.rb" ] && RAILS_HELPER=spec/rails_helper.rb
```

### JavaScript/TypeScript

```bash
# From package.json scripts and deps
grep -E '"(jest|vitest|mocha|ava|tap|playwright|cypress|@testing-library)' package.json

# Config files
[ -f "jest.config.js" ] || [ -f "jest.config.ts" ] || [ -f "jest.config.mjs" ] && TEST_RUNNER=jest
[ -f "vitest.config.js" ] || [ -f "vitest.config.ts" ] || [ -f "vitest.config.mts" ] && TEST_RUNNER=vitest
[ -f ".mocharc.yml" ] || [ -f ".mocharc.json" ] && TEST_RUNNER=mocha
[ -f "cypress.config.js" ] || [ -f "cypress.config.ts" ] && TEST_RUNNER_E2E=cypress
[ -f "playwright.config.ts" ] || [ -f "playwright.config.js" ] && TEST_RUNNER_E2E=playwright

# Test directory
[ -d "__tests__" ] && TEST_DIR=__tests__
[ -d "tests" ] && TEST_DIR=tests
[ -d "test" ] && TEST_DIR=test
[ -d "src/__tests__" ] && TEST_DIR=src/__tests__

# Test file pattern
find . -name "*.test.ts" -o -name "*.test.js" -o -name "*.spec.ts" -o -name "*.spec.js" 2>/dev/null | head -5
```

### Python

```bash
# pytest
[ -f "pytest.ini" ] || [ -f "conftest.py" ] && TEST_RUNNER=pytest
grep -q '\[tool\.pytest' pyproject.toml 2>/dev/null && TEST_RUNNER=pytest
grep -qE '^pytest' requirements.txt requirements-dev.txt 2>/dev/null && TEST_RUNNER=pytest

# unittest (default if no pytest)
[ -d "tests" ] && find tests/ -name "test_*.py" -print -quit 2>/dev/null | grep -q . && TEST_RUNNER=${TEST_RUNNER:-unittest}

# Test directory
[ -d "tests" ] && TEST_DIR=tests
[ -d "test" ] && TEST_DIR=test
```

### Go

```bash
# Go always uses go test
find . -name "*_test.go" -print -quit 2>/dev/null | grep -q . && TEST_RUNNER="go test"

# testify
grep 'stretchr/testify' go.mod 2>/dev/null && TEST_LIB=testify
```

### Rust

```bash
# Rust always uses cargo test — detect integration test dir
[ -d "tests" ] && TEST_DIR=tests
TEST_RUNNER="cargo test"
```

### Java/Kotlin

```bash
# JUnit
grep -q 'junit' pom.xml build.gradle build.gradle.kts 2>/dev/null && TEST_RUNNER=junit
[ -d "src/test" ] && TEST_DIR=src/test

# TestNG
grep -q 'testng' pom.xml build.gradle 2>/dev/null && TEST_RUNNER=testng
```

### Elixir

```bash
TEST_RUNNER="mix test"
[ -d "test" ] && TEST_DIR=test
```

### PHP

```bash
grep -E '"phpunit/phpunit"' composer.json 2>/dev/null && TEST_RUNNER=phpunit
[ -f "phpunit.xml" ] || [ -f "phpunit.xml.dist" ] && TEST_RUNNER=phpunit
grep -E '"pestphp/pest"' composer.json 2>/dev/null && TEST_RUNNER=pest
```

---

## 5. Coverage Tool

```bash
# Ruby
grep -E "gem ['\"]simplecov['\"]" Gemfile && COVERAGE=simplecov
[ -f "coverage/.resultset.json" ] && COVERAGE_PATH=coverage/.resultset.json

# JS/TS
grep -q '"c8"' package.json && COVERAGE=c8
grep -q '"istanbul"' package.json && COVERAGE=istanbul
grep -q '"nyc"' package.json && COVERAGE=nyc
grep -qE '"--coverage"' package.json && COVERAGE=jest-coverage  # jest built-in

# Python
grep -qiE '^(coverage|pytest-cov)' requirements.txt requirements-dev.txt 2>/dev/null && COVERAGE=pytest-cov
grep -qi 'pytest-cov\|coverage' pyproject.toml 2>/dev/null && COVERAGE=pytest-cov
[ -f ".coveragerc" ] && COVERAGE=coverage.py

# Go: built-in
# go test -cover

# Rust
grep -q 'cargo-tarpaulin\|llvm-cov' Cargo.toml 2>/dev/null && COVERAGE=tarpaulin

# Java
grep -q 'jacoco' pom.xml build.gradle build.gradle.kts 2>/dev/null && COVERAGE=jacoco

# PHP
grep -q 'phpunit' composer.json && COVERAGE=phpunit  # built-in with xdebug/pcov

# Elixir
grep -q ':excoveralls' mix.exs 2>/dev/null && COVERAGE=excoveralls
```

---

## 6. Security Tools

```bash
# Ruby
grep -E "gem ['\"]brakeman['\"]" Gemfile && SAST+=("brakeman")
grep -E "gem ['\"]bundler-audit['\"]" Gemfile && SAST+=("bundler-audit")
grep -E "gem ['\"]rubocop-security['\"]" Gemfile && SAST+=("rubocop-security")

# JS/TS
grep -q '"eslint-plugin-security"' package.json && SAST+=("eslint-security")
grep -q '"snyk"' package.json && SAST+=("snyk")
grep -q '"npm-audit"' package.json 2>/dev/null && SAST+=("npm-audit")
[ -f ".snyk" ] && SAST+=("snyk")

# Python
grep -qiE '^bandit' requirements.txt requirements-dev.txt 2>/dev/null && SAST+=("bandit")
grep -qiE '^safety' requirements.txt requirements-dev.txt 2>/dev/null && SAST+=("safety")
[ -f ".bandit" ] || [ -f "bandit.yaml" ] && SAST+=("bandit")

# Go
grep -q 'securego/gosec' go.mod 2>/dev/null && SAST+=("gosec")
command -v gosec >/dev/null 2>&1 && SAST+=("gosec")

# Rust
grep -q 'cargo-audit' Cargo.toml 2>/dev/null && SAST+=("cargo-audit")

# Java
grep -qE 'spotbugs|findsecbugs|owasp.*dependency-check' pom.xml build.gradle 2>/dev/null && SAST+=("spotbugs")

# PHP
grep -q '"roave/security-advisories"' composer.json 2>/dev/null && SAST+=("roave-security")

# Generic
[ -f ".semgrep.yml" ] || [ -f ".semgrep" ] && SAST+=("semgrep")
[ -f "trivy.yaml" ] && SAST+=("trivy")
[ -f ".gitleaks.toml" ] && SAST+=("gitleaks")
[ -f "sonar-project.properties" ] && SAST+=("sonarqube")
```

---

## 7. Linters & Formatters

```bash
# Ruby
grep -E "gem ['\"]rubocop['\"]" Gemfile && LINTER=rubocop
[ -f ".rubocop.yml" ] && LINTER=rubocop

# JS/TS
[ -f ".eslintrc" ] || [ -f ".eslintrc.js" ] || [ -f ".eslintrc.json" ] || [ -f ".eslintrc.yml" ] || [ -f "eslint.config.js" ] || [ -f "eslint.config.mjs" ] && LINTER=eslint
[ -f ".prettierrc" ] || [ -f ".prettierrc.json" ] || [ -f "prettier.config.js" ] && FORMATTER=prettier
grep -q '"biome"' package.json 2>/dev/null && LINTER=biome
[ -f "biome.json" ] && LINTER=biome

# Python
[ -f ".flake8" ] || grep -q 'flake8' pyproject.toml 2>/dev/null && LINTER=flake8
grep -q 'ruff' pyproject.toml 2>/dev/null && LINTER=ruff
[ -f "ruff.toml" ] || [ -f ".ruff.toml" ] && LINTER=ruff
grep -qi 'pylint' pyproject.toml requirements-dev.txt 2>/dev/null && LINTER=pylint
grep -qi 'black' pyproject.toml 2>/dev/null && FORMATTER=black
[ -f "mypy.ini" ] || grep -q '\[tool\.mypy\]' pyproject.toml 2>/dev/null && TYPE_CHECKER=mypy
grep -q 'pyright' pyproject.toml 2>/dev/null && TYPE_CHECKER=pyright

# Go
# gofmt/goimports are standard — always available
LINTER=gofmt
command -v golangci-lint >/dev/null 2>&1 && LINTER=golangci-lint
[ -f ".golangci.yml" ] || [ -f ".golangci.yaml" ] && LINTER=golangci-lint

# Rust
# rustfmt and clippy are standard
LINTER=clippy
FORMATTER=rustfmt

# Java/Kotlin
grep -q 'checkstyle\|ktlint\|spotless' pom.xml build.gradle build.gradle.kts 2>/dev/null && LINTER=checkstyle

# PHP
[ -f ".php-cs-fixer.php" ] || [ -f ".php_cs" ] && LINTER=php-cs-fixer
grep -q '"phpstan/phpstan"' composer.json 2>/dev/null && TYPE_CHECKER=phpstan

# Elixir
[ -f ".credo.exs" ] && LINTER=credo
[ -f ".formatter.exs" ] && FORMATTER=mix_format

# Generic
[ -f ".editorconfig" ] && EDITOR_CONFIG=true
```

---

## 8. Database

```bash
# From config files
# Rails: config/database.yml
[ -f "config/database.yml" ] && grep -iE '(postgresql|postgres|mysql|sqlite|mariadb|oracle|sqlserver)' config/database.yml

# Django: settings.py
find . -name "settings.py" -path "*/settings.py" 2>/dev/null | head -1 | xargs grep -iE "(postgresql|mysql|sqlite|oracle)" 2>/dev/null

# From dependency files
grep -iE '(pg|mysql2|sqlite3|oracle|sequel)' Gemfile 2>/dev/null
grep -iE '(pg|mysql|psycopg|sqlite|pymongo|motor|redis|asyncpg|aiopg)' requirements.txt pyproject.toml 2>/dev/null
grep -iE '(pgx|go-sqlite3|mysql-driver|mongo-driver|go-redis)' go.mod 2>/dev/null

# From docker-compose
[ -f "docker-compose.yml" ] || [ -f "docker-compose.yaml" ] && grep -iE '(postgres|mysql|mariadb|mongo|redis|elasticsearch|rabbitmq|kafka)' docker-compose.yml docker-compose.yaml 2>/dev/null

# From .env files (just DB type, not credentials)
[ -f ".env.example" ] && grep -iE 'DATABASE_URL|DB_HOST|DB_ENGINE|DB_DRIVER' .env.example 2>/dev/null
[ -f ".env.sample" ] && grep -iE 'DATABASE_URL|DB_HOST|DB_ENGINE|DB_DRIVER' .env.sample 2>/dev/null
```

---

## 9. Auth Pattern Detection

Detect authentication approach by reading existing code and tests:

```bash
# Search for auth middleware/libraries in dependencies
# Ruby
grep -E "gem ['\"]devise['\"]" Gemfile && AUTH_LIB=devise
grep -E "gem ['\"]cancancan['\"]" Gemfile && AUTHZ_LIB=cancancan
grep -E "gem ['\"]pundit['\"]" Gemfile && AUTHZ_LIB=pundit
grep -E "gem ['\"]jwt['\"]" Gemfile && AUTH_STRATEGY=token
grep -E "gem ['\"]omniauth['\"]" Gemfile && AUTH_STRATEGY=oauth

# JS/TS
grep -qE '"(passport|@auth/core|next-auth|lucia|clerk|auth0|@supabase/auth|jsonwebtoken|jose|@clerk/nextjs)' package.json && AUTH_LIB_DETECTED=true
grep -q '"passport"' package.json && AUTH_LIB=passport
grep -q '"next-auth"\|"@auth/core"' package.json && AUTH_LIB=next-auth
grep -q '"jsonwebtoken"\|"jose"' package.json && AUTH_STRATEGY=token

# Python
grep -qiE '(django-allauth|django-rest-framework-simplejwt|flask-login|flask-jwt-extended|fastapi-users|python-jose|authlib)' requirements.txt pyproject.toml 2>/dev/null

# Go
grep -qE '(golang-jwt/jwt|coreos/go-oidc|markbates/goth)' go.mod 2>/dev/null && AUTH_STRATEGY=token

# Search existing test files for auth patterns
# This reveals how tests currently authenticate
grep -rlE '(sign_in|login|authenticate|force_authenticate|set_auth_header|auth_headers|Authorization|Bearer|session\[|current_user)' \
  spec/ test/ tests/ __tests__/ src/**/*.test.* 2>/dev/null | head -10

# Read a sample test file to extract the actual auth helper
# (the onboarding agent will read these files to understand the pattern)
```

### Auth pattern inference

| Detected | Inference |
|----------|-----------|
| `devise` + `sign_in(user)` in specs | Session-based, helper = `sign_in(user)` |
| `login(user, pessoa)` in specs | Custom session, helper = `login(user, pessoa)` |
| `force_authenticate(user=user)` in tests | Token-based (DRF), helper = `self.client.force_authenticate(user=user)` |
| `jsonwebtoken`/`jose` + `Authorization` in tests | Token-based, extract header pattern |
| `next-auth` + session mock | Session-based, need session provider mock |
| `passport` | Depends on strategy — check passport config |
| No auth lib detected | Ask developer (Q2 becomes mandatory) |

---

## 10. Factory / Fixture Pattern

```bash
# Ruby
grep -E "gem ['\"]factory_bot['\"]" Gemfile && FACTORY_LIB=factory_bot
[ -d "spec/factories" ] && FACTORY_DIR=spec/factories
[ -d "test/factories" ] && FACTORY_DIR=test/factories
[ -d "spec/fixtures" ] && FIXTURE_STRATEGY=fixtures
[ -d "test/fixtures" ] && FIXTURE_STRATEGY=fixtures

# JS/TS
grep -q '"fishery"' package.json && FACTORY_LIB=fishery
grep -q '"@mswjs/data"' package.json && FACTORY_LIB=msw-data
grep -q '"factory.ts"' package.json && FACTORY_LIB=factory.ts
grep -q '"@faker-js/faker"\|"faker"' package.json && FAKER=true
# Check for factory files
find . -path "*/factories/*" -o -path "*/__factories__/*" -o -name "*.factory.ts" -o -name "*.factory.js" 2>/dev/null | head -5

# Python
grep -qiE '^(factory.boy|factory-boy|pytest-factoryboy)' requirements.txt requirements-dev.txt pyproject.toml 2>/dev/null && FACTORY_LIB=factory_boy
grep -qiE '^faker' requirements.txt requirements-dev.txt 2>/dev/null && FAKER=true
# Django fixtures
find . -path "*/fixtures/*.json" -o -path "*/fixtures/*.yaml" 2>/dev/null | head -5

# Go — typically hand-written helpers
grep -rl 'func.*Factory\|func.*Fixture\|func.*Create.*Test' . --include="*_test.go" 2>/dev/null | head -5

# PHP
grep -q '"fakerphp/faker"' composer.json 2>/dev/null && FAKER=true
# Laravel factories
[ -d "database/factories" ] && FACTORY_LIB=laravel_factories

# Elixir
grep -q ':ex_machina' mix.exs 2>/dev/null && FACTORY_LIB=ex_machina

# Analyze existing test files for data creation patterns
grep -rn 'create(\|factory(\|build(\|FactoryBot\|Factory\.' \
  spec/ test/ tests/ __tests__/ 2>/dev/null | head -10
```

---

## 11. API Style

```bash
# GraphQL
[ -f "schema.graphql" ] || [ -f "schema.gql" ] && API_STYLE=graphql
find . -name "*.graphql" -o -name "*.gql" 2>/dev/null | head -1 | grep -q . && API_STYLE=graphql
grep -q 'graphql\|apollo\|@nestjs/graphql\|ariadne\|strawberry\|absinthe\|graphql-go\|juniper\|async-graphql' \
  package.json requirements.txt pyproject.toml go.mod Cargo.toml Gemfile mix.exs 2>/dev/null && API_STYLE=graphql

# gRPC
find . -name "*.proto" 2>/dev/null | head -1 | grep -q . && API_STYLE=grpc
grep -q 'grpc\|protobuf\|tonic' package.json go.mod Cargo.toml requirements.txt 2>/dev/null && API_STYLE=grpc

# REST (default if endpoints exist)
API_STYLE=${API_STYLE:-rest}

# OpenAPI/Swagger
[ -f "openapi.yaml" ] || [ -f "openapi.json" ] || [ -f "swagger.yaml" ] || [ -f "swagger.json" ] && API_SPEC=openapi
grep -q 'swagger\|openapi\|@nestjs/swagger\|drf-spectacular\|rswag' \
  package.json requirements.txt pyproject.toml Gemfile 2>/dev/null && API_SPEC=openapi
```

---

## 12. CI/CD Platform

```bash
# GitHub Actions
[ -d ".github/workflows" ] && CI_PLATFORM=github
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null | head -10

# GitLab CI
[ -f ".gitlab-ci.yml" ] && CI_PLATFORM=gitlab

# CircleCI
[ -f ".circleci/config.yml" ] && CI_PLATFORM=circleci

# Travis CI
[ -f ".travis.yml" ] && CI_PLATFORM=travis

# Jenkins
[ -f "Jenkinsfile" ] && CI_PLATFORM=jenkins

# Bitbucket Pipelines
[ -f "bitbucket-pipelines.yml" ] && CI_PLATFORM=bitbucket

# Azure DevOps
[ -f "azure-pipelines.yml" ] && CI_PLATFORM=azure

# Buildkite
[ -f ".buildkite/pipeline.yml" ] && CI_PLATFORM=buildkite

# Vercel
[ -f "vercel.json" ] && DEPLOY_PLATFORM=vercel

# Netlify
[ -f "netlify.toml" ] && DEPLOY_PLATFORM=netlify

# Render
[ -f "render.yaml" ] && DEPLOY_PLATFORM=render

# Fly.io
[ -f "fly.toml" ] && DEPLOY_PLATFORM=fly

# Heroku
[ -f "Procfile" ] && DEPLOY_PLATFORM=heroku

# AWS
[ -f "serverless.yml" ] || [ -f "serverless.yaml" ] && DEPLOY_PLATFORM=serverless
[ -f "template.yaml" ] || [ -f "samconfig.toml" ] && DEPLOY_PLATFORM=aws-sam
[ -d ".aws" ] || [ -f "cdk.json" ] && DEPLOY_PLATFORM=aws-cdk
```

---

## 13. Container / Infra

```bash
[ -f "Dockerfile" ] && CONTAINER=docker
[ -f "docker-compose.yml" ] || [ -f "docker-compose.yaml" ] || [ -f "compose.yml" ] || [ -f "compose.yaml" ] && COMPOSE=true
[ -d "k8s" ] || [ -d "kubernetes" ] || [ -d "helm" ] && ORCHESTRATION=kubernetes
[ -f "skaffold.yaml" ] && ORCHESTRATION=skaffold
[ -f "Tiltfile" ] && ORCHESTRATION=tilt
[ -f "devcontainer.json" ] || [ -d ".devcontainer" ] && DEVCONTAINER=true
```

---

## 14. Monorepo Detection

```bash
# Workspaces
grep -q '"workspaces"' package.json 2>/dev/null && MONOREPO=npm-workspaces
[ -f "pnpm-workspace.yaml" ] && MONOREPO=pnpm-workspaces
[ -f "lerna.json" ] && MONOREPO=lerna
[ -f "turbo.json" ] && MONOREPO=turborepo
[ -f "nx.json" ] && MONOREPO=nx

# Multiple apps
ls apps/*/package.json packages/*/package.json 2>/dev/null | head -10

# Multiple Gemfiles (Rails engines)
find . -maxdepth 2 -name "Gemfile" 2>/dev/null | wc -l

# Multiple go.mod (Go workspace)
find . -maxdepth 3 -name "go.mod" 2>/dev/null | wc -l
[ -f "go.work" ] && MONOREPO=go-workspace

# If monorepo: ask which app/package is the primary target
```

---

## Output

The detection produces a `DETECTED` object with all findings:

```yaml
detected:
  language: <ruby|javascript|typescript|python|go|rust|java|kotlin|elixir|php|csharp|dart|swift>
  language_version: <version string or null>
  package_manager: <bundler|npm|yarn|pnpm|bun|pip|poetry|pipenv|uv|gomod|cargo|maven|gradle|mix|composer|dotnet|spm|pub>
  framework: <rails|express|fastify|next|nuxt|nest|django|flask|fastapi|gin|echo|fiber|chi|actix|axum|phoenix|laravel|spring|...>
  framework_version: <version string or null>
  framework_type: <backend|frontend|fullstack|mobile|desktop>
  orm: <activerecord|sequelize|prisma|drizzle|typeorm|sqlalchemy|django-orm|gorm|diesel|ecto|eloquent|null>
  database: <postgresql|mysql|sqlite|mongodb|redis|null>
  api_style: <rest|graphql|grpc>
  api_spec: <openapi|null>
  test_runner: <rspec|minitest|jest|vitest|mocha|pytest|unittest|go-test|cargo-test|junit|phpunit|pest|mix-test|null>
  test_runner_e2e: <cypress|playwright|capybara|selenium|null>
  test_dir: <path>
  test_file_pattern: <*_spec.rb|*.test.ts|test_*.py|*_test.go|...>
  coverage_tool: <simplecov|istanbul|c8|nyc|pytest-cov|coverage.py|jacoco|tarpaulin|null>
  coverage_path: <path or null>
  security_tools: [<brakeman|bandit|gosec|eslint-security|semgrep|snyk|...>]
  linter: <rubocop|eslint|biome|ruff|flake8|pylint|golangci-lint|clippy|checkstyle|credo|phpstan|null>
  formatter: <prettier|black|rustfmt|gofmt|mix_format|null>
  type_checker: <mypy|pyright|typescript|phpstan|null>
  auth_lib: <devise|passport|next-auth|django-allauth|jwt|null>
  auth_strategy: <session|token|oauth|null>
  authz_lib: <cancancan|pundit|casl|null>
  factory_lib: <factory_bot|fishery|factory_boy|ex_machina|laravel_factories|null>
  factory_dir: <path or null>
  faker: <true|false>
  ci_platform: <github|gitlab|circleci|travis|jenkins|bitbucket|azure|buildkite|null>
  deploy_platform: <vercel|netlify|heroku|fly|render|aws|null>
  container: <docker|null>
  compose: <true|false>
  monorepo: <npm-workspaces|pnpm-workspaces|turborepo|nx|lerna|go-workspace|null>
  source_dirs: "<space-separated>"
  file_patterns:
    model: "<glob>"
    endpoint: "<glob>"
    service: "<glob>"
    view: "<glob>"
    job: "<glob>"
    migration: "<glob>"
    config: "<glob>"
    test: "<glob>"
```

## What Can Be Fully Auto-Detected (skip question)

| Info | Detection Confidence |
|------|---------------------|
| Language | HIGH — from dependency files |
| Framework | HIGH — from dependency files |
| Test runner | HIGH — from configs and deps |
| Coverage tool | HIGH — from configs and deps |
| Security tools | HIGH — from configs and deps |
| Linter/Formatter | HIGH — from configs |
| Database type | MEDIUM — from deps and config |
| CI platform | HIGH — from config dirs |
| Directory structure | HIGH — from filesystem |
| Auth library | HIGH — from deps |
| Factory library | HIGH — from deps |
| API style | MEDIUM — from deps and file patterns |

## What Needs Confirmation or Questions

| Info | Why |
|------|-----|
| Auth test helper syntax | Even if we detect devise, the exact test helper may vary |
| Multi-tenancy model | Business logic, not detectable from deps alone |
| Tenant FK field | Business logic |
| Custom factory patterns | If no standard library detected |
| Report language | User preference |

## Reading Existing Tests for Pattern Extraction

[MUST] After detecting the test runner and test directory, read 2-3 existing test files to extract:

1. **Auth helper** — how tests authenticate (exact function call)
2. **Factory usage** — how tests create data (exact syntax)
3. **Assertion style** — assertion patterns used
4. **Setup/teardown** — how tests prepare state
5. **Tenant creation** — how tests create users/tenants

```bash
# Find the most recent test files (likely most representative)
find ${TEST_DIR} -name "${TEST_FILE_PATTERN}" -type f 2>/dev/null | \
  xargs ls -t 2>/dev/null | head -3
```

Read these files and extract patterns via grep:

```bash
# Auth patterns in test files
grep -n 'login\|sign_in\|authenticate\|force_authenticate\|auth_header\|Authorization' ${TEST_DIR}/**/* 2>/dev/null | head -10

# Factory patterns in test files
grep -n 'create(\|factory(\|build(\|FactoryBot\|Factory\.\|fixture' ${TEST_DIR}/**/* 2>/dev/null | head -10

# Tenant/user creation patterns
grep -n 'test_tenant\|create.*user\|create.*account\|create.*org' ${TEST_DIR}/**/* 2>/dev/null | head -10
```

This extraction allows the adapter to be generated with the **exact patterns** the project already uses, not generic templates.
