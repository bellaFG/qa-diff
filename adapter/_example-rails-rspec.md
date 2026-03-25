---
name: Rails + RSpec Adapter (Reference Example)
description: Complete adapter for a Ruby on Rails project using RSpec + FactoryBot. NOT loaded automatically — use as reference for /qa-setup or manual adapter creation.
version: 1.0.0
stack: Ruby on Rails + RSpec + FactoryBot
---

# Rails + RSpec Adapter

> This is a **reference example**. Projects should generate their own `adapter/project.md` via `/qa-setup`.

## file_patterns

```yaml
file_patterns:
  model: "app/models/**/*.rb"
  endpoint: "app/controllers/**/*.rb"
  service: "app/models/**/*_{service,calculator,generator,builder}.rb"
  view: "app/views/**/*.{erb,haml}"
  script: "app/javascript/**/*.{js,vue}"
  migration: "db/migrate/*.rb"
  helper: "app/helpers/**/*.rb"
  job: "app/jobs/**/*.rb"
  config: "config/**/*.{rb,yml}"
  test: "spec/**/*_spec.rb"

source_dirs: "app/ lib/ config/"
```

## auth_pattern

```yaml
auth_pattern:
  setup: |
    let(:user) { test_tenant }
    let(:pessoa) { create(:pessoa, user: user, email: "test@example.com") }
    before { login(user, pessoa) }
  cross_tenant: |
    let(:other_user) { yet_another_tenant }
    let(:foreign_resource) { create(:factory_name, user: other_user) }
  notes: |
    - `test_tenant` and `yet_another_tenant` are helper methods that create complete tenant users
    - `login(user, pessoa)` sets up a real session (not a mock)
    - NEVER use `allow(controller).to receive(:current_user)` — always use `login()`
    - `skip_authorization(controller)` is acceptable when CanCanCan authorization is not the focus
```

## idor_detection

```yaml
idor_detection:
  scan_command: "grep -n 'params\\[:.*_id\\]' {file}"
  safe_patterns:
    - "current_user.association.find(params[:id])"
    - "current_user.association.where(id: params[:id])"
  vulnerable_patterns:
    - "Model.find(params[:id])"
    - "Model.where(id: params[:id])"
    - "params[:campo_id] in assign without tenant validation"
```

## test_execution

```yaml
test_execution:
  command: "bundle exec rspec {specs} --require ./spec/rails_helper --format documentation"
  coverage_flag: "COVERAGE=true"
  temp_dir: "/tmp/qa-specs"
  structure:
    - controllers/
    - requests/
    - models/
    - load/
  file_suffix: "_spec.rb"
  framework: "rspec"
```

## assertions

```yaml
assertions:
  mock_assertion_pattern: "have_received"
  last_record_pattern: "\\.(last|first)(?![_a-zA-Z])"
  weak_assertions:
    - "be_present"
    - "be_truthy"
    - "be_nil"
    - "be_falsy"
    - "be_empty"
  status_only_pattern: "have_http_status"
  mock_syntax: "allow\\(.+\\)\\.to receive|instance_double|class_double"
  auth_mock_pattern: "allow\\(controller\\)\\.to receive\\(:current_user\\)"
  exact_value_assertion: "eq("
```

## factory_pattern

```yaml
factory_pattern:
  create: "create(:factory_name, attribute: value, user: user)"
  build_forbidden: "build_stubbed"
  discovery_command: "grep -rl 'factory :{model_name}' spec/factories/ | head -5"
  notes: |
    - ALWAYS use `create()` for real persistence (touches database)
    - NEVER use `build_stubbed()` — it doesn't persist to the database
    - Use factory traits for common variations: `create(:invoice, :with_installments)`
    - Use `let` for lazy evaluation, `let!` for side-effects needed in setup
```

## spec_type_mapping

```yaml
spec_type_mapping:
  model: "controller/request spec via HTTP; model spec with create() only if no controller exists"
  endpoint: "controller spec (type: :controller)"
  helper: "integration spec within controller context"
  job: "job spec with perform_now (real execution)"
  view: "controller spec verifying response.body"
```

## coverage

```yaml
coverage:
  tool: "simplecov"
  result_path: "coverage/.resultset.json"
  extraction_script: |
    ruby -e "
    require 'json'
    resultset = JSON.parse(File.read('coverage/.resultset.json'))
    coverage_data = resultset.values.first['coverage']
    ARGV.each do |file|
      entry = coverage_data.find { |k, _| k.include?(file) }
      next puts \"#{file}: not found\" unless entry
      lines = entry[1]['lines']
      base = ENV.fetch('QA_DIFF_BASE', 'main')
      diff = %x(git diff origin/#{base}...HEAD -U0 -- #{file})
      nums = diff.scan(/^@@\\s+-\\d+(?:,\\d+)?\\s+\\+(\\d+)(?:,(\\d+))?\\s+@@/).flat_map { |s, c|
        c = (c || 1).to_i; (s.to_i...(s.to_i + c)).to_a
      }
      diff_lines = nums.map { |n| [n, lines[n-1]] }.select { |_,v| !v.nil? }
      next puts \"#{file}: 0 executable lines\" if diff_lines.empty?
      total = diff_lines.size
      covered = diff_lines.count { |_,v| v.to_i > 0 }
      pct = (covered.to_f / total * 100).round(1)
      puts \"#{file}: #{pct}% (#{covered}/#{total})\"
    end
    " {files}
```

## security_tools

```yaml
security_tools:
  - name: "brakeman"
    command: "bundle exec brakeman --no-progress --quiet -w 2 --only SqlInjection,RemoteCodeExecution,CrossSiteScripting,UnsafeReflection"
  - name: "rubocop-security"
    command: "bundle exec rubocop --only Security app/controllers/ app/models/"
```

## mutation_testing

```yaml
mutation_testing:
  command: "bundle exec mutant run --include app --require rails_helper --use rspec \"{class}\" --since origin/main"
  fallback: "property-based testing manual (loop with random inputs)"
  available: false  # Set to true if mutant-rspec is in Gemfile
```

## load_testing

```yaml
load_testing:
  benchmark_template: |
    require_relative "../../spec/rails_helper"
    require "benchmark"
    ITERATIONS = 50
    results = Benchmark.bm(25) do |x|
      x.report("endpoint") { ITERATIONS.times { get "{path}", headers: auth_headers } }
    end
    avg_ms = (results.first.real / ITERATIONS * 1000).round(2)
    puts "Avg: #{avg_ms}ms"
  query_counter_template: |
    require_relative "../../spec/rails_helper"
    queries = []
    ActiveSupport::Notifications.subscribe("sql.active_record") do |*, payload|
      queries << payload[:sql] unless payload[:sql].match?(/SCHEMA|BEGIN|COMMIT|SAVEPOINT/)
    end
    # Execute the code under test here
    puts "Queries: #{queries.size} | Unique: #{queries.uniq.size}"
```

## lint

```yaml
lint:
  command: "bundle exec rubocop {files}"
```

## next_steps

```yaml
next_steps:
  - "/check-ci"
  - "/review-kody"
  - "/review-humano"
  - "/pr-ready"
```

## Integration Rules (Rails-Specific)

These rules are HARD requirements for Rails integration specs:

1. **NEVER** `build_stubbed` — use `create()` for real persistence
2. **NEVER** `allow(controller).to receive(:current_user)` — use `login(user, pessoa)`
3. **NEVER** `instance_double` for internal models — use `create(:factory)`
4. **NEVER** test model in isolation if a controller exists
5. `allow` ONLY for external deps without global stubs (SDK, binaries, paid APIs)
6. The only acceptable `allow` for internal code: `skip_authorization(controller)` when CanCanCan is not the focus
