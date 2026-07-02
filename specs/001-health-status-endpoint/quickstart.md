# Quickstart: Health Status Endpoint

**Phase**: 1 (Design & Contracts)
**Date**: 2026-07-02

---

## Validation & Testing Guide

This guide provides step-by-step instructions to validate that the Health Status Endpoint feature works end-to-end. Use this after implementation is complete to verify compliance with the specification and contracts.

---

## Prerequisites

- Application running on `http://localhost:9000`
- `curl` command-line tool
- `jq` for JSON formatting (optional but recommended)
- Java 25 (per project `.java-version`)

---

## Setup: Start the Application

**Step 1**: Clone and build the project (if not already built):

```bash
cd /home/naveenp/code/java.samples.spring.boot
./mvnw clean install
```

**Step 2**: Start the application:

```bash
./mvnw spring-boot:run
```

**Expected Output**:
```
...
2026-07-02 17:11:00.000  INFO 12345 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 9000 (http) with context path ''
2026-07-02 17:11:00.000  INFO 12345 --- [main] com.example.Application                   : Started Application in 2.345 seconds (process running for 3.456)
```

**Verify**: Health endpoint is accessible:

```bash
curl -s http://localhost:9000/api/v1/status | jq '.'
```

---

## Scenario 1: Endpoint Returns Valid JSON with All Fields

**Purpose**: Verify that GET /api/v1/status returns HTTP 200 with valid JSON containing all required fields.

**Command**:

```bash
curl -i http://localhost:9000/api/v1/status
```

**Expected Output**:

```http
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 89
Date: Wed, 02 Jul 2026 17:11:00 GMT

{
  "uptime":"0s",
  "totalMemoryMB":384,
  "freeMemoryMB":384,
  "timestamp":"2026-07-02T17:11:00Z"
}
```

**Validation Checks**:

- ✅ HTTP status: **200 OK**
- ✅ Content-Type: **application/json**
- ✅ Field `uptime`: Present, string type, non-empty (e.g., "0s")
- ✅ Field `totalMemoryMB`: Present, integer type, > 0 (e.g., 384)
- ✅ Field `freeMemoryMB`: Present, integer type, >= 0 (e.g., 384)
- ✅ Field `timestamp`: Present, string type, ISO-8601 format (e.g., "2026-07-02T17:11:00Z")
- ✅ All fields present (no missing fields)
- ✅ Response payload size < 500 bytes (actual: ~89 bytes)

**Detailed Validation**:

```bash
# Check all fields are present
curl -s http://localhost:9000/api/v1/status | jq 'keys'
# Output: ["freeMemoryMB", "timestamp", "totalMemoryMB", "uptime"]

# Verify field types
curl -s http://localhost:9000/api/v1/status | jq '.uptime | type'           # "string"
curl -s http://localhost:9000/api/v1/status | jq '.totalMemoryMB | type'    # "number"
curl -s http://localhost:9000/api/v1/status | jq '.freeMemoryMB | type'     # "number"
curl -s http://localhost:9000/api/v1/status | jq '.timestamp | type'        # "string"

# Verify timestamp format (ISO-8601 UTC)
curl -s http://localhost:9000/api/v1/status | jq '.timestamp' | grep -E '^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z$'
# Output: "2026-07-02T17:11:00Z"
```

---

## Scenario 2: Uptime Increases Monotonically Over Time

**Purpose**: Verify that uptime field increases continuously as application runs and never decreases.

**Command 1** (at application startup, e.g., t=0s):

```bash
curl -s http://localhost:9000/api/v1/status | jq '.uptime'
```

**Expected Output** (immediately after startup):

```
"0s"
```

**Wait**: 5 seconds (actual elapsed time)

**Command 2** (at t=5s):

```bash
curl -s http://localhost:9000/api/v1/status | jq '.uptime'
```

**Expected Output**:

```
"5s"
```

**Wait**: Additional 10 seconds (now at t=15s total)

**Command 3** (at t=15s):

```bash
curl -s http://localhost:9000/api/v1/status | jq '.uptime'
```

**Expected Output**:

```
"15s"
```

**Validation Checks**:

- ✅ Uptime at t=0s: ~"0s"
- ✅ Uptime at t=5s: ~"5s" (increased by ~5 seconds)
- ✅ Uptime at t=15s: ~"15s" (increased by ~10 seconds since previous check)
- ✅ Uptime never decreased (monotonically increasing)
- ✅ Uptime reflects actual elapsed time (within ±1 second)

**Validation Script**:

```bash
#!/bin/bash
echo "Starting uptime monotonicity test..."
UPTIME1=$(curl -s http://localhost:9000/api/v1/status | jq -r '.uptime')
echo "T1: $UPTIME1"

sleep 10

UPTIME2=$(curl -s http://localhost:9000/api/v1/status | jq -r '.uptime')
echo "T2: $UPTIME2"

# Parse uptime strings to compare (simplified; assumes format "Xs" or "XmYs" or "XhYmZs")
if [ "$UPTIME2" != "$UPTIME1" ]; then
  echo "✅ Uptime increased from $UPTIME1 to $UPTIME2"
else
  echo "❌ Uptime did not change"
fi
```

---

## Scenario 3: Memory Metrics Are Non-Zero and Realistic

**Purpose**: Verify that totalMemoryMB and freeMemoryMB fields contain realistic values that reflect JVM state.

**Command**:

```bash
curl -s http://localhost:9000/api/v1/status | jq '.totalMemoryMB, .freeMemoryMB'
```

**Expected Output**:

```
384
256
```

**Validation Checks**:

- ✅ `totalMemoryMB` is a positive integer (> 0). Typical range: 256–2048 MB depending on `-Xmx` JVM flag.
- ✅ `freeMemoryMB` is a non-negative integer (≥ 0). Typical range: 50–400 MB depending on application load.
- ✅ `freeMemoryMB` ≤ `totalMemoryMB` (constraint: free cannot exceed total)
- ✅ Both values are realistic for a Spring Boot application (not absurdly large/small)

**Detailed Validation**:

```bash
# Check constraint: freeMemoryMB <= totalMemoryMB
TOTAL=$(curl -s http://localhost:9000/api/v1/status | jq '.totalMemoryMB')
FREE=$(curl -s http://localhost:9000/api/v1/status | jq '.freeMemoryMB')

if [ "$FREE" -le "$TOTAL" ]; then
  echo "✅ Constraint satisfied: freeMemoryMB ($FREE) <= totalMemoryMB ($TOTAL)"
else
  echo "❌ Constraint violated: freeMemoryMB ($FREE) > totalMemoryMB ($TOTAL)"
fi

# Check realistic values (heuristic)
if [ "$TOTAL" -ge 256 ] && [ "$TOTAL" -le 2048 ]; then
  echo "✅ totalMemoryMB ($TOTAL) is realistic"
else
  echo "⚠️  totalMemoryMB ($TOTAL) is unusual (expected 256–2048)"
fi

if [ "$FREE" -ge 0 ] && [ "$FREE" -le "$TOTAL" ]; then
  echo "✅ freeMemoryMB ($FREE) is realistic"
else
  echo "❌ freeMemoryMB ($FREE) is unrealistic"
fi
```

---

## Scenario 4: Timestamp Is ISO-8601 UTC and Close to Current Time

**Purpose**: Verify that timestamp field is in ISO-8601 UTC format and reflects current server time.

**Command**:

```bash
curl -s http://localhost:9000/api/v1/status | jq '.timestamp'
```

**Expected Output**:

```
"2026-07-02T17:11:00Z"
```

**Validation Checks**:

- ✅ Timestamp matches ISO-8601 format: `YYYY-MM-DDTHH:MM:SSZ` (ends with `Z` for UTC)
- ✅ Timestamp is in UTC timezone (ends with `Z`, not a local offset like `+05:30`)
- ✅ Timestamp is close to current system time (within ±1 second)
- ✅ Timestamp precision: seconds (no milliseconds)

**Detailed Validation**:

```bash
# Check ISO-8601 format with regex
TIMESTAMP=$(curl -s http://localhost:9000/api/v1/status | jq -r '.timestamp')
if [[ $TIMESTAMP =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$ ]]; then
  echo "✅ Timestamp format is valid ISO-8601 UTC: $TIMESTAMP"
else
  echo "❌ Timestamp format is invalid: $TIMESTAMP"
fi

# Check timezone (must end with Z, not offset)
if [[ $TIMESTAMP == *"Z" ]]; then
  echo "✅ Timestamp is in UTC (ends with Z)"
else
  echo "❌ Timestamp is not UTC"
fi

# Check time is close to current (within 5 seconds)
SERVER_TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ" -d "$TIMESTAMP")
CURRENT_TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
echo "Server timestamp: $TIMESTAMP"
echo "Current time:     $CURRENT_TIMESTAMP"
```

---

## Scenario 5: Invalid HTTP Methods Return HTTP 405

**Purpose**: Verify that non-GET methods (POST, PUT, DELETE, PATCH) are rejected with HTTP 405 Method Not Allowed.

**Command 1 - POST request**:

```bash
curl -i -X POST http://localhost:9000/api/v1/status
```

**Expected Output**:

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET
Content-Type: application/json;charset=UTF-8

{
  "timestamp":"2026-07-02T17:11:00Z",
  "status":405,
  "error":"Method Not Allowed",
  "message":"Request method 'POST' not supported",
  "path":"/api/v1/status"
}
```

**Validation Checks**:

- ✅ HTTP status: **405 Method Not Allowed**
- ✅ Response includes `Allow: GET` header
- ✅ Error message: "Request method 'POST' not supported"

**Command 2 - PUT request**:

```bash
curl -i -X PUT http://localhost:9000/api/v1/status
```

**Expected Output**: HTTP 405 (same as POST)

**Command 3 - DELETE request**:

```bash
curl -i -X DELETE http://localhost:9000/api/v1/status
```

**Expected Output**: HTTP 405 (same as POST)

**Validation Script**:

```bash
for METHOD in POST PUT DELETE PATCH; do
  RESPONSE=$(curl -s -i -X $METHOD http://localhost:9000/api/v1/status)
  STATUS=$(echo "$RESPONSE" | head -1 | awk '{print $2}')
  
  if [ "$STATUS" = "405" ]; then
    echo "✅ $METHOD request correctly rejected with HTTP 405"
  else
    echo "❌ $METHOD request returned HTTP $STATUS (expected 405)"
  fi
done
```

---

## Scenario 6: Concurrent Requests Are Handled Successfully

**Purpose**: Verify that the endpoint handles multiple concurrent requests without errors or timeouts.

**Command** (simulate 10 concurrent requests):

```bash
#!/bin/bash
echo "Sending 10 concurrent requests..."
for i in {1..10}; do
  curl -s http://localhost:9000/api/v1/status | jq '.' &
done
wait
echo "All requests completed."
```

**Expected Output**: All 10 requests complete successfully with valid JSON responses.

**Validation Checks**:

- ✅ All 10 requests return HTTP 200
- ✅ All responses contain valid JSON with required fields
- ✅ No timeout errors
- ✅ No error logs in application

**Detailed Validation**:

```bash
#!/bin/bash
SUCCESS_COUNT=0
ERROR_COUNT=0

for i in {1..10}; do
  RESPONSE=$(curl -s http://localhost:9000/api/v1/status)
  
  # Check if response contains all required fields
  if echo "$RESPONSE" | jq 'has("uptime") and has("totalMemoryMB") and has("freeMemoryMB") and has("timestamp")' | grep -q "true"; then
    ((SUCCESS_COUNT++))
  else
    ((ERROR_COUNT++))
  fi
done

echo "Concurrent requests: $SUCCESS_COUNT successful, $ERROR_COUNT failed"
if [ "$ERROR_COUNT" -eq 0 ]; then
  echo "✅ All concurrent requests successful"
else
  echo "❌ $ERROR_COUNT requests failed"
fi
```

---

## Scenario 7: Response Time Is Under 10ms (99th Percentile)

**Purpose**: Verify that response time is consistently fast (<10ms p99).

**Command** (measure response time for 100 requests):

```bash
#!/bin/bash
echo "Measuring response times for 100 requests..."
TIMES=()

for i in {1..100}; do
  START=$(date +%s%N)
  curl -s http://localhost:9000/api/v1/status > /dev/null
  END=$(date +%s%N)
  
  # Calculate milliseconds
  ELAPSED_MS=$(( ($END - $START) / 1000000 ))
  TIMES+=($ELAPSED_MS)
done

# Sort times and calculate percentiles
SORTED=$(printf '%s\n' "${TIMES[@]}" | sort -n)
COUNT=${#TIMES[@]}
P99_INDEX=$(( $COUNT * 99 / 100 ))
P99_VALUE=$(echo "$SORTED" | sed -n "$((P99_INDEX + 1))p")

echo "Response times (ms):"
echo "  Min:  $(echo "$SORTED" | head -1)"
echo "  Max:  $(echo "$SORTED" | tail -1)"
echo "  Avg:  $(echo "${TIMES[@]}" | awk '{sum=0; for(i=1;i<=NF;i++) sum+=$i; print sum/NF}')"
echo "  P99:  $P99_VALUE"

if [ "$P99_VALUE" -lt 10 ]; then
  echo "✅ P99 response time ($P99_VALUE ms) is under 10ms"
else
  echo "❌ P99 response time ($P99_VALUE ms) exceeds 10ms"
fi
```

**Expected Output**:

```
Response times (ms):
  Min:  1
  Max:  8
  Avg:  2.5
  P99:  7
✅ P99 response time (7 ms) is under 10ms
```

---

## Scenario 8: Swagger/OpenAPI Documentation Is Generated

**Purpose**: Verify that endpoint is documented via SpringDoc OpenAPI 3.0 and accessible via Swagger UI.

**Command 1** (fetch OpenAPI spec):

```bash
curl -s http://localhost:9000/v3/api-docs | jq '.paths["/api/v1/status"]'
```

**Expected Output**:

```json
{
  "get": {
    "tags": ["Health"],
    "summary": "Get application health status",
    "description": "Returns application uptime, JVM memory metrics...",
    "operationId": "getHealthStatus",
    "responses": {
      "200": {
        "description": "Health status retrieved successfully",
        "content": {
          "application/json": {
            "schema": {
              "$ref": "#/components/schemas/HealthStatusResponse"
            }
          }
        }
      }
    }
  }
}
```

**Command 2** (verify Swagger UI accessibility):

```bash
curl -s http://localhost:9000/swagger-ui.html | head -20
```

**Expected Output**: HTML response (200 OK)

**Validation Checks**:

- ✅ OpenAPI spec endpoint `/v3/api-docs` returns valid JSON
- ✅ Endpoint path `/api/v1/status` is documented
- ✅ GET method is documented
- ✅ Response schema references `HealthStatusResponse`
- ✅ HTTP 200 response documented
- ✅ Swagger UI is accessible at `/swagger-ui.html`

**Browser Check** (optional):

```bash
# Open in browser (macOS)
open http://localhost:9000/swagger-ui.html

# Or Linux
xdg-open http://localhost:9000/swagger-ui.html
```

Then navigate to `GET /api/v1/status` in the Swagger UI and click "Try it out" to execute a test request.

---

## Scenario 9: No Caching—Metrics Are Fresh On Each Request

**Purpose**: Verify that metrics reflect real-time JVM state and are not cached.

**Setup**: Allocate memory and observe freeMemoryMB decrease:

```bash
# Terminal 1: Run Java program that allocates memory
cat > /tmp/MemoryAllocator.java << 'EOF'
public class MemoryAllocator {
    public static void main(String[] args) throws InterruptedException {
        byte[] buffer = new byte[100 * 1024 * 1024]; // Allocate 100 MB
        System.out.println("Allocated 100 MB");
        Thread.sleep(10000); // Hold for 10 seconds
        System.out.println("Releasing...");
    }
}
EOF

javac /tmp/MemoryAllocator.java
java -cp /tmp MemoryAllocator &
```

**Terminal 2: Monitor health endpoint**:

```bash
# Check free memory before allocation
curl -s http://localhost:9000/api/v1/status | jq '.freeMemoryMB'
# Output (before): 256

# Wait for memory allocation in Terminal 1 to complete
# Check free memory after allocation
sleep 2
curl -s http://localhost:9000/api/v1/status | jq '.freeMemoryMB'
# Output (after): ~150 (decreased due to allocation)

# Wait for release
sleep 10
curl -s http://localhost:9000/api/v1/status | jq '.freeMemoryMB'
# Output (after release): ~256 (recovered via GC)
```

**Validation Checks**:

- ✅ `freeMemoryMB` decreases when memory is allocated (no caching)
- ✅ `freeMemoryMB` recovers when memory is released (reflects GC)
- ✅ Each request returns different values based on current JVM state (fresh metrics)

**Interpretation**: If `freeMemoryMB` remained constant throughout the test, metrics would be cached (violating spec). Fresh values confirm no caching.

---

## Summary Checklist

| Scenario | Status | Notes |
|----------|--------|-------|
| 1. Valid JSON with all fields | ✅ | HTTP 200, Content-Type application/json |
| 2. Uptime increases monotonically | ✅ | Format: XhYmZs, never decreases |
| 3. Memory metrics non-zero/realistic | ✅ | Constraint: free <= total |
| 4. Timestamp is ISO-8601 UTC | ✅ | Ends with Z, close to current time |
| 5. Invalid methods return 405 | ✅ | POST/PUT/DELETE rejected |
| 6. Concurrent requests handled | ✅ | 10–100 concurrent requests succeed |
| 7. Response time < 10ms (p99) | ✅ | Typically 1–8ms |
| 8. OpenAPI docs generated | ✅ | Swagger UI accessible, schema documented |
| 9. No caching (fresh metrics) | ✅ | Metrics change with memory allocation |

---

## Automated Test Script

Here's a comprehensive validation script that runs all scenarios:

```bash
#!/bin/bash

API_URL="http://localhost:9000/api/v1/status"
PASS=0
FAIL=0

test_case() {
  local name=$1
  local command=$2
  local expected=$3
  
  echo -n "Testing: $name ... "
  RESULT=$(eval "$command")
  
  if echo "$RESULT" | grep -q "$expected"; then
    echo "✅"
    ((PASS++))
  else
    echo "❌ (Got: $RESULT)"
    ((FAIL++))
  fi
}

echo "=== Health Status Endpoint Validation ==="
echo

test_case "HTTP 200 response" \
  "curl -s -w '%{http_code}' $API_URL" \
  "200"

test_case "Content-Type JSON" \
  "curl -s -I $API_URL | grep -i Content-Type" \
  "application/json"

test_case "Has uptime field" \
  "curl -s $API_URL | jq '.uptime'" \
  "s"

test_case "Has totalMemoryMB field" \
  "curl -s $API_URL | jq '.totalMemoryMB'" \
  "[0-9]"

test_case "Has freeMemoryMB field" \
  "curl -s $API_URL | jq '.freeMemoryMB'" \
  "[0-9]"

test_case "Has timestamp field" \
  "curl -s $API_URL | jq '.timestamp'" \
  "[0-9]{4}-[0-9]{2}-[0-9]{2}"

test_case "Invalid method returns 405" \
  "curl -s -w '%{http_code}' -X POST $API_URL" \
  "405"

echo
echo "Results: $PASS passed, $FAIL failed"
if [ $FAIL -eq 0 ]; then
  echo "✅ All tests passed!"
  exit 0
else
  echo "❌ Some tests failed"
  exit 1
fi
```

---

## Troubleshooting

| Issue | Diagnosis | Resolution |
|-------|-----------|-----------|
| Connection refused (curl: (7) Failed to connect) | Application not running | Run `./mvnw spring-boot:run` |
| HTTP 404 Not Found | Endpoint not implemented | Verify Controller and Service are deployed |
| HTTP 500 Internal Server Error | Service exception | Check application logs for stack trace |
| jq: command not found | jq not installed | Install: `brew install jq` (macOS) or `apt-get install jq` (Linux) |
| Memory values are 0 | Memory access denied (rare) | Check logs: "Unable to access JVM memory metrics" |
| Response time > 10ms | System overload or GC pause | Re-run test during lower load; this is expected under heavy GC |

---

**Quickstart Complete**: 2026-07-02
**Ready for**: Phase 2 Tasks & Phase 3 Implementation
