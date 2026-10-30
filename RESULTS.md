# K6 lab results

## Part 0
- Step 0.1, http_req_duration: 639.7ms (avg=639.7ms, min=639.7ms, med=639.7ms, max=639.7ms, p(90)=639.7ms, p(95)=639.7ms)
- Step 0.2, requests, rate and p(95), and why the rate stays below 5:
  - Total requests: 69
  - Request rate: 3.25 req/s (3.245534/s)
  - p(95): 966.87ms
  - Why the rate stays below 5: Each virtual user runs in a loop executing iterations sequentially. Each iteration includes an HTTP GET request (taking ~470ms on average) plus an explicit think time pause of `sleep(1)` (1000ms), resulting in an iteration duration of ~1.52s. Because each VU can only make roughly `1 / 1.52s ≈ 0.65` requests per second, 5 concurrent VUs generate approximately `5 * 0.65 ≈ 3.25` requests per second, which is below 5 req/s.
- Step 0.6, exit code of the normal run and of the run with P95=300:
  - Exit code of the normal run: `0` (all thresholds passed, measured p(95) was 586.46ms < 1500ms)
  - Exit code of the run with P95=300: `99` (threshold `p(95)<300` failed because measured p(95) was 445.31ms > 300ms)
- Step 0.7, rules that passed, p(95), p(99), exit code after your change:
  - Rules that passed: All rules passed (`http_req_failed` < 0.01 with 0.00%, `http_req_duration` p(95) < 1000ms, and p(99) < 1500ms)
  - Measured p(95): 535.94ms
  - Measured p(99): 823.47ms
  - Exit code after change: `99` (changed requirement threshold to `p(95)<200`, causing k6 to exit with code 99 due to crossed threshold as measured p(95) was 1.16s)
- Step 0.8, what was wrong, and how the checks showed it:
  - What was wrong: The request payload in `08-bug-hunt.js` passed `excludedIngredients: ['pepperoni']` and `excludedTools: ['knife']` in all-lowercase. Calling `/api/ingredients/topping` and `/api/tools` via `explore.js` revealed that QuickPizza uses case-sensitive naming (`'Pepperoni'` and `'Knife'`). Because the API performs case-sensitive matching, the lowercase entries were ignored, causing the service to still generate pizzas containing Pepperoni and requiring a Knife.
  - How the checks showed it: Under load (5 VUs for 20 seconds, generating 65 requests), the checks `no pepperoni` and `no knife needed` failed repeatedly (pepperoni check failed 8 times, knife check failed 22 times). A single manual request might randomly return a pizza without pepperoni and without a knife by pure chance, giving a false sense of correctness. Load testing repeated the request enough times to reliably expose the bug. After updating the values to `'Pepperoni'` and `'Knife'`, all 126 checks passed (100.00%).

## Part A results
| Step | Test type | Users | Req/s | p(95) | Max | Failed | Checks |
| ---- | ------------ | ----- | ----- | ----- | --- | ------ | ------ |
| A1 | Smoke | 1 | 0.48 | 944.45ms | 956.71ms | 0.00% | 100.00% |
| A2 | Average load | 3 | 1.17 | 989.62ms | 1.74s | 0.00% | 100.00% |
| A3 | Stress | 5 | 1.79 | 1.53s | 3.90s | 0.00% | 100.00% |
| A4 | Spike | 5 | 0.91 | 1.24s | 1.69s | 0.00% | 100.00% |
| A5 | Soak | 2 | 0.82 | 1.50s | 5.44s | 0.00% | 100.00% |

## Part B
1. The results table for Steps A1 to A5 has been populated above with the exact measurements recorded during execution.

2. In Step 0.2, the request rate stayed below 5 requests per second because each virtual user executes an iteration synchronously. In each iteration, the user sends an HTTP GET request (taking ~470ms) and immediately executes `sleep(1)` (1000ms), giving an average iteration duration of ~1.52s. One VU can therefore complete only ~0.65 requests per second (`1 / 1.52s`). With 5 concurrent VUs, the aggregate throughput is `5 * 0.65 ≈ 3.25` requests per second. The think time and network round-trip time prevent the system from reaching 1 request/VU/second.

3. Yes, the response time grew noticeably from the average-load test to the stress test:
   - Average load (3 VUs): p(95) was 989.62ms, max was 1.74s.
   - Stress test (5 VUs): p(95) rose to 1.53s, max rose to 3.90s.
   In terms of the **operational envelope**, the system operates smoothly under average load (3 VUs) well inside its operational envelope where resources (CPU, worker threads, database locks) process requests without significant queuing. When load is pushed to 5 VUs during the stress test, the system approaches the boundary of its operational envelope. Resource contention and request queuing cause latency to rise and the response time thresholds to cross, demonstrating the graceful slowdown characteristic of reaching the edge of the operational envelope.

4. In the spike test, response time rose during the sudden surge to 5 virtual users (reaching a peak of 1.69s). After dropping back to 1 VU (at the 1m10s mark), the response time quickly returned to its baseline normal level (~400–500ms) within approximately 10 to 15 seconds, demonstrating that the system successfully recovered from the traffic burst without permanent degradation.

5. In our 10-minute soak test, the error rate stayed at 0.00% throughout the entire test, and response times remained stable without progressive degradation between the initial minutes and the final minutes (p(95) was 1.50s, checks succeeded 100%).
   A soak test running for several hours or days can reveal defects that only appear after extended continuous operation, such as:
   - **Memory leaks** (gradual consumption of heap memory leading eventually to an Out-Of-Memory crash)
   - **Connection pool / thread exhaustion** (unreleased database connections or thread leaks)
   - **File descriptor leaks** or unrotated log file growth exhausting server disk space
   - **Cache fragmentation** and degradation in database performance over sustained operations.

6. Thresholds use p(95) rather than the average because the arithmetic average hides the experience of the slowest users. For example, if 95 out of 100 requests complete in 100ms and 5 requests take 5000ms, the average is ~345ms (which might look acceptable), yet 5% of users experienced an unacceptable 5-second lag. The 95th percentile (p(95)) guarantees that 95% of all requests completed faster than the specified threshold, making it a much more reliable metric for SLA compliance and real-world user experience.

7. Two reasons why running the same tests against a system in a dedicated lab environment could give different numbers:
   - **Network variability and latency**: Testing over the public internet introduces variable routing hops, ISP latency fluctuations, jitter, and packet buffering. In a dedicated lab environment, network latency is near zero and consistent.
   - **Shared multi-tenant workload**: QuickPizza is a public test service shared by learners and automation worldwide, meaning background traffic and server CPU/memory usage fluctuate uncontrollably. A dedicated lab server has isolated, predictable hardware resources and zero extraneous background load.

8. You must never run stress, spike, or breakpoint tests with hundreds of users against a site you do not own because:
   - It can overwhelm the target server infrastructure, exhausting bandwidth, CPU, and database connections, causing a **Denial of Service (DoS)** for genuine users.
   - It can incur substantial financial costs for the site owner in terms of cloud bandwidth, auto-scaling compute, or third-party API fees.
   - Without written authorization, generating heavy stress loads against third-party systems is considered an unauthorized cyberattack and is illegal under cybersecurity laws (such as the Computer Fraud and Abuse Act or equivalent legislation).

9. **Vague requirement for an everyday application** (e.g. food delivery / e-commerce mobile app):
   - *Vague requirement*: "The menu search should be fast and reliable for all users."
   - *Testable requirement*: "Under a sustained load of 50 concurrent users over a period of 5 minutes, 95% of menu search requests must respond in under 500ms, 99% of requests must respond in under 800ms, and the HTTP failure rate must not exceed 0.5%."
   - *K6 thresholds*:
   ```javascript
   export const options = {
     vus: 50,
     duration: '5m',
     thresholds: {
       http_req_duration: ['p(95)<500', 'p(99)<800'],
       http_req_failed: ['rate<0.005'],
     },
   };
   ```
