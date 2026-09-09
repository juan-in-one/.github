# .github

Repo especial de la organización [juan-in-one](https://github.com/juan-in-one) — GitHub le da un
significado propio a este nombre exacto:

- **`profile/README.md`** es la página de perfil pública de la organización
  ([github.com/juan-in-one](https://github.com/juan-in-one)) — ahí está la arquitectura completa de la
  plataforma.
- **`.github/workflows/`** contiene los workflows reutilizables (`workflow_call`) que llaman `car-api`,
  `sport-api`, `academy-api` y `web`:
  - `fastapi-service-ci.yml` — la CD real: build, escaneos bloqueantes, firma con Cosign, SBOM,
    publicación en GHCR. Dispara solo al fusionar a `main`.
  - `pr-checks.yml` — todo lo que hay que validar antes de fusionar (lint, tests, Gitleaks, Dependency
    Review, un build de prueba). No tiene ni login a GHCR: no puede publicar nada.

Centralizar esto aquí significa que las cuatro apps no duplican su pipeline — cada `ci.yml`/`pr-checks.yml`
de cada repo es solo unas pocas líneas que llaman a estos dos workflows.
