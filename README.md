Maintained by [Ankara Yazılım](https://github.com/Ankara-Software) | [Project page](https://ankarayazilim.org/acik-kaynak/mockforge)

**Status: stable.** It builds, the 2 unit tests pass and the flows in this README (tenant routing, stateful CRUD, rules, hot reload) were run against a local instance. Resource figures such as memory use and latency have not been measured.

# MockForge

<p align="center">
  <img src="assets/banner.png" alt="MockForge Banner" width="100%" />
</p>

<p align="center">
  <img src="assets/logo.png" alt="MockForge Logo" width="150" />
</p>

<h3 align="center">A multi-tenant mock API gateway</h3>

<p align="center">
  <strong>A mock server that simulates several backend services and simple CRUD data from one YAML file, with hot reloading and in-memory state.</strong>
</p>

## Türkçe

MockForge, tek bir YAML dosyasından birden fazla arka uç servisini ve basit CRUD verisini taklit eden bir mock API sunucusudur. İstekleri sunucu adına (`host`) ya da yol önekine (`prefix`) göre ilgili servise yönlendirir. `POST`, `PUT`, `PATCH` ve `DELETE` istekleri bellekteki veriyi günceller, sonraki `GET` istekleri güncel veriyi döndürür. Yanıtlar `params`, `query`, `headers` ve `body` üzerinde yazılan koşullara göre değişebilir. Yapılandırma dosyası değiştiğinde sunucu yeniden başlatılmadan güncellenir. Veri yalnızca bellekte tutulur, sunucu kapanınca kaybolur.

---

## The Pain Point

Setting up many backend services locally for integration tests or frontend development is slow and uses a lot of system resources. MockForge replaces them with a single compiled Rust binary. It routes traffic for multiple tenants (services) using one YAML configuration file, evaluates matching conditions, and keeps in-memory CRUD state. No Node.js runtime, Docker or external database is needed to run it.

---

## Features

- **Multi-Tenancy**: Route mock API requests by **Hostname** (e.g. `payment.local`) or **Path Prefix** (e.g. `/auth`, `/payment`).
- **In-Memory Stateful CRUD**: Bind mock endpoints to a state store. `POST`, `PUT`, `PATCH`, and `DELETE` requests update the state in RAM, so later `GET` requests return the updated data. `PUT` replaces the whole object with the request body.
- **Rules Engine**: Serve different responses with simple conditions on `params`, `query`, `headers` and `body`, using `==`, `!=`, `contains`, `exists` and `!exists` (e.g. `params.id == '404'` or `headers.x-api-key != 'secret'`).
- **Hot Reloading**: Watches the config file. Changing it updates the routes without a restart.
- **CORS**: Permissive CORS headers are enabled on every response.

---

## Installation & Usage

### 1. Build From Source (Rust Cargo)
```bash
cargo install --git https://github.com/Ankara-Software/mockforge
mockforge --config mockforge.yaml
```

### 2. Run via npx (Node.js required)
The `mockforge-gateway` npm package is a small wrapper that downloads the release binary for your OS:
```bash
npx mockforge-gateway --config mockforge.yaml --port 8080
```

### 3. Global Install via npm
```bash
npm install -g mockforge-gateway
mockforge --config mockforge.yaml
```

### Options
```
-c, --config <FILE>   Config file [default: mockforge.yaml]
-p, --port <PORT>     Port [default: 8080]
-H, --host <HOST>     Address to bind [default: 127.0.0.1]
```

---

## Configuration Guide (`mockforge.yaml`)

Define all your mock microservices in a single, simple configuration file:

```yaml
tenants:
  - name: auth-service
    prefix: /auth
    routes:
      # GET collection - binds to state store "users"
      - path: /users
        method: GET
        state: users
        default:
          status: 200
          body:
            - id: "1"
              name: "Anilcan Kara"
              role: "admin"
            - id: "2"
              name: "Nozich Bot"
              role: "assistant"

      # GET single item - matches id param
      - path: /users/:id
        method: GET
        state: users
        rules:
          - if: "params.id == '404'"
            status: 404
            body: { "error": "User not found" }
        default:
          status: 404
          body: { "error": "User not found in state store" }

      # POST item - inserts JSON payload into "users" list
      - path: /users
        method: POST
        state: users

      # PUT item - updates the object by ID
      - path: /users/:id
        method: PUT
        state: users

      # DELETE item - removes from state
      - path: /users/:id
        method: DELETE
        state: users

      # Conditional Headers & Response matching
      - path: /status
        method: GET
        rules:
          - if: "headers.x-api-key != 'secret'"
            status: 401
            body: { "error": "Unauthorized" }
        default:
          status: 200
          body: { "status": "healthy" }

  - name: payment-service
    host: payment.local
    routes:
      - path: /charge
        method: POST
        default:
          status: 200
          body: { "chargeId": "ch_12345", "status": "succeeded" }
```

---

## Architecture & Request Life Cycle

```
                      [ Client Request ]
                              │
                              ▼
                      [ Resolves Tenant ]
                     ├── Host Match: host.local
                     └── Prefix Match: /prefix
                              │
                              ▼
                      [ Matches Route ]
                     (Method + Path Pattern)
                              │
                              ▼
                     [ Evaluates Rules ]
             (Evaluates conditions against request)
                ├── Match? ──► Return custom response
                └── No Match
                              │
                              ▼
                     [ Stateful CRUD? ]
                ├── Yes ──► Read/Write state store (RAM)
                └── No  ──► Return route default
```

---

## Verification & Manual Testing

1. **Start MockForge**:
   ```bash
   npx mockforge-gateway --config mockforge.yaml
   ```
2. **Fetch all users** (GET collection):
   ```bash
   curl http://localhost:8080/auth/users
   ```
3. **Insert a new user** (POST):
   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"name": "Alice"}' http://localhost:8080/auth/users
   ```
4. **Fetch all users again** to verify the state was updated:
   ```bash
   curl http://localhost:8080/auth/users
   ```
5. **Verify path param matching and rules**:
   ```bash
   curl http://localhost:8080/auth/users/404
   ```

---

## Development

```bash
cargo build
cargo test
```

See [CONTRIBUTING.md](CONTRIBUTING.md), [SECURITY.md](SECURITY.md) and [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

## License

MIT. See [LICENSE](LICENSE).
