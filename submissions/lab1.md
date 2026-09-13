\# Lab 1 



Liubov Utenysheva, CBS-03



\---



\## Task 1 



\### 1.1  All 5 services running



```

$ cd app \&\& docker compose up --build -d

$ docker compose ps

NAME             IMAGE                COMMAND                  SERVICE    CREATED          STATUS                    PORTS

app-events-1     app-events           "uvicorn main:app --…"   events     20 seconds ago   Up 13 seconds             0.0.0.0:8081->8081/tcp, \[::]:8081->8081/tcp

app-gateway-1    app-gateway          "uvicorn main:app --…"   gateway     20 seconds ago   Up 13 seconds             0.0.0.0:3080->8080/tcp, \[::]:3080->8080/tcp

app-payments-1   app-payments         "uvicorn main:app --…"   payments   20 seconds ago   Up 19 seconds             0.0.0.0:8082->8082/tcp, \[::]:8082->8082/tcp

app-postgres-1   postgres:17-alpine   "docker-entrypoint.s…"   postgres   20 seconds ago   Up 19 seconds (healthy)   0.0.0.0:5432->5432/tcp, \[::]:5432->5432/tcp

app-redis-1      redis:7-alpine       "docker-entrypoint.s…"   redis      20 seconds ago   Up 19 seconds (healthy)   0.0.0.0:6379->6379/tcp, \[::]:6379->6379/tcp

```



\### 1.2 



```

$ curl -s http://localhost:3080/events | python3 -m json.tool

\[

&#x20;   {

&#x20;       "id": 1,

&#x20;       "name": "Go Conference 2026",

&#x20;       "venue": "Main Hall A",

&#x20;       "date": "2026-09-15T09:00:00+00:00",

&#x20;       "total\_tickets": 100,

&#x20;       "price\_cents": 5000,

&#x20;       "available": 100

&#x20;   },

&#x20;   {

&#x20;       "id": 4,

&#x20;       "name": "Python Workshop",

&#x20;       "venue": "Lab 301",

&#x20;       "date": "2026-09-22T14:00:00+00:00",

&#x20;       "total\_tickets": 25,

&#x20;       "price\_cents": 2000,

&#x20;       "available": 25

&#x20;   },

&#x20;   {

&#x20;       "id": 2,

&#x20;       "name": "SRE Meetup",

&#x20;       "venue": "Room 204",

&#x20;       "date": "2026-10-01T18:00:00+00:00",

&#x20;       "total\_tickets": 30,

&#x20;       "price\_cents": 0,

&#x20;       "available": 30

&#x20;   },

&#x20;   {

&#x20;       "id": 5,

&#x20;       "name": "Kubernetes Deep Dive",

&#x20;       "venue": "Auditorium B",

&#x20;       "date": "2026-10-10T10:00:00+00:00",

&#x20;       "total\_tickets": 80,

&#x20;       "price\_cents": 8000,

&#x20;       "available": 80

&#x20;   },

&#x20;   {

&#x20;       "id": 3,

&#x20;       "name": "Cloud Native Summit",

&#x20;       "venue": "Expo Center",

&#x20;       "date": "2026-11-20T10:00:00+00:00",

&#x20;       "total\_tickets": 500,

&#x20;       "price\_cents": 15000,

&#x20;       "available": 500

&#x20;   }

]



$ curl -s -X POST http://localhost:3080/events/1/reserve \\

&#x20;   -H "Content-Type: application/json" -d '{"quantity": 1}' | python3 -m json.tool

{

&#x20;   "reservation\_id": "70afc33f-1144-4f7c-9031-edeb43166c2c",

&#x20;   "event\_id": 1,

&#x20;   "quantity": 1,

&#x20;   "total\_cents": 5000,

&#x20;   "expires\_in\_seconds": 300

}



$ curl -s -X POST http://localhost:3080/reserve/70afc33f-1144-4f7c-9031-edeb43166c2c/pay | python3 -m json.tool

{

&#x20;   "order\_id": "70afc33f-1144-4f7c-9031-edeb43166c2c",

&#x20;   "event\_id": 1,

&#x20;   "quantity": 1,

&#x20;   "total\_cents": 5000,

&#x20;   "status": "confirmed"

}



$ curl -s http://localhost:3080/health | python3 -m json.tool

{

&#x20;   "status": "healthy",

&#x20;   "checks": {

&#x20;       "events": "ok",

&#x20;       "payments": "ok",

&#x20;       "circuit\_payments": "CLOSED"

&#x20;   }

}

```



\### 1.3 





```mermaid

graph LR

&#x20;   U\[User / loadgen] -->|HTTP :3080| GW\[gateway]

&#x20;   GW -->|HTTP| EV\[events]

&#x20;   GW -->|HTTP /charge| PAY\[payments]

&#x20;   EV -->|psycopg2| PG\[(postgres)]

&#x20;   EV -->|redis-py| RD\[(redis)]

```



```

gateway → events    (list, get, reserve, confirm)

gateway → payments  (only /charge, during pay)

events  → postgres  (catalog, availability, orders)

events  → redis     (reservation holds with 300s TTL, held-ticket counters)

payments → nothing (self-contained mock)

```



gateway is stateless and can only report what its two upstreams report. events is the hub: every path touches it, and it holds both stateful dependencies. payments is the only component with no dependencies, so its failure stays isolated to the pay step.



\### 1.4



Each component was stopped with `docker compose stop <svc>`, the four endpoints were tested, then it was started back up and recovery was checked.



\#### payments



```

$ docker compose stop payments

$ curl -s http://localhost:3080/events

&#x20; -> HTTP 200: \[{"id":1,"name":"Go Conference 2026","venue":"Main Hall A",...

$ curl -s -X POST http://localhost:3080/events/1/reserve -H 'Content-Type: application/json' -d '{"quantity": 1}'

&#x20; -> HTTP 200: {"reservation\_id":"441b6145-091a-4048-b1fd-1e8aab1a8718","event\_id":1,"quantity":1,"total\_cents":5000,"expires\_in\_seconds":300}

$ curl -s -X POST http://localhost:3080/reserve/28ad1e00-1f92-4f31-ac94-ad6b804a21db/pay

&#x20; -> HTTP 502: {"detail":"Payment service unavailable"}

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 503: {"status":"degraded","checks":{"events":"ok","payments":"down","circuit\_payments":"CLOSED"}}

$ docker compose start payments

$ curl -s -X POST http://localhost:3080/reserve/28ad1e00-1f92-4f31-ac94-ad6b804a21db/pay   (retry after recovery)

&#x20; -> HTTP 200: {"order\_id":"28ad1e00-1f92-4f31-ac94-ad6b804a21db","event\_id":1,"quantity":1,"total\_cents":5000,"status":"confirmed"}

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 200: {"status":"healthy","checks":{"events":"ok","payments":"ok","circuit\_payments":"CLOSED"}}

```



The cleanest outage. Browse and reserve keep working. Pay fails with a generic 502. After the restart the same reservation pays through, so the hold really did survive.



\#### events



I reserved first (`6296c8ee-…`) so I could pay with a real held reservation after the kill.



```

$ docker compose stop events

$ curl -s http://localhost:3080/events

&#x20; -> HTTP 502: {"detail":"Events service unavailable"}

$ curl -s -X POST http://localhost:3080/events/1/reserve -H 'Content-Type: application/json' -d '{"quantity": 1}'

&#x20; -> HTTP 502: {"detail":"Events service unavailable"}

$ curl -s -X POST http://localhost:3080/reserve/28f68e29-0de4-468c-8f50-8e9c02cecd04/pay   (pre-killed reservation)

&#x20; -> HTTP 500: {"detail":"Payment succeeded but confirmation failed — contact support"}

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 503: {"status":"degraded","checks":{"events":"down","payments":"ok","circuit\_payments":"CLOSED"}}

$ docker compose start events

$ curl -s http://localhost:3080/events   (recovery check)

&#x20; -> HTTP 200: \[{"id":1,"name":"Go Conference 2026",...

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 200: {"status":"healthy","checks":{"events":"ok","payments":"ok","circuit\_payments":"CLOSED"}}

```



The bad part: the charge on the payments side succeeded, but the order never got written. That means a customer is charged with no order and no ticket to point at.



\#### redis



Reservation `7e5b06a0-…` was made before the kill. I waited 6 seconds after the kill so the events service's 5-second redis health cache expired.



```

$ docker compose stop redis

$ curl -s http://localhost:3080/events

&#x20; -> HTTP 200: \[{"id":1,"name":"Go Conference 2026",...

$ curl -s -X POST http://localhost:3080/events/1/reserve -H 'Content-Type: application/json' -d '{"quantity": 1}'

&#x20; -> HTTP 504: {"detail":"Events service timeout"}

$ curl -s -X POST http://localhost:3080/reserve/7e5b06a0-037b-42d4-b9f5-3a6fd6f2b8a0/pay

&#x20; -> HTTP 500: {"detail":"Payment succeeded but confirmation failed — contact support"}

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 503: {"status":"degraded","checks":{"events":"down","payments":"ok","circuit\_payments":"CLOSED"}}

$ docker compose start redis

$ sleep 6

$ curl -s -X POST http://localhost:3080/events/1/reserve -H 'Content-Type: application/json' -d '{"quantity": 1}'  (recovery)

&#x20; -> HTTP 200: {"reservation\_id":"354ccb89-b1e0-4088-9f42-def0e04b6b61","event\_id":1,"quantity":1,"total\_cents":5000,"expires\_in\_seconds":300}

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 200: {"status":"healthy","checks":{"events":"ok","payments":"ok","circuit\_payments":"CLOSED"}}

```



The 504 on reserve is a delayed failure. The events endpoint blocks on redis socket timeouts, the gateway hits its 5 second budget first and gives up. But the request kept running on the server side. When redis came back, the stuck confirm finished and wrote the order \~60 seconds after the user was already told to contact support:



```

$ docker compose exec postgres psql -U quickticket -d quickticket -c "SELECT id, payment\_ref, created\_at FROM orders WHERE id='7e5b06a0-…';"

&#x20;7e5b06a0-037b-42d4-b9f5-3a6fd6f2b8a0 | PAY-3D04C960 | 2026-09-13 19:40:07.749736+00

$ docker compose logs events | grep 7e5b06a0

&#x20;{"time":"2026-09-13 19:40:07,754",... "msg":"Order confirmed: 7e5b06a0-037b-42d4-b9f5-3a6fd6f2b8a0"}

```



A timeout does not cancel the server-side request. That silently creates duplicate side effects.



\#### postgres



Reservation `abc0c1c4-…` was made before the kill.



```

$ docker compose stop postgres

$ curl -s http://localhost:3080/events

&#x20; -> HTTP 502: {"detail":"Events service unavailable"}

$ curl -s -X POST http://localhost:3080/events/1/reserve -H 'Content-Type: application/json' -d '{"quantity": 1}'

&#x20; -> HTTP 500: Internal Server Error

$ curl -s -X POST http://localhost:3080/reserve/abc0c1c4-5a55-41c7-bb5a-c45d90c408af/pay

&#x20; -> HTTP 500: {"detail":"Payment succeeded but confirmation failed — contact support"}

$ curl -s http://localhost:3080/health

&#x20; -> HTTP 503: {"status":"degraded","checks":{"events":"down","payments":"ok","circuit\_payments":"CLOSED"}}

$ docker compose start postgres

$ curl -s http://localhost:3080/events   (recovery check right after healthy)

&#x20; -> HTTP 200: \[{"id":1,"name":"Go Conference 2026","venue":"Main Hall A","date":"2026-09-15T09:00:00+00:00","total\_tickets":100,"price\_cents":5000,"available":97},...]

```



The raw 500 on reserve is a gateway bug. events returns a 500 with a non-JSON body, and the gateway's status-error handler calls `e.response.json()` on it, which raises and leaks an unformatted 500. I checked the DB after recovery: `SELECT count(\*) FROM orders WHERE id='abc0c1c4-…'` gave 0. Charged again, no order. Recovery itself was instant. The DB pool threw away the dead connections and reconnected on the next request, no restart of events needed.



\### Failure table



| Component Killed | Events List | Reserve | Pay | Health Check | User Impact |

|-----------------|-------------|---------|-----|--------------|-------------|

| payments | 200 OK | 200 OK | 502 `Payment service unavailable` | 503 degraded, payments down | Browsing and reserving fine. Purchase fails with a generic 502. The hold survives (300s TTL), so retrying after recovery works. |

| events | 502 `Events service unavailable` | 502 same | 500 `Payment succeeded but confirmation failed` | 503 degraded, events down | Whole store down. The charge can still succeed while the order cannot, so the customer pays and gets nothing. |

| redis | 200 OK | 504 `Events service timeout` (gateway gave up at 5s, request kept running server-side) | 500 `Payment succeeded but confirmation failed` — order still landed \~60s later | 503 degraded, events down | Catalog still browsable. Reserves time out. Pay reports failure but the order can appear later with a payment ref. Most confusing state for the customer. |

| postgres | 502 `Events service unavailable` | raw 500 `Internal Server Error` (gateway fails to parse the non-JSON body) | 500 `Payment succeeded but confirmation failed`, order not persisted (0 rows in `orders`) | 503 degraded, events down | Whole store down, same charged-without-order exposure. Recovery is fast and the DB pool self-heals. |



`/health` is the only endpoint that tells the operator the truth. It went 503 in every scenario. What users see is either "everything works" or a generic 502/504. And in 3 of the 4 cases the money path fails after the charge already succeeded.



\### 1.5 — Load generator, error rate spike



Baseline, all healthy:



```

$ ./app/loadgen/run.sh 5 30

QuickTicket Load Generator

Target: http://localhost:3080 | RPS: 5 | Duration: 30s

\---

\[10s] requests=43 success=43 fail=0 error\_rate=0%

\[20s] requests=88 success=88 fail=0 error\_rate=0%

\---

Done. total=133 success=133 fail=0 error\_rate=0%

```



Same run, but I stopped payments at +10s (and started it back after the run):



```

$ ./app/loadgen/run.sh 5 30   (payments killed at +10s)

QuickTicket Load Generator

Target: http://localhost:3080 | RPS: 5 | Duration: 30s

\---

\[10s] requests=44 success=44 fail=0 error\_rate=0%

\---

Done. total=79 success=75 fail=4 error\_rate=5.0%

```



Errors went from 0% to 5.0%, and all 4 failures happened after the kill. The load mix is 70% list, 20% reserve, 10% full purchase, and list and reserve do not touch payments. So basically every purchase attempt inside the fault window failed. Throughput also dropped (79 vs 133 requests) because the purchase path is two sequential calls.



\---



\## Task 2 





One new `except` clause in `pay\_reservation()` in `app/gateway/main.py`, between the `HTTPStatusError` handler and the catch-all. It catches `httpx.ConnectError` only and answers with a 503 that says the reservation is safe:



```diff

diff --git a/app/gateway/main.py b/app/gateway/main.py

index c86db33..6dbb9a2 100644

\--- a/app/gateway/main.py

+++ b/app/gateway/main.py

@@ -336,6 +336,16 @@ async def pay\_reservation(reservation\_id: str):

&#x20;        raise HTTPException(504, "Payment service timeout")

&#x20;    except httpx.HTTPStatusError as e:

&#x20;        raise HTTPException(e.response.status\_code, "Payment failed")

\+    except httpx.ConnectError as e:

\+        log.error(f"payments service unreachable: {e}")

\+        return JSONResponse(

\+            status\_code=503,

\+            content={

\+                "error": "payments\_unavailable",

\+                "message": "Payment service is temporarily down. Your reservation is held — try again in a few minutes.",

\+                "reservation\_id": reservation\_id,

\+            },

\+        )

&#x20;    except Exception as e:

&#x20;        log.error(f"payment error: {e}")

&#x20;        raise HTTPException(502, "Payment service unavailable")

```



list, get and reserve never call payments, so they needed no change. The reservation lives in redis, not in payments, so it really does outlive the outage. The clause sits before the catch-all, which would otherwise swallow `ConnectError` and send back the old 502.



\### Verification



I rebuilt the gateway with `docker compose up -d --build gateway`, then stopped payments:



```

$ docker compose stop payments

$ curl -s -X POST http://localhost:3080/events/1/reserve \\

&#x20;   -H "Content-Type: application/json" -d '{"quantity": 1}'

&#x20; {

&#x20;     "reservation\_id": "a8243e95-dc4e-4ef7-a5d4-d6f5a2c68fb3",

&#x20;     "event\_id": 1,

&#x20;     "quantity": 1,

&#x20;     "total\_cents": 5000,

&#x20;     "expires\_in\_seconds": 300

&#x20; }



$ curl -s -X POST http://localhost:3080/reserve/a8243e95-dc4e-4ef7-a5d4-d6f5a2c68fb3/pay

&#x20; -> HTTP 503

&#x20; {

&#x20;     "error": "payments\_unavailable",

&#x20;     "message": "Payment service is temporarily down. Your reservation is held — try again in a few minutes.",

&#x20;     "reservation\_id": "a8243e95-dc4e-4ef7-a5d4-d6f5a2c68fb3"

&#x20; }



$ docker compose start payments

$ curl -s -X POST http://localhost:3080/reserve/a8243e95-dc4e-4ef7-a5d4-d6f5a2c68fb3/pay   (retry after recovery)

&#x20; -> HTTP 200

&#x20; {

&#x20;     "order\_id": "a8243e95-dc4e-4ef7-a5d4-d6f5a2c68fb3",

&#x20;     "event\_id": 1,

&#x20;     "quantity": 1,

&#x20;     "total\_cents": 5000,

&#x20;     "status": "confirmed"

&#x20; }



$ curl -s http://localhost:3080/health | python3 -m json.tool

&#x20; {

&#x20;     "status": "healthy",

&#x20;     "checks": {

&#x20;         "events": "ok",

&#x20;         "payments": "ok",

&#x20;         "circuit\_payments": "CLOSED"

&#x20;     }

&#x20; }

```



Reserve works, pay degrades to a clear 503, and the same reservation\_id pays through right after the restart.



\---



\## Task 3 



Starring a repo is a bookmark you keep for yourself, and it is also a public signal of community trust. It helps maintainers and new contributors tell which projects people actually rely on. Following developers turns their work into a feed you can read. You see how peers structure repos and solve problems in real time, and that builds the network you will work with on future teams.



\---



\## Bonus Task 



I sampled `docker stats --no-stream` in three states: idle, under a 10 rps load run (`./app/loadgen/run.sh 10 30`, sampled at \~15s), and under the same load with fault-injected payments (`PAYMENT\_FAILURE\_RATE=0.3 PAYMENT\_LATENCY\_MS=500`). Payments was then restored with `PAYMENT\_FAILURE\_RATE=0.0 PAYMENT\_LATENCY\_MS=0`.



\### B.1  Idle



| NAME | CPU % | MEM USAGE | NET I/O | PIDS |

|------|------:|-----------|---------|-----:|

| app-gateway-1 | 0.18% | 38.04 MiB / 15.19 GiB | 8.45 kB / 7.63 kB | 2 |

| app-events-1 | 0.17% | 40.81 MiB / 15.19 GiB | 8.56 kB / 7.71 kB | 2 |

| app-payments-1 | 0.19% | 33.4 MiB / 15.19 GiB | 2.15 kB / 1.15 kB | 2 |

| app-postgres-1 | 0.01% | 24.16 MiB / 15.19 GiB | 164 kB / 188 kB | 8 |

| app-redis-1 | 2.34% | 3.805 MiB / 15.19 GiB | 52.5 kB / 22.1 kB | 6 |



\### B.2  Under load (10 rps × 30s)



Load run finished with `total=241 success=229 fail=12 error\_rate=4.9%`. See the finding below for where those failures came from.



| NAME | CPU % | MEM USAGE | NET I/O | PIDS |

|------|------:|-----------|---------|-----:|

| app-gateway-1 | 5.39% | 38.59 MiB / 15.19 GiB | 838 kB / 818 kB | 2 |

| app-events-1 | 2.91% | 41.34 MiB / 15.19 GiB | 715 kB / 959 kB | 2 |

| app-payments-1 | 0.20% | 34.93 MiB / 15.19 GiB | 7.14 kB / 4.71 kB | 2 |

| app-postgres-1 | 1.25% | 24.42 MiB / 15.19 GiB | 556 kB / 648 kB | 8 |

| app-redis-1 | 0.54% | 3.918 MiB / 15.19 GiB | 147 kB / 62.5 kB | 6 |



\### B.3  Chaos (payments: 30% failures, +500ms latency)



Load run finished with `total=158 success=152 fail=6 error\_rate=3.7%` (2.0% at 10s, 3.9% at 20s).



| NAME | CPU % | MEM USAGE | NET I/O | PIDS |

|------|------:|-----------|---------|-----:|

| app-payments-1 | 0.46% | 34.98 MiB / 15.19 GiB | 7.48 kB / 5.53 kB | 2 |

| app-gateway-1 | 2.09% | 38.52 MiB / 15.19 GiB | 516 kB / 503 kB | 2 |

| app-events-1 | 1.29% | 41.28 MiB / 15.19 GiB | 443 kB / 595 kB | 2 |

| app-postgres-1 | 0.41% | 24.42 MiB / 15.19 GiB | 406 kB / 469 kB | 8 |

| app-redis-1 | 0.43% | 3.672 MiB / 15.19 GiB | 114 kB / 48.1 kB | 6 |



\### Analysis



\*\*Most memory: events (\~41 MiB), then gateway (\~38.5 MiB), and it does not change under load.\*\* Memory is flat across all three scenarios. These services hold small fixed structures, like the psycopg2 pool and the shared httpx client, and nothing allocates meaningfully per request at this scale. redis is the cheapest (\~4 MiB), postgres is lighter than you would expect (alpine image, tiny schema).



\*\*Most CPU under load: gateway (5.39% vs events 2.91%).\*\* The gateway sits on 100% of the traffic. It proxies every request, handles the JSON on both sides of every hop, and runs the metrics and rate-limit middleware on each one. events is second because it does the real work: SQL joins and redis round trips. In the chaos run the gateway's absolute CPU dropped to 2.09%. That is not because it got lighter. Total throughput dropped to 158 requests because the 500ms injected latency slowed each purchase down, and the serial loadgen loop sends fewer requests overall.



\*\*How the payments fault shows up on the gateway: hold time, not CPU.\*\* With +500ms on `/charge`, every in-flight purchase keeps a connection open in the gateway for longer. The gateway cannot close the request until the slow or failed charge returns. So in-flight concurrency rises even as the request rate falls. Net I/O backs this up. The gateway moved several times the bytes of any other service in both load scenarios, because it is the only container on the user-facing path. Also, payments' own CPU stayed near zero (0.20-0.46%) even while failing. The fault injection just sleeps, and returning a 500 is cheap. The cost lands on the callers, not the broken service.



\*\*Extra finding: the "healthy" load run was not 0%, and the cause is a capacity leak.\*\* The B.2 run failed 4.9% of requests, and the repeat run got worse over time (4.8% at 10s, 9.0% final). Events 2 (30 tickets) and 4 (25 tickets) started returning 409 Not enough tickets while everything was up. The reason: `event:{id}:held` in redis goes up on every reserve but is never brought back down on confirm or on TTL expiry. After \~45 reservations the counters alone (37/21/29/18/27) had already "sold out" both small events, for example `available = max(0, 25 - 8 - 18) = 0`. The `/events` endpoint does not even show these held counts, it only subtracts confirmed orders. So the store displays tickets that the reserve path refuses. Left alone, normal usage alone would exhaust every event. No outage required. I reset the leaked counters before the chaos run so B.3 isolates the injected fault.



