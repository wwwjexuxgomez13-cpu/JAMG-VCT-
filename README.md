
curl -I https://jamgvct.co/
curl -I https://jamgvct.co/manifest.webmanifest
curl -I https://jamgvct.co/service-worker.js
curl -I https://jamgvct.co/assets/icon-192.png
curl -I https://jamgvct.co/assets/icon-512.png


/manifest.webmanifest
  Content-Type: application/manifest+json
  Cache-Control: public, max-age=3600

/service-worker.js
  Content-Type: application/javascript
  Cache-Control: no-cache

/assets/icon-192.png
  Content-Type: image/png

/assets/icon-512.png
  Content-Type: image/png

@noCache path /service-worker.js /manifest.webmanifest
header @noCache Cache-Control "no-cache"
 https://github.com/wwwjexuxgomez13-cpu/jamg-vct-os.git

cd jamg-vct-os

git status
git branch --show-current
git log -1 --oneline

git checkout -b security/1377-demonstrable-gate

jamg-vct-os/
├── .gitlab-ci.yml
├── .gitlab/
│   ├── CODEOWNERS
│   └── security/
├── GOVERNANCE.md
├── evidence/
│   └── README.md
├── tests/
├── package.json
├── package-lock.json
├── requirements.txt
└── tsconfig.json

find . -maxdepth 3 -type f \
  \( -name "package.json" \
  -o -name "package-lock.json" \
  -o -name "requirements.txt" \
  -o -name "pyproject.toml" \
  -o -name "tsconfig.json" \
  -o -name "pytest.ini" \) \
  -not -path "./node_modules/*" \
  -not -path "./.git/*"

repositorio/imagen:version@sha256:DIGEST_VERIFICADO

# JAMG VCT — DEMONSTRABLE GOVERNANCE GATE
# HUMAN ETHICS FIRST
# Checkpoint: 13,677
# Production and settlement remain disabled.

stages:
  - test
  - security
  - compliance
  - governance_gate

workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH'
    - when: never

variables:
  GIT_DEPTH: "0"
  PRODUCTION_AUTHORIZATION: "false"
  SETTLEMENT_STATUS: "ARMED_STANDBY_MAINTAINED"

  PYTHON_IMAGE: "$PYTHON_IMAGE"
  NODE_IMAGE: "$NODE_IMAGE"
  SEMGREP_IMAGE: "$SEMGREP_IMAGE"
  GITLEAKS_IMAGE: "$GITLEAKS_IMAGE"
  SYFT_IMAGE: "$SYFT_IMAGE"

default:
  interruptible: true

# --------------------------------------------------
# 01. PYTHON TESTS
# --------------------------------------------------

python_tests:
  stage: test
  image: "$PYTHON_IMAGE"
  script:
    - |
      set +e

      if [ ! -f requirements.txt ] && [ ! -f pyproject.toml ]; then
        echo "NOT_APPLICABLE" > python-test.exit
        exit 0
      fi

      python -m pip install --no-cache-dir pytest

      if [ -f requirements.txt ]; then
        python -m pip install --no-cache-dir -r requirements.txt
      fi

      if [ -f pyproject.toml ]; then
        python -m pip install --no-cache-dir .
      fi

      python -m pytest -q \
        --junitxml=python-tests.xml

      RESULT=$?
      echo "$RESULT" > python-test.exit
      exit "$RESULT"

  artifacts:
    when: always
    expire_in: 1 year
    paths:
      - python-test.exit
      - python-tests.xml

# --------------------------------------------------
# 02. NODE / TYPESCRIPT TESTS
# --------------------------------------------------

node_tests:
  stage: test
  image: "$NODE_IMAGE"
  script:
    - |
      set +e

      if [ ! -f package.json ]; then
        echo "NOT_APPLICABLE" > node-test.exit
        exit 0
      fi

      if [ ! -f package-lock.json ]; then
        echo "ERROR: package-lock.json missing"
        echo "1" > node-test.exit
        exit 1
      fi

      npm ci

      RESULT=$?

      if [ "$RESULT" -ne 0 ]; then
        echo "$RESULT" > node-test.exit
        exit "$RESULT"
      fi

      npm test

      RESULT=$?

      if [ "$RESULT" -eq 0 ] && [ -f tsconfig.json ]; then
        npx --no-install tsc --noEmit
        RESULT=$?
      fi

      echo "$RESULT" > node-test.exit
      exit "$RESULT"

  artifacts:
    when: always
    expire_in: 1 year
    paths:
      - node-test.exit

# --------------------------------------------------
# 03. SAST — SEMGREP
# --------------------------------------------------

sast_semgrep:
  stage: security
  image:
    name: "$SEMGREP_IMAGE"
    entrypoint: [""]
  script:
    - |
      set +e

      semgrep scan \
        --config p/default \
        --error \
        --json \
        --output semgrep-report.json .

      RESULT=$?

      echo "$RESULT" > semgrep.exit
      exit "$RESULT"

  artifacts:
    when: always
    expire_in: 1 year
    paths:
      - semgrep-report.json
      - semgrep.exit

# --------------------------------------------------
# 04. SECRET DETECTION — GITLEAKS
# --------------------------------------------------

secret_detection:
  stage: security
  image:
    name: "$GITLEAKS_IMAGE"
    entrypoint: [""]
  script:
    - |
      set +e

      gitleaks detect \
        --source . \
        --report-format json \
        --report-path gitleaks-report.json \
        --exit-code 1

      RESULT=$?

      echo "$RESULT" > gitleaks.exit
      exit "$RESULT"

  artifacts:
    when: always
    expire_in: 1 year
    paths:
      - gitleaks-report.json
      - gitleaks.exit

# --------------------------------------------------
# 05. SOFTWARE BILL OF MATERIALS
# --------------------------------------------------

generate_sbom:
  stage: compliance
  image:
    name: "$SYFT_IMAGE"
    entrypoint: [""]
  script:
    - |
      set +e

      syft dir:. \
        -o cyclonedx-json=sbom-cyclonedx.json

      RESULT=$?

      echo "$RESULT" > sbom.exit
      exit "$RESULT"

  artifacts:
    when: always
    expire_in: 1 year
    paths:
      - sbom-cyclonedx.json
      - sbom.exit

# --------------------------------------------------
# 06. FINAL GOVERNANCE GATE
# --------------------------------------------------

governance_gate:
  stage: governance_gate
  image: alpine:3.20

  needs:
    - job: python_tests
      artifacts: true

    - job: node_tests
      artifacts: true

    - job: sast_semgrep
      artifacts: true

    - job: secret_detection
      artifacts: true

    - job: generate_sbom
      artifacts: true

  when: always
  allow_failure: false

  script:
    - apk add --no-cache python3 coreutils
    - |
      set -eu

      echo "JAMG VCT GOVERNANCE VALIDATION"

      # A. Tests must have been executed
      PYTHON_RESULT="$(cat python-test.exit)"
      NODE_RESULT="$(cat node-test.exit)"

      if [ "$PYTHON_RESULT" != "0" ] &&
         [ "$PYTHON_RESULT" != "NOT_APPLICABLE" ]; then
        echo "BLOCKED: Python tests failed"
        exit 1
      fi

      if [ "$NODE_RESULT" != "0" ] &&
         [ "$NODE_RESULT" != "NOT_APPLICABLE" ]; then
        echo "BLOCKED: Node tests failed"
        exit 1
      fi

      if [ "$PYTHON_RESULT" = "NOT_APPLICABLE" ] &&
         [ "$NODE_RESULT" = "NOT_APPLICABLE" ]; then
        echo "BLOCKED: No supported project stack detected"
        exit 1
      fi

      # B. Security scanners must exit successfully
      test "$(cat semgrep.exit)" = "0"
      test "$(cat gitleaks.exit)" = "0"

      # C. SBOM generation must succeed
      test "$(cat sbom.exit)" = "0"
      test -s sbom-cyclonedx.json

      # D. Validate the SBOM
      python3 - <<'PY'
      import json

      with open("sbom-cyclonedx.json", encoding="utf-8") as f:
          sbom = json.load(f)

      assert sbom.get("bomFormat") == "CycloneDX"
      assert isinstance(sbom.get("components"), list)

      print("SBOM_SCHEMA_VALID")
      PY

      # E. Verify reports exist
      test -s semgrep-report.json
      test -s gitleaks-report.json

      # F. Verify governance lock
      test "$PRODUCTION_AUTHORIZATION" = "false"
      test "$SETTLEMENT_STATUS" = "ARMED_STANDBY_MAINTAINED"

      # G. Hash evidence artifacts
      mkdir -p evidence

      sha256sum \
        semgrep-report.json \
        gitleaks-report.json \
        sbom-cyclonedx.json \
        > evidence/SHA256SUMS

      # H. Create evidence manifest
      python3 - <<'PY'
      import json
      import os
      from datetime import datetime, timezone

      manifest = {
          "project": os.environ["CI_PROJECT_PATH"],
          "commit": os.environ["CI_COMMIT_SHA"],
          "pipeline_id": os.environ["CI_PIPELINE_ID"],
          "job_id": os.environ["CI_JOB_ID"],
          "checkpoint": "13,677",
          "governance": "HUMAN_ETHICS_FIRST",
          "technical_status": "CI_CONTROLS_PASSED",
          "production_authorization": False,
          "settlement_status": "ARMED_STANDBY_MAINTAINED",
          "timestamp": datetime.now(timezone.utc).isoformat()
      }

      with open("evidence/manifest.json", "w") as f:
          json.dump(manifest, f, indent=2)
      PY

      echo "TECHNICAL_GATE_PASSED"
      echo "PRODUCTION_REMAINS_BLOCKED"

  artifacts:
    when: always
    expire_in: 1 year
    paths:
      - evidence/
      - sbom-cyclonedx.json
      - semgrep-report.json
      - gitleaks-report.json

# JAMG VCT — GOVERNANCE POLICY

Governance: HUMAN_ETHICS_FIRST

## Technical controls

- Unit tests mandatory.
- SAST mandatory.
- Secret detection mandatory.
- CycloneDX SBOM mandatory.
- SHA-256 artifact integrity mandatory.

## Production controls

PRODUCTION_AUTHORIZATION=false

Production deployment requires:
- Protected production environment.
- Independent human approval.
- Verified release artifact.
- Approved security evidence.

## Financial controls

SETTLEMENT_STATUS=ARMED_STANDBY_MAINTAINED

No financial settlement is permitted
without independent financial authorization
and verified external evidence.

## AI governance

AI agents cannot independently:
- Authorize production.
- Change financial balances.
- Approve settlement.
- Override human controls.

/.gitlab-ci.yml       @wwwjexuxgomez13-cpu
/GOVERNANCE.md        @wwwjexuxgomez13-cpu
/.gitlab/             @wwwjexuxgomez13-cpu

Environment: production

Allowed to deploy:
  Authorized Maintainers

Required approvals:
  Independent authorized approver

Self approval:
  Disabled

Deployment:
  Protected

PYTHON_IMAGE: "$PYTHON_IMAGE"
NODE_IMAGE: "$NODE_IMAGE"
SEMGREP_IMAGE: "$SEMGREP_IMAGE"
GITLEAKS_IMAGE: "$GITLEAKS_IMAGE"
SYFT_IMAGE: "$SYFT_IMAGE"

git add .gitlab-ci.yml GOVERNANCE.md .gitlab/CODEOWNERS

git status

git commit -m \
"security: implement demonstrable JAMG VCT governance gate"

git push -u origin security/1377-demonstrable-gate

python_tests: FAILED
governance_gate: FAILED or BLOCKED
production: BLOCKED

python -m json.tool sbom-cyclonedx.json

python -c '
import json
d=json.load(open("sbom-cyclonedx.json"))
assert d["bomFormat"] == "CycloneDX"
assert isinstance(d["components"], list)
print("SBOM VALID")
'

SBOM VALID

sha256sum -c evidence/SHA256SUMS

semgrep-report.json: OK
gitleaks-report.json: OK
sbom-cyclonedx.json: OK

CI execution
    |
    v
Evidence manifest
    |
    v
SHA-256 verification
    |
    v
Digital signature
    |
    v
Public key verification
    |
    v
Independent timestamp
    |
    v
Evidence archive

{
  "checkpoint": "13,677",
  "project": "jamg-vct-os",
  "governance": "HUMAN_ETHICS_FIRST",
  "technical_gate": {
    "configuration": "VALIDATED",
    "execution": "COMPLETED",
    "python_tests": "PASS",
    "node_tests": "PASS_OR_NOT_APPLICABLE",
    "sast": "PASS",
    "secret_detection": "PASS",
    "sbom": "VALID",
    "artifact_integrity": "VERIFIED"
  },
  "authorization": {
    "production": false,
    "settlement": "ARMED_STANDBY_MAINTAINED"
  },
  "financial_evidence": "PENDING",
  "deployment": "BLOCKED",
  "signature_verification": "PENDING"
}

export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Estado sincronizado con el manifiesto JSON (Variables de entorno en Cloudflare)
    const PRODUCTION_AUTHORIZATION = env.PRODUCTION_AUTHORIZATION === "true"; // false por defecto
    const CHECKPOINT = "13,637-ACK";
    const NODE_REF = "1045668717-MS-ONE";

    // 1. Bloqueo Fail-Closed en rutas de liquidación o ejecución financiera
    if (url.pathname.startsWith("/api/v1/settlement")) {
      if (!PRODUCTION_AUTHORIZATION) {
        return new Response(
          JSON.stringify({
            node_ref: NODE_REF,
            declared_checkpoint: CHECKPOINT,
            settlement_state: "PENDING_CRYPTOGRAPHIC_VERIFICATION",
            fail_closed_enforced: true,
            message: "Operación bloqueada: falta hash de certificación bancaria y firma ML-DSA-65."
          }),
          {
            status: 503,
            headers: {
              "Content-Type": "application/json",
              "X-Node-Ref": NODE_REF,
              "X-Governance-Label": "JAMG — HUMAN ETHICS FIRST",
              "Cache-Control": "no-store"
            }
          }
        );
      }

      // 2. Verificación mTLS de Cloudflare (Client Certificate) si producción está activa
      const certAuth = request.cf?.tlsClientAuth;
      if (!certAuth || certAuth.certVerified !== "SUCCESS") {
        return new Response(
          JSON.stringify({ error: "MTLS_VERIFICATION_FAILED", node_ref: NODE_REF }),
          { status: 403, headers: { "Content-Type": "application/json" } }
        );
      }
    }

    // 3. Reenvío seguro al servidor origen con cabeceras de trazabilidad
    const modifiedRequest = new Request(request, {
      headers: new Headers(request.headers)
    });
    modifiedRequest.headers.set("X-Node-Ref", NODE_REF);
    modifiedRequest.headers.set("X-External-Reference", "080016001067202519619");

    return fetch(modifiedRequest);
  }
};
_format_version: "3.0"
_transform: true

services:
  - name: jamg-ledger-core
    url: http://10.0.1.50:3000
    protocol: http
    connect_timeout: 5000
    write_timeout: 5000
    read_timeout: 5000
    routes:
      # Ruta de liquidación (Bloqueada por Fail-Closed hasta evidencia bancaria)
      - name: settlement-execution-gate
        paths:
          - /api/v1/settlement
        methods:
          - POST
        protocols:
          - https
        plugins:
          - name: request-termination
            enabled: true # Cambiar a false solo cuando production_authorization = true
            config:
              status_code: 503
              content_type: "application/json; charset=utf-8"
              body: >
                {
                  "node_ref": "1045668717-MS-ONE",
                  "checkpoint": "13,637-ACK",
                  "status": "PENDING_CRYPTOGRAPHIC_VERIFICATION",
                  "reason": "AWAITING_BANK_CERTIFICATION_HASH"
                }
          - name: mtls-auth
            config:
              ca_certificates:
                - "<UUID_CERTIFICADO_CA_JAMG>"
              revocation_check_mode: "STRICT"
          - name: acl
            config:
              allow:
                - supervisor-sod-verified
              hide_groups_header: true

      # Ruta de auditoría y telemetría (Solo lectura)
      - name: audit-checkpoint-read
        paths:
          - /api/v1/checkpoint/13637-ack
        methods:
          - GET
        protocols:
          - https
        plugins:
          - name: rate-limiting
            config:
              minute: 30
              policy: local
          - name: response-transformer
            config:
              add:
                headers:
                  - "X-Node-Ref: 1045668717-MS-ONE"
                  - "X-Governance: JAMG-HUMAN-ETHICS-FIRST"
# /etc/nginx/conf.d/jamg-fintech-os.conf

# Interruptor maestro de gobernanza (0 = Bloqueado / Fail-Closed, 1 = Habilitado tras verificación)
map $host $jamg_production_auth {
    default 0; # Corresponde a "production_authorization": false
}

# Formato de auditoría inmutable con trazabilidad del nodo
log_format jamg_audit_json escape=json
  '{"timestamp":"$time_iso8601",'
  '"node_ref":"1045668717-MS-ONE",'
  '"checkpoint":"13,637-ACK",'
  '"external_ref":"080016001067202519619",'
  '"client_ip":"$remote_addr",'
  '"mtls_verify":"$ssl_client_verify",'
  '"method":"$request_method",'
  '"uri":"$request_uri",'
  '"status":$status}';

server {
    listen 443 ssl http2;
    server_name api.jamgvct.co gateway.jamgvct.co;

    # Cifrado estricto TLS 1.3
    ssl_certificate     /etc/ssl/jamg/fullchain.pem;
    ssl_certificate_key /etc/ssl/jamg/privkey.pem;
    ssl_protocols       TLSv1.3;
    ssl_prefer_server_ciphers off;

    # Configuración mTLS para autenticación mutua
    ssl_client_certificate /etc/ssl/jamg/ca-client-trust.pem;
    ssl_verify_client      optional;

    # Cabeceras de seguridad y trazabilidad
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header Content-Security-Policy "default-src 'none'; frame-ancestors 'none';" always;
    add_header X-Node-Ref "1045668717-MS-ONE" always;
    add_header X-Governance-Policy "HUMAN_ETHICS_FIRST" always;

    access_log /var/log/nginx/jamg_gateway_audit.log jamg_audit_json;

    # Compuerta Fail-Closed para liquidación y pagos ISO 20022
    location /api/v1/settlement/ {
        if ($jamg_production_auth = 0) {
            add_header Content-Type application/json always;
            return 503 '{"error":"FAIL_CLOSED_ENFORCED","state":"PENDING_CRYPTOGRAPHIC_VERIFICATION","checkpoint":"13,637-ACK"}';
        }

        # Exigir mTLS válido si la producción se llegara a habilitar
        if ($ssl_client_verify != SUCCESS) {
            return 403 '{"error":"MTLS_CLIENT_CERT_REQUIRED"}';
        }

        proxy_pass http://127.0.0.1:3000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Node-Ref "1045668717-MS-ONE";
    }

    # Endpoints de solo lectura / auditoría pública del manifiesto
    location /api/v1/status {
        limit_except GET { deny all; }
        proxy_pass http://127.0.0.1:3000;
    }
}


{
  "name": "JAMG VCT · JAMG-FINTECH-OS",
  "short_name": "JAMG OS",
  "description": "Nodo Soberano Post-Quantum · ISO 20022 & Blockchain Base COP",
  "start_url": "./index.html",
  "scope": "./",
  "display": "standalone",
  "background_color": "#07101b",
  "theme_color": "#07101b",
  "orientation": "portrait",
  "icons": [
    {
      "src": "data:image/svg+xml;utf8,<svg xmlns='http://w3.org' viewBox='0 0 512 512'><rect width='512' height='512' rx='110' fill='%2307101b'/><circle cx='256' cy='256' r='190' fill='none' stroke='%232dd4bf' stroke-width='20'/><text x='50%' y='56%' dominant-baseline='middle' text-anchor='middle' font-family='sans-serif' font-weight='900' font-size='210' fill='%2338bdf8'>℣</text></svg>",
      "sizes": "192x192 512x512",
      "type": "image/svg+xml",
      "purpose": "any maskable"
    }
  ]
}

const CACHE_NAME = 'jamg-fintech-v2';
const urlsToCache = [
  './',
  './index.html',
  './manifest.json'
];

self.addEventListener('install', event => {
  self.skipWaiting();
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(urlsToCache))
  );
});

self.addEventListener('activate', event => {
  event.waitUntil(clients.claim());
});

self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request).then(response => response || fetch(event.request))
  );
});

<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <meta name="theme-color" content="#07101b" />
  <meta name="mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
  <title>JAMG VCT · JAMG-FINTECH-OS</title>

  <link rel="manifest" href="manifest.json" />
  <link rel="icon" type="image/svg+xml" href="data:image/svg+xml;utf8,<svg xmlns='http://w3.org' viewBox='0 0 100 100'><rect width='100' height='100' rx='22' fill='%230284c7'/><text x='50%' y='56%' dominant-baseline='middle' text-anchor='middle' font-family='sans-serif' font-weight='900' font-size='56' fill='%23041019'>℣</text></svg>" />

  <style>
    :root {
      --bg-dark: #07101b;
      --bg-panel: #0b1626;
      --bg-card: #0e1c30;
      --bg-code: #050b14;
      --border: #1b314d;
      --text-main: #f8fafc;
      --text-muted: #8fa4bd;
      --cyan: #38bdf8;
      --cyan-bg: rgba(56, 189, 248, 0.14);
      --teal: #2dd4bf;
      --teal-bg: rgba(45, 212, 191, 0.14);
      --amber: #fbbf24;
      --amber-bg: rgba(251, 191, 36, 0.14);
      --red: #f87171;
      --red-bg: rgba(248, 113, 113, 0.15);
      --purple: #c084fc;
      --purple-bg: rgba(192, 132, 252, 0.14);
    }

    * { box-sizing: border-box; margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; -webkit-tap-highlight-color: transparent; }
    body { background-color: var(--bg-dark); color: var(--text-main); line-height: 1.45; padding-bottom: 32px; }
    .app-container { max-width: 520px; margin: 0 auto; padding: 16px; }
    .gov-header { border-bottom: 1px solid var(--border); padding-bottom: 16px; margin-bottom: 18px; }
    .brand-title-row { display: flex; align-items: center; justify-content: space-between; margin-bottom: 4px; }
    .brand-left { display: flex; align-items: center; gap: 10px; }
    .trident-logo { width: 30px; height: 30px; border-radius: 8px; background: linear-gradient(135deg, #0284c7, #2dd4bf); display: flex; align-items: center; justify-content: center; font-weight: 900; font-size: 17px; color: #041019; }
    .brand-title { font-size: 16px; font-weight: 800; letter-spacing: 0.4px; }
    .brand-location { font-size: 12px; color: var(--text-muted); margin-bottom: 14px; }
    .top-tabs { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 14px; }
    .tab-pill { background: var(--bg-card); border: 1px solid var(--border); color: var(--text-muted); font-size: 12px; font-weight: 600; padding: 8px 12px; border-radius: 10px; cursor: pointer; }
    .tab-pill.active { background: rgba(14, 165, 233, 0.16); border-color: var(--cyan); color: var(--cyan); }
    .pqc-pill { display: inline-flex; align-items: center; gap: 8px; background: var(--bg-panel); border: 1px solid var(--border); border-radius: 999px; padding: 6px 14px; font-family: "Courier New", monospace; font-size: 11.5px; color: var(--text-muted); }
    .dot-live { width: 8px; height: 8px; border-radius: 50%; background-color: var(--teal); box-shadow: 0 0 8px var(--teal); animation: pulse 2s infinite; }
    @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }
    .module-view { display: none; }
    .module-view.active { display: block; }
    .panel-card { background: var(--bg-panel); border: 1px solid var(--border); border-radius: 16px; padding: 18px; margin-bottom: 16px; }
    .panel-card.highlight-cyan { border-color: rgba(56, 189, 248, 0.45); background: linear-gradient(180deg, #0d1d33 0%, #091322 100%); }
    .eyebrow { font-size: 11px; font-weight: 800; color: var(--cyan); text-transform: uppercase; letter-spacing: 0.6px; margin-bottom: 8px; }
    .hero-h2 { font-size: 18px; font-weight: 800; margin-bottom: 14px; line-height: 1.3; }
    .spec-strip { background: var(--bg-code); border: 1px solid var(--border); border-radius: 10px; padding: 10px 12px; font-family: "Courier New", monospace; font-size: 12px; color: #e2e8f0; margin-bottom: 8px; }
    .pqc-metrics { margin-top: 14px; padding-top: 12px; border-top: 1px solid var(--border); font-family: "Courier New", monospace; font-size: 12px; color: var(--text-muted); display: grid; gap: 4px; }
    .pqc-metrics span { color: var(--teal); font-weight: 700; }
    .sub-nav { display: grid; grid-template-columns: repeat(4, 1fr); gap: 6px; background: var(--bg-panel); border: 1px solid var(--border); border-radius: 12px; padding: 6px; margin-bottom: 16px; }
    .sub-btn { background: transparent; border: none; color: var(--text-muted); font-size: 11px; font-weight: 700; padding: 8px 4px; border-radius: 8px; cursor: pointer; }
    .sub-btn.active { background: var(--teal-bg); color: var(--teal); }
    .fintech-subview { display: none; }
    .fintech-subview.active { display: block; }
    .btn-action { background: #5eead4; color: #042f2e; border: none; border-radius: 10px; padding: 10px 16px; font-size: 13px; font-weight: 800; cursor: pointer; }
    .btn-apk { background: linear-gradient(135deg, #38bdf8, #2dd4bf); color: #041019; border: none; border-radius: 8px; padding: 6px 11px; font-size: 11px; font-weight: 800; cursor: pointer; }
    .btn-outline { background: transparent; border: 1px solid var(--border); color: var(--text-muted); border-radius: 8px; padding: 6px 10px; font-size: 11px; font-weight: 600; cursor: pointer; }
    .search-box { width: 100%; background: var(--bg-card); border: 1px solid var(--border); border-radius: 10px; padding: 10px 12px; color: var(--text-main); font-size: 13px; margin-bottom: 14px; outline: none; }
    .kpi-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 16px; }
    .kpi-box { background: var(--bg-card); border: 1px solid var(--border); border-radius: 12px; padding: 12px; }
    .kpi-title { font-size: 10px; font-weight: 700; color: var(--text-muted); text-transform: uppercase; margin-bottom: 6px; }
    .kpi-num { font-size: 20px; font-weight: 800; margin-bottom: 4px; }
    .kpi-foot { font-size: 11px; color: var(--teal); }
    .bars-wrap { display: flex; align-items: flex-end; justify-content: space-around; height: 130px; padding-top: 16px; border-bottom: 1px solid var(--border); }
    .bar-item { width: 16%; height: 100%; display: flex; flex-direction: column; justify-content: flex-end; align-items: center; }
    .bar-fill { width: 100%; max-width: 30px; background: linear-gradient(180deg, #60a5fa, #818cf8); border-radius: 5px 5px 0 0; min-height: 4px; }
    .bar-fill.alt { background: linear-gradient(180deg, #c084fc, #9333ea); }
    .bar-lbl { font-size: 11px; color: var(--text-muted); margin-top: 6px; font-weight: 700; }
    .table-wrap { width: 100%; overflow-x: auto; }
    table { width: 100%; min-width: 450px; border-collapse: collapse; font-size: 12px; }
    th { text-align: left; font-size: 10px; color: var(--text-muted); text-transform: uppercase; padding: 8px 6px; border-bottom: 1px solid var(--border); }
    td { padding: 9px 6px; border-bottom: 1px solid rgba(27, 49, 77, 0.5); white-space: nowrap; }
    .tag-pill { font-size: 10px; font-weight: 800; padding: 3px 7px; border-radius: 6px; text-transform: uppercase; }
    .tag-ok { background: var(--teal-bg); color: var(--teal); }
    .tag-fail { background: var(--red-bg); color: var(--red); }
    .tag-pend { background: var(--amber-bg); color: var(--amber); }
    .tag-crypto { background: var(--purple-bg); color: var(--purple); }
    .tag-fiat { background: var(--cyan-bg); color: var(--cyan); }
    .field { margin-bottom: 12px; }
    .field label { display: block; font-size: 11.5px; color: var(--text-muted); margin-bottom: 5px; }
    .field input, .field select { width: 100%; background: var(--bg-card); border: 1px solid var(--border); border-radius: 9px; padding: 10px; color: var(--text-main); font-size: 13px; }
    .filter-pills { display: flex; gap: 6px; margin-bottom: 12px; flex-wrap: wrap; }
    .filter-chip { background: var(--bg-card); border: 1px solid var(--border); color: var(--text-muted); font-size: 11px; font-weight: 700; padding: 5px 10px; border-radius: 999px; cursor: pointer; }
    .filter-chip.active { background: var(--cyan-bg); border-color: var(--cyan); color: var(--cyan); }
    .result-box { background: var(--bg-code); border: 1px solid var(--border); border-radius: 10px; padding: 12px 14px; margin-top: 12px; font-family: "Courier New", monospace; font-size: 13px; }
    .result-box .value { color: var(--teal); font-weight: 800; font-size: 17px; margin-top: 6px; }
    .status-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 6px; font-size: 11.5px; color: var(--text-muted); }
    .arch-scroll { overflow-x: auto; background: var(--bg-code); border: 1px solid var(--border); border-radius: 12px; padding: 16px; margin-top: 12px; }
    .arch-canvas { min-width: 680px; display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
    .arch-col { display: flex; flex-direction: column; gap: 12px; }
    .arch-node { background: var(--bg-card); border: 1px solid var(--border); border-radius: 10px; padding: 10px; font-size: 11.5px; }
    .arch-node.blue { border-color: var(--cyan); }
    .arch-node.amber { border-color: var(--amber); }
    .arch-node.purple { border-color: var(--purple); }
    .arch-node.teal { border-color: var(--teal); }
    .arch-node strong { display: block; margin-bottom: 4px; font-size: 12px; }
    .arch-node small { color: var(--text-muted); font-size: 10.5px; }
    .muted { color: var(--text-muted); font-size: 12.5px; }
    .flex-between { display: flex; justify-content: space-between; align-items: center; gap: 10px; flex-wrap: wrap; }
    .mt-12 { margin-top: 12px; }
    .mb-8 { margin-bottom: 8px; }
  </style>
</head>


jamgvct.co {
    root * /var/www/jamgvct-landing

    handle /manifest.webmanifest {
        header Content-Type "application/manifest+json"
        header Cache-Control "public, max-age=3600"
        file_server
    }

    handle /service-worker.js {
        header Cache-Control "no-cache"
        file_server
    }

    handle /assets/icon-192.png {
        header Cache-Control "public, max-age=31536000"
        file_server
    }

    handle /assets/icon-512.png {
        header Cache-Control "public, max-age=31536000"
        file_server
    }

    handle {
        try_files {path} {path}.html {path}/ /index.html
        file_server
    }
}
sudo caddy validate \
  --config /etc/caddy/Caddyfile

sudo systemctl reload caddy

sudo systemctl status caddy --no-pager

for path in \
  / \
  /manifest.webmanifest \
  /service-worker.js \
  /assets/icon-192.png \
  /assets/icon-512.png
do
  echo "===== $path ====="
  curl -sS -o /dev/null -D - \
    "https://jamgvct.co$path" |
    grep -Ei 'HTTP/|content-type:|cache-control:'
done