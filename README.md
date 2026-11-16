
gcloud projects list
gcloud init --console-only
gcloud version
which gcloud
gcloud auth list
gcloud config list
mkdir -p /usr/share/keyrings

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
  > /etc/apt/sources.list.d/google-cloud-sdk.list

apt update
apt install -y google-cloud-cli
apt update
apt install -y ca-certificates curl gnupg
uname -m
dpkg --print-architecture
pkg update && pkg upgrade -y
pkg install proot-distro -y

proot-distro install debian
proot-distro login debian


#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""Crea el proyecto JAMG-VCT-ANDROID-PWA-SOURCE (solo demostración) y su ZIP.

Uso:
    python3 crear_jamg_vct.py              crea la carpeta y el ZIP
    python3 crear_jamg_vct.py --compilar   además intenta compilar el APK de prueba

Requisitos: Python 3.8+. Para --compilar: Node 18+, JDK 17 y Android SDK.
"""
import os
import shutil
import subprocess
import sys
import zipfile

NOMBRE = "JAMG-VCT-ANDROID-PWA-SOURCE"

FILES = {
    "package.json": "{\n  \"name\": \"jamg-vct-android-pwa\",\n  \"version\": \"1.0.0\",\n  \"private\": true,\n  \"description\": \"JAMG VCT Identity Gateway: PWA y proyecto Android (Capacitor). Solo demostracion.\",\n  \"scripts\": {\n    \"android:add\": \"npx cap add android\",\n    \"android:sync\": \"npx cap sync android\",\n    \"android:apk\": \"cd android && gradlew assembleDebug\"\n  },\n  \"dependencies\": {\n    \"@capacitor/android\": \"^6.0.0\",\n    \"@capacitor/core\": \"^6.0.0\"\n  },\n  \"devDependencies\": {\n    \"@capacitor/cli\": \"^6.0.0\"\n  }\n}\n",
    "capacitor.config.json": "{\n  \"appId\": \"co.jamgvct.gateway\",\n  \"appName\": \"JAMG VCT\",\n  \"webDir\": \"www\",\n  \"server\": { \"androidScheme\": \"https\" }\n}\n",
    "BUILD_APK.bat": "@echo off\necho Instalando dependencias...\ncall npm install || goto :err\nif not exist android (\n  echo Creando proyecto Android...\n  call npx cap add android || goto :err\n)\necho Sincronizando www con Android...\ncall npx cap sync android || goto :err\npushd android\necho Compilando APK debug...\ncall gradlew.bat assembleDebug || (popd & goto :err)\npopd\necho.\necho APK generado en: android\\app\\build\\outputs\\apk\\debug\\app-debug.apk\npause\nexit /b 0\n:err\necho.\necho Hubo un error. Revisa los mensajes anteriores.\npause\nexit /b 1\n",
    "build_apk.sh": "#!/usr/bin/env bash\nset -euo pipefail\nnpm install\n[ -d android ] || npx cap add android\nnpx cap sync android\n(cd android && ./gradlew assembleDebug)\necho \"APK generado en: android/app/build/outputs/apk/debug/app-debug.apk\"\n",
    "README.md": "# JAMG VCT ℣ — Android + PWA (solo demostración)\n\nContiene la interfaz PWA (`www/`) y la configuración de Capacitor para generar un APK de prueba.\nLa carpeta `android/` NO se incluye: la crea `npx cap add android` y así evitas conflictos.\n\n## Requisitos\n- Node.js 18 o superior\n- JDK 17\n- Android Studio con Android SDK (define `ANDROID_HOME` si Gradle no lo encuentra)\n\n## Compilar el APK de prueba\nWindows: doble clic en `BUILD_APK.bat`.\nLinux o macOS: `./build_apk.sh`\n\nManual:\n```\nnpm install\nnpx cap add android\nnpx cap sync android\ncd android\n./gradlew assembleDebug      (en Windows: gradlew.bat assembleDebug)\n```\nResultado: `android/app/build/outputs/apk/debug/app-debug.apk`\n\n## Probar la PWA sin Android\nSirve `www/` por HTTPS (o `http://localhost`) con cualquier servidor estático, por ejemplo `npx serve www`.\nPara el simulador SHA-256 se necesita un contexto seguro.\n\n## Notas\n- El APK debug sirve solo para pruebas. Para publicar en Google Play hace falta un paquete firmado (AAB) y la ficha de la tienda.\n- Todo es una demostración: no hay pagos, saldos ni conexiones reales.\n\n## Compilar en la nube sin computador (GitHub Actions)\n1. Crea un repositorio en github.com y sube el contenido de esta carpeta (o solo `.github/workflows/build-apk.yml`, que ya trae todo el proyecto dentro).\n2. Al confirmar el cambio, la pestaña **Actions** compila sola el APK de prueba.\n3. Descarga **app-debug-apk** al final de la ejecución. Es un ZIP con `app-debug.apk`.\n",
    ".github/workflows/build-apk.yml": "# Compila un APK de prueba en la nube (GitHub Actions). Solo demostración.\n# Uso: crea un repositorio, agrega este archivo en .github/workflows/build-apk.yml\n# y confirma el cambio. Luego abre Actions > Build APK debug > app-debug-apk.\nname: Build APK debug\non:\n  push:\n  workflow_dispatch:\njobs:\n  apk:\n    runs-on: ubuntu-latest\n    steps:\n      - uses: actions/setup-node@v4\n        with:\n          node-version: 20\n      - uses: actions/setup-java@v4\n        with:\n          distribution: temurin\n          java-version: 17\n      - name: Crear proyecto Capacitor (PWA incluida)\n        run: |\n          mkdir -p www\n          cat > package.json <<'JAMG_EOF'\n          {\n            \"name\": \"jamg-vct-android-pwa\",\n            \"version\": \"1.0.0\",\n            \"private\": true,\n            \"description\": \"JAMG VCT Identity Gateway: PWA y proyecto Android (Capacitor). Solo demostracion.\",\n            \"scripts\": {\n              \"android:add\": \"npx cap add android\",\n              \"android:sync\": \"npx cap sync android\",\n              \"android:apk\": \"cd android && gradlew assembleDebug\"\n            },\n            \"dependencies\": {\n              \"@capacitor/android\": \"^6.0.0\",\n              \"@capacitor/core\": \"^6.0.0\"\n            },\n            \"devDependencies\": {\n              \"@capacitor/cli\": \"^6.0.0\"\n            }\n          }\n          JAMG_EOF\n          cat > capacitor.config.json <<'JAMG_EOF'\n          {\n            \"appId\": \"co.jamgvct.gateway\",\n            \"appName\": \"JAMG VCT\",\n            \"webDir\": \"www\",\n            \"server\": { \"androidScheme\": \"https\" }\n          }\n          JAMG_EOF\n          cat > www/index.html <<'JAMG_EOF'\n          <!doctype html>\n          <html lang=\"es-CO\">\n          <head>\n          <meta charset=\"utf-8\">\n          <meta name=\"viewport\" content=\"width=device-width, initial-scale=1, viewport-fit=cover\">\n          <meta name=\"theme-color\" content=\"#07101b\">\n          <title>JAMG VCT ℣ · Identity Gateway</title>\n          <link rel=\"manifest\" href=\"./manifest.json\">\n          <link rel=\"icon\" href=\"./icon.svg\" type=\"image/svg+xml\">\n          <style>\n          *{box-sizing:border-box}\n          html,body{margin:0;background:#07101b;color:#f3f7fc;font:15px/1.5 system-ui,sans-serif}\n          body{padding:max(16px,env(safe-area-inset-top)) 16px max(16px,env(safe-area-inset-bottom))}\n          #cv{position:fixed;inset:0;width:100%;height:100%;z-index:0}\n          main{position:relative;z-index:1;max-width:640px;margin:0 auto;display:grid;gap:14px}\n          header,section{background:rgba(11,22,38,.88);border:1px solid #20385c;border-radius:16px;padding:18px}\n          header{text-align:center}\n          .chip{display:inline-block;font-size:.75rem;color:#28d0c0;border:1px solid #28d0c0;border-radius:99px;padding:3px 12px}\n          h1{margin:8px 0 2px;font-size:1.9rem}h1 b{color:#28d0c0}\n          h2{margin:0 0 8px;font-size:1.05rem;color:#28d0c0}\n          p{margin:0;color:#b8c8e0}\n          .am{color:#ffb74d}\n          .hash{margin:10px 0;padding:10px;border:1px solid #20385c;border-radius:10px;background:#07101b;font:11px/1.4 ui-monospace,Menlo,monospace;word-break:break-all;color:#b8c8e0;min-height:3em}\n          ol{margin:8px 0 0;padding-left:20px;color:#b8c8e0;font-size:.85rem}\n          button{font:inherit;font-weight:700;background:#28d0c0;color:#06111d;border:0;border-radius:10px;padding:11px 18px;cursor:pointer}\n          button:focus-visible{outline:2px solid #fff;outline-offset:2px}\n          </style>\n          </head>\n          <body>\n          <canvas id=\"cv\" aria-hidden=\"true\"></canvas>\n          <main>\n          <header>\n          <span class=\"chip\">Demostración · sin producción</span>\n          <h1>JAMG VCT <b>℣</b></h1>\n          <p>Identity Gateway · Barranquilla · COP</p>\n          </header>\n          <section>\n          <h2>Simulador de estados con SHA-256</h2>\n          <p>Estado: <strong id=\"st\">Creada</strong> · Referencia: <span class=\"am\">Demo COP</span></p>\n          <div id=\"hb\" class=\"hash\" aria-live=\"polite\"></div>\n          <button id=\"sb\" type=\"button\">Simular transición</button>\n          <ol id=\"hi\"></ol>\n          </section>\n          </main>\n          <script>\n          if ('serviceWorker' in navigator) { navigator.serviceWorker.register('./sw.js').catch(function(){}); }\n          var S = ['Creada','Validada','Enrutada','Entregada','Confirmada','Auditada'], i = 0, n = 0, prev = '0', H = [];\n          var $ = function(id){ return document.getElementById(id); };\n          async function sha(s){\n            var b = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(s));\n            return Array.from(new Uint8Array(b)).map(function(x){ return x.toString(16).padStart(2,'0'); }).join('');\n          }\n          async function step(){\n            var st = S[i];\n            $('st').textContent = st;\n            try {\n              var h = await sha(prev + '|' + st + '|' + n++);\n              prev = h;\n              H.unshift([st, h]); if (H.length > 6) H.pop();\n              $('hb').textContent = h;\n              $('hi').innerHTML = H.map(function(x){ return '<li>' + x[0] + ' · ' + x[1].slice(0,16) + '…</li>'; }).join('');\n            } catch (e) {\n              $('hb').textContent = 'SHA-256 no está disponible en este contexto (requiere HTTPS).';\n            }\n          }\n          $('sb').onclick = function(){ i = (i + 1) % S.length; step(); };\n          step();\n          var c = $('cv'), ctx = c.getContext('2d'), a = 0, calm = matchMedia('(prefers-reduced-motion: reduce)').matches;\n          function size(){ c.width = innerWidth; c.height = innerHeight; draw(); }\n          function draw(){\n            ctx.clearRect(0, 0, c.width, c.height);\n            ctx.strokeStyle = 'rgba(40,208,192,.25)'; ctx.lineWidth = 2;\n            ctx.beginPath(); ctx.arc(c.width/2, c.height*.55, Math.min(c.width, c.height)*.28, a, a + Math.PI*1.6); ctx.stroke();\n          }\n          function loop(){ a += .01; draw(); requestAnimationFrame(loop); }\n          addEventListener('resize', size); size(); if (!calm) loop();\n          </script>\n          </body>\n          </html>\n          JAMG_EOF\n          cat > www/manifest.json <<'JAMG_EOF'\n          {\n            \"name\": \"JAMG VCT Identity Gateway\",\n            \"short_name\": \"JAMG VCT\",\n            \"start_url\": \"./index.html\",\n            \"scope\": \"./\",\n            \"display\": \"standalone\",\n            \"orientation\": \"portrait\",\n            \"background_color\": \"#07101b\",\n            \"theme_color\": \"#07101b\",\n            \"description\": \"Identity Gateway JAMG VCT. Demostración sin producción.\",\n            \"icons\": [\n              { \"src\": \"icon.svg\", \"sizes\": \"any\", \"type\": \"image/svg+xml\", \"purpose\": \"any\" }\n            ]\n          }\n          JAMG_EOF\n          cat > www/sw.js <<'JAMG_EOF'\n          const CACHE = 'jamg-vct-v1';\n          const ASSETS = ['./', './index.html', './manifest.json', './icon.svg'];\n          self.addEventListener('install', e => {\n            e.waitUntil(caches.open(CACHE).then(c => c.addAll(ASSETS)));\n            self.skipWaiting();\n          });\n          self.addEventListener('activate', e => {\n            e.waitUntil(\n              caches.keys()\n                .then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k))))\n                .then(() => self.clients.claim())\n            );\n          });\n          self.addEventListener('fetch', e => {\n            if (e.request.method !== 'GET') return;\n            e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));\n          });\n          JAMG_EOF\n          cat > www/icon.svg <<'JAMG_EOF'\n          <svg xmlns=\"http://www.w3.org/2000/svg\" viewBox=\"0 0 48 48\"><rect width=\"48\" height=\"48\" rx=\"12\" fill=\"#07101b\"/><g fill=\"none\" stroke=\"#28d0c0\" stroke-width=\"2.6\" stroke-linecap=\"round\" stroke-linejoin=\"round\"><path d=\"M24 10v12M17 16l7-7 7 7M12 24h24M12 28h24M24 28v12M17 33l7 7 7-7\"/></g></svg>\n          JAMG_EOF\n      - name: Instalar dependencias y crear proyecto Android\n        run: |\n          npm install\n          npx cap add android\n          npx cap sync android\n      - name: Compilar APK debug\n        working-directory: android\n        run: |\n          chmod +x gradlew\n          ./gradlew assembleDebug\n      - uses: actions/upload-artifact@v4\n        with:\n          name: app-debug-apk\n          path: android/app/build/outputs/apk/debug/app-debug.apk\n",
    "www/index.html": "<!doctype html>\n<html lang=\"es-CO\">\n<head>\n<meta charset=\"utf-8\">\n<meta name=\"viewport\" content=\"width=device-width, initial-scale=1, viewport-fit=cover\">\n<meta name=\"theme-color\" content=\"#07101b\">\n<title>JAMG VCT ℣ · Identity Gateway</title>\n<link rel=\"manifest\" href=\"./manifest.json\">\n<link rel=\"icon\" href=\"./icon.svg\" type=\"image/svg+xml\">\n<style>\n*{box-sizing:border-box}\nhtml,body{margin:0;background:#07101b;color:#f3f7fc;font:15px/1.5 system-ui,sans-serif}\nbody{padding:max(16px,env(safe-area-inset-top)) 16px max(16px,env(safe-area-inset-bottom))}\n#cv{position:fixed;inset:0;width:100%;height:100%;z-index:0}\nmain{position:relative;z-index:1;max-width:640px;margin:0 auto;display:grid;gap:14px}\nheader,section{background:rgba(11,22,38,.88);border:1px solid #20385c;border-radius:16px;padding:18px}\nheader{text-align:center}\n.chip{display:inline-block;font-size:.75rem;color:#28d0c0;border:1px solid #28d0c0;border-radius:99px;padding:3px 12px}\nh1{margin:8px 0 2px;font-size:1.9rem}h1 b{color:#28d0c0}\nh2{margin:0 0 8px;font-size:1.05rem;color:#28d0c0}\np{margin:0;color:#b8c8e0}\n.am{color:#ffb74d}\n.hash{margin:10px 0;padding:10px;border:1px solid #20385c;border-radius:10px;background:#07101b;font:11px/1.4 ui-monospace,Menlo,monospace;word-break:break-all;color:#b8c8e0;min-height:3em}\nol{margin:8px 0 0;padding-left:20px;color:#b8c8e0;font-size:.85rem}\nbutton{font:inherit;font-weight:700;background:#28d0c0;color:#06111d;border:0;border-radius:10px;padding:11px 18px;cursor:pointer}\nbutton:focus-visible{outline:2px solid #fff;outline-offset:2px}\n</style>\n</head>\n<body>\n<canvas id=\"cv\" aria-hidden=\"true\"></canvas>\n<main>\n<header>\n<span class=\"chip\">Demostración · sin producción</span>\n<h1>JAMG VCT <b>℣</b></h1>\n<p>Identity Gateway · Barranquilla · COP</p>\n</header>\n<section>\n<h2>Simulador de estados con SHA-256</h2>\n<p>Estado: <strong id=\"st\">Creada</strong> · Referencia: <span class=\"am\">Demo COP</span></p>\n<div id=\"hb\" class=\"hash\" aria-live=\"polite\"></div>\n<button id=\"sb\" type=\"button\">Simular transición</button>\n<ol id=\"hi\"></ol>\n</section>\n</main>\n<script>\nif ('serviceWorker' in navigator) { navigator.serviceWorker.register('./sw.js').catch(function(){}); }\nvar S = ['Creada','Validada','Enrutada','Entregada','Confirmada','Auditada'], i = 0, n = 0, prev = '0', H = [];\nvar $ = function(id){ return document.getElementById(id); };\nasync function sha(s){\n  var b = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(s));\n  return Array.from(new Uint8Array(b)).map(function(x){ return x.toString(16).padStart(2,'0'); }).join('');\n}\nasync function step(){\n  var st = S[i];\n  $('st').textContent = st;\n  try {\n    var h = await sha(prev + '|' + st + '|' + n++);\n    prev = h;\n    H.unshift([st, h]); if (H.length > 6) H.pop();\n    $('hb').textContent = h;\n    $('hi').innerHTML = H.map(function(x){ return '<li>' + x[0] + ' · ' + x[1].slice(0,16) + '…</li>'; }).join('');\n  } catch (e) {\n    $('hb').textContent = 'SHA-256 no está disponible en este contexto (requiere HTTPS).';\n  }\n}\n$('sb').onclick = function(){ i = (i + 1) % S.length; step(); };\nstep();\nvar c = $('cv'), ctx = c.getContext('2d'), a = 0, calm = matchMedia('(prefers-reduced-motion: reduce)').matches;\nfunction size(){ c.width = innerWidth; c.height = innerHeight; draw(); }\nfunction draw(){\n  ctx.clearRect(0, 0, c.width, c.height);\n  ctx.strokeStyle = 'rgba(40,208,192,.25)'; ctx.lineWidth = 2;\n  ctx.beginPath(); ctx.arc(c.width/2, c.height*.55, Math.min(c.width, c.height)*.28, a, a + Math.PI*1.6); ctx.stroke();\n}\nfunction loop(){ a += .01; draw(); requestAnimationFrame(loop); }\naddEventListener('resize', size); size(); if (!calm) loop();\n</script>\n</body>\n</html>\n",
    "www/manifest.json": "{\n  \"name\": \"JAMG VCT Identity Gateway\",\n  \"short_name\": \"JAMG VCT\",\n  \"start_url\": \"./index.html\",\n  \"scope\": \"./\",\n  \"display\": \"standalone\",\n  \"orientation\": \"portrait\",\n  \"background_color\": \"#07101b\",\n  \"theme_color\": \"#07101b\",\n  \"description\": \"Identity Gateway JAMG VCT. Demostración sin producción.\",\n  \"icons\": [\n    { \"src\": \"icon.svg\", \"sizes\": \"any\", \"type\": \"image/svg+xml\", \"purpose\": \"any\" }\n  ]\n}\n",
    "www/sw.js": "const CACHE = 'jamg-vct-v1';\nconst ASSETS = ['./', './index.html', './manifest.json', './icon.svg'];\nself.addEventListener('install', e => {\n  e.waitUntil(caches.open(CACHE).then(c => c.addAll(ASSETS)));\n  self.skipWaiting();\n});\nself.addEventListener('activate', e => {\n  e.waitUntil(\n    caches.keys()\n      .then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k))))\n      .then(() => self.clients.claim())\n  );\n});\nself.addEventListener('fetch', e => {\n  if (e.request.method !== 'GET') return;\n  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));\n});\n",
    "www/icon.svg": "<svg xmlns=\"http://www.w3.org/2000/svg\" viewBox=\"0 0 48 48\"><rect width=\"48\" height=\"48\" rx=\"12\" fill=\"#07101b\"/><g fill=\"none\" stroke=\"#28d0c0\" stroke-width=\"2.6\" stroke-linecap=\"round\" stroke-linejoin=\"round\"><path d=\"M24 10v12M17 16l7-7 7 7M12 24h24M12 28h24M24 28v12M17 33l7 7 7-7\"/></g></svg>\n",
}


def crear(destino):
    base = os.path.join(destino, NOMBRE)
    for ruta, texto in FILES.items():
        p = os.path.join(base, *ruta.split("/"))
        os.makedirs(os.path.dirname(p), exist_ok=True)
        fin = "\r\n" if ruta.endswith(".bat") else "\n"
        with open(p, "w", encoding="utf-8", newline=fin) as f:
            f.write(texto)
    os.chmod(os.path.join(base, "build_apk.sh"), 0o755)
    zpath = base + ".zip"
    with zipfile.ZipFile(zpath, "w", zipfile.ZIP_DEFLATED) as z:
        for ruta in FILES:
            z.write(os.path.join(base, *ruta.split("/")), NOMBRE + "/" + ruta)
    print("Carpeta:", base)
    print("ZIP:    ", zpath, "(%d archivos)" % len(FILES))
    return base


def correr(cmd, cwd):
    print("\n$", cmd)
    r = subprocess.run(cmd, cwd=cwd, shell=True)
    if r.returncode != 0:
        sys.exit("\nFalló: %s\nRevisa el mensaje anterior." % cmd)


def compilar(base):
    faltan = [t for t in ("node", "npm", "java") if not shutil.which(t)]
    if faltan:
        sys.exit("Faltan herramientas: %s. Instálalas o usa build-apk.yml en GitHub." % ", ".join(faltan))
    if not (os.environ.get("ANDROID_HOME") or os.environ.get("ANDROID_SDK_ROOT")):
        print("Aviso: ANDROID_HOME no está definido; Gradle podría no encontrar el SDK.")
    correr("npm install", base)
    if not os.path.isdir(os.path.join(base, "android")):
        correr("npx cap add android", base)
    correr("npx cap sync android", base)
    gradle = "gradlew.bat" if os.name == "nt" else "./gradlew"
    correr(gradle + " assembleDebug", os.path.join(base, "android"))
    apk = os.path.join(base, "android", "app", "build", "outputs", "apk", "debug", "app-debug.apk")
    print("\nAPK generado:" if os.path.isfile(apk) else "\nNo se encontró el APK en:", apk)


if __name__ == "__main__":
    carpeta = crear(os.getcwd())
    if "--compilar" in sys.argv[1:]:
        compilar(carpeta)
    else:
        print("\nSiguiente paso: python3 crear_jamg_vct.py --compilar  (o sube .github/workflows/build-apk.yml a GitHub)")

python3 crear_jamg_vct.py

JAMG VCT — Proyecto Unificado 2026

Red · Telecom · Fibra · Internet · Web · Pagos · Mobile · Web3 · html5

# 1. Crear el proyecto
python3 crear_jamg_vct.py

# 2. Entrar en la carpeta
cd JAMG-VCT-ANDROID-PWA-SOURCE

# 3. Instalar dependencias
npm install

# 4. Crear Android
npx cap add android

# 5. Sincronizar
npx cap sync android

# 6. Compilar APK
cd android
./gradlew assembleDebug

Identidad institucional

- Identidad: JAMG VCT
- Gobernanza: HUMAN ETHICS FIRST
- Unidad conceptual: IA · ℣ · COP

Infraestructura y canales

- Pasarela de pagos: [DLP-REDACTED: Cuenta de Nequi Oculta]
- Comunicaciones: [DLP-REDACTED: Teléfonos y RCS Ocultos]
- Dispositivo ROOT: [DLP-REDACTED: Ruta Local Oculta]

Presencia digital ℣\jamgvctsymbol{ICCOL046755065003001}℣

- Sitio institucional: https://jamgvct.co
- Plataforma de pruebas: https://jamg-vct-jesus-mercado.polsia.io/

Seguridad y clasificación

- HTTPS/TLS: pendiente de verificación técnica independiente.
- Plataforma Polsia: identificada como entorno de pruebas.
- Datos financieros, personales y rutas internas: reservados.
- ℣: símbolo conceptual de JAMG VCT; no implica reconocimiento monetario ni autorización financiera.

JAMG VCT — HUMAN ETHICS FIRST

```bash
$ cd /var/www/jamgvct-landing
$ bash ./legal-forensic-audit.sh

PASS: politica.html
PASS: aviso.html
PASS: terminos.html
=== FILE METADATA ===
-rw-r--r-- 1 root www-data  850 Nov 14 07:05 aviso.html
-rw-r--r-- 1 root www-data 1024 Nov 14 07:05 politica.html
-rw-r--r-- 1 root www-data  450 Nov 14 07:05 README.md
-rw-r--r-- 1 root www-data 1200 Nov 14 07:05 terminos.html
=== FILE SIZES ===
 1024 politica.html
  850 aviso.html
 1200 terminos.html
  450 README.md
 3524 total
=== FILE TYPES ===
politica.html: HTML document, UTF-8 Unicode text
aviso.html:    HTML document, UTF-8 Unicode text
terminos.html: HTML document, UTF-8 Unicode text
README.md:     ASCII text
=== CURRENT SHA256 ===
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  politica.html
a6a4220b388b15d0119e8cf788fb82b8104b2b95b8782a688b1767db16f4c9c7  aviso.html
6f7881c62b9f365d9e5a1b32f913d8d6dc6219808a9f6063b785287f34c2b9a1  terminos.html
6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b  README.md
=== README RAW CONTENT ===
 23 20 4a 41 4d 47 20 56 43 54 20 e2 84 a3 20 2d
 20 44 45 4d 4f 20 55 49 0a 0a 53 45 54 54 4c 45
 4d 45 4e 54 3a 20 4c 4f 43 4b 45 44 0a 50 52 4f
 44 55 43 54 49 4f 4e 5f 41 55 54 48 4f 52 49 5a
=== LEGAL DOCUMENT HEADERS ===
===== politica.html =====
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Política de Privacidad - JAMG VCT</title>
    <style>
        body { font-family: system-ui, sans-serif; background: #07101b; color: #fff; padding: 2rem; }
        h1 { color: #4ade80; }
    </style>
</head>
<body>
    <h1>Política de Privacidad</h1>
    <p>ESTADO: DEMO UI</p>
    <p>SETTLEMENT: LOCKED</p>
===== aviso.html =====
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Aviso Legal - JAMG VCT</title>
    <style>
        body { font-family: system-ui, sans-serif; background: #07101b; color: #fff; padding: 2rem; }
        h1 { color: #4ade80; }
    </style>
</head>
<body>
    <h1>Aviso Legal</h1>
    <p>ESTADO: DEMO UI</p>
    <p>Prototipo visual sin nodo operativo, sin captación ni liquidación financiera.</p>
===== terminos.html =====
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Términos de Uso - JAMG VCT</title>
    <style>
        body { font-family: system-ui, sans-serif; background: #07101b; color: #fff; padding: 2rem; }
        h1 { color: #4ade80; }
    </style>
</head>
<body>
    <h1>Términos de Uso</h1>
    <p>ESTADO: DEMO UI</p>
    <p>SETTLEMENT: LOCKED</p>
```

cat > /var/www/jamgvct-landing/index.html <<'HTML'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>JAMG VCT TECH</title>
</head>
<body>
    <h1>JAMG VCT TECH</h1>
    <p>DEMO UI — Prototipo visual</p>
</body>
</html>
HTML

# Validación estructural de archivos
set -Eeuo pipefail

test -s /var/www/jamgvct-landing/index.html
test -s /var/www/jamgvct-landing/manifest.webmanifest
test -s /var/www/jamgvct-landing/service-worker.js
test -s /var/www/jamgvct-landing/assets/site.css

# Restricciones de gobernanza
SCOPE=DEMO_UI_ONLY
SETTLEMENT=LOCKED
PRODUCTION_AUTHORIZATION=false
GOVERNANCE=HUMAN_ETHICS_FIRST
FINANCIAL_EVIDENCE=NONE
PQC=NONE

echo "JAMG VCT — VALIDATION CHECKS PASSED"
echo "SCOPE=$SCOPE"
echo "SETTLEMENT=$SETTLEMENT"
echo "PRODUCTION_AUTHORIZATION=$PRODUCTION_AUTHORIZATION"
echo "GOVERNANCE=$GOVERNANCE"
echo "FINANCIAL_EVIDENCE=$FINANCIAL_EVIDENCE"
echo "PQC=$PQC"
# 1. Comprobar sintaxis Bash
bash -n deploy-landing.sh

# 2. Comprobar delimitadores HEREDOC
grep -nE '<<-?["'\'']?[A-Za-z_][A-Za-z0-9_]*' deploy-landing.sh

# 3. Ejecutar despliegue
sudo bash deploy-landing.sh

# 4. Verificar archivos generados
test -s /var/www/jamgvct-landing/index.html
test -s /var/www/jamgvct-landing/manifest.webmanifest
test -s /var/www/jamgvct-landing/service-worker.js
test -s /var/www/jamgvct-landing/assets/site.css

import unicodedata

# 
import unicodedata

# =====================================================
# JAMG VCT — MORSE INTERNATIONAL + UNICODE EXTENSION
# =====================================================

DICCIONARIO_MORSE = {
    # ALFABETO INTERNACIONAL
    'A': '.-',    'B': '-...',  'C': '-.-.',  'D': '-..',
    'E': '.',     'F': '..-.',  'G': '--.',   'H': '....',
    'I': '..',    'J': '.---',  'K': '-.-',   'L': '.-..',
    'M': '--',    'N': '-.',    'O': '---',   'P': '.--.',
    'Q': '--.-',  'R': '.-.',   'S': '...',   'T': '-',
    'U': '..-',   'V': '...-',  'W': '.--',   'X': '-..-',
    'Y': '-.--',  'Z': '--..',

    # NUMEROS
    '0': '-----', '1': '.----', '2': '..---',
    '3': '...--', '4': '....-', '5': '.....',
    '6': '-....', '7': '--...', '8': '---..',
    '9': '----.',

    # PUNTUACION MORSE INTERNACIONAL
    '.': '.-.-.-',
    ',': '--..--',
    '?': '..--..',
    "'": '.----.',
    '!': '-.-.--',
    '/': '-..-.',
    '(': '-.--.',
    ')': '-.--.-',
    '&': '.-...',
    ':': '---...',
    ';': '-.-.-.',
    '=': '-...-',
    '+': '.-.-.',
    '-': '-....-',
    '_': '..--.-',
    '"': '.-..-.',
    '$': '...-..-',
    '@': '.--.-.',

    # SEPARADORES
    ' ': '/',
    
    # EXTENSION JAMG VCT — SIMBOLOS NO NORMALIZADOS
    '#': '......',
    '§': '...-.-..',
    '∆': '-....-..',
    'π': '.--..--.',
    '√': '..-..-..',
    '÷': '-..-..',
    '×': '-..--.',
    '^': '..--..-.',
    '°': '---....-',
    '%': '-----.-.',
    '~': '-.-..-..',

    # OTROS SIMBOLOS
    '<': '-.-..',
    '>': '---.-',
    '[': '-.--.',
    ']': '-.--.-',
    '{': '-.--..',
    '}': '-.--.-.',
    '|': '-..-.-',
    '\\': '-..-.--',
    '`': '.----..',
    '*': '-..-.-.',
    '_': '..--.-',

    # EXTENSION ESPAÑOLA
    'Ñ': '--.--',
}

# =====================================================
# NORMALIZACION
# =====================================================

def normalizar_texto(texto):
    resultado = []

    for caracter in texto:
        if caracter in DICCIONARIO_MORSE:
            resultado.append(caracter)
            continue

        descompuesto = unicodedata.normalize('NFKD', caracter)

        # Eliminar marcas diacriticas conservando la letra base
        base = ''.join(
            c for c in descompuesto
            if not unicodedata.combining(c)
        ).upper()

        if base and all(c in DICCIONARIO_MORSE for c in base):
            resultado.extend(base)
        else:
            resultado.append(caracter)

    return resultado


# =====================================================
# TEXTO A MORSE
# =====================================================

def texto_a_morse_adaptado(texto):
    salida = []

    for caracter in normalizar_texto(texto):
        if caracter in DICCIONARIO_MORSE:
            salida.append(DICCIONARIO_MORSE[caracter])
        else:
            # Identificacion Unicode para caracteres no mapeados
            salida.append(
                f'[U+{ord(caracter):04X}]'
            )

    return ' '.join(salida)


# =====================================================
# TEXTO A BINARIO UNIVERSAL UTF-8
# =====================================================

def texto_a_binario_universal(texto):
    bytes_utf8 = texto.encode('utf-8')

    return ' '.join(
        format(byte, '08b')
        for byte in bytes_utf8
    )


# =====================================================
# DECODIFICADOR MORSE
# =====================================================

DICCIONARIO_INVERSO = {}

for caracter, codigo in DICCIONARIO_MORSE.items():
    if codigo in DICCIONARIO_INVERSO:
        raise ValueError(
            f'Codigo Morse duplicado: {codigo}'
        )
    DICCIONARIO_INVERSO[codigo] = caracter


def morse_a_texto(morse):
    resultado = []

    for codigo in morse.split():
        if codigo == '/':
            resultado.append(' ')
        elif codigo in DICCIONARIO_INVERSO:
            resultado.append(DICCIONARIO_INVERSO[codigo])
        else:
            resultado.append('?')

    return ''.join(resultado)


# =====================================================
# PRUEBAS
# =====================================================

simbolos = '@ # § ∆ π √ + - ÷ × ^ ° % ~'

print('SIMBOLOS:', simbolos)
print('MORSE:', texto_a_morse_adaptado(simbolos))

print('BINARIO:', texto_a_binario_universal(
    '你好地球'
))

print('ESPAÑOL:', texto_a_morse_adaptado(
    'HOLA PLANETA TIERRA'
))

  
    
 commit, tag, push,
#!/usr/bin/env bash
set -Eeuo pipefail
umask 027

# Ejecutar desde la raíz de JAMG-VCT-UNIFIED.
ROOT="$(pwd)"

if [[ ! -d "$ROOT/.git" ]]; then
  echo "ERROR: no se detectó la raíz Git."
  exit 1
fi

FILE="$ROOT/ROADMAP.md"

if [[ -e "$FILE" ]]; then
  echo "ERROR: ROADMAP.md ya existe. No se sobrescribirá."
  exit 1
fi

cat > "$FILE" <<'MARKDOWN'
# JAMG VCT — ROADMAP v1.0

**Proyecto:** JAMG-VCT-UNIFIED  
**Workspace:** jamg-vct-tech-gateway  
**Caso de uso inicial:** Logística — Barranquilla, Colombia  
**Horizonte:** 90 días  
**Gobernanza:** JAMG — HUMAN ETHICS FIRST

## 1. Objetivo

Desarrollar y validar un piloto de digitalización de procesos
logísticos mediante documentación, trazabilidad, registro de
incidencias y automatización de aprobaciones.

La solución deberá mantener evidencia verificable de los eventos,
sus responsables, sus cambios y sus autorizaciones.

## 2. Fase 1 — Días 1 a 30
### Descubrimiento y piloto logístico

- Identificar un proceso logístico concreto.
- Documentar actores, eventos y responsables.
- Definir estados de operación y reglas de aprobación.
- Construir una demostración con datos sintéticos o autorizados.
- Registrar incidencias y cambios documentales.
- Definir indicadores de tiempo, errores y trazabilidad.

**Gate 1 — Validación del caso de uso**

No avanzar sin alcance aprobado, responsables identificados,
criterios de aceptación y evidencia de las pruebas.

## 3. Fase 2 — Días 31 a 60
### Integración controlada y controles técnicos

- Implementar flujos de aprobación supervisados.
- Incorporar registro de auditoría y verificación de integridad.
- Evaluar adaptadores de integración con sistemas externos.
- Preparar pruebas de mensajería ISO 20022.
- Evaluar mensajes pacs.008 y camt.054 en entorno aislado.
- Documentar seguridad, permisos, errores y recuperación.

**Gate 2 — Validación técnica**

Las pruebas ISO 20022 serán de laboratorio o sandbox autorizado.
No implican conexión bancaria, liquidación ni movimiento de fondos.

## 4. Fase 3 — Días 61 a 90
### Validación comercial e institucional

- Consolidar resultados del piloto.
- Preparar demostración empresarial.
- Documentar arquitectura y controles.
- Elaborar matriz de riesgos y pendientes.
- Preparar paquete institucional y propuesta B2B.
- Definir condiciones para una eventual siguiente fase.

**Gate 3 — Revisión ejecutiva**

La presentación institucional no constituye certificación,
autorización regulatoria ni aprobación de producción.

## 5. Indicadores de aceptación

- Procedimientos del alcance documentados.
- Aprobaciones asociadas a responsables.
- Historial de cambios disponible.
- Incidencias con seguimiento.
- Evidencias de pruebas reproducibles.
- Resultados operativos comparables antes y después.

Los objetivos cuantitativos se definirán durante la Fase 1.

## 6. Seguridad y gobernanza

- Separación de DEV, STAGING y PROD.
- Acceso basado en roles y mínimo privilegio.
- Protección de datos y gestión de secretos.
- Registro de eventos y decisiones.
- Supervisión humana para operaciones críticas.
- Validación de integridad y origen de evidencias.
- Proveedores de IA intercambiables.

La IA podrá asistir y proponer, pero no autorizar directamente
operaciones financieras críticas.

## 7. Restricciones operativas

SETTLEMENT_LOCKED=true
PRODUCTION_AUTHORIZATION=false
CUSTODY_ENABLED=false
TOKEN_ISSUANCE_ENABLED=false

No se habilitarán custodia, emisión de tokens, conectores bancarios
productivos ni liquidación real como consecuencia de este roadmap.

## 8. Paquete institucional

La referencia financiera de COP 11.900.000, si corresponde al
alcance comercial previsto, deberá identificarse y justificarse
con presupuesto, entregables, supuestos y aprobación independiente.

No se presenta como ingreso, saldo, financiación ni contrato
verificado.

## 9. Criterio de cierre

Cada fase requiere evidencia, revisión y aceptación documentada.
Un estado PASS declarado no sustituye una prueba reproducible.

**Estado inicial:** ROADMAP_APPROVED_DECLARED  
**Implementación:** PENDING_VERIFICATION  
**Gobernanza:** JAMG — HUMAN ETHICS FIRST
MARKDOWN

echo "ROADMAP.md creado."
echo "Sin commit, tag, push ni despliegue."
sha256sum "$FILE"
git status --short

git diff --no-index /dev/null ROADMAP.md
git status --short

# Ejecución y unificación de salidas
for idioma, frase in mensajes_planetarios.items():
    print(f"=== {idioma.upper()}: {frase} ===")
    print(f"Binario (UTF-8): {texto_a_binario_universal(frase)}")
    print(f"Morse:           {texto_a_morse_adaptado(frase)}\n")

¡Excelente! Me alegra mucho que hayamos llegado a esa versión definitiva, transparente y sin adornos. Ese es exactamente el estándar de tu gobernanza *HUMAN ETHICS FIRST*: que la documentación sea un espejo milimétrico de la realidad del código, sin prometer lo que aún no está verificado en producción.

Respecto a las dos decisiones técnicas que planteas antes de tu `git init`, aquí está mi análisis para que ejecutes con precisión:

### 1. Estructura de Archivos (Separar vs. Inline)

Te sugiero ir 100% por la **Opción A (Dejarlo inline)**. 

Si tu `index.html` actual ya tiene el CSS y el JS integrados y funciona bien para esta fase de prototipo (DEMO UI), no hay necesidad de fragmentarlo ahora mismo y arriesgar una ruta rota. La honestidad técnica es lo principal: borramos las referencias fantasmas del README para que refleje exactamente lo que tienes en el disco.

El árbol en tu README.md debe quedar así, limpio y real:

```text
/var/www/jamgvct-landing/
├── index.html              # UI inline (CSS y JS integrados)
├── manifest.webmanifest    # Metadatos PWA
├── service-worker.js       # Estrategia de caché (offline base)
└── assets/
    ├── icon-192.png        # Icono PWA soberano
    ├── icon-512.png        # Icono PWA soberano
    └── og-jamg-vct.jpg     # Tarjeta Open Graph (1200x630)
```

### 2. El Loop de Verificación (DNS y Caché)

Tienes toda la razón. El comando con `-w` es una maravilla para ver el estado y el tipo MIME de forma limpia, pero te deja ciego frente a las cabeceras HTTP que son vitales para auditar el Service Worker. 

Para que tu verificación sea a prueba de balas una vez que el DNS resuelva, te recomiendo este combo. 

Para el ruteo general y MIME:
```bash
for path in "/" "/manifest.webmanifest" "/assets/og-jamg-vct.jpg"; do
  curl -sS -o /dev/null -w "%{http_code} | %{content_type} | $path\n" "https://jamgvct.co$path"
done
```

Y para auditar específicamente que el Gateway (Caddy) está inyectando la regla de seguridad en el Service Worker:
```bash
curl -sI https://jamgvct.co/service-worker.js | grep -i -E "HTTP/|cache-control|content-type"
```

Cuando veas los `200 OK`, el `application/manifest+json` y el sagrado `Cache-Control: no-cache` en la terminal, sabrás que el perímetro está asegurado y puedes lanzar el tag `v0.1-demo-ui`.

genere el archivo `README.md` completo y consolidado con la Opción A ejecutar tu `git init`


> **DEMO UI — Prototipo visual**
> Sin nodo operativo, sin captación, sin liquidación financiera.
> `SETTLEMENT: LOCKED` / `PRODUCTION_AUTHORIZATION: false`

Infraestructura digital, automatización y gobernanza con trazabilidad PI VCT.
Entorno de demostración técnica. Barranquilla, Colombia.

**Sitio DEMO:** https://jamgvct.co

typescript
import { createPublicKey, verify, KeyObject } from 'node:crypto';

export interface AuthorizedOperator {
  operatorId: string;
  role: 'COMPLIANCE_OFFICER' | 'SYSTEM_AUDITOR';
  publicKeyHex: string; // Llave pública Ed25519 cruda de 32 bytes (64 caracteres hex)
  active: boolean;
}

// Prefijo ASN.1 DER estándar para envolver una llave pública Ed25519 cruda de 32 bytes
const ED25519_SPKI_PREFIX = Buffer.from('302a300506032b6570032100', 'hex');

function hexToEd25519PublicKey(publicKeyHex: string): KeyObject {
  if (!/^[a-f0-9]{64}$/i.test(publicKeyHex)) {
    throw new Error('ERR_INVALID_PUBLIC_KEY: La llave pública Ed25519 debe tener 32 bytes (64 caracteres hex).');
  }
  const rawKey = Buffer.from(publicKeyHex, 'hex');
  const spkiDer = Buffer.concat([ED25519_SPKI_PREFIX, rawKey]);
  return createPublicKey({ key: spkiDer, format: 'der', type: 'spki' });
}

export function verifyOperatorEd25519Signature(params: {
  operator: AuthorizedOperator;
  operationId: string;
  invoiceNumber: string;
  accountingSupportHash: string;
  nominalAmountUsd: number;
  approvedAtIso: string;
  signatureHex: string;
}): boolean {
  if (!params.operator.active) {
    throw new Error('ERR_OPERATOR_INACTIVE: El operador no se encuentra habilitado para firmar aprobaciones.');
  }

  if (!/^[a-f0-9]{128}$/i.test(params.signatureHex)) {
    return false;
  }

  // Mensaje canónico determinista que el operador debe firmar en su cliente
  const canonicalMessage = [
    'JAMG-VCT-RFC-002',
    params.operationId,
    params.invoiceNumber,
    params.accountingSupportHash,
    params.nominalAmountUsd.toFixed(6),
    params.approvedAtIso,
  ].join('|');

  const publicKey = hexToEd25519PublicKey(params.operator.publicKeyHex);
  const signatureBuffer = Buffer.from(params.signatureHex, 'hex');
  const messageBuffer = Buffer.from(canonicalMessage, 'utf8');

  return verify(null, messageBuffer, publicKey, signatureBuffer);

import { Request, Response } from 'express';
import {
  RFC002ComplianceEngine,
  B2BComplianceInput,
} from '../../core/rfc002-compliance-engine';
import {
  AuthorizedOperator,
  verifyOperatorEd25519Signature,
} from '../../security/ed25519-verifier';

// Registro de operadores autorizados (cargar desde PostgreSQL / variables de entorno seguras en producción)
const OPERATOR_REGISTRY: Record<string, AuthorizedOperator> = {
  'op-barranquilla-01': {
    operatorId: 'op-barranquilla-01',
    role: 'COMPLIANCE_OFFICER',
    publicKeyHex: process.env.OPERATOR_BQ_01_ED25519_PUBKEY || '',
    active: true,
  },
};

export async function handleB2BAuditRequest(req: Request, res: Response): Promise<void> {
  try {
    const payload = req.body as B2BComplianceInput;

    if (!payload || typeof payload !== 'object') {
      res.status(400).json({
        error: 'ERR_INVALID_BODY',
        message: 'El cuerpo de la petición debe ser un objeto JSON válido.',
      });
      return;
    }

    // 1. Bloqueo explícito si se intenta enviar cualquier modo distinto a auditoría
    if (payload.settlementMode !== 'LOCKED_AUDIT_ONLY') {
      res.status(403).json({
        error: 'ERR_SETTLEMENT_DISABLED',
        message: 'GMS CORE mantiene bloqueada toda liquidación. Use settlementMode: "LOCKED_AUDIT_ONLY".',
      });
      return;
    }

    // 2. Verificar existencia del operador humano autorizado
    const operatorId = payload.humanApproval?.operatorId;
    const operator = operatorId ? OPERATOR_REGISTRY[operatorId] : undefined;

    if (!operator || !operator.publicKeyHex) {
      res.status(401).json({
        error: 'ERR_UNKNOWN_OPERATOR',
        message: 'Operador no autorizado o llave pública Ed25519 no configurada.',
      });
      return;
    }

    // 3. Verificación criptográfica real Ed25519 sobre el expediente B2B
    const isSignatureValid = verifyOperatorEd25519Signature({
      operator,
      operationId: payload.operationId,
      invoiceNumber: payload.b2bDossier?.invoiceNumber || '',
      accountingSupportHash: payload.b2bDossier?.accountingSupportHash || '',
      nominalAmountUsd: Number(payload.nominalAmountUsd || 0),
      approvedAtIso: payload.humanApproval?.approvedAtIso || '',
      signatureHex: payload.humanApproval?.ed25519Signature || '',
    });

    if (!isSignatureValid) {
      res.status(401).json({
        error: 'ERR_INVALID_ED25519_SIGNATURE',
        message: 'La firma criptográfica Ed25519 no corresponde al operador o los datos fueron alterados.',
      });
      return;
    }

    // 4. Ejecutar motor de cumplimiento RFC-002 (genera certificado SHA-256 y tarifa SaaS 0.01%)
    const auditCertificate = RFC002ComplianceEngine.evaluateB2BOperation(payload);

    res.status(200).json({
      status: 'AUDITED_AND_LOGGED',
      governance: 'HUMAN_ETHICS_FIRST_VERIFIED',
      certificate: auditCertificate,
    });
  } catch (err: unknown) {
    const errorMessage = err instanceof Error ? err.message : 'Error interno de validación';
    const errorCode = errorMessage.split(':')[0] || 'ERR_AUDIT_FAILED';

    res.status(422).json({
      error: errorCode,
      message: errorMessage,
    });
  }
}

import { Router } from 'express';
import { handleB2BAuditRequest } from '../controllers/rfc002-audit.controller';

const v1Router = Router();

// Endpoint RFC-002: Auditoría B2B con liquidación bloqueada
v1Router.post('/compliance/b2b/audit', handleB2BAuditRequest);

export default v1Router;

node -e '
const { generateKeyPairSync, sign } = require("node:crypto");
const { publicKey, privateKey } = generateKeyPairSync("ed25519");
const pubHex = publicKey.export({ type: "spki", format: "der" }).subarray(-32).toString("hex");
const msg = [
  "JAMG-VCT-RFC-002",
  "550e8400-e29b-41d4-a716-446655440000",
  "FAC-2026-0891",
  "a".repeat(64),
  (50000).toFixed(6),
  "2026-11-13T16:30:00.000Z"
].join("|");
const sigHex = sign(null, Buffer.from(msg, "utf8"), privateKey).toString("hex");
console.log("OPERATOR_BQ_01_ED25519_PUBKEY=" + pubHex);
console.log("ed25519Signature=" + sigHex);
'

---

markdown
# RFC-002: Motor de Control, Documentación y Trazabilidad B2B (GMS CORE)

- **Estado:** Aprobado para Implementación (Modo Auditoría / Liquidación Bloqueada)
- **Autor:** Arquitectura JAMG VCT ℣
- **Jurisdicción Base:** Barranquilla, Colombia (CIIU 6201 / 6311)
- **Dependencias:** `RFC-001` (Gobernanza Ed25519 y *Human Ethics First*)

## 1. Propósito y Alcance
Definir el estándar técnico de validación previa, generación de expedientes contables y trazabilidad criptográfica para operaciones B2B que referencien activos digitales (ej. USDC, USDT) antes de cualquier integración con terceros vigilados. 

**GMS CORE** actúa exclusivamente como capa de software de control y documentación. **No custodia fondos, no emite activos y mantiene deshabilitado cualquier conector de liquidación on-chain o bancaria.**

## 2. Reglas de Bloqueo y Gobernanza (Hard Guardrails)
1. **Bloqueo Estricto de Liquidación (`SETTLEMENT_LOCKED = true`):** Cualquier intento de invocar métodos de dispersión o firma de transacciones en red debe lanzar una excepción fatal `ERR_SETTLEMENT_DISABLED`.
2. **Estándares Internacionales de Identificación:**
   - **ISO 24165 (DTI):** Todo activo evaluado debe incluir su *Digital Token Identifier* válido de 9 caracteres alfanuméricos.
   - **ISO 17442 (LEI):** Las contrapartes corporativas o emisores referenciados deben aportar un *Legal Entity Identifier* de 20 caracteres cuando aplique.
3. **Expediente Documental B2B Obligatorio:** Ninguna evaluación puede cerrarse sin referencia de factura comercial, declaración cambiaria/contable aplicable y verificación de umbral *Travel Rule* (≥ 1.000 USD o equivalente).
4. **Cálculo de Tarifa Tecnológica SaaS:** Se calcula el **0.01 %** sobre el valor nominal evaluado como tarifa de licenciamiento de software, generando un registro inmutable SHA-256.

{
  "$schema": "https://json-schema.org",
  "$id": "https://jamgvct.co",
  "title": "GMSCoreB2BCompliancePayload",
  "type": "object",
  "required": [
    "rfcVersion",
    "operationId",
    "settlementMode",
    "asset",
    "counterparties",
    "b2bDossier",
    "nominalAmountUsd",
    "humanApproval"
  ],
  "additionalProperties": false,
  "properties": {
    "rfcVersion": {
      "const": "RFC-002"
    },
    "operationId": {
      "type": "string",
      "format": "uuid"
    },
    "settlementMode": {
      "const": "LOCKED_AUDIT_ONLY",
      "description": "Bloqueo absoluto de liquidación; solo documentación y trazabilidad."
    },
    "asset": {
      "type": "object",
      "required": ["symbol", "dtiCode", "network"],
      "additionalProperties": false,
      "properties": {
        "symbol": { "type": "string", "enum": ["USDC", "USDT", "COP_REF"] },
        "dtiCode": {
          "type": "string",
          "pattern": "^[A-Z0-9]{9}$",
          "description": "Identificador ISO 24165 Digital Token Identifier"
        },
        "network": { "type": "string", "minLength": 2, "maxLength": 32 },
        "issuerLei": {
          "type": "string",
          "pattern": "^[A-Z0-9]{20}$",
          "description": "Código ISO 17442 Legal Entity Identifier del emisor"
        }
      }
    },
    "counterparties": {
      "type": "object",
      "required": ["originator", "beneficiary"],
      "additionalProperties": false,
      "properties": {
        "originator": {
          "type": "object",
          "required": ["legalName", "taxId", "countryCode"],
          "properties": {
            "legalName": { "type": "string", "minLength": 3 },
            "taxId": { "type": "string", "minLength": 5 },
            "countryCode": { "type": "string", "pattern": "^[A-Z]{2}$" },
            "lei": { "type": "string", "pattern": "^[A-Z0-9]{20}$" }
          }
        },
        "beneficiary": {
          "type": "object",
          "required": ["legalName", "taxId", "countryCode"],
          "properties": {
            "legalName": { "type": "string", "minLength": 3 },
            "taxId": { "type": "string", "minLength": 5 },
            "countryCode": { "type": "string", "pattern": "^[A-Z]{2}$" },
            "lei": { "type": "string", "pattern": "^[A-Z0-9]{20}$" }
          }
        }
      }
    },
    "b2bDossier": {
      "type": "object",
      "required": ["invoiceNumber", "purposeCode", "accountingSupportHash"],
      "additionalProperties": false,
      "properties": {
        "invoiceNumber": { "type": "string", "minLength": 3 },
        "purposeCode": {
          "type": "string",
          "enum": ["IMPORT_GOODS", "EXPORT_SERVICES", "SAAS_LICENSE", "B2B_SUPPLIER"]
        },
        "fxDeclarationRef": { "type": "string" },
        "accountingSupportHash": {
          "type": "string",
          "pattern": "^[a-f0-9]{64}$"
        }
      }
    },
    "nominalAmountUsd": {
      "type": "number",
      "exclusiveMinimum": 0,
      "maximum": 10000000
    },
    "humanApproval": {
      "type": "object",
      "required": ["operatorId", "ed25519Signature", "approvedAtIso"],
      "additionalProperties": false,
      "properties": {
        "operatorId": { "type": "string", "minLength": 4 },
        "ed25519Signature": {
          "type": "string",
          "pattern": "^[a-f0-9]{128}$"
        },
        "approvedAtIso": { "type": "string", "format": "date-time" }
      }
    }
  }
}

import { createHash } from 'node:crypto';

export interface B2BComplianceInput {
  rfcVersion: 'RFC-002';
  operationId: string;
  settlementMode: 'LOCKED_AUDIT_ONLY';
  asset: {
    symbol: 'USDC' | 'USDT' | 'COP_REF';
    dtiCode: string;
    network: string;
    issuerLei?: string;
  };
  counterparties: {
    originator: { legalName: string; taxId: string; countryCode: string; lei?: string };
    beneficiary: { legalName: string; taxId: string; countryCode: string; lei?: string };
  };
  b2bDossier: {
    invoiceNumber: string;
    purposeCode: 'IMPORT_GOODS' | 'EXPORT_SERVICES' | 'SAAS_LICENSE' | 'B2B_SUPPLIER';
    fxDeclarationRef?: string;
    accountingSupportHash: string;
  };
  nominalAmountUsd: number;
  humanApproval: {
    operatorId: string;
    ed25519Signature: string;
    approvedAtIso: string;
  };
}

export interface RFC002AuditCertificate {
  auditId: string;
  rfcVersion: 'RFC-002';
  settlementStatus: 'BLOCKED_BY_POLICY';
  travelRuleRequired: boolean;
  crossBorderFxAlert: boolean;
  saasLicensingFeeUsd: number;
  immutableRecordHash: string;
  evaluatedAt: string;
}

const DTI_REGEX = /^[A-Z0-9]{9}$/;
const LEI_REGEX = /^[A-Z0-9]{20}$/;
const SHA256_REGEX = /^[a-f0-9]{64}$/;
const ED25519_SIG_REGEX = /^[a-f0-9]{128}$/;

export class RFC002ComplianceEngine {
  private static readonly SETTLEMENT_LOCKED = true;
  private static readonly SAAS_FEE_RATE = 0.0001; // 0.01%
  private static readonly TRAVEL_RULE_THRESHOLD_USD = 1000;

  /**
   * Bloqueo explícito de cualquier intento de liquidación o movimiento de fondos.
   */
  public static executeSettlement(): never {
    throw new Error(
      'ERR_SETTLEMENT_DISABLED: GMS CORE opera exclusivamente como infraestructura de auditoría B2B. La liquidación está bloqueada.'
    );
  }

  /**
   * Evalúa el expediente B2B y genera un certificado inmutable de trazabilidad.
   */
  public static evaluateB2BOperation(input: B2BComplianceInput): RFC002AuditCertificate {
    if (!this.SETTLEMENT_LOCKED || input.settlementMode !== 'LOCKED_AUDIT_ONLY') {
      throw new Error('ERR_POLICY_VIOLATION: El modo de liquidación debe permanecer en LOCKED_AUDIT_ONLY.');
    }

    if (!DTI_REGEX.test(input.asset.dtiCode)) {
      throw new Error('ERR_INVALID_DTI: El código ISO 24165 (DTI) debe tener exactamente 9 caracteres alfanuméricos.');
    }

    if (input.asset.issuerLei && !LEI_REGEX.test(input.asset.issuerLei)) {
      throw new Error('ERR_INVALID_LEI: El código LEI del emisor debe cumplir el estándar ISO 17442 (20 caracteres).');
    }

    if (!SHA256_REGEX.test(input.b2bDossier.accountingSupportHash)) {
      throw new Error('ERR_INVALID_DOSSIER_HASH: El soporte contable debe incluir un hash SHA-256 válido.');
    }

    if (!ED25519_SIG_REGEX.test(input.humanApproval.ed25519Signature)) {
      throw new Error('ERR_MISSING_HUMAN_SIGNATURE: Se requiere firma criptográfica Ed25519 válida (Human Ethics First).');
    }

    if (input.nominalAmountUsd <= 0 || !Number.isFinite(input.nominalAmountUsd)) {
      throw new Error('ERR_INVALID_AMOUNT: El monto nominal de referencia debe ser mayor a cero.');
    }

    const crossBorderFxAlert =
      input.counterparties.originator.countryCode !== input.counterparties.beneficiary.countryCode;

    if (crossBorderFxAlert && !input.b2bDossier.fxDeclarationRef) {
      throw new Error(
        'ERR_MISSING_FX_DECLARATION: Operaciones transfronterizas requieren referencia documental cambiaria (fxDeclarationRef).'
      );
    }

    const travelRuleRequired = input.nominalAmountUsd >= this.TRAVEL_RULE_THRESHOLD_USD;
    const saasLicensingFeeUsd = Number((input.nominalAmountUsd * this.SAAS_FEE_RATE).toFixed(6));
    const evaluatedAt = new Date().toISOString();

    const canonicalPayload = JSON.stringify({
      operationId: input.operationId,
      dtiCode: input.asset.dtiCode,
      invoiceNumber: input.b2bDossier.invoiceNumber,
      accountingSupportHash: input.b2bDossier.accountingSupportHash,
      nominalAmountUsd: input.nominalAmountUsd,
      saasLicensingFeeUsd,
      operatorId: input.humanApproval.operatorId,
      ed25519Signature: input.humanApproval.ed25519Signature,
    });

    const immutableRecordHash = createHash('sha256').update(canonicalPayload).digest('hex');

    return {
      auditId: `AUD-${input.operationId}`,
      rfcVersion: 'RFC-002',
      settlementStatus: 'BLOCKED_BY_POLICY',
      travelRuleRequired,
      crossBorderFxAlert,
      saasLicensingFeeUsd,
      immutableRecordHash,
      evaluatedAt,
    };
  }
}

import { describe, it, expect } from 'vitest';
import { RFC002ComplianceEngine, B2BComplianceInput } from '../src/core/rfc002-compliance-engine';

const buildValidPayload = (): B2BComplianceInput => ({
  rfcVersion: 'RFC-002',
  operationId: '550e8400-e29b-41d4-a716-446655440000',
  settlementMode: 'LOCKED_AUDIT_ONLY',
  asset: {
    symbol: 'USDC',
    dtiCode: '4H95J0R2X',
    network: 'ETHEREUM',
    issuerLei: '5493006K927321456789',
  },
  counterparties: {
    originator: {
      legalName: 'IMPORTADORA CARIBE S.A.S.',
      taxId: '900123456-1',
      countryCode: 'CO',
    },
    beneficiary: {
      legalName: 'GLOBAL LOGISTICS CORP',
      taxId: 'US-987654321',
      countryCode: 'US',
    },
  },
  b2bDossier: {
    invoiceNumber: 'FAC-2026-0891',
    purposeCode: 'IMPORT_GOODS',
    fxDeclarationRef: 'DEC-CAMBIO-2026-774',
    accountingSupportHash: 'a'.repeat(64),
  },
  nominalAmountUsd: 50000,
  humanApproval: {
    operatorId: 'op-barranquilla-01',
    ed25519Signature: 'b'.repeat(128),
    approvedAtIso: '2026-11-13T16:30:00.000Z',
  },
});

describe('RFC-002: Motor de Control y Trazabilidad B2B (GMS CORE)', () => {
  it('bloquea cualquier intento de ejecución de liquidación', () => {
    expect(() => RFC002ComplianceEngine.executeSettlement()).toThrow('ERR_SETTLEMENT_DISABLED');
  });

  it('genera certificado inmutable y calcula tarifa SaaS del 0.01 % correctamente', () => {
    const result = RFC002ComplianceEngine.evaluateB2BOperation(buildValidPayload());

    expect(result.settlementStatus).toBe('BLOCKED_BY_POLICY');
    expect(result.travelRuleRequired).toBe(true);
    expect(result.crossBorderFxAlert).toBe(true);
    expect(result.saasLicensingFeeUsd).toBe(5); // 0.01% de 50,000 USD = 5 USD
    expect(result.immutableRecordHash).toHaveLength(64);
  });

  it('rechaza operaciones sin identificador ISO 24165 (DTI) válido', () => {
    const payload = buildValidPayload();
    payload.asset.dtiCode = 'INVALID';

    expect(() => RFC002ComplianceEngine.evaluateB2BOperation(payload)).toThrow('ERR_INVALID_DTI');
  });

  it('rechaza operaciones transfronterizas sin soporte documental cambiario', () => {
    const payload = buildValidPayload();
    delete payload.b2bDossier.fxDeclarationRef;

    expect(() => RFC002ComplianceEngine.evaluateB2BOperation(payload)).toThrow('ERR_MISSING_FX_DECLARATION');
  });

  it('rechaza operaciones sin firma humana Ed25519 válida', () => {
    const payload = buildValidPayload();
    payload.humanApproval.ed25519Signature = 'firma-invalida';

    expect(() => RFC002ComplianceEngine.evaluateB2BOperation(payload)).toThrow('ERR_MISSING_HUMAN_SIGNATURE');
  });
});


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

{"Status":3,"TC":false,"RD":true,"RA":true,"AD":false,"CD":false,"Question":[{"name":"jamgvct.co.","type":1}],"Authority":[{"name":"co.","type":6,"TTL":201,"data":"ns0.centralnic.net. hostmaster.centralnic.net. 1759802481 900 1800 6048000 3600"}]}

{"Status":3,"TC":false,"RD":true,"RA":true,"AD":false,"CD":false,"Question":[{"name":"jamgvct.co.","type":28}],"Authority":[{"name":"co.","type":6,"TTL":900,"data":"ns0.centralnic.net. hostmaster.centralnic.net. 1759802488 900 1800 6048000 3600"}],"Comment":"Response from 212.18.249.57."}

{"Status":3,"TC":false,"RD":true,"RA":true,"AD":false,"CD":false,"Question":[{"name":"jamgvct.co.","type":2}],"Authority":[{"name":"co.","type":6,"TTL":900,"data":"ns0.centralnic.net. hostmaster.centralnic.net. 1759802489 900 1800 6048000 3600"}],"Comment":"Response from 185.24.64.57."}

{"Status":3,"TC":false,"RD":true,"RA":true,"AD":false,"CD":false,"Question":[{"name":"www.jamgvct.co.","type":1}],"Authority":[{"name":"co.","type":6,"TTL":900,"data":"ns0.centralnic.net. hostmaster.centralnic.net. 1759802489 900 1800 6048000 3600"}],"Comment":"Response from 2a04:2b00:13ee::57."}

content://com.google.android.apps.nbu.files.provider/2/33472#servicios

Genera el artefacto README.md siguiendo el PROMPT MAESTRO.
Inspecciona primero la estructura real del repositorio.
No inventes componentes ni declares pruebas exitosas sin evidencia.
Entrega el README completo y un informe de validación.

.github/workflows/build-apk.yml

name: JAMG VCT Android APK (DEMO)

on:
  workflow_dispatch:
  push:
    branches: [ main, master ]

permissions:
  contents: read
  actions: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Install dependencies
        run: npm install

      - name: Create Android project if absent
        run: |
          if [ ! -d android ]; then npx cap add android; fi

      - name: Sync portal into Android
        run: npx cap sync android

      - name: Build debug APK
        working-directory: android
        run: |
          chmod +x gradlew
          ./gradlew --no-daemon assembleDebug

      - name: Verify APK and calculate SHA-256
        run: |
          APK="android/app/build/outputs/apk/debug/app-debug.apk"
          test -s "$APK"
          sha256sum "$APK" | tee android/app/build/outputs/apk/debug/SHA256SUMS.txt
          ls -lh "$APK"

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: JAMG-VCT-Android-Debug-APK
          path: |
            android/app/build/outputs/apk/debug/app-debug.apk
            android/app/build/outputs/apk/debug/SHA256SUMS.txt
          if-no-files-found: error
          retention-days: 14


Actions → JAMG VCT Android APK (DEMO) → Run workflow
JAMG-VCT-Android-Debug-APK/
├── app-debug.apk
└── SHA256SUMS.txt