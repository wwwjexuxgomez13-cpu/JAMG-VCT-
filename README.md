Perfecto, aquí te lo dejo listo para GitHub. Copia y pégalo tal cual como `README.md` en la raíz. Este sí te protege porque dice DEMO en todos lados.
# JAMG VCT TECH — JAMG-FINTECH-OS

> **DEMO UI — Prototipo visual**
> Sin nodo operativo, sin captación, sin liquidación financiera.
> `SETTLEMENT: LOCKED` / `PRODUCTION_AUTHORIZATION: false`

Infraestructura digital, automatización y gobernanza con trazabilidad PI VCT.
Entorno de demostración técnica. Barranquilla, Colombia.

**Sitio DEMO:** https://jamgvct.co

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