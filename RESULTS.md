# K6 lab results

## Part 0
- Step 0.1, http_req_duration: `avg=347.9ms min=347.9ms med=347.9ms max=347.9ms p(90)=347.9ms p(95)=347.9ms`
- Step 0.2, requests, rate and p(95), and why the rate stays below 5: `66` requests, `3.11 req/s` (`3.105355/s`), `p(95)=2.02s` (`avg=530.67ms`, `med=307.01ms`). The rate stays below 5 requests/s because in each iteration a virtual user must wait for the HTTP response (~0.3–0.5s) AND then pause for `sleep(1)` (1 second of think time). Because one complete iteration takes ~1.57s on average (> 1s), each VU completes fewer than 1 iteration per second (~0.62 req/s per user), keeping the total rate across 5 VUs around 3.11 req/s (< 5 req/s).
- Step 0.6, exit code of the normal run and of the run with P95=300: Normal run exit code = `0` (all thresholds passed, measured `p(95)=417.28ms < 1500ms`). Run with `-e P95=300` exit code = `99` (`http_req_duration` threshold `p(95)<300` failed with measured `p(95)=440.32ms`).
- Step 0.7, rules that passed, p(95), p(99), exit code after your change: All 4 rules passed (`checks: rate>0.99` at `100.00%`, `http_req_failed: rate<0.01` at `0.00%`, `http_req_duration: p(95)<1000` with measured `p(95)=418.1ms`, and `p(99)<1500` with measured `p(99)=424.41ms`). After changing the `p(95)` threshold to a stricter value (`p(95)<300`) so the test fails, the exit code was `99`.
- Step 0.8, what was wrong, and how the checks showed it: In the buggy run (75 requests / 225 checks), `checks_succeeded` was `86.22%` (194/225): `'no pepperoni'` passed 93% (70 pass / 5 fail) and `'no knife needed'` passed 65% (49 pass / 26 fail). Running `explore.js` against `/api/ingredients/topping` and `/api/tools` revealed that QuickPizza stores capitalized names (`"Pepperoni"` and `"Knife"`), whereas the buggy script sent lowercase `excludedIngredients: ['pepperoni']` and `excludedTools: ['knife']`. Because the API matches exclusions case-sensitively, it ignored the lowercase strings and still randomly picked `"Pepperoni"` and `"Knife"`. A single manual request might randomly receive a pizza without Pepperoni or a Knife and appear to work, whereas automated checks under load ran 75 iterations and immediately exposed the intermittent failures. Fixing the payload to `['Pepperoni']` and `['Knife']` made 100.00% (225/225) of checks pass.

## Part A results
| Step | Test type    | Users | Req/s   | p(95)    | Max      | Failed | Checks  |
| ---- | ------------ | ----- | ------- | -------- | -------- | ------ | ------- |
| A1   | Smoke        | 1     | 0.49/s  | 415.52ms | 419.9ms  | 0.00%  | 100.00% |
| A2   | Average load | 3     | 1.28/s  | 406.5ms  | 451.18ms | 0.00%  | 100.00% |
| A3   | Stress       | 5     | 2.15/s  | 397.63ms | 1.83s    | 0.00%  | 100.00% |
| A4   | Spike        | 5     | 1.02/s  | 423.26ms | 1.17s    | 0.00%  | 100.00% |
| A5   | Soak         | 2     | 0.96/s  | 434.47ms | 2.97s    | 0.00%  | 100.00% |

## Part B
1. **Results table for Steps A1 to A5**: Completed in the `## Part A results` table above.
2. **In Step 0.2, why was the request rate below 5 requests per second with 5 virtual users?**
   Each virtual user runs the default function sequentially in a loop: it sends an HTTP GET request (which takes ~300–530 ms over the internet) and then calls `sleep(1)` to model 1 second of user think time. Because each iteration takes `http_req_duration + 1s ≈ 1.57s` (> 1 second), a single VU can only send about `1 / 1.57 ≈ 0.62` requests/s. Five VUs running in parallel therefore achieve `5 / 1.57 ≈ 3.11` requests/s, which is always strictly below 5 requests/s.
3. **Did the response time grow from the average-load test to the stress test? Explain your answer with the operational envelope.**
   The `p(95)` response time did not grow significantly from the average-load test (`406.5ms` at 3 VUs, `1.28 req/s`) to the stress test (`397.63ms` at 5 VUs, `2.15 req/s`), although the maximum outlier (`Max`) increased from `451.18ms` to `1.83s`. In terms of the **operational envelope** (Pezzè & Young, Section 22.2), QuickPizza is a production-hosted public demo service whose capacity (operational envelope) is much larger than 5 concurrent users. Because our "stress" test is scaled down to a polite 5 VUs on a shared public site, both 3 VUs and 5 VUs operate well inside the system's normal operational envelope rather than near or beyond its saturation boundary, so throughput scales linearly (`1.28/s` to `2.15/s`) while `p(95)` latency remains flat (~400 ms).
4. **In the spike test, how long did the response time take to return to normal after the spike?**
   During the sudden jump from 1 VU to 5 VUs (at `t = 30s`), the 10-second window `p(95)` briefly rose to `~899.7ms` (as the 4 new VUs established TLS connections) and one request reached `1.17s` during the 5-VU plateau (`35s–65s`), while most requests stayed around `~420–432ms`. As soon as the load dropped back from 5 VUs to 1 VU (`65s–70s`), the response time returned to its normal baseline (`p(95) ≈ 331–414ms`, `avg ≈ 295–350ms`) **almost immediately (within under 10 seconds, i.e., the very next 10-second reporting interval at `70s–80s`)**.
5. **In the soak test, did the response time or the error rate get worse over time? Name one kind of defect that a soak test of several hours could reveal.**
   No—neither the response time nor the error rate degraded over the 11-minute soak test. The error rate stayed at `0.00%` (0 failed out of 639 requests), and the response time in the last minutes (`avg ≈ 311–362ms`, `p(95) ≈ 346–441ms` in minutes 9–11) was virtually identical to the first minutes after warm-up (`avg ≈ 318–348ms`, `p(95) ≈ 372–410ms` in minutes 1–3). A real soak test running for several hours could reveal **memory leaks** (gradual heap growth leading to heavy garbage collection or `OutOfMemory` crashes), as well as **connection pool / file descriptor leaks**, **database connection exhaustion**, or **disk/log storage exhaustion**.
6. **Why do the thresholds in this lab use p(95) rather than the average?**
   An arithmetic average (`avg`) hides tail latency: if 95% of requests finish in 200 ms but 5% of users wait 5 seconds, the average is still ~440 ms, masking the fact that 1 in 20 requests gives a terrible user experience. Conversely, a single extreme network outlier can skew the average upward. The 95th percentile (`p(95)`) directly expresses a service-level objective for user experience: **at least 95% of all requests must finish within the target time** (only 1 in 20 may be slower).
7. **Your results come from a shared public site reached over the internet. Give two reasons why the same tests against a system in a lab could give different numbers.**
   - **Network latency and jitter over the public internet**: Reaching `quickpizza.grafana.com` over the public internet introduces wide-area network round-trip time (~250–300 ms baseline RTT, TLS handshakes, routing hops, and ISP congestion), whereas a local lab server on a LAN or localhost has near-zero network latency (< 1–5 ms) and much lower variance.
   - **Shared background traffic (noisy neighbors) and unknown server state**: QuickPizza is simultaneously used by other K6 learners worldwide, so background load fluctuates unpredictably outside our control, whereas a dedicated lab environment is isolated and starts from a known, repeatable hardware/database state.
8. **Why must you never run a stress, spike or breakpoint test with hundreds of users against a site that you do not own?**
   Generating high-volume traffic against a server without the owner's written permission is functionally indistinguishable from a **Denial of Service (DoS) attack**. It can degrade or crash the service for legitimate users, trigger auto-scaling or bandwidth bills that cost the owner money, get your IP/network blocked by firewalls/WAFs, and violate computer misuse laws and acceptable-use policies.
9. **Write one vague performance requirement for an app you use every day. Rewrite it as a testable requirement, and show the K6 thresholds that check it.**
   - **Vague requirement**: *"The food delivery app's restaurant search must be fast and reliable even during dinner rush hour."*
   - **Testable requirement**: *"Under a concurrent load of 50 virtual users searching for restaurants continuously for 5 minutes, the `GET /api/restaurants/search` endpoint must respond within 800 ms for at least 95% of requests and within 1,500 ms for at least 99% of requests, with an HTTP failure rate below 0.5% and more than 99% of functional checks passing."*
   - **K6 thresholds**:
     ```javascript
     export const options = {
       vus: 50,
       duration: '5m',
       thresholds: {
         http_req_duration: ['p(95)<800', 'p(99)<1500'],
         http_req_failed: ['rate<0.005'],
         checks: ['rate>0.99'],
       },
     };
     ```
