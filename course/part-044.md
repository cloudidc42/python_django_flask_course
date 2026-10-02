# Part 044: CI/CD Pipeline
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ CI/CD และประโยชน์
- เขียน GitHub Actions workflow
- รัน tests อัตโนมัติ
- ตั้งค่า linting ด้วย flake8/ruff
- ตรวจสอบ type ด้วย mypy
- Build และ push Docker images
- Deploy อัตโนมัติ

---

## 1. CI/CD คืออะไร?

```
CI (Continuous Integration):
- ทุกครั้งที่ push code → รัน tests, lint, type check อัตโนมัติ
- ตรวจหา bugs เร็วขึ้น
- ทุกคนใน team มั่นใจว่า code ที่ merge ผ่าน tests

CD (Continuous Deployment/Delivery):
- Continuous Delivery: สร้าง build พร้อม deploy (manual trigger)
- Continuous Deployment: deploy อัตโนมัติเมื่อ tests ผ่าน

Benefits:
✅ ค้นพบ bugs เร็ว (หลัง commit ไม่ใช่หลัง release)
✅ Deploy บ่อยขึ้น, risk น้อยลง
✅ Consistent build process
✅ Audit trail ของทุก deployment
```

---

## 2. GitHub Actions Basics

```yaml
# .github/workflows/ci.yml
name: CI

# กำหนดว่าจะ trigger เมื่อไหร่
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  # Manual trigger
  workflow_dispatch:

# Jobs คือชุดของ steps ที่รันบน runner
jobs:
  test:
    # Runner OS
    runs-on: ubuntu-latest
    
    # Matrix: รันหลาย Python versions
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
      fail-fast: false  # อย่าหยุดถ้า 1 version fail
    
    # Steps ใน job
    steps:
      # Check out the code
      - name: Checkout code
        uses: actions/checkout@v4
      
      # ตั้ง Python
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      # Cache pip dependencies
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      
      # ติดตั้ง dependencies
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      # รัน tests
      - name: Run tests
        run: |
          pytest tests/ -v --tb=short
        env:
          DATABASE_URL: sqlite:///test.db
          SECRET_KEY: test-secret-key
```

---

## 3. Full CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop, "feature/**"]
  pull_request:
    branches: [main, develop]

env:
  PYTHON_VERSION: "3.12"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===== Job 1: Lint =====
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install linting tools
        run: |
          pip install ruff flake8 black isort
      
      # ruff (เร็วกว่า flake8 มาก)
      - name: Run ruff
        run: |
          ruff check . --output-format=github
          ruff format --check .
      
      # flake8 (ถ้าใช้แทน ruff)
      - name: Run flake8
        run: |
          flake8 src/ tests/ \
            --max-line-length=88 \
            --exclude=.git,__pycache__,migrations \
            --statistics
        if: false  # Disable ถ้าใช้ ruff แล้ว
      
      # black (code formatter)
      - name: Check black formatting
        run: black --check --diff src/ tests/
        if: false  # Disable ถ้าใช้ ruff แล้ว
      
      # isort (import sorting)
      - name: Check import sorting
        run: isort --check-only --diff src/ tests/
        if: false  # Disable ถ้าใช้ ruff แล้ว
  
  
  # ===== Job 2: Type Check =====
  type-check:
    name: Type Check
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install mypy types-requests types-redis
      
      - name: Run mypy
        run: |
          mypy src/ \
            --ignore-missing-imports \
            --strict \
            --show-error-codes \
            --pretty
  
  
  # ===== Job 3: Test =====
  test:
    name: Test (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    
    # Services (ถ้าต้องการ)
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
    
    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379/0
      SECRET_KEY: ${{ secrets.TEST_SECRET_KEY || 'test-secret-key-for-ci-pipeline' }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-asyncio pytest-xdist
      
      - name: Run database migrations
        run: |
          python manage.py migrate
        if: false  # สำหรับ Django projects
      
      - name: Run tests with coverage
        run: |
          pytest tests/ \
            -v \
            --tb=short \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -n auto  # parallel tests
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          flags: ${{ matrix.python-version }}
          token: ${{ secrets.CODECOV_TOKEN }}
  
  
  # ===== Job 4: Security Check =====
  security:
    name: Security Check
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install security tools
        run: |
          pip install bandit safety
      
      # bandit: ตรวจหา security issues ใน Python code
      - name: Run bandit
        run: |
          bandit -r src/ \
            -f json \
            -o bandit-report.json \
            --severity-level medium
        continue-on-error: true
      
      # safety: ตรวจสอบ dependencies ที่มี vulnerabilities
      - name: Check dependencies with safety
        run: |
          safety check --json -o safety-report.json
        continue-on-error: true
      
      - name: Upload security reports
        uses: actions/upload-artifact@v4
        with:
          name: security-reports
          path: |
            bandit-report.json
            safety-report.json
```

---

## 4. Docker Build และ Push

```yaml
# .github/workflows/docker.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]  # Semantic versioning tags

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  docker:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    
    # ต้องการ permissions สำหรับ write packages
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      # ตั้ง QEMU สำหรับ multi-platform builds
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      # ตั้ง Docker Buildx
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      # Login to GitHub Container Registry
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      # Extract metadata (tags, labels)
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
      
      # Build and push Docker image
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          # Build cache
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 5. Dockerfile สำหรับ Python App

```dockerfile
# Dockerfile
# Multi-stage build สำหรับ production

# === Stage 1: Builder ===
FROM python:3.12-slim as builder

WORKDIR /build

# ติดตั้ง build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements ก่อน (Docker cache)
COPY requirements.txt .

# สร้าง virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# ติดตั้ง dependencies
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt


# === Stage 2: Production ===
FROM python:3.12-slim as production

# Non-root user สำหรับ security
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# ติดตั้ง runtime dependencies เท่านั้น
RUN apt-get update && apt-get install -y \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Copy virtual environment จาก builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=8000

EXPOSE ${PORT}

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:${PORT}/health')" \
    || exit 1

# Entrypoint
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

---

## 6. Deployment Workflows

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  # === Deploy to Staging ===
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/myapp
            git pull origin main
            docker compose pull
            docker compose up -d --no-build
            docker compose exec web python manage.py migrate
            echo "Deployed to staging!"
  
  
  # === Deploy to Production (Manual Approval) ===
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging  # ต้อง deploy staging ก่อน
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
      
      # ===  Blue-Green Deployment ===
      - name: Blue-Green Deploy
        run: |
          # เปลี่ยน traffic ไป "green" environment
          # หลัง deploy สำเร็จ
          echo "Blue-Green deployment complete"
      
      # === Notify on success ===
      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deployed to production successfully!"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      # === Notify on failure ===
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Production deployment failed! Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 7. Tools Configuration

```toml
# pyproject.toml - รวม config ทุกอย่างไว้ที่เดียว

[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "N",   # pep8-naming
]
ignore = ["E501"]  # line too long (handled by formatter)

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]  # assert ok in tests

[tool.ruff.format]
quote-style = "double"
indent-style = "space"


[tool.mypy]
python_version = "3.12"
strict = true
ignore_missing_imports = true
pretty = true


[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",
    "--tb=short",
    "--strict-markers",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]


[tool.coverage.run]
source = ["src"]
omit = ["*/migrations/*", "*/tests/*", "*/conftest.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise NotImplementedError",
    "if TYPE_CHECKING:",
]
show_missing = true
fail_under = 80
```

---

## 8. Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
# pip install pre-commit
# pre-commit install

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-json
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ["--maxkb=1000"]
      - id: detect-private-key
      - id: debug-statements
  
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
  
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.8
    hooks:
      - id: bandit
        args: ["-ll"]
        files: .py$
        exclude: tests/
```

---

## 9. Complete Example Project Structure

```
myproject/
├── .github/
│   └── workflows/
│       ├── ci.yml          # Tests + Lint + Type check
│       ├── docker.yml      # Build + Push Docker
│       └── deploy.yml      # Deploy to environments
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── main.py
│       └── models.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── .pre-commit-config.yaml
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── requirements.txt
└── requirements-dev.txt
```

```ini
# requirements-dev.txt
-r requirements.txt

# Testing
pytest>=8.0
pytest-cov>=5.0
pytest-asyncio>=0.23
pytest-xdist>=3.5
faker>=24.0

# Linting
ruff>=0.3
mypy>=1.9
bandit>=1.7

# Pre-commit
pre-commit>=3.7
```

---

## 10. สรุป Part 044

✅ **CI/CD Concepts** - ประโยชน์ของ automation  
✅ **GitHub Actions** - workflows, jobs, steps, matrix  
✅ **Linting** - ruff, flake8 ใน CI pipeline  
✅ **Type Checking** - mypy strict mode  
✅ **Testing** - pytest, coverage, parallel tests  
✅ **Security** - bandit, safety checks  
✅ **Docker** - multi-stage builds, push to registry  
✅ **Deployment** - staging → production, manual approval  
✅ **Pre-commit** - ตรวจสอบ code ก่อน commit  
✅ **pyproject.toml** - รวม tools configuration  

**Best Practices:**
- ทุก PR ต้องผ่าน CI ก่อน merge
- แยก dev/staging/production environments
- ใช้ secrets สำหรับ sensitive values
- Cache dependencies ลดเวลา build
- Notify เมื่อ deploy ล้มเหลว

## ➡️ ถัดไป: Part 045 - Security Basics
*Part 044/100+ | Python Course - Beginner to World-Class*
