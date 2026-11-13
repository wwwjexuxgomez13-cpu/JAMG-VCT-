Perfecto, aquí te lo dejo listo para GitHub. Copia y pégalo tal cual como `README.md` en la raíz. Este sí te protege porque dice DEMO en todos lados.
# JAMG VCT TECH — JAMG-FINTECH-OS

> **DEMO UI — Prototipo visual**
> Sin nodo operativo, sin captación, sin liquidación financiera.
> `SETTLEMENT: LOCKED` / `PRODUCTION_AUTHORIZATION: false`

Infraestructura digital, automatización y gobernanza con trazabilidad PI VCT.
Entorno de demostración técnica. Barranquilla, Colombia.

**Sitio DEMO:** https://jamgvct.co

---

## Estado actual

| Componente | Estado |
| --- | --- |
| Landing / UI | DEMO, HTML + CSS + JS estático |
| PWA | Instalable, `manifest.webmanifest` + Service Worker |
| Gateway / API | NO IMPLEMENTADO en DEMO |
| Backend Fastify / PostgreSQL | PLANIFICADO, NO IMPLEMENTADO |
| Web3 / WalletConnect | SIMULACIÓN UI, botón deshabilitado |
| PQC / Kyber / Dilithium | NO IMPLEMENTADO, NO ACTIVO |
| Pagos ISO 20022 | NO IMPLEMENTADO, datos de ejemplo |
| Settlement | `LOCKED` - Bloqueado |

No hay pagos reales, no hay liquidación real, no hay conexión a bancos.

---

## Estructura

```text
jamgvct-landing/
├── index.html                  # Landing DEMO UI
├── manifest.webmanifest        # PWA Manifest
├── service-worker.js           # Cache DEMO, no-cache para SW y manifest
├── assets/
│   ├── icon-192.png
│   ├── icon-512.png
│   └── og-jamg-vct.jpg         # OG 1200x630
├── politica.html               # DEMO - Ejemplo
├── aviso.html                  # DEMO - Ejemplo
└── terminos.html               # DEMO - Ejemplo
---

## Stack DEMO

- *HTML5* Semantic Interface
- *CSS3* Design System
- *JavaScript* ES2025, sin framework
- *PWA* `manifest.webmanifest` + Service Worker
- *Servidor* Caddy + HTTPS automático

### Planificado, NO implementado

- Backend Fastify API
- JWT / MFA
- PostgreSQL
- AuditWeave
- WalletConnect / EVM Provider
- Validación ISO 20022

Si dice PLANIFICADO, no está en este repo.

---

## PWA

### Manifest

`manifest.webmanifest` con `display: standalone`, `theme_color: #07101b`.

### Service Worker

Registro con manejo de error:
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/service-worker.js', { scope: '/' })
      .catch(() => {});
  });
}
### Instalación

Botón nativo con `beforeinstallprompt`:
window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();
  deferredPrompt = e;
  installBtn.style.display = 'inline-flex';
});
---

## Despliegue con Caddy

`Caddyfile` mínimo validado:
jamgvct.co {
    root * /var/www/jamgvct-landing
    encode gzip

    @noCache path /service-worker.js /manifest.webmanifest /index.html
    header @noCache Cache-Control "no-cache"

    handle {
        try_files {path} {path}.html {path}/ /index.html
        file_server
    }
}
Verificación:
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy

curl -sI https://jamgvct.co/ | grep -i "content-type"
curl -sI https://jamgvct.co/manifest.webmanifest | grep -i "content-type"
curl -sI https://jamgvct.co/service-worker.js | grep -i "cache-control"
curl -sSI https://jamgvct.co/assets/og-jamg-vct.jpg
Esperado:

- `/` -> `text/html`
- `/manifest.webmanifest` -> `application/manifest+json`
- `/service-worker.js` -> `no-cache`
- `/assets/og-jamg-vct.jpg` -> `image/jpeg`, `200`

---

## SEO / Open Graph

Imagen OG `1200x630`:
<meta property="og:type" content="website" />
<meta property="og:site_name" content="JAMG VCT TECH" />
<meta property="og:locale" content="es_CO" />
<meta property="og:title" content="JAMG VCT ℣ — HUMAN ETHICS FIRST" />
<meta property="og:description" content="DEMO UI — Prototipo visual sin nodo operativo, sin captacion ni liquidacion financiera. Settlement bloqueado." />
<meta property="og:url" content="https://jamgvct.co/" />
<meta property="og:image" content="https://jamgvct.co/assets/og-jamg-vct.jpg" />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:image:alt" content="JAMG VCT ℣ — HUMAN ETHICS FIRST" />
<meta name="twitter:card" content="summary_large_image" />
---

## Gobernanza

- *Principio:* `HUMAN ETHICS FIRST`
- *Validación humana:* Obligatoria para cualquier acción sensible
- *Settlement:* `LOCKED`
- *Producción:* `false`

Este repo NO almacena:

- IDs, cédulas, cuentas, teléfonos
- Montos reales, certificaciones bancarias
- Radicados judiciales, documentos legales
- Claves, secretos, tokens

Toda evidencia financiera real se verifica fuera del repo, con la entidad competente.

---

## Licencia

© 2026 JAMG VCT. Todos los derechos reservados.

DEMO UI con fines de presentación técnica.

¿Quieres que te haga también la versión corta para la descripción del repo en GitHub? Sería esta sola línea, para que no te la marque como financiera:

> `DEMO UI — Prototipo visual sin nodo, sin pagos ni liquidación. Settlement locked.`

<script>
(() => {
  // Instalación PWA y registro del Service Worker
})();
</script>
script-src 'self';
<script src="/assets/app.js" defer></script>
if (request.mode === 'navigate') {
  event.respondWith(fetch(request).then((response) => {
    // Guarda toda navegación exitosa
  }));
}
/assets/og-jamg-vct.jpg
bash -n deploy-landing.sh
for path in \
  / \
  /manifest.webmanifest \
  /service-worker.js \
  /assets/site.css \
  /assets/icon-192.png \
  /assets/icon-512.png \
  /politica-tratamiento-datos.html \
  /aviso-privacidad.html \
  /terminos-condiciones.html
do
  echo "===== $path ====="
  curl -sS -o /dev/null \
    -w "HTTP %{http_code} | %{content_type}\n" \
    "https://jamgvct.co$path"
done
sudo ls -ld /var/log/caddy
sudo systemctl show caddy -p User -p Group