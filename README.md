# Postgres-Hub

### **Postgres-Hub** is a centralized PostgreSQL service designed for multi-stack Docker environments. It enables multiple application stacks to connect securely and efficiently to a shared database backend via a dedicated external Docker network.

## 🌴 Why a Centralized Postgres-Hub?

Instead of deploying isolated PostgreSQL instances per application stack, Postgres-Hub serves as a unified, secure database backend for all Dockerized services. This architecture offers several key advantages:

- 🥥 **Security & Access Control**
  Centralizing authentication and user management simplifies privilege auditing and reduces surface area for misconfigurations.
- 🥥 **Performance Optimization**
  Shared caching, connection pooling, and resource tuning can be applied globally, improving efficiency across stacks.
- 🥥 **Modular Stack Integration**
  Application stacks become lighter and more portable by offloading database responsibilities to the hub. Each stack simply connects via the postgres-hub-network.
- 🥥 **Simplified Backups & Monitoring**
  One database to monitor, one backup routine to maintain. This streamlines disaster recovery and observability.
- 🥥 **Reusability & Scalability**
  New services can be onboarded quickly by provisioning new databases and users—without duplicating infrastructure.
- 🥥 **Developer Experience**
  Local development mirrors production more closely, reducing friction and improving consistency across environments.
  This design reflects the 305 Automations philosophy: modular, secure, and client-friendly infrastructure that scales with clarity.

## 🌴 Creating the External Network

**Before** launching Postgres-Hub or any dependent application stack, create the shared bridge network:

```bash
docker network create postgres-hub-network
```

### This command establishes a user-defined bridge network that persists independently of any single Docker Compose project. It’s the linchpin of the cross-stack architecture, allowing seamless inter-container communication across isolated stacks.

🥥 **Note:** Any container that needs access to a PostgreSQL database must be connected to the `postgres-hub-network`. This ensures secure, direct communication with the Postgres-Hub service while maintaining stack isolation.

To connect an app container to the network, add this to its Compose file:

```Yaml
networks:
  postgres-hub-network:
    external: true
```

🥥 Alternatively, if you're using multiple networks:

```Yaml
services:
  your-app:
    image: your-app-image
    networks:
      - postgres-hub-network
      - other-internal-network

networks:
  postgres-hub-network:
    external: true
  other-internal-network:
    driver: bridge
```

🥥 **Best Practice:** Keep your internal service networks separate from shared infrastructure like Postgres-Hub. This improves security and makes debugging easier.

## 🌴 Per-Container Database Isolation

To maximize security and maintain clean separation of concerns, each Dockerized application stack should use its own dedicated database and user credentials within Postgres-Hub. This approach ensures:

- 🥥 Least privilege access — apps only interact with the data they own
- 🥥 Simplified auditing — track usage and performance per container
- 🥥 Easier maintenance — drop or migrate individual stacks without affecting others
- 🥥 Improved fault isolation — one misbehaving app won’t compromise others

## 🌴 Creating a New Database

To provision a new database and user with full privileges:

```Bash
docker exec -i postgres-hub psql -U postgres_super_user -d the_lord_of_dbs <<EOSQL
CREATE USER new_user WITH PASSWORD 'new_pass';
CREATE DATABASE new_db OWNER new_user;
GRANT ALL PRIVILEGES ON DATABASE new_db TO new_user;
EOSQL
```

```Bash
docker exec -i postgres-hub psql -U postgres_super_user -d new_db <<EOSQL
ALTER SCHEMA public OWNER TO new_user;
GRANT ALL PRIVILEGES ON SCHEMA public TO new_user;
GRANT CREATE ON SCHEMA public TO new_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO new_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO new_user;
EOSQL
```

## 🌴 Fixing Collation Warnings

To resolve collation-related warnings (especially after system upgrades or locale changes):

```Bash
docker exec -i postgres-hub psql -U db_user -d app_db <<EOSQL
REINDEX DATABASE app_db;
ALTER DATABASE app_db REFRESH COLLATION VERSION;
EOSQL
```

## 🌴 Deleting a Database and User (Use with Caution)

To safely remove a database and its associated user:

```Bash
docker exec -i postgres-hub psql -U postgres_super_user -d the_lord_of_dbs <<'EOSQL'
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'new_db';
DROP DATABASE IF EXISTS new_db;
REASSIGN OWNED BY new_user TO postgres_super_user;
DROP OWNED BY new_user;
DROP USER IF EXISTS new_user;
EOSQL

```

⚠️ Warning: This operation is irreversible. Ensure backups are in place before executing.

## 🌴 pgAdmin Integration

pgAdmin is bundled into the Postgres-Hub stack to provide a secure, browser-based interface for managing databases, users, and queries. It’s ideal for:

- 🥥 Visualizing schema relationships
- 🥥 Running ad-hoc queries and diagnostics
- 🥥 Managing roles, privileges, and backups
- 🥥 Empowering less technical users with GUI access
