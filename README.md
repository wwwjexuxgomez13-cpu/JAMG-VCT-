git clone https://github.com/wwwjexuxgomez13-cpu/jamg-vct-os.git

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

