# DevHex

A hexagonal-architecture Go library for abstracting development environment backends behind a single port interface. DevHex provides uniform access to Docker, Podman, Nix, process-compose, and native process management, enabling seamless switching between containerized and local development environments.

## Architecture

```
cmd/devenv/          CLI entry point (optional)
pkg/
  domain/            Core ports + domain types (no external deps)
    environment.go   Environment port interface + Config/Status types
    registry.go      Adapter registry (Register + New)
  adapters/
    docker/          Docker/Podman adapter (wraps docker/docker SDK)
    nix/             Nix flake / nix-shell adapter
    native/          process-compose / bare-process adapter
```

The `domain` package has zero external dependencies. Adapters import it and implement the `Environment` interface. Consumers depend only on `domain`.

## Usage

```go
import (
    "context"
    "github.com/KooshaPari/devenv-abstraction/pkg/domain"
    "github.com/KooshaPari/devenv-abstraction/pkg/adapters/docker"
)

func main() {
    reg := domain.NewRegistry()
    reg.Register(domain.BackendDocker, func() domain.Environment {
        a, err := docker.New()
        if err != nil {
            panic(err)
        }
        return a
    })

    env, err := reg.New(domain.BackendDocker)
    if err != nil {
        panic(err)
    }

    ctx := context.Background()
    if err := env.Start(ctx, domain.Config{
        Name:    "my-service",
        Backend: domain.BackendDocker,
        Image:   "golang:1.23",
        Ports:   []domain.PortMapping{{HostPort: 8080, ContainerPort: 8080, Protocol: "tcp"}},
    }); err != nil {
        panic(err)
    }
    defer env.Stop(ctx)

    result, err := env.Exec(ctx, []string{"go", "build", "./..."})
    if err != nil {
        panic(err)
    }
}
```

## Supported Backends

| Backend | Status | Notes |
|---------|--------|-------|
| `docker` | WIP (WP02) | Wraps `docker/docker` SDK v27 |
| `podman` | Planned (WP02) | Reuses docker adapter via socket |
| `nix` | WIP (WP05) | `nix develop` / `nix-shell` |
| `native` | WIP (WP04) | process-compose / bare exec |

## Design Philosophy

- **Hexagonal Architecture** — Domain layer has zero external dependencies; adapters implement the port interface
- **Pluggable Backends** — Register and switch backends at runtime without code changes
- **Type Safety** — Go interfaces ensure compile-time correctness
- **Minimal Wrapping** — Thin adapter layer over native SDKs (docker-go, nix CLI)

## Development

```bash
go build ./...
go test ./...
go mod tidy
```

## Project Status

- **Status**: Active
- **Language**: Go 1.23+
- **Type**: Abstraction Library
- **Part of**: Phenotype Ecosystem
- **Integrates With**: Helios, DevEnv, AgilePlus

## Testing & Quality

- Unit tests for each adapter and domain logic
- Integration tests with real backends (Docker, Nix)
- Functional requirement traceability in AgilePlus
- Code review and linting with golangci-lint

## License

MIT
