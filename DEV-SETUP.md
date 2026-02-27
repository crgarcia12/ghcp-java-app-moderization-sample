# Development Environment Setup Guide

Clone the repo, open a dev container, and start coding. Everything runs in Docker — no local JDK, Maven, PostgreSQL, or RabbitMQ installation required.

---

## Prerequisites

- [Git](https://git-scm.com/downloads)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- An IDE with [dev container support](https://containers.dev/supporting):
  - **VS Code** — install the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
  - **JetBrains (IntelliJ, etc.)** — built-in support (2023.1+)
  - **Visual Studio** — built-in support (17.4+)
  - **GitHub Codespaces** — works with no local setup at all
  - **Dev container CLI** — `npm install -g @devcontainers/cli` for terminal-only workflows

## Getting Started

**1. Clone the repository**

```bash
git clone https://github.com/microsoft/ghcp-java-app-moderization-sample.git
cd ghcp-java-app-moderization-sample
```

**2. Open in a dev container**

Choose the configuration you need:

| Configuration | Use case |
|---|---|
| **Asset Manager - JDK 8 (Source)** | Running the app in its original state (before migration) |
| **Asset Manager - JDK 25 (Target)** | Running the app after migration to modern Java |

How to open depends on your IDE:

- **VS Code**: Command Palette → *Dev Containers: Reopen in Container* → pick a configuration
- **JetBrains**: File → Remote Development → Dev Containers → select the `.devcontainer` folder
- **Visual Studio**: File → Open → Folder → select the repo, then choose the dev container when prompted
- **CLI**: `devcontainer up --workspace-folder .`

The container will automatically:
1. Pull and start PostgreSQL and RabbitMQ (with health checks — they're ready before you are)
2. Install the correct JDK and Maven
3. Pre-download all Maven dependencies

**3. Run the application**

Open a terminal inside the container and run:

```bash
# Start the web module
./mvnw -pl web spring-boot:run 

# Start the worker module
./mvnw -pl worker spring-boot:run 
```

**4. Open the app**

| Service | URL | Credentials |
|---|---|---|
| Web app | http://localhost:8080 | — |
| RabbitMQ management | http://localhost:15672 | `guest` / `guest` |

## Switching JDK Versions

Rebuild the container with a different configuration:

- **VS Code**: Command Palette → *Dev Containers: Reopen in Container* → pick the other configuration
- **JetBrains / Visual Studio**: Close the container, reopen with the other `.devcontainer` config
- **CLI**: `devcontainer up --workspace-folder . --config .devcontainer/jdk25/devcontainer.json`

## What's Included

| Component | Details |
|---|---|
| JDK | 8 or 25 (depending on configuration) |
| Maven | Latest version |
| PostgreSQL | Port 5432 — user: `postgres`, password: `postgres`, database: `assets_manager` |
| RabbitMQ | Port 5672 — management UI on port 15672 (`guest` / `guest`) |
| Port forwarding | 8080, 5432, 5672, 15672 |

## File Structure

```
.devcontainer/
  docker-compose.yml       # App + PostgreSQL + RabbitMQ services (with health checks)
  jdk8/
    devcontainer.json      # JDK 8 dev container configuration
  jdk25/
    devcontainer.json      # JDK 25 dev container configuration
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `localhost:8080` connection refused | Check terminal for build errors. The JDK 8 config runs the original app; JDK 25 is for the migrated version. |
| Lombok errors (`cannot find symbol`) | Switch to the JDK 8 configuration — the original app requires JDK 8. |
| `git status` shows "dubious ownership" | Should auto-resolve on container start. If not: `git config --global --add safe.directory /workspaces/app` |
| Container build fails | Ensure Docker Desktop is running and has enough resources (4 GB RAM minimum recommended). |

