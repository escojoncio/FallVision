# Bitácora FallVision

## Estado actual

- Repo recién creado. Solo documentación: `README.md`, `PLAN.md`, este fichero.
- Sin código, sin workflow de CI, sin build.

## Decisiones tomadas

- Pila: Wine ARM64EC nativo en un solo proceso + FEX como capa WoW64 (JIT vía StikDebug, igual que AstroVisionPro) + DXVK sobre VPVulkan (VPKEngine) + runtime OpenXR propio sobre Compositor Services.
- Puertas estrictas en orden (ver `PLAN.md`); nada empieza hasta pasar la puerta 1 (Wine single-process con `hello.exe`).
- Launcher, descarga de Steam y entrada de mandos Sense se reutilizan de AVP.
- Pendiente de decisión del propietario antes de la puerta 3: cómo se trata el requisito de la EA app.

## Pendiente para la siguiente sesión

1. Esperar a que AVP / AstroVisionPro / VPKEngine no estén en medio de builds.
2. Puerta 1: scaffolding del IPA (SwiftUI launcher + log), scripts de build de Wine ARM64EC contra el SDK de visionOS, `wineserver` en proceso.
3. Crear `.github/workflows/build.yml` (solo `workflow_dispatch` o `[build]` en el commit; caché de toolchains).
