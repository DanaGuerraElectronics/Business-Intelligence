# Business-Intelligence
import argparse, csv, os, statistics, time
from concurrent.futures import ThreadPoolExecutor
import requests

BASE = "http://127.0.0.1:8000"

def measure(name, fn, repeats):
    times, errors = [], 0
    for _ in range(repeats):
        start = time.perf_counter()
        try:
            response = fn()
            response.raise_for_status()
        except Exception as exc:
            errors += 1
            print(f"[WARN] {name}: {exc}")
        times.append((time.perf_counter() - start) * 1000)
    total_s = sum(times) / 1000
    return {
        "test": name, "repetitions": repeats,
        "median_latency_ms": round(statistics.median(times), 3),
        "mean_latency_ms": round(statistics.mean(times), 3),
        "requests_per_second": round(repeats / total_s, 3) if total_s else 0,
        "errors": errors
    }

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--requests", type=int, default=100)
    parser.add_argument("--records", type=int, default=1000)
    args = parser.parse_args()
    n = max(1, args.requests)
    records = [{"product": f"Producto-{i % 20 + 1}", "quantity": i % 5 + 1,
                "unit_price": round(10 + (i % 50) * 1.25, 2)} for i in range(args.records)]
    tests = [
        measure("batch_ingestion", lambda: requests.post(f"{BASE}/ingest/batch", json={"records": records}, timeout=120), max(3, min(10, n // 10 or 3))),
        measure("event_ingestion", lambda: requests.post(f"{BASE}/ingest/event", json=records[0], timeout=30), n),
        measure("REST_query", lambda: requests.get(f"{BASE}/sales?limit=100", timeout=30), n),
        measure("GraphQL_query", lambda: requests.post(f"{BASE}/graphql", json={"query": "{ sales(limit: 100) { id product quantity unitPrice createdAt } }"}, timeout=30), n),
    ]
    os.makedirs("results", exist_ok=True)
    path = "results/benchmark.csv"
    with open(path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=tests[0].keys())
        writer.writeheader()
        writer.writerows(tests)
    print(f"Resultados guardados en {path}")
    for row in tests:
        print(row)

if __name__ == "__main__":
    main()
