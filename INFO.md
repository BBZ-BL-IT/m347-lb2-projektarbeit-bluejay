# Nivala – Projektdokumentation

## Projektübersicht

**Nivala** ist eine Full-Stack Food-Delivery-Webapplikation, bestehend aus vier Docker-Containern, die über Docker Compose orchestriert werden.

| Container  | Beschreibung                                    | Port  |
|------------|-------------------------------------------------|-------|
| `mongodb`  | MongoDB 7 Datenbank (persistenter Datenspeicher) | intern |
| `backend`  | Node.js / Express REST-API                      | 4000  |
| `frontend` | React (Vite) – Kundenansicht, via Nginx          | 5173  |
| `admin`    | React (Vite) – Adminpanel, via Nginx             | 5174  |

**Tech-Stack:** React 18, Vite, Node.js 20, Express, MongoDB 7, Mongoose, JWT, Nginx, Docker

---

## Voraussetzungen

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installiert und gestartet
- Git (optional, zum Klonen)

---

## Applikation starten

### Mit lokalen Images (Build)

Baut alle Images lokal und startet die Applikation:

```bash
docker compose -f docker-compose-build.yml up --build
```

### Mit Images von Docker Hub

Zieht fertige Images von Docker Hub und startet die Applikation:

```bash
docker compose -f docker-compose-hub.yml up
```

### Applikation stoppen

```bash
docker compose -f docker-compose-build.yml down
# oder
docker compose -f docker-compose-hub.yml down
```

> **Wichtig:** Die MongoDB-Daten bleiben durch das Volume `mongo_data` erhalten. Selbst nach `docker compose down` und erneutem `up` sind alle Daten noch vorhanden.

---

## Applikation aufrufen

Nach dem Start sind folgende URLs erreichbar:

| Dienst    | URL                        |
|-----------|----------------------------|
| Frontend  | http://localhost:5173       |
| Admin     | http://localhost:5174       |
| Backend   | http://localhost:4000       |

---

## Umgebungsvariablen (.env)

Die Umgebungsvariablen sind in `.env`-Dateien aufgeteilt:

**`.env` (Root – für Docker Compose):**
```env
MONGO_ROOT_USER=admin
MONGO_ROOT_PASSWORD=secret123
MONGO_DB=nivala
BACKEND_PORT=4000
FRONTEND_PORT=5173
ADMIN_PORT=5174
```

**`Backend/.env` (für den Backend-Container):**
```env
PORT=4000
DB_URI=mongodb://admin:secret123@mongodb:27017/nivala?authSource=admin
JWT_SECRET=<geheimschlüssel>
CORS_ORIGIN=http://localhost:5173
```

Die Docker-Compose-Dateien lesen die Root-`.env` automatisch ein. Das Backend lädt `Backend/.env` via `env_file`.

---

## Persistente Datenspeicherung

Die MongoDB-Daten werden in einem Docker Named Volume gespeichert:

```yaml
volumes:
  mongo_data:
```

Das Volume wird dem Container unter `/data/db` eingebunden. Damit bleiben alle Daten (Benutzer, Bestellungen, Speisekarte usw.) auch nach einem `docker compose down` vollständig erhalten.

**Persistenz demonstrieren:**
1. Applikation starten und Daten erfassen (z. B. Gericht hinzufügen)
2. `docker compose down` ausführen
3. `docker compose up` erneut ausführen
4. Daten sind weiterhin vorhanden

---

## Dockerfile-Erklärungen

### Backend (`Backend/Dockerfile`)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev        # nur Produktionsabhängigkeiten

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 4000
CMD ["node", "server.js"]
```

**Multistage Build:** Im ersten Stage werden die Dependencies installiert (`--omit=dev` entfernt devDependencies). Im zweiten Stage wird nur der notwendige Code und die sauberen `node_modules` kopiert. Dadurch enthält das finale Image keinen Build-Overhead und keine Dev-Tools.

### Frontend & Admin (`Frontend/Dockerfile`, `Admin/Dockerfile`)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # erstellt /app/dist

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**Multistage Build:** Im Builder-Stage wird die React-App kompiliert. Im finalen Stage wird nur das statische Build-Artefakt (`dist/`) in ein schlankes Nginx-Image kopiert. Der Sourcecode und die `node_modules` (typisch > 200 MB) landen nicht im finalen Image.

---

## Dev Container

Der Dev Container ermöglicht eine vollständig reproduzierbare Entwicklungsumgebung direkt in VS Code, ohne lokale Installationen.

### Dev Container starten

1. VS Code mit der Extension [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) öffnen
2. Projekt-Ordner in VS Code öffnen
3. Befehlspalette öffnen (`Ctrl+Shift+P`) → **"Dev Containers: Reopen in Container"** wählen
4. VS Code baut den Container und öffnet das Projekt darin

### Entwicklung im Dev Container

**Backend starten (mit Autoreload via Nodemon):**
```bash
cd Backend
npm run server
```
Nodemon überwacht alle `.js`-Dateien und startet den Server automatisch neu, sobald du eine Datei speicherst.

**Frontend starten (mit Hot Module Replacement):**
```bash
cd Frontend
npm run dev
```
Vite's HMR lädt Änderungen sofort im Browser nach, ohne manuellen Refresh.

**Admin starten:**
```bash
cd Admin
npm run dev
```

### Debugging

Das Dev Container enthält die **ESLint**-Extension für automatisches Linting sowie den integrierten VS Code Debugger. Für Node.js-Debugging:

1. In VS Code die Ansicht **Run & Debug** öffnen (`Ctrl+Shift+D`)
2. Konfiguration `Node.js: Launch Program` wählen oder eine `launch.json` anlegen:

```json
{
  "type": "node",
  "request": "launch",
  "name": "Debug Backend",
  "program": "${workspaceFolder}/Backend/server.js",
  "runtimeExecutable": "node",
  "env": { "NODE_ENV": "development" }
}
```

3. Breakpoints im Code setzen und Debugging starten

### Installierte Extensions (`.devcontainer/devcontainer.json`)

| Extension | Zweck |
|-----------|-------|
| `dbaeumer.vscode-eslint` | Linting für JS/JSX – zeigt Fehler direkt im Editor |
| `esbenp.prettier-vscode` | Autoformatierung beim Speichern |
| `mongodb.mongodb-vscode` | Direkte Datenbankverbindung und Abfragen in VS Code |
| `ms-azuretools.vscode-docker` | Docker-Container und Images direkt verwalten |
| `christian-kohler.path-intellisense` | Autocomplete für Import-Pfade |

### Datenbankverbindung im Dev Container

Mit der MongoDB-Extension kann direkt aus VS Code auf die Datenbank zugegriffen werden:

1. Extension öffnen (MongoDB-Symbol in der Seitenleiste)
2. **"Add Connection"** → Verbindungsstring eingeben:
   ```
   mongodb://admin:secret123@localhost:27017/nivala?authSource=admin
   ```
3. Collections durchsuchen, Dokumente lesen und bearbeiten

---

## Netzwerk

Alle Container sind im selben Docker-Netzwerk `nivala_net` verbunden. Das Backend erreicht MongoDB intern über den Hostnamen `mongodb` (Container-Name). Frontend und Admin kommunizieren über den Browser direkt mit dem Backend auf Port 4000.

---

## Projektstruktur

```
m347-lb2-projektarbeit-bluejay/
├── .env                        # Umgebungsvariablen für Docker Compose
├── docker-compose-build.yml    # Compose: lokaler Build
├── docker-compose-hub.yml      # Compose: Docker Hub Images
├── .devcontainer/
│   └── devcontainer.json       # Dev Container Konfiguration
├── Backend/
│   ├── Dockerfile
│   ├── .env
│   ├── server.js
│   ├── config/, controllers/, models/, routes/, middleware/
├── Frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── src/
└── Admin/
    ├── Dockerfile
    ├── nginx.conf
    └── src/
```
