# k6-lab: Performance Testing with Grafana K6

Performance testing repository for ITDS362 Software Quality Assurance and Testing.
Tests are conducted against [QuickPizza](https://quickpizza.grafana.com), a public demo site provided by Grafana for K6 practice.

## Repository Contents

```
k6-lab/
├── README.md              # Project overview and instructions
├── RESULTS.md             # Experimental results and reflection answers
├── 01-first-test.js       # Step 0.1-0.3: First HTTP GET test and load control
├── 04-pizza-api.js        # Step 0.4: API POST testing with JSON response checks
├── 05-no-token.js         # Step 0.5: Security test verifying 401 Unauthorized handling
├── 06-thresholds.js       # Step 0.6: Pass/fail criteria using performance thresholds
├── 07-requirement.js      # Step 0.7: Translating product requirements into thresholds
├── 08-bug-hunt.js         # Step 0.8: Fixed bug hunt script testing pizza restrictions
├── explore.js             # Step 0.8: Utility script discovering API toppings and tools
├── journey.js             # Part A: Modular multi-step user journey module
├── a1-smoke.js            # Step A1: Smoke test (1 user for 30s)
├── a2-average.js          # Step A2: Average load test (ramps to 3 users for 2m)
├── a3-stress.js           # Step A3: Stress test (ramps to 5 users for 2m)
├── a4-spike.js            # Step A4: Spike test (1 user -> sudden surge to 5 -> recovery)
├── a5-soak.js             # Step A5: Soak test (2 users for 10 minutes)
└── reports/               # HTML dashboards exported from K6 runs
    ├── a2-average.html
    ├── a3-stress.html
    ├── a4-spike.html
    └── a5-soak.html
```

## Running the Tests

Make sure [k6](https://k6.io/) is installed on your system.

### Part 0 Basics
```powershell
# Step 0.1: Basic single request test
k6 run 01-first-test.js

# Step 0.2: More users for longer
k6 run --vus 5 --duration 20s 01-first-test.js

# Step 0.3: Fixed iterations
k6 run --vus 5 --iterations 20 01-first-test.js

# Step 0.4: API POST test
k6 run 04-pizza-api.js

# Step 0.5: Expecting 401 without token
k6 run 05-no-token.js

# Step 0.6: Performance thresholds
k6 run 06-thresholds.js
k6 run -e P95=300 06-thresholds.js

# Step 0.7: Product requirement thresholds
k6 run 07-requirement.js

# Step 0.8: Fixed bug hunt & API exploration
k6 run explore.js
k6 run --vus 5 --duration 20s 08-bug-hunt.js
```

### Part A User Journey & Load Test Types
```powershell
# Step A1: Smoke Test
k6 run a1-smoke.js

# Step A2: Average Load Test (generates HTML report)
k6 run --out web-dashboard=export=reports/a2-average.html a2-average.js

# Step A3: Stress Test (generates HTML report)
k6 run --out web-dashboard=export=reports/a3-stress.html a3-stress.js

# Step A4: Spike Test (generates HTML report)
k6 run --out web-dashboard=export=reports/a4-spike.html a4-spike.js

# Step A5: Soak Test (generates HTML report)
k6 run --out web-dashboard=export=reports/a5-soak.html a5-soak.js
```

## Results & Analysis

Detailed metrics, tables, and reflections are documented in [RESULTS.md](RESULTS.md).
HTML reports for Steps A2 to A5 can be viewed directly in any web browser from the `reports/` folder.