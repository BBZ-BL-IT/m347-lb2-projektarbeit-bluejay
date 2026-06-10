# Nivala – Projektdokumentation

## Projektübersicht

**Nivala** ist eine Full-Stack Food-Delivery-Webapplikation, bestehend aus vier Docker-Containern, die über Docker Compose orchestriert werden.

| Container  | Beschreibung                                     | Port   |
| ---------- | ------------------------------------------------ | ------ |
| `mongodb`  | MongoDB 7 Datenbank (persistenter Datenspeicher) | intern |
| `backend`  | Node.js / Express REST-API                       | 4000   |
| `frontend` | React (Vite) – Kundenansicht, via Nginx          | 5173   |
| `admin`    | React (Vite) – Adminpanel, via Nginx             | 5174   |

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

| Dienst   | URL                   |
| -------- | --------------------- |
| Frontend | http://localhost:5173 |
| Admin    | http://localhost:5174 |
| Backend  | http://localhost:4000 |

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

Der Dev Container ermöglicht eine vollständig reproduzierbare Entwicklungsumgebung direkt in VS Code. Alle Extensions und Einstellungen werden automatisch installiert — egal ob Windows, Mac oder Linux.

### Voraussetzungen

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installiert und gestartet
- VS Code mit der Extension [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installiert

### Dev Container starten

1. Projekt-Ordner in VS Code öffnen
2. `Ctrl+Shift+P` → **"Dev Containers: Reopen in Container"** wählen
3. VS Code startet den Container automatisch und installiert alle Extensions
4. Nach dem Start ist der Workspace unter `/workspace` verfügbar

### Autoreload aktivieren

**Backend (Nodemon):**

```bash
cd /workspace/Backend
npm run server
```

Nodemon überwacht alle `.js`-Dateien und startet den Server automatisch neu, sobald eine Datei gespeichert wird.

**Frontend (Vite HMR):**

```bash
cd /workspace/Frontend
npm run dev
```

Vite's Hot Module Replacement lädt Änderungen sofort im Browser nach, ohne manuellen Refresh.

**Admin:**

```bash
cd /workspace/Admin
npm run dev
```

### Debugging (Breakpoints)

Die Konfiguration liegt in `.vscode/launch.json`. So debuggt man das Backend:

1. `Ctrl+Shift+D` (Run & Debug öffnen)
2. Oben **"Debug Backend"** auswählen
3. Grünen Play-Button klicken
4. Breakpoints setzen (roter Punkt links neben eine Codezeile klicken)
5. Der Server stoppt automatisch an den Breakpoints

### Installierte Extensions

| Extension                            | Zweck                                          |
| ------------------------------------ | ---------------------------------------------- |
| `dbaeumer.vscode-eslint`             | Linting – zeigt JS/JSX Fehler direkt im Editor |
| `esbenp.prettier-vscode`             | Autoformatierung beim Speichern                |
| `mongodb.mongodb-vscode`             | Datenbankverbindung direkt in VS Code          |
| `ms-azuretools.vscode-docker`        | Docker-Container und Images verwalten          |
| `christian-kohler.path-intellisense` | Autocomplete für Import-Pfade                  |

### Datenbankverbindung im Dev Container

1. MongoDB-Symbol in der linken Seitenleiste klicken
2. **"Add Connection"** → folgenden Verbindungsstring eingeben:
   ```
   mongodb://admin:secret123@localhost:27017/nivala?authSource=admin
   ```
3. Collections durchsuchen, Dokumente lesen und bearbeiten

### Dev Container verlassen

`Ctrl+Shift+P` → **"Dev Containers: Reopen Folder Locally"**

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
