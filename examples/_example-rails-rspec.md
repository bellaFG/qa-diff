---
name: Rails + RSpec Examples (Reference)
description: Concrete RSpec examples for QA pipeline concepts — integration, IDOR, adversarial, boundary, export, mutation, load testing.
version: 1.0.0
stack: Ruby on Rails + RSpec + FactoryBot
---

# Rails + RSpec Examples

> Reference examples. Your project's concrete examples are generated in `examples/project.md` by `/qa-setup`.

## 1. Integration-Only — Real Stack (NEVER unit test)

```ruby
# CORRECT — integration with real persistence
RSpec.describe(OrdersController, type: :controller) do
  let(:user) { test_tenant }
  let(:pessoa) { create(:pessoa, user: user, email: "test@example.com") }
  let(:payment_method) { create(:documento) }

  before { login(user, pessoa) }

  it "creates order with data persisted in database" do
    expect {
      post(:create, params: { order: { amount: 100.0, payment_id: payment_method.id } })
    }.to(change(Order, :count).by(1))

    order = Order.last
    expect(order.amount).to(eq(100.0))
    expect(order.payment_id).to(eq(payment_method.id))
  end
end

# WRONG — unit test (FORBIDDEN)
# order = build_stubbed(:order, amount: 100.0)
# expect(order.total_with_tax).to(eq(110.0))

# CORRECT — same calculation tested via integration
it "calculates total with tax via request" do
  order = create(:order, amount: 100.0, tax_rate: 10.0, user: user)
  get(:show, params: { id: order.id }, format: :json)
  expect(parsed_body["total_with_tax"]).to(eq(110.0))
end
```

## 2. AAA with Real Factory (never build_stubbed)

```ruby
# CORRECT — AAA pattern with create (real persistence)
it "returns total with taxes" do
  item = create(:line_item, unit_price: 100.0, quantity: 10, user: user)
  result = item.total_with_tax
  expect(result).to(eq(1100.0))
end

# WRONG — build_stubbed doesn't touch database
# item = build_stubbed(:line_item, unit_price: 100.0)

# WRONG — stubs internal method
# allow(item).to(receive(:shipping_cost).and_return(50.0))

# CORRECT — use factory with data that produces the behavior
item = create(:line_item, :with_shipping, unit_price: 100.0, quantity: 10, user: user)
```

## 3. Spec Levels (CRITICAL / IMPORTANT / COVERAGE)

```ruby
# CRITICAL — verify bind parameters (SQL injection prevention)
it "uses bind parameters instead of interpolation" do
  conn = ActiveRecord::Base.connection
  allow(conn).to(receive(:exec_query).and_wrap_original) do |original, sql, name, binds|
    expect(sql).not_to(include("'#{"))  # no string interpolation
    original.call(sql, name, binds)
  end
  get(:index, params: { start_date: "2024-01-01" }, format: :json)
  expect(response).to(have_http_status(:ok))
end

# IMPORTANT — verify filter with real data
it "returns only active products" do
  active = create(:product, active: true, user: user)
  create(:product, active: false, user: user)
  get(:index, params: { active: true }, format: :json)
  expect(parsed_body.map { |p| p["id"] }).to(eq([active.id]))
end

# COVERAGE — exercise path
it "renders JS template" do
  get(:index, params: { start_date: "2024-01-01" }, format: :js)
  expect(response).to(have_http_status(:ok))
end
```

## 4. Adversarial — 3-Assertion Template

```ruby
context "when input is invalid (null amount)" do
  it "handles without exception, returns error, no corruption" do
    # 1. No unhandled exception
    expect {
      post(:create, params: { order: { amount: nil } })
    }.not_to(raise_error)

    # 2. Friendly response
    expect(response.status).to(be_in([200, 302, 400, 422]))

    # 3. No corrupted data
    expect(Order.where(amount: nil)).not_to(exist)
  end
end
```

## 5. IDOR — Cross-Tenant Spec

```ruby
RSpec.describe(OrdersController, type: :controller) do
  let(:user) { test_tenant }
  let(:pessoa) { create(:pessoa, user: user, email: "test@example.com") }
  let(:other_user) { yet_another_tenant }

  before { login(user, pessoa) }

  describe "IDOR: cross-tenant access" do
    context "when accessing resource from another tenant via params[:id]" do
      let(:foreign_order) { create(:order, user: other_user) }

      it "returns 404 or redirects" do
        get(:show, params: { id: foreign_order.id })
        expect(response.status).to(be_in([302, 403, 404]))
      end
    end

    context "when associating FK from another tenant" do
      let(:foreign_product) { create(:product, user: other_user) }

      it "does not persist association with foreign resource" do
        post(:create, params: { order: { product_id: foreign_product.id } })
        expect(Order.where(product_id: foreign_product.id, user: user)).not_to(exist)
      end
    end

    context "when modifying resource from another tenant" do
      let(:foreign_order) { create(:order, user: other_user) }

      it "does not allow modification" do
        put(:update, params: { id: foreign_order.id, order: { name: "hacked" } })
        expect(response.status).to(be_in([302, 403, 404]))
        expect(foreign_order.reload.name).not_to(eq("hacked"))
      end
    end
  end
end
```

## 6. Integration Spec — No-Mock Stack

```ruby
# CORRECT — real auth, real data, real database, zero mocks
RSpec.describe(InvoicesController, type: :controller) do
  let(:user) { test_tenant }
  let(:pessoa) { create(:pessoa, user: user, email: "test@example.com") }
  let(:payment_method) { create(:documento) }
  let(:account) { create(:account, user: user) }

  before { login(user, pessoa) }

  describe "POST #create" do
    let(:valid_params) do
      { invoice: { amount: 100.0, payment_id: payment_method.id, account_id: account.id } }
    end

    it "creates invoice with persisted data" do
      expect { post(:create, params: valid_params) }.to(change(Invoice, :count).by(1))

      invoice = Invoice.last
      expect(invoice.amount).to(eq(100.0))
      expect(invoice.payment_id).to(eq(payment_method.id))
    end
  end
end

# WRONG — mock auth (H9): allow(controller).to(receive(:current_user).and_return(user))
# WRONG — instance_double (H9): invoice = instance_double('Invoice', amount: 100.0)
# WRONG — build_stubbed: invoice = build_stubbed(:invoice, amount: 100.0)
# ONLY ACCEPTABLE — external deps: allow(ExternalApi).to(receive(:call).and_return({status: "ok"}))
```

## 7. H10 — Mock of Newly-Created Method

```ruby
# If diff creates method `calculate_interest` on Invoice:

# WRONG (H10) — tests the mock, not the code
it "calculates interest" do
  invoice = create(:invoice, amount: 1000.0)
  allow(invoice).to(receive(:calculate_interest).and_return(50.0))
  expect(invoice.calculate_interest).to(eq(50.0))
end

# CORRECT — exercises the real method with real data
it "calculates interest based on rate" do
  invoice = create(:invoice, amount: 1000.0, interest_rate: 5.0, user: user)
  expect(invoice.calculate_interest).to(eq(50.0))
end
```

## 8. Boundary Values — Integration

```ruby
context "when amount is 0" do
  it "handles zero amount" do
    post(:create, params: { invoice: { amount: 0, user_id: user.id } })
    expect(response.status).to(be_in([200, 302, 422]))
  end
end

context "when amount is negative" do
  it "rejects negative amount" do
    post(:create, params: { invoice: { amount: -1 } })
    expect(Invoice.where(amount: -1)).not_to(exist)
  end
end

context "when name has XSS" do
  it "sanitizes without exception" do
    expect {
      post(:create, params: { product: { name: "<script>alert(1)</script>" } })
    }.not_to(raise_error)
    expect(Product.last&.name).not_to(include("<script>"))
  end
end

context "when date is invalid" do
  it "rejects without exception" do
    expect {
      post(:create, params: { event: { date: "not-a-date" } })
    }.not_to(raise_error)
    expect(response.status).to(be_in([200, 302, 422]))
  end
end
```

## 9. Export Validation — Real File

```ruby
it "exports XLSX with correct data" do
  invoice = create(:invoice, amount: 1500.0, user: user)
  get(:index, params: { format: :xlsx })

  expect(response.content_type).to(include("spreadsheet").or(include("excel")))
  expect(response.body.size).to(be > 100)

  workbook = RubyXL::Parser.parse_buffer(response.body)
  sheet = workbook.worksheets.first
  values = sheet.map { |row| row&.cells&.map(&:value) }.compact
  expect(values.flatten).to(include(1500.0))
end

it "exports valid PDF" do
  create(:invoice, user: user)
  get(:show, params: { id: Invoice.last.id, format: :pdf })
  expect(response.content_type).to(include("pdf"))
  expect(response.body[0..3]).to(eq("%PDF"))
end
```

## 10. Job with Real Side-Effects

```ruby
it "processes import with real side-effects" do
  import = create(:import, user: user, file: fixture_file("data.xlsx"))

  ImportJob.perform_now(import.id)

  import.reload
  expect(import.status).to(eq("completed"))
  expect(import.imported_count).to(be > 0)
  expect(Product.where(user: user).count).to(be > 0)
end
```

## 11. Mutation Testing — Property-Based

```ruby
context "property: interest is never negative (100 inputs)" do
  100.times do
    amount = rand(0.01..999_999.99).round(2)
    rate = rand(0.0..100.0).round(2)

    it "interest >= 0 for amount=#{amount}, rate=#{rate}" do
      account = create(:account, amount: amount, interest_rate: rate, user: user)
      expect(account.calculated_interest).to(be >= 0)
    end
  end
end

context "property: sum of installments == total (100 inputs)" do
  100.times do
    amount = rand(0.01..999_999.99).round(2)
    count = rand(1..36)

    it "amount=#{amount}, installments=#{count}" do
      invoice = create(:invoice, amount: amount, user: user)
      installments = invoice.generate_installments(count)
      total = installments.sum(&:amount)
      expect(total).to(be_within(0.01).of(amount))
    end
  end
end
```

## 12. Load Testing

### 12.1 Benchmark Ruby (inline)

```ruby
require_relative "../../spec/rails_helper"
require "benchmark"

ITERATIONS = 50
VOLUME = 500

user = test_tenant
VOLUME.times { |i| create(:resource, user: user, name: "Load #{i}") }

results = Benchmark.bm(25) do |x|
  x.report("GET /resources") do
    ITERATIONS.times { get "/resources", headers: auth_headers(user) }
  end
end

avg_ms = (results.first.real / ITERATIONS * 1000).round(2)
puts "Avg: #{avg_ms}ms | Total: #{results.first.real.round(2)}s"
puts avg_ms < 100 ? "STATUS: GOOD" : avg_ms < 500 ? "STATUS: ACCEPTABLE" : "STATUS: BAD"
```

### 12.2 Query Counter (N+1)

```ruby
require_relative "../../spec/rails_helper"

VOLUME = 200
user = test_tenant
VOLUME.times { create(:resource, user: user) }

queries = []
ActiveSupport::Notifications.subscribe("sql.active_record") do |*, payload|
  queries << payload[:sql] unless payload[:sql].match?(/SCHEMA|BEGIN|COMMIT|SAVEPOINT/)
end

# Execute the code under test
MyService.new(user).call

puts "Queries: #{queries.size} | Unique: #{queries.uniq.size}"
puts queries.size > 10 ? "N+1 DETECTED" : "OK"
```
