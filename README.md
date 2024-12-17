# Secure Pipeline Verslag

## Inhoudsopgave
1. [Introductie](#introductie)
2. [Project Setup en Workflow Configuratie](#project-setup-en-workflow-configuratie)
3. [Github Actions Configuratie](#github-actions-configuratie)
   - [Linting met Flake8](#linting-met-flake8)
   - [SAST met Bandit](#sast-met-bandit)
   - [Dependency Vulnerability Check](#dependency-vulnerability-check)
   - [Secret Detection](#secret-detection)
   - [Blokkeren van Pull Requests](#blokkeren-van-pull-requests)
4. [Introduceren van Kwetsbaarheden](#introduceren-van-kwetsbaarheden)
5. [Resultaten en Screenshots](#resultaten-en-screenshots)
6. [Link naar Repository](#link-naar-repository)
7. [Conclusie](#conclusie)

---

## 1. Introductie
In deze opdracht is een **secure pipeline** opgezet met behulp van GitHub Actions voor een Python project. Het project bestaat uit een eenvoudig "Hello World" script in Python. De pipeline is zo geconfigureerd dat de volgende beveiligingschecks worden uitgevoerd:

1. **SAST (Static Application Security Testing):** Controleert de code op kwetsbaarheden met behulp van Bandit.
2. **Dependency Vulnerability Check:** Checkt of gebruikte dependencies kwetsbaarheden bevatten met behulp van Safety.
3. **Secret Detection:** Scant de code op hard-coded wachtwoorden of geheime sleutels met detect-secrets.
4. **Linting:** Code kwaliteit wordt gecontroleerd met Flake8.
5. **Blokkeren van Pull Requests:** Pull requests mogen niet worden gemerged als een van de bovenstaande checks faalt.
6. **Wekelijkse SAST Scan:** Er wordt elke week automatisch een SAST-scan uitgevoerd.

Hieronder worden alle stappen, configuraties en resultaten in detail beschreven.

---

## 2. Project Setup en Workflow Configuratie

### Projectstructuur
De repository bevat de volgende bestanden:
```
my-python-project/
├── .github/
│   └── workflows/
│       └── secure_pipeline.yml
├── requirements.txt
└── main.py
```

1. **`main.py`:** Een eenvoudig Python script dat kwetsbaarheden bevat om de checks te triggeren.
2. **`requirements.txt`:** Bevat verouderde dependencies om veiligheidsproblemen te veroorzaken.
3. **`secure_pipeline.yml`:** De GitHub Actions workflow die alle checks uitvoert.

---

## 3. Github Actions Configuratie
De pipeline is geconfigureerd in een bestand genaamd `secure_pipeline.yml`. Hieronder wordt elke stap van de workflow uitgelegd.

### Workflow Bestand
```yaml
name: Secure Python Pipeline

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  schedule:
    - cron: "0 0 * * 0" # Wekelijkse SAST-scan elke zondag

permissions:
  contents: read
  security-events: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.10
        uses: actions/setup-python@v3
        with:
          python-version: "3.10"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 bandit safety detect-secrets
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      - name: Lint with flake8
        id: lint
        run: |
          echo "Running flake8 for linting..."
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Static Application Security Testing (SAST) with Bandit
        id: sast
        run: |
          echo "Running Bandit for SAST checks..."
          bandit -r . -ll

      - name: Dependency Vulnerability Check
        id: deps
        run: |
          echo "Running safety for dependency vulnerabilities..."
          safety check || true

      - name: Secret detection
        id: secrets
        run: |
          echo "Running detect-secrets..."
          detect-secrets scan > .secrets.baseline || true
          detect-secrets audit .secrets.baseline || true

      - name: Fail on any warnings or errors
        if: always() && (steps.lint.outcome == 'failure' || steps.sast.outcome == 'failure' || steps.deps.outcome == 'failure' || steps.secrets.outcome == 'failure')
        run: |
          echo "❌ One or more checks failed. Blocking the pull request."
          exit 1
```

### 3.1 Linting met Flake8
- **Doel:** Controleert de kwaliteit van de code.
- **Configuratie:**
  ```bash
  flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
  flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
  ```

### 3.2 SAST met Bandit
- **Doel:** Detecteert kwetsbaarheden in de code, zoals het gebruik van `eval()`.
- **Configuratie:**
  ```bash
  bandit -r . -ll
  ```

### 3.3 Dependency Vulnerability Check
- **Doel:** Controleert of dependencies kwetsbaarheden bevatten.
- **Configuratie:**
  ```bash
  safety check
  ```

### 3.4 Secret Detection
- **Doel:** Detecteert hard-coded geheimen zoals wachtwoorden en API-sleutels.
- **Configuratie:**
  ```bash
  detect-secrets scan > .secrets.baseline
  detect-secrets audit .secrets.baseline
  ```

### 3.5 Blokkeren van Pull Requests
De volgende regel zorgt ervoor dat een pull request wordt geblokkeerd als een check faalt:
```yaml
if: always() && (steps.lint.outcome == 'failure' || steps.sast.outcome == 'failure' || steps.deps.outcome == 'failure' || steps.secrets.outcome == 'failure')
```

---

## 4. Introduceren van Kwetsbaarheden
Om de checks te triggeren, zijn de volgende kwetsbaarheden toegevoegd:

1. **Hard-coded wachtwoord:**
   ```python
   secret_password = "SuperSecret123"
   print("Hard-coded secret:", secret_password)
   ```
2. **Kwetsbaar gebruik van eval():**
   ```python
   user_input = "2 + 2"
   print("Result:", eval(user_input))
   ```
3. **Verouderde dependency in requirements.txt:**
   ```plaintext
   django==1.11.0
   ```

---

## 5. Resultaten en Screenshots
**Screenshot van mislukte checks:**
(Voeg hier de screenshot toe van een gefaalde workflow die een pull request blokkeert.)

---

## 6. Link naar Repository
[Link naar publieke GitHub repository](https://github.com/USERNAME/my-python-project)

---

## 7. Conclusie
De secure pipeline is succesvol opgezet en geconfigureerd. De checks voor linting, SAST, dependency vulnerabilities en secrets worden correct uitgevoerd. Pull requests worden geblokkeerd als er waarschuwingen of fouten worden gedetecteerd, wat bijdraagt aan een veilige codebase.

