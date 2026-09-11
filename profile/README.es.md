[English](README.md) · **Español**

# Juan in One

Una plataforma de Kubernetes con forma de producción, construida desde cero para aprender cada una de sus capas — desde el Dockerfile hasta el admission controller que decide si un Pod puede llegar a ejecutarse.

Tres microservicios en FastAPI y un front en React, entregados por GitOps, con una cadena de suministro que firma cada imagen y un clúster que se niega a ejecutar nada que no pueda verificar.

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Cosign](https://img.shields.io/badge/Cosign-keyless-2F855A)
![Kyverno](https://img.shields.io/badge/Kyverno-Deny-1D4ED8)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![React](https://img.shields.io/badge/React%2019-61DAFB?logo=react&logoColor=black)

<p align="center">
  <img src="img/web-home.png" alt="Inicio de Juan in One" width="620">
</p>

---

## Contenidos

- [Por qué existe esto](#por-qué-existe-esto)
- [Arquitectura](#arquitectura)
- [Acceso](#acceso)
- [Repositorios](#repositorios)
- [Cadena de suministro](#cadena-de-suministro)
- [Stack tecnológico](#stack-tecnológico)
- [Decisiones de ingeniería](#decisiones-de-ingeniería)

---

## Por qué existe esto

Lo construí para aprender, y la forma del proyecto sale de ahí.

Las aplicaciones resuelven problemas reales míos — el mantenimiento de mi coche, mis carreras, mis certificaciones — porque quería algo que fuese a usar de verdad y no otra demo de lista de tareas. Pero **el código de aplicación es a propósito la parte menos cuidada**. Lo importante nunca fue el código, sino todo lo que hay alrededor.

Corre entero en mi propia máquina, en un clúster de un solo nodo sobre OrbStack. Dejar la nube fuera fue una decisión: en parte para no pagar por un proyecto de aprendizaje, y en parte porque quería montarlo todo sobre hardware que es mío y puedo destripar. Esa restricción es justamente el punto — aquí no hay ningún servicio gestionado que aparezca haciendo clic. **Cada pieza de esta plataforma es una que instalé, configuré y depuré yo**: el stack de monitorización desde cero con Prometheus, Grafana, Loki, Tempo, Alloy y OpenTelemetry; los secretos con Vault y External Secrets; la entrega con Argo CD; y las herramientas de seguridad a lo largo de toda la pipeline.

Lo que me propuse aprender: cómo se monta una pipeline de CI/CD y dónde va de verdad un gate; qué detecta y qué se le escapa a SAST, DAST, SCA, la detección de secretos y los SBOM; cómo se firma y se verifica una cadena de suministro de punta a punta; cómo se conectan realmente los tres pilares de la observabilidad; y cómo se comporta GitOps cuando algo falla. Casi todo lo que sé ahora salió de que las cosas se rompieran — y por eso las decisiones de más abajo están escritas como están.

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

## Arquitectura

```mermaid
flowchart TB
    user["Navegador"] --> ing["ingress-nginx<br/>juan-in-one.local"]

    ing -->|"/"| web["web<br/>React + Vite + nginx"]
    ing -->|"/api/car-api"| car["car-api<br/>FastAPI"]
    ing -->|"/api/sport-api"| sport["sport-api<br/>FastAPI"]
    ing -->|"/api/academy-api"| acad["academy-api<br/>FastAPI"]

    car --> pg[("PostgreSQL")]
    sport --> pg
    acad --> pg

    vault["Vault"] --> eso["External Secrets<br/>Operator"]
    eso -.->|"DATABASE_URL"| car
    eso -.->|"DATABASE_URL"| sport
    eso -.->|"DATABASE_URL"| acad

    car -.->|"métricas + trazas"| obs["Prometheus · Loki<br/>Tempo · Grafana"]
    sport -.-> obs
    acad -.-> obs
```

Cada servicio tiene su propio chart de Helm. Cada chart es una `Application` de Argo CD. Cada `Application` está declarada en el repo [gitops](https://github.com/juan-in-one/gitops), gestionada por una única aplicación raíz — el patrón **App of Apps**, así que añadir un servicio es un `git push`, nunca un `kubectl apply`.

Un único host de ingress reparte por prefijo de ruta. El front siempre llama a URLs relativas (`/api/car-api/...`), lo que significa **cero CORS en ningún sitio**: en local, Vite proxya esas rutas hacia el clúster; en producción las enruta ingress-nginx. El código de React no distingue una situación de la otra, y no le hace falta.

<p align="center">
  <img src="img/argocd-app-of-apps.png" alt="Árbol App of Apps de Argo CD" width="420">
</p>

<p align="center"><i>Una única <code>Application</code> raíz gestiona todas las demás. Añadir un servicio es añadir un fichero a <code>apps/</code>.</i></p>

![Aplicaciones en Argo CD](img/argocd-apps.png)

<p align="center"><i>La plataforma completa bajo Argo CD: las tres APIs, el front y la capa de plataforma que los sostiene.</i></p>

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

## Acceso

Todo se sirve desde un único host de ingress-nginx, resuelto en local a través de `/etc/hosts`:

```
<IP_DEL_INGRESS>  juan-in-one.local
<IP_DEL_INGRESS>  argocd.juan-in-one.local
```

`<IP_DEL_INGRESS>` es la dirección que OrbStack asigna al LoadBalancer de ingress-nginx — `kubectl get svc -n ingress-nginx` la devuelve.

![IP del LoadBalancer de ingress y entradas del hosts](img/ingress-hosts.png)

<p align="center"><i>La misma dirección en los dos sitios: la <code>EXTERNAL-IP</code> del Service de ingress-nginx, y las dos entradas de <code>/etc/hosts</code> que apuntan los dominios locales hacia ella.</i></p>

| URL | Qué sirve |
|---|---|
| `http://juan-in-one.local/` | El front |
| `http://juan-in-one.local/api/car-api/` | car-api |
| `http://juan-in-one.local/api/sport-api/` | sport-api |
| `http://juan-in-one.local/api/academy-api/` | academy-api |
| `http://juan-in-one.local/grafana/` | Grafana |
| `http://argocd.juan-in-one.local/` | Argo CD |

Las aplicaciones comparten un único host y se reparten por prefijo de ruta, que es lo que mantiene al front en URLs relativas y fuera del CORS por completo. Argo CD tiene su propio subdominio porque es herramienta de plataforma, no parte del producto.

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

## Repositorios

| Repositorio | Qué es |
|---|---|
| [gitops](https://github.com/juan-in-one/gitops) | Las `Applications` de Argo CD y toda la capa de plataforma. La fuente de verdad del clúster. |
| [.github](https://github.com/juan-in-one/.github) | Los dos workflows reutilizables — los checks de PR y la pipeline de entrega — que comparten las tres APIs. |
| [car-api](https://github.com/juan-in-one/car-api) | Mantenimiento del coche — revisiones, ITV, kilómetros. |
| [sport-api](https://github.com/juan-in-one/sport-api) | Carreras y retos de montaña, conseguidos y pendientes. |
| [academy-api](https://github.com/juan-in-one/academy-api) | Certificaciones y objetivos diarios de estudio con check-ins. |
| [web](https://github.com/juan-in-one/web) | Front en React 19 + TypeScript para todo lo anterior. |

<table>
<tr>
<td width="33%"><img src="img/web-coche.png" alt="Pantalla de mantenimiento del coche"></td>
<td width="33%"><img src="img/web-retos.png" alt="Pantalla de retos"></td>
<td width="33%"><img src="img/web-academia.png" alt="Pantalla de academia"></td>
</tr>
</table>

<p align="center"><i>Las tres pantallas. El front es pequeño y sin alardes a propósito — existe para que la plataforma de debajo tenga algo real que entregar.</i></p>

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

## Cadena de suministro

Dos workflows, y nunca corren a la vez. Abrir un PR lo valida; solo un merge a `main` puede publicar algo.

```mermaid
flowchart LR
    pr["Pull request<br/>desde una rama feat/*"] --> checks

    subgraph checks["pr-checks — ningún paso de este fichero puede publicar nada"]
        direction TB
        a["branch-name<br/>obliga a feat/*"] --> b["lint"]
        b --> c["secrets-scan<br/>Gitleaks, historial completo"]
        c --> d["dependency-review<br/>solo dependencias nuevas"]
        d --> e["build-sanity<br/>build + Trivy — sin login a GHCR en todo el fichero"]
        e --> f["coverage<br/>pytest, comentado en el PR"]
    end

    checks -->|"todo en verde"| ready["Merge permitido"]
```

Fusionar es lo único que dispara la entrega de verdad:

```mermaid
flowchart LR
    push["Merge a main"] --> gate

    subgraph gate["ci — todo esto bloquea"]
        direction TB
        a["Ruff"] --> b["pytest<br/>Postgres real"]
        b --> c["Gitleaks<br/>historial completo"]
        c --> d["Trivy config"]
        d --> e["Semgrep SAST"]
        e --> f["build, sin publicar"]
        f --> g["Trivy imagen"]
    end

    gate --> pushimg["publicar en GHCR"]
    pushimg --> sign["Cosign<br/>firma keyless"]
    sign --> sbom["SBOM con Syft<br/>attestation firmada"]

    sbom --> argo["Argo CD sincroniza"]
    argo --> api["API server"]
    api --> kyv{"Kyverno<br/>¿la firmó mi CI?"}
    kyv -->|no| deny["Pod denegado"]
    kyv -->|sí| run["El Pod arranca"]
```

La firma es **keyless**: no hay ninguna clave privada guardada en ningún sitio. El job de CI demuestra su identidad ante Sigstore con un token OIDC de GitHub, Fulcio emite un certificado de diez minutos atado a esa identidad, y la firma queda registrada en el log público de transparencia Rekor. El clúster comprueba después que quien firmó fue exactamente ese workflow, desde `main`:

```yaml
subject: "https://github.com/juan-in-one/.github/.github/workflows/fastapi-service-ci.yml@refs/heads/main"
issuer:  "https://token.actions.githubusercontent.com"
```

Un atacante que robe un token del registro puede publicar una imagen. Lo que no puede es hacer que se ejecute.

<p align="center">
  <img src="img/ci-jobs.png" alt="Grafo de jobs del CI" width="700">
</p>

<p align="center"><i>El grafo de jobs de <code>ci</code>, al fusionar a <code>main</code>: lint, tests y detección de secretos corren en paralelo; <code>build-and-push</code> solo arranca si los tres pasan. El DAST corre a la vez, contra la app levantada con un Postgres real.</i></p>

![Pasos de build-and-push](img/ci-build-and-push.png)

<p align="center"><i>Toda la cadena de suministro en un job, en orden. Fíjate en los pasos 8 al 12: la imagen se <b>construye sin publicar</b>, se escanea, y solo entonces se publica — así que un escaneo fallido significa que la imagen no llega siquiera al registro. Después se lee el digest real ya publicado, se firma, y el SBOM se adjunta como attestation firmada.</i></p>

Dos políticas vigilan la admisión, y entre las dos responden a las dos mitades de la pregunta — *de dónde viene esta imagen* y *quién la construyó*:

![Kyverno denegando tres Pods](img/kyverno-deny.png)

<p align="center"><i>Tres rechazos, en directo. <b>El primero</b>: una imagen mía, de un registro permitido, pero nunca firmada — la tumba la política de firmas. <b>El segundo y el tercero</b>: <code>ubuntu</code> y <code>mongo</code> de Docker Hub — los tumba la política de registros, antes siquiera de plantearse las firmas. Nada de fuera de los registros aprobados entra, y nada mío se ejecuta si no lo firmó mi CI.</i></p>

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

## Stack tecnológico

<table>
<tr>
<td valign="top" width="50%">

### 🧱 Plataforma

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white)
![ingress--nginx](https://img.shields.io/badge/ingress--nginx-269539?logo=nginx&logoColor=white)
![HPA](https://img.shields.io/badge/HPA-autoscaling-326CE5)

### 🔁 GitOps

![Argo CD](https://img.shields.io/badge/Argo%20CD-EF7B4D?logo=argo&logoColor=white)
![App of Apps](https://img.shields.io/badge/App%20of%20Apps-patr%C3%B3n-EF7B4D)

### 🔑 Secretos

![Vault](https://img.shields.io/badge/HashiCorp%20Vault-000000?logo=vault&logoColor=FFEC6E)
![External Secrets Operator](https://img.shields.io/badge/External%20Secrets%20Operator-6B46C1)

### 📊 Observabilidad

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-F46800)
![Tempo](https://img.shields.io/badge/Tempo-F46800)
![Grafana Alloy](https://img.shields.io/badge/Grafana%20Alloy-F46800)
![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-000000?logo=opentelemetry&logoColor=white)

</td>
<td valign="top" width="50%">

### 🛡️ DevSecOps

**SAST** — análisis estático
<br>![Semgrep](https://img.shields.io/badge/Semgrep-1B2532)
![Ruff](https://img.shields.io/badge/Ruff-D7FF64)
![oxlint](https://img.shields.io/badge/oxlint-181717)

**SCA** — escaneo de dependencias e imágenes
<br>![Trivy](https://img.shields.io/badge/Trivy-1904DA)
![Dependency Review](https://img.shields.io/badge/Dependency%20Review-2F855A?logo=github&logoColor=white)

**Detección de secretos**
<br>![Gitleaks](https://img.shields.io/badge/Gitleaks-EE5A00)
![GitHub Secret Scanning](https://img.shields.io/badge/GitHub%20Secret%20Scanning-181717?logo=github&logoColor=white)

**DAST**
<br>![OWASP ZAP](https://img.shields.io/badge/OWASP%20ZAP-8C1A1A)

**Cadena de suministro**
<br>![Cosign](https://img.shields.io/badge/Cosign-keyless-2F855A)
![Syft](https://img.shields.io/badge/Syft-SBOM-2F855A)
![Kyverno](https://img.shields.io/badge/Kyverno-Deny-1D4ED8)

**CI/CD**
<br>![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)
![GHCR](https://img.shields.io/badge/GHCR-181717?logo=github&logoColor=white)

</td>
</tr>
</table>

### 💻 Aplicaciones

![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-D71F00)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![pytest](https://img.shields.io/badge/pytest-0A9EDC?logo=pytest&logoColor=white)
![React](https://img.shields.io/badge/React%2019-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-646CFF?logo=vite&logoColor=white)
![Vitest](https://img.shields.io/badge/Vitest-6E9F18?logo=vitest&logoColor=white)

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

### Observabilidad, montada desde cero

![Dashboard RED en Grafana](img/grafana-red.png)

<p align="center"><i>Métricas RED por servicio — peticiones por segundo, porcentaje de errores y latencia p95 — junto a la salud de los pods. La carga de esta captura es sintética: peticiones y errores provocados a propósito para ejercitar los paneles y confirmar que cada uno reacciona a lo que dice medir. El selector de servicio cambia el dashboard entero entre car-api, sport-api y academy-api.</i></p>

![Logs y trazas en Grafana](img/grafana-logs-traces.png)

<p align="center"><i>Logs en vivo desde Loki y trazas recientes desde Tempo, en el mismo dashboard. Junto a las métricas de infraestructura hay un contador de negocio propio, porque la pregunta interesante suele ser cuántos eventos de mantenimiento se han creado, no solo cuántas peticiones han entrado.</i></p>

![Traza distribuida en Tempo](img/tempo-trace.png)

<p align="center"><i>Una única petición a través de car-api, span a span: recepción HTTP, conexión, INSERT, SELECT, respuesta. OpenTelemetry instrumenta la app directamente, sin colector intermedio.</i></p>

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

## Decisiones de ingeniería

Lo interesante de este proyecto no es la lista de herramientas, sino por qué cada pieza está configurada como está.

<details>
<summary><b>El escaneo va antes del push, no después</b></summary>

El build usaba originalmente `push: true` y escaneaba a continuación. Eso parecía un gate de seguridad pero no lo era: para cuando Trivy hacía fallar el job, la imagen vulnerable ya estaba publicada en el registro y cualquiera podía descargarla. La pipeline se ponía roja; en realidad no se impedía nada.

Ahora el build corre con `push: false, load: true`, manteniendo la imagen dentro del runner mientras se escanea, y publica en un paso aparte solo si todo ha pasado. Un gate que avisa de la brecha cuando ya ha ocurrido es un informe, no un gate.
</details>

<details>
<summary><b>Bloquear solo por vulnerabilidades que se pueden arreglar</b></summary>

Trivy hace fallar el build ante CRITICAL y HIGH, pero con `ignore-unfixed`. Bloquear por una CVE sin parche disponible no hace nada más seguro — deja la pipeline en rojo permanente hasta que alguien desactiva el escáner, que es peor que no tenerlo.

La vez que esto importó de verdad: tres CVEs CRITICAL sin arreglo en `perl-base`, que venía en la imagen base slim de Debian y que ninguna de estas apps usa para nada. En vez de silenciarlas o esperar a Debian, el paquete se elimina de la imagen. El código que no está en la imagen no se puede explotar.
</details>

<details>
<summary><b>La identidad que firma es el workflow reutilizable, no quien lo llama</b></summary>

Las tres APIs comparten un workflow reutilizable. Fulcio saca la identidad del certificado del claim OIDC `job_workflow_ref` — el workflow que contiene físicamente el job que se está ejecutando — así que las tres firman como `.github/.github/workflows/fastapi-service-ci.yml`, no como su propio `ci.yml`.

El front tiene su propio workflow, así que firma con una identidad distinta. Por eso la política de Kyverno declara dos attestors, y cualquier servicio futuro que adopte el workflow compartido se acepta sin tocar la política.

`id-token: write` hay que concederlo dos veces, en el workflow que llama y en el reutilizable: quien llama pone el techo, y el bloque `permissions` explícito del llamado filtra sobre él.
</details>

<details>
<summary><b>El motor legacy de Kyverno no veía firmas válidas</b></summary>

La primera implementación usaba una `ClusterPolicy` (`kyverno.io/v1`), que insistía en reportar `no signatures found` para imágenes cuyas firmas se verificaban perfectamente a mano con `cosign verify` y `crane ls`, usando las mismas credenciales.

El diagnóstico salió de los logs del admission controller, más verificar el token del registro por separado para descartarlo. La solución fue migrar a `ImageValidatingPolicy` (`policies.kyverno.io`), el motor basado en CEL que el propio Kyverno recomienda en sus avisos de deprecación. Funcionó a la primera.

Llegar hasta ahí exigió además tres intentos de autenticación contra el registro: RBAC sobre los `imagePullSecrets` existentes, luego referencias explícitas al secreto entre namespaces — que fallan por un bug conocido de Kyverno — y por último un secreto en el propio namespace `kyverno`.
</details>

<details>
<summary><b>La mutación del digest está desactivada a propósito</b></summary>

El motor CEL activa `mutateDigest` por defecto, reescribiendo la imagen del Deployment a su digest resuelto en el momento de la admisión. Es una funcionalidad genuinamente útil — y dejó a car-api en `OutOfSync` permanente sin un solo commit en el repo de GitOps, porque Git decía `sha-61c886a` y el clúster decía `@sha256:...`.

Está desactivada, con el motivo anotado al lado. Dos buenas prácticas en conflicto directo, resuelto a favor de la coherencia de GitOps; el problema de los tags mutables se ataja desplegando tags inmutables por commit.
</details>

<details>
<summary><b>El PR y el merge son dos ficheros separados, no un workflow con un <code>if</code></b></summary>

La primera versión tenía un único workflow reutilizable escuchando a `push` y a `pull_request`, con los pasos de publicar y firmar condicionados con `if: github.event_name == 'push'`. Funcionaba, pero en cada PR aparecía un job llamado `ci` que en realidad nunca podía publicar nada — confuso, y duplicaba lo que ya cubría un job de checks dedicado.

Ahora `ci.yml` se dispara solo al hacer push a `main`, y `pr-checks.yml` solo en `pull_request`. Verificado con un PR real (solo corre `pr-checks`, `ci` aparece como `skipping`) y con un merge real justo después (lo contrario, `pr-checks` sale como `skipped`). `pr-checks` absorbió el lint, Gitleaks y un job de build-and-scan de comprobación, sin ningún paso de login a GHCR en todo el fichero — no desactivado por una condición, sino estructuralmente incapaz de publicar.

Montar el job de cobertura en el PR destapó de paso tres bugs reales sin relación entre sí, en este orden: `python-coverage-comment-action` necesita `contents: write`, no solo `pull-requests: write`, para guardar su histórico en una rama huérfana — y el error que lanza en ese caso es un mensaje genérico de "permissions" que despista; `coverage.py` necesita `relative_files = true` en `pyproject.toml` o la action no consigue leer su propio informe; y la Dependency Review Action de GitHub exige activar `vulnerability-alerts` explícitamente en cada repositorio — no viene incluido al hacer público un repo, y no es el mismo ajuste que Secret Scanning ni que las actualizaciones de Dependabot.
</details>

<details>
<summary><b>Server-side apply, e ignorar solo las diferencias reales</b></summary>

Los CRDs de Kyverno llevan dentro todo su esquema de validación, lo que desborda la anotación de 256 KB que el `kubectl apply` de cliente usa para guardar la configuración anterior. `ServerSideApply=true` mueve esa contabilidad al API server.

Las dos diferencias que quedaban eran artefactos de serialización, no desviaciones reales: `spec.conversion`, que el API server rellena por defecto con `{strategy: None}`, y un mapa vacío `labels: {}` que Kubernetes omite por completo. Ambas se verificaron campo a campo con `argocd app diff` antes de añadirlas a `ignoreDifferences` — nunca a ciegas.
</details>

<details>
<summary><b>Qué es en realidad el Kubernetes de OrbStack</b></summary>

Con curiosidad suficiente como para comprobarlo en vez de suponerlo: `kubectl get nodes` reporta `v1.35.6+orb1` — un build propio de Kubernetes upstream, ni k3s (`+k3s1`) ni kind (que corre los "nodos" como contenedores compartiendo el kernel del host). El kernel del nodo es `7.0.14-orbstack-...`, un Linux ligero propio de OrbStack corriendo en una VM sobre el Virtualization framework de Apple, no sobre un hipervisor más pesado.

Dos piezas están tomadas directamente del ecosistema k3s en vez de reinventarlas: `rancher/klipper-lb` sostiene cada Service `LoadBalancer`, y `rancher/local-path-provisioner` sostiene el `StorageClass` por defecto. No hay ningún Pod `kube-proxy` en `kube-system` — quien sea que gestione el enrutado de Services está integrado en el propio runtime, no como Pod aparte.
</details>

<p align="right">(<a href="#contenidos">volver arriba</a>)</p>

---

*Construido y mantenido por [Juan Álvarez Gayoso](https://github.com/juan-cloudops) — Cloud & DevOps Engineer.*
