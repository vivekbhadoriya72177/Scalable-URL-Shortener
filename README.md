# Scalable URL Shortener (System Design Implementation)

**Short Description:** This project moves beyond standard CRUD architecture to solve real-world scaling bottlenecks. By decoupling ID generation into a background worker (KGS) and layering Redis for cache-aside reads, this URL shortener is engineered to survive database write-locks and viral traffic spikes (the "Thundering Herd" problem).

A high-performance URL shortener built to demonstrate core system design principles, including high-availability read paths, write-lock mitigation, and concurrent traffic handling. 

## 🚀 Features & Architecture

* **Key Generation Service (KGS):** A background worker that pre-generates unique 7-character Base62 strings into an in-memory queue. This eliminates on-the-fly computational overhead and prevents database write-lock collisions under heavy concurrent `POST` traffic.
* **Redis Caching Layer:** Handles read-heavy workloads (100:1 read-to-write ratio). Implements a cache-aside pattern to serve `GET` redirects directly from RAM, protecting the database from I/O spikes during viral link surges.
* **Optimized Database Schema:** Utilizes a relational MySQL database with a `UNIQUE INDEX` (B-Tree) on the short alias, ensuring O(log N) lookup times for cache misses rather than O(N) full table scans.
* **302 Temporary Redirects:** Forces the browser to hit the server for every redirect, laying the architectural groundwork for asynchronous click-tracking and analytics.
* **Vanilla Frontend:** A lightweight, zero-dependency HTML/JS interface to interact with the API.

## 🛠 Tech Stack

* **Backend:** Node.js, Express
* **Primary Database:** MySQL (using parameterized queries to prevent SQL injection)
* **Cache:** Redis
* **Frontend:** Vanilla HTML/CSS/JavaScript
* **Dependencies:** `mysql2`, `redis`, `nanoid` (for Base62 generation)

## 🧠 System Design Flow

### The Write Path (Creating a Link)
1. The background KGS worker continuously fills a memory array with unique Base62 IDs.
2. Client sends a `POST` request with a long URL.
3. The server pops a pre-validated ID from the KGS queue in O(1) time.
4. The server maps the ID to the URL and saves it to MySQL.

### The Read Path (Redirecting)
1. Client requests the short link (`GET /:short_id`).
2. Server checks Redis.
   * **Cache Hit:** Redirects immediately from RAM (sub-millisecond latency).
   * **Cache Miss:** Queries MySQL via the B-Tree index, fetches the long URL, writes it to Redis with a 1-hour TTL (Time To Live), and redirects the client.
