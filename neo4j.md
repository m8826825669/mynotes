Here's a complete guide to installing and using Neo4j on Ubuntu.

---

## What is Neo4j?

Neo4j is a **graph database** — instead of rows and tables, it stores data as **nodes** (entities) and **relationships** (connections between them). It excels at querying deeply connected data like social networks, recommendation engines, fraud detection, and knowledge graphs.

---

## 1. Installation

### Prerequisites

Neo4j 2025.x requires Java 21 (default). OpenJDK 21 is usually available by default on supported Ubuntu versions, so no extra setup is typically needed.

### Add the Neo4j Repository

```bash
# Install dependencies
sudo apt install apt-transport-https ca-certificates curl -y

# Add GPG key
curl -fsSL https://debian.neo4j.com/neotechnology.gpg.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/neo4j-archive-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/neo4j-archive-keyring.gpg] \
  https://debian.neo4j.com stable latest" \
  | sudo tee /etc/apt/sources.list.d/neo4j.list

# Install
sudo apt update && sudo apt install neo4j -y
```

### Start & Enable the Service

```bash
sudo systemctl enable neo4j
sudo systemctl start neo4j
sudo systemctl status neo4j
```

### Set Initial Password

```bash
sudo neo4j-admin dbms set-initial-password YourStrongPassword
```

---

## 2. Configuration (`/etc/neo4j/neo4j.conf`)

By default, Neo4j is configured to accept connections from localhost only (127.0.0.1). This ensures it is not exposed to the public internet. To allow remote access, edit the config:

```bash
sudo nano /etc/neo4j/neo4j.conf
```

Key settings to adjust:

```ini
# Allow connections from all interfaces
server.default_listen_address=0.0.0.0

# HTTP Browser UI
server.http.listen_address=:7474

# Bolt protocol (main connection port)
server.bolt.listen_address=:7687

# Memory tuning
server.memory.heap.initial_size=512m
server.memory.heap.max_size=1G
server.memory.pagecache.size=512m
```

Then restart:
```bash
sudo systemctl restart neo4j
```

---

## 3. Firewall Rules

Neo4j creates two network sockets in a default installation — one on port **7474** for the built-in HTTP interface, and the main Bolt protocol on port **7687**. Neo4j recommends not using the HTTP port in production.

```bash
sudo ufw allow 7474/tcp   # Browser UI
sudo ufw allow 7687/tcp   # Bolt protocol (apps connect here)
sudo ufw reload
```

---

## 4. Access Neo4j Browser (Web UI)

Open a web browser and navigate to `http://localhost:7474`. Connect using the username `neo4j` with your password, or the default password `neo4j` — you'll be prompted to change it on first login.

---

## 5. Using Cypher Shell (CLI)

```bash
cypher-shell -u neo4j -p YourStrongPassword
```

---

## 6. Basic Cypher Query Language

Neo4j uses **Cypher** — a SQL-like language for graphs.

### Create Nodes
```cypher
CREATE (alice:Person {name: "Alice", age: 30})
CREATE (bob:Person {name: "Bob", age: 25})
CREATE (techcorp:Company {name: "TechCorp"})
```

### Create Relationships
```cypher
MATCH (a:Person {name: "Alice"}), (b:Person {name: "Bob"})
CREATE (a)-[:FRIENDS_WITH]->(b)

MATCH (a:Person {name: "Alice"}), (c:Company {name: "TechCorp"})
CREATE (a)-[:WORKS_AT]->(c)
```

### Query Data
```cypher
-- Find all persons
MATCH (p:Person) RETURN p

-- Find Alice's friends
MATCH (a:Person {name: "Alice"})-[:FRIENDS_WITH]->(friend)
RETURN friend.name

-- Find who works at TechCorp
MATCH (p:Person)-[:WORKS_AT]->(c:Company {name: "TechCorp"})
RETURN p.name
```

### Update & Delete
```cypher
-- Update a property
MATCH (p:Person {name: "Alice"})
SET p.age = 31

-- Delete a node (must delete relationships first)
MATCH (p:Person {name: "Bob"})
DETACH DELETE p
```

### Indexes (for performance)
```cypher
CREATE INDEX FOR (p:Person) ON (p.name)
```

---

## 7. Key Ports Summary

| Port | Protocol | Purpose |
|------|----------|---------|
| 7474 | HTTP | Neo4j Browser (web UI) |
| 7473 | HTTPS | Secure Browser UI |
| 7687 | Bolt | Application connections |

---

## 8. Useful Commands

```bash
# View logs
sudo tail -f /var/log/neo4j/neo4j.log
sudo tail -f /var/log/neo4j/debug.log

# Check service
sudo systemctl status neo4j

# Restart
sudo systemctl restart neo4j
```

---

**Tip:** For production, tune `server.memory.heap.max_size` and `server.memory.pagecache.size` to roughly 25–50% of your RAM each for best performance.