# Part 95: Advanced Testing Strategies
## ขั้นตอนที่ 3401-3440

> **ระดับ:** ระดับโลก (World-Class)  
> **เวลาเรียน:** 5-6 ชั่วโมง  
> **เป้าหมาย:** Test Spring Boot applications at production scale

---

## ขั้นตอนที่ 3401: Testing Maturity Model

```
Testing Maturity Levels:

Level 1: Unit Tests only
  - Fast, but miss integration bugs
  
Level 2: Unit + Integration Tests
  - More confidence, slower CI
  
Level 3: + Contract Tests
  - Catch API breaking changes early
  
Level 4: + Load/Performance Tests  
  - Prevent performance regressions
  
Level 5: + Chaos Tests
  - Validate resilience under failure
  
Level 6: + Synthetic Monitoring
  - Real user scenario verification in prod
  
World-class = Level 5-6 with automated gates
```

---

## ขั้นตอนที่ 3402: Chaos Testing with Chaos Monkey

```java
// Chaos Monkey for Spring Boot - inject failures in test/staging
// pom.xml:
// <dependency>
//   <groupId>de.codecentric</groupId>
//   <artifactId>chaos-monkey-spring-boot</artifactId>
// </dependency>

// application-chaos.yml
/*
chaos:
  monkey:
    enabled: true
    assaults:
      level: 5              # 1-10, how often to assault
      latency-active: true
      latency-range-start: 1000
      latency-range-end: 3000
      exceptions-active: true
      kill-application-active: false
    watcher:
      service: true
      repository: true
      rest-controller: false  # don't chaos the API layer
*/

// Chaos test: verify circuit breaker activates under chaos
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles({"test", "chaos"})
class CircuitBreakerChaosTest {

    @Autowired private TestRestTemplate restTemplate;
    @Autowired private CircuitBreakerRegistry registry;

    @Test
    void circuitBreaker_ShouldOpen_UnderChaos() throws InterruptedException {
        CircuitBreaker cb = registry.circuitBreaker("productService");

        // Fire enough requests to trigger circuit breaker
        for (int i = 0; i < 20; i++) {
            restTemplate.getForObject("/api/v1/products/1", String.class);
        }

        Thread.sleep(1000);

        // Circuit breaker should be OPEN now
        assertThat(cb.getState())
            .isIn(CircuitBreaker.State.OPEN, CircuitBreaker.State.HALF_OPEN);
    }
}
```

---

## ขั้นตอนที่ 3403: Load Testing Gates in CI

```yaml
# GitHub Actions: fail build if p99 latency > 200ms
name: Performance Gate

on:
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Start application
        run: docker-compose up -d
        
      - name: Wait for app
        run: |
          until curl -sf http://localhost:8080/actuator/health; do
            sleep 2
          done
          
      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.0
        with:
          filename: tests/load/baseline.js
          flags: --out json=results.json
          
      - name: Check performance thresholds
        run: |
          P99=$(jq '[.[] | select(.metric=="http_req_duration") | .data.value] | sort | .[(length * 0.99 | floor)]' results.json)
          echo "p99 latency: ${P99}ms"
          if (( $(echo "$P99 > 200" | bc -l) )); then
            echo "FAIL: p99 latency ${P99}ms exceeds 200ms threshold"
            exit 1
          fi
```

```javascript
// tests/load/baseline.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 50 },   // ramp up
        { duration: '60s', target: 50 },   // steady state
        { duration: '10s', target: 0 },    // ramp down
    ],
    thresholds: {
        http_req_duration: ['p(99)<200'],  // 99% under 200ms
        http_req_failed: ['rate<0.01'],    // <1% errors
    },
};

export default function () {
    const res = http.get('http://localhost:8080/api/v1/products?page=0&size=20');
    check(res, {
        'status 200': (r) => r.status === 200,
        'has products': (r) => JSON.parse(r.body).content.length > 0,
    });
    sleep(0.1);
}
```

---

## ขั้นตอนที่ 3404: Synthetic Monitoring

```java
// Playwright synthetic tests that run against production
// Verify critical user journeys work end-to-end

// tests/synthetic/CheckoutFlow.java (run every 5 minutes via cron)
@Test
void syntheticTest_BrowseAndCheckout() {
    try (Playwright playwright = Playwright.create()) {
        Browser browser = playwright.chromium().launch();
        Page page = browser.newPage();

        // Test product search
        page.navigate(PROD_URL + "/products?q=laptop");
        assertThat(page.locator(".product-card")).hasCount(GreaterThan(0));

        // Test add to cart
        page.locator(".product-card").first().click();
        page.locator("#add-to-cart").click();
        assertThat(page.locator(".cart-count")).hasText("1");

        // Alert if any step fails
        browser.close();
    }
}
```

---

## ขั้นตอนที่ 3405: Testing with Production Data Snapshots

```bash
#!/bin/bash
# Create anonymized production data snapshot for testing

# 1. Create snapshot
pg_dump --data-only --table=products \
    -h prod-db.example.com \
    -U readonly_user \
    myapp > /tmp/prod-snapshot.sql

# 2. Anonymize PII
sed -i "s/[a-zA-Z0-9._%+-]\+@[a-zA-Z0-9.-]\+\.[a-zA-Z]\+/anon@test.com/g" /tmp/prod-snapshot.sql
sed -i "s/[0-9]\{10\}/0000000000/g" /tmp/prod-snapshot.sql  # phone numbers

# 3. Load into test DB
psql -h test-db.example.com -U test_user test_db < /tmp/prod-snapshot.sql
```

```java
// Use snapshot in integration tests
@SpringBootTest
@Sql("/test-data/prod-snapshot-anonymized.sql")
class ProductSearchIntegrationTest {

    @Test
    void search_WithProductionDataVolume_ShouldRespondFast() {
        long start = System.currentTimeMillis();
        Page<Product> result = productService.search("laptop", PageRequest.of(0, 20));
        long elapsed = System.currentTimeMillis() - start;

        assertThat(elapsed).isLessThan(100);  // < 100ms even with prod data volume
        assertThat(result.getContent()).isNotEmpty();
    }
}
```

---

## ขั้นตอนที่ 3406-3440: Security Testing in CI

```yaml
# OWASP ZAP scan in CI
- name: Run OWASP ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.10.0
  with:
    target: 'http://localhost:8080'
    rules_file_name: '.zap/rules.tsv'
    cmd_options: '-I'  # don't fail on warnings
    
# Dependency vulnerability scan
- name: OWASP Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    path: '.'
    format: 'HTML'
    
# Secret scanning
- name: Detect secrets
  uses: reviewdog/action-detect-secrets@master
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    reporter: github-pr-review
    fail_on_error: true
```

---

*[← Part 94: Machine Learning](./part-94-machine-learning.md) | [Part 96: Capstone Design →](./part-96-capstone-design.md)*
