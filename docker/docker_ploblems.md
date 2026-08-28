# React + Docker Development Environment

This project demonstrates how to run a **React application with Vite inside Docker**, including live code updates using Docker volumes and Vite polling.

The setup uses:

* React
* Vite
* Node.js 20 Alpine
* Docker
* Docker Desktop
* Docker volumes
* Git Bash on Windows

---

## Project Structure

```text
react-docker/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
├── vite.config.js
├── index.html
└── src/
    ├── App.jsx
    ├── main.jsx
    └── ...
```

---

# 1. Dockerfile

The Dockerfile creates a Node.js environment for the React application.

Example:

```dockerfile
FROM node:20-alpine

RUN addgroup app && adduser -S -G app app

WORKDIR /app

COPY package*.json ./

RUN chown -R app:app .

USER app

RUN npm install

COPY . .

EXPOSE 5173

CMD ["npm", "run", "dev"]
```

### Important parts

### Base image

```dockerfile
FROM node:20-alpine
```

Uses Node.js 20 with Alpine Linux.

### Working directory

```dockerfile
WORKDIR /app
```

The application will live inside `/app`.

### Dependencies

```dockerfile
COPY package*.json ./
RUN npm install
```

The package files are copied first so Docker can cache the dependency installation layer.

### Port

```dockerfile
EXPOSE 5173
```

Documents that the application uses port `5173`.

### Start application

```dockerfile
CMD ["npm", "run", "dev"]
```

Starts Vite.

---

# 2. Vite Configuration

When Vite runs inside Docker, it must listen on all network interfaces.

Create:

```text
vite.config.js
```

with:

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    host: '0.0.0.0',
    watch: {
      usePolling: true,
    },
  },
})
```

## Why `host: '0.0.0.0'`?

By default, Vite may listen only on:

```text
localhost
```

inside the container.

That means Docker cannot expose the application correctly to the Windows host.

Using:

```js
host: '0.0.0.0'
```

makes Vite listen on all interfaces.

The expected output becomes:

```text
VITE ready

➜ Local:   http://localhost:5173/
➜ Network: http://172.17.x.x:5173/
```

---

# 3. Vite File Watching and Docker

When using Docker volumes on Windows, filesystem events may not be detected correctly by Vite.

For example, this can happen:

```text
Windows file
     ↓
Docker volume
     ↓
/app
     ↓
Vite
     ↓
No update detected
```

To solve this, Vite polling is enabled:

```js
watch: {
  usePolling: true,
}
```

Polling makes Vite actively check for file changes.

This is especially useful for:

* Docker Desktop
* Windows
* Git Bash
* bind-mounted development directories

---

# 4. `package.json`

The development script should be:

```json
{
  "scripts": {
    "dev": "vite --host 0.0.0.0"
  }
}
```

## Important: Do NOT use `--watch`

We initially tried:

```json
"dev": "vite --host 0.0.0.0 --watch"
```

This caused:

```text
CACError: Unknown option `--watch`
```

Vite's development server does not require `--watch` for normal development.

File watching is configured through:

```js
server: {
  watch: {
    usePolling: true
  }
}
```

inside `vite.config.js`.

Therefore:

```json
"dev": "vite --host 0.0.0.0"
```

is the correct command.

---

# 5. Building the Docker Image

One of the first problems encountered was:

```bash
docker build -t react-docker
```

which produced:

```text
docker: 'docker buildx build' requires 1 argument
```

## Why?

Docker requires a **build context**.

The missing argument was:

```text
.
```

The dot means:

> Use the current directory as the build context.

The correct command is:

```bash
docker build -t react-docker .
```

### Explanation

```text
docker build
    ↓
-t react-docker
    ↓
.
    ↓
current directory
```

Docker then searches the current directory for the Dockerfile and other files required for the build.

---

# 6. Successful Docker Build

A successful build looks similar to:

```text
[+] Building ... FINISHED

...
=> exporting to image
=> naming to docker.io/library/react-docker:latest
```

Verify the image:

```bash
docker images
```

You should see:

```text
REPOSITORY     TAG       IMAGE ID
react-docker   latest    ...
```

---

# 7. Running the Container

The basic command is:

```bash
docker run -p 5173:5173 react-docker
```

The port mapping means:

```text
-p HOST_PORT:CONTAINER_PORT

5173:5173
```

So:

```text
Windows
localhost:5173
       │
       ▼
Docker
       │
       ▼
Container:5173
       │
       ▼
Vite
```

Then open:

```text
http://localhost:5173
```

---

# 8. Port Already Allocated

One of the first errors encountered was:

```text
Bind for 0.0.0.0:5173 failed:
port is already allocated
```

This means another process was already using port `5173`.

## Find the process

On Windows:

```bash
netstat -ano | findstr :5173
```

Example:

```text
TCP    0.0.0.0:5173    0.0.0.0:0    LISTENING    25984
TCP    [::]:5173       [::]:0       LISTENING    25984
```

The last number is the PID:

```text
25984
```

Find the process:

```bash
tasklist | findstr 25984
```

---

# 9. Git Bash and `taskkill`

When using Git Bash, this command:

```bash
taskkill /PID 25984 /F
```

can be interpreted incorrectly.

Git Bash may produce:

```text
ERROR: Invalid argument/option -
'C:/ProgramFiles/Git/PID'
```

Use:

```bash
winpty taskkill /PID 25984 /F
```

Alternatively:

```bash
cmd.exe /c "taskkill /PID 25984 /F"
```

Then verify:

```bash
netstat -ano | findstr :5173
```

If there is no output, port `5173` is free.

---

# 10. Using Another Host Port

Instead of stopping the process using `5173`, another option is:

```bash
docker run -p 5174:5173 react-docker
```

This means:

```text
Windows:5174
      ↓
Docker:5173
      ↓
Vite:5173
```

Open:

```text
http://localhost:5174
```

Notice that the container port remains `5173`.

---

# 11. Docker Engine Not Running

Another error encountered was:

```text
docker: error during connect:
open //./pipe/dockerDesktopLinuxEngine:
The system cannot find the file specified.
```

This usually means Docker Desktop's Linux engine isn't running.

## Solution

Start:

```text
Docker Desktop
```

Wait until Docker Desktop finishes starting.

Then test:

```bash
docker version
```

A working Docker installation should show both:

```text
Client:
...

Server:
...
```

You can also run:

```bash
docker info
```

---

# 12. Docker Containers vs Docker Images

An important distinction:

## Image

An image is the template used to create containers.

Check images:

```bash
docker images
```

Example:

```text
react-docker
```

## Container

A container is a running or stopped instance of an image.

Check containers:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

Stop a container:

```bash
docker stop CONTAINER_ID
```

For example:

```bash
docker stop 6865afe197e3
```

Do NOT use:

```bash
docker stop 6865afe197e3...
```

if that value is actually an image ID.

---

# 13. Docker Container Cleanup

We used:

```bash
docker container prune
```

This removes **stopped containers**.

Docker asks:

```text
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N]
```

Answer:

```text
y
```

This does NOT remove:

* Running containers
* Docker images
* Dockerfiles
* Source code

It only removes stopped containers.

---

# 14. Bind Mounts for Development

For development, we used:

```bash
docker run -it \
  -p 5173:5173 \
  -v "$(pwd):/app" \
  -v /app/node_modules \
  react-docker
```

On one line:

```bash
docker run -it -p 5173:5173 -v "$(pwd):/app" -v /app/node_modules react-docker
```

## First volume

```bash
-v "$(pwd):/app"
```

This mounts the current Windows project directory into `/app`.

Therefore:

```text
Windows project
      │
      ▼
Docker /app
```

Changes made locally become available inside the container.

---

# 15. Why `/app/node_modules` Is a Separate Volume

We also use:

```bash
-v /app/node_modules
```

This prevents the host's `node_modules` from replacing the container's Linux `node_modules`.

The idea is:

```text
Windows project
      │
      ├── source files
      │
      └── package.json
      │
      ▼
    /app
      │
      └── node_modules
          │
          └── container-managed dependencies
```

This is useful when developing Node applications on Windows while the application runs inside a Linux container.

---

# 16. Development Command

The recommended development command is:

```bash
docker run -it -p 5173:5173 -v "$(pwd):/app" -v /app/node_modules react-docker
```

Then Vite should output:

```text
VITE ready

➜ Local:   http://localhost:5173/
➜ Network: http://172.17.x.x:5173/
```

Open:

```text
http://localhost:5173
```

---

# 17. Live Reload

The complete live-reload setup requires three things.

### 1. Vite listens externally

```js
server: {
  host: '0.0.0.0',
}
```

### 2. Vite polls for changes

```js
watch: {
  usePolling: true,
}
```

### 3. Docker mounts the source code

```bash
-v "$(pwd):/app"
```

Together:

```text
Windows source code
       │
       │ bind mount
       ▼
Docker /app
       │
       ▼
Vite
       │
       │ polling
       ▼
Detect file change
       │
       ▼
HMR
       │
       ▼
Browser updates
```

---

# 18. Common Problems and Solutions

| Problem                                                | Cause                                     | Solution                         |
| ------------------------------------------------------ | ----------------------------------------- | -------------------------------- |
| `docker buildx build requires 1 argument`              | Missing build context                     | `docker build -t react-docker .` |
| `port is already allocated`                            | Port 5173 already in use                  | Find process or use `5174:5173`  |
| `dockerDesktopLinuxEngine` not found                   | Docker Desktop not running                | Start Docker Desktop             |
| `Unknown option --watch`                               | Invalid Vite CLI argument                 | Remove `--watch`                 |
| Vite works inside container but browser cannot connect | Vite listening on localhost               | Use `host: '0.0.0.0'`            |
| Files don't update                                     | Windows/Docker filesystem events          | Enable `usePolling: true`        |
| `docker stop` says no such container                   | ID belongs to an image or wrong container | Check `docker ps -a`             |
| `taskkill /PID` fails in Git Bash                      | Git Bash interprets Windows option        | Use `winpty taskkill ...`        |

---

# 19. Recommended Workflow

Every time you start working on the project:

### Start Docker Desktop

Make sure Docker Desktop is running.

### Navigate to the project

```bash
cd ~/Order/programming/DevOpss/npm-docker/react-docker
```

### Build the image when Dockerfile/dependencies change

```bash
docker build -t react-docker .
```

### Start development

```bash
docker run -it -p 5173:5173 -v "$(pwd):/app" -v /app/node_modules react-docker
```

### Open the application

```text
http://localhost:5173
```

### Modify React code

Edit files locally:

```text
src/App.jsx
src/main.jsx
...
```

Save the changes.

Vite should detect them through polling and update the browser.

### Stop the development container

Press:

```text
Ctrl + C
```

---

# 20. Useful Docker Commands

### List running containers

```bash
docker ps
```

### List all containers

```bash
docker ps -a
```

### List images

```bash
docker images
```

### Build image

```bash
docker build -t react-docker .
```

### Run container

```bash
docker run -p 5173:5173 react-docker
```

### Run development container with volumes

```bash
docker run -it -p 5173:5173 -v "$(pwd):/app" -v /app/node_modules react-docker
```

### Stop container

```bash
docker stop CONTAINER_ID
```

### Remove container

```bash
docker rm CONTAINER_ID
```

### Remove stopped containers

```bash
docker container prune
```

### Remove an image

```bash
docker rmi react-docker
```

### See container logs

```bash
docker logs CONTAINER_ID
```

---

# 21. Final Working Configuration

The important pieces are:

### `package.json`

```json
{
  "scripts": {
    "dev": "vite --host 0.0.0.0"
  }
}
```

### `vite.config.js`

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    host: '0.0.0.0',
    watch: {
      usePolling: true,
    },
  },
})
```

### Build

```bash
docker build -t react-docker .
```

### Run

```bash
docker run -it -p 5173:5173 -v "$(pwd):/app" -v /app/node_modules react-docker
```

### Application

```text
http://localhost:5173
```

---

# Conclusion

The main issues encountered during the setup were not problems with React itself. They were related to the interaction between:

```text
Windows
   ↓
Git Bash
   ↓
Docker Desktop
   ↓
Linux container
   ↓
Vite
```

The final solution uses:

* Docker port mapping
* Vite `0.0.0.0` host configuration
* Docker bind mounts
* A separate `node_modules` volume
* Vite polling for Windows filesystem changes

With this configuration, the React application runs inside Docker while the source code remains editable from the local Windows environment with live updates.
