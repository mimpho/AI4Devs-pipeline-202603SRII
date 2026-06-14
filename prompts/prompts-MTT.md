# Historial de prompts – AI4Devs Pipeline

> **Contexto del Entorno:** Todo este historial se ha desarrollado utilizando el **IDE Antigravity**. Se empezó utilizando el modelo **Claude Sonnet 4.6** y, posteriormente, se cambió al modelo **Gemini 3.1 Pro**.

---

## Sesión 0 – Preparación del Prompt (En LLM Gemini web)

### 🧑 Usuario
*Antes de abrir el IDE, se le proporcionó al chat web de Gemini todo el enunciado del ejercicio del máster. A través de un par de iteraciones, se le pidió al LLM que definiera un rol específico (Ingeniero Cloud y DevOps Senior) y extrajera las tareas concretas para crear un único "Prompt Maestro".*

### 🤖 Gemini
*Generó el prompt optimizado que se utilizó a continuación como punto de partida dentro del IDE Antigravity.*

---

## Sesión 1 – Análisis del proyecto y generación del pipeline

### 🧑 Usuario
Rol: Actúa como un Ingeniero Cloud y DevOps Senior experto en automatización de pipelines de CI/CD con GitHub Actions y despliegues en AWS.

Contexto: Necesito automatizar el flujo de integración y despliegue continuo del proyecto.

Tarea: > 1. Analiza los archivos de la raíz de este proyecto (package.json, README.md y docker-compose.yml) para identificar qué tecnología utiliza este backend, qué comandos usa para ejecutar los tests y cómo genera el build de producción.
2. Con esa información, genera el código YAML completo para el archivo .github/workflows/pipeline.yml que cumpla estrictamente con:

Trigger: Debe dispararse únicamente cuando se haga un push a una rama con un Pull Request abierto hacia la rama main.

Job de Tests: Checkout del código, configuración del entorno detectado, instalación de dependencias y ejecución de sus tests.

Job de Build: Generación de la compilación/build según la tecnología detectada.

Job de Despliegue: Conexión segura a AWS EC2 mediante SSH usando secretos de GitHub para las credenciales (AWS_EC2_HOST, AWS_EC2_USER y AWS_SSH_KEY), transfiriendo el build y reiniciando el servicio.

Requisito de formato: Dame el archivo YAML limpio, optimizado y optimiza el uso de cachés para las dependencias según la tecnología que hayas encontrado. Explícame brevemente las decisiones que has tomado tras analizar mi código.

Ves guardando en el archivo prompts/prompts.md las conversaciones que vayamos teniendo aqui sintetizando para que no quede demasiado verboso

### 🤖 Agente (análisis y decisiones)

**Stack detectado:**
| Componente | Tecnología |
|---|---|
| Runtime | Node.js 20 (LTS) |
| Lenguaje | TypeScript 4.9 |
| Framework | Express 4 |
| ORM | Prisma 5 |
| Tests | Jest 29 + ts-jest |
| Build | `tsc` → `backend/dist/` |
| DB (dev) | PostgreSQL vía Docker |

**Comandos clave del `backend/package.json`:**
- Test: `npm test` (jest con preset ts-jest, entorno node)
- Build: `npm run build` (ejecuta `tsc`, output en `dist/`)
- Start prod: `node dist/index.js`

**Decisiones técnicas:**
1. **Trigger `pull_request` → `main`**: GitHub dispara este evento en cada push sobre una rama cuya PR apunta a `main`. Es el equivalente exacto al requisito.
2. **`concurrency`**: cancela runs redundantes si se hace push rápido (ahorra minutos de Actions).
3. **Caché npm**: se usa `actions/setup-node@v4` con `cache: 'npm'` y `cache-dependency-path: backend/package-lock.json`. Cachea `~/.npm` por hash del lock file.
4. **`npm ci`** en lugar de `npm install`: reproducible, más rápido en CI, falla si el lock no está sincronizado.
5. **Secuencia con `needs`**: `test → build → deploy`. El deploy nunca se ejecuta si los tests o el build fallan.
6. **Artefacto empaquetado**: se comprime solo `dist/`, `package.json`, `package-lock.json` y `prisma/` en un `build.tar.gz`. Se excluyen `node_modules` y código fuente `.ts` para reducir el tamaño de transferencia.
7. **Deploy vía SSH nativo**: se usa `ssh-keyscan` para añadir el host a `known_hosts` (evita prompt interactivo). La clave privada se escribe desde el secret `AWS_SSH_KEY` con permisos `600`.
8. **`npm ci --omit=dev`** en EC2: instala solo dependencias de producción, más ligero.
9. **`npx prisma generate`** en servidor: regenera el cliente Prisma adaptado a la arquitectura del servidor (necesario si difiere de la del runner).
10. **PM2**: gestor de procesos recomendado por el propio README del proyecto. Se hace `pm2 restart || pm2 start` para cubrir tanto el primer despliegue como los siguientes.

**Archivo generado:** `.github/workflows/pipeline.yml`

---

## Sesión 2 – Configuración de Repositorios y Servidor AWS (EC2)

### 🧑 Usuario
- Preguntas sobre qué instancia de EC2 crear (Ubuntu 24 vs 26, t2.micro vs t3.micro).
- Duda sobre la regla SSH en el Security Group de AWS (origen `0.0.0.0/0` vs "Mi IP").

### 🤖 Agente (resolución y guía)

**1. Creación de la instancia EC2:**
- **OS**: Ubuntu 26.04 LTS (o 24.04 LTS, ambas compatibles).
- **Tipo**: `t3.micro` (mejor rendimiento que t2.micro, 2 vCPUs y entra en el Free Tier).
- **Security Group**: 
  - Puerto 22 (SSH) abierto a `0.0.0.0/0` (necesario para que las GitHub Actions, que tienen IPs dinámicas, puedan desplegar).
  - Puerto 3010 (TCP Custom) abierto a `0.0.0.0/0` para exponer la API.

**2. Troubleshooting SSH y permisos:**
- Se aplicaron los permisos correctos (`chmod 400`) y se proporcionó el comando SSH exacto con `-o StrictHostKeyChecking=accept-new` para evitar bloqueos por el fingerprint.
- Se proporcionaron los scripts de instalación de Node.js 20, PM2 y creación de carpetas necesarias (`~/app`).

**3. Configuración de GitHub Secrets:**
Instrucciones para configurar en el fork:
- `AWS_EC2_HOST`: IP pública de la instancia.
- `AWS_EC2_USER`: `ubuntu`
- `AWS_SSH_KEY`: Contenido del archivo `.pem`.

---

## Sesión 3 – Validación del Pull Request y Ajustes Finales

### 🧑 Usuario
* Crea la PR hacia la organización y se encuentra con el bloqueo: "1 workflow awaiting approval".
* Tras desplegar con éxito, pregunta por un warning de PM2: `Error: [ERROR] Process or Namespace backend not found`.
* Pregunta por un warning de GitHub Actions relacionado con la deprecación de Node.js 20.

### 🤖 Agente
**1. Bloqueo de la PR en la Organización:**
Se le explicó que por motivos de seguridad, las GitHub Actions están bloqueadas para contribuidores externos en el repositorio de la organización. Para validar el ejercicio y sacar capturas, se le instruyó a **abrir una PR dentro de su propio fork** (`pipeline-MTT` → `main` en su cuenta personal), donde las Actions tienen permiso automático.

**2. Warning de PM2:**
Se le aclaró que el error `Process backend not found` es normal en el **primer despliegue** porque el proceso de PM2 aún no existe. Gracias al comando `pm2 restart backend || pm2 start dist/index.js --name backend`, el operador `||` captura ese fallo y arranca el proceso exitosamente, tal y como se evidenció en los logs.

**3. Warning de Node.js 20 en GitHub Actions:**
Para entregar un pipeline perfectamente limpio, se le añadió la variable de entorno `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: 'true'` en el archivo `pipeline.yml`. Esto fuerza a las actions a usar internamente Node 24 y elimina el mensaje de deprecación amarillo, mejorando la presentación visual de la ejecución.
