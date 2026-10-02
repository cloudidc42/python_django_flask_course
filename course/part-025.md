# Part 025 - Virtual Environments and Package Management

## เป้าหมาย
- สร้างและจัดการ Virtual Environments ด้วย `venv`
- ใช้ `pip` อย่างมีประสิทธิภาพ
- จัดการ dependencies ด้วย `requirements.txt`
- ใช้ `pip-tools` สำหรับ deterministic builds
- ใช้ `poetry` สำหรับ modern dependency management
- เข้าใจ `conda` และ `pyenv`
- Best practices สำหรับ dependency management

---

## 1. Virtual Environments (venv)

Virtual environment คือ Python environment ที่แยกออกจาก system Python

```bash
# === สร้าง Virtual Environment ===

# สร้าง venv ชื่อ "venv" ในโฟลเดอร์ปัจจุบัน
python -m venv venv

# สร้างชื่ออื่น
python -m venv myenv
python -m venv .venv  # นิยมขึ้นต้นด้วย . เพื่อซ่อน

# สร้างโดยระบุ Python version
python3.11 -m venv venv
/usr/bin/python3.11 -m venv venv

# === Activate ===

# Linux/Mac
source venv/bin/activate

# Windows Command Prompt
venv\Scripts\activate.bat

# Windows PowerShell
venv\Scripts\Activate.ps1

# Fish shell
source venv/bin/activate.fish

# === ตรวจสอบ ===
which python   # ควรชี้ไปที่ venv
python --version
pip --version

# === Deactivate ===
deactivate

# === ลบ venv ===
# ลบโฟลเดอร์ venv ทิ้ง
rm -rf venv
```

```python
# Python script สำหรับ manage venv
import subprocess
import sys
import os
from pathlib import Path

def create_venv(venv_path: str = "venv") -> Path:
    """สร้าง virtual environment"""
    venv_dir = Path(venv_path)
    
    if venv_dir.exists():
        print(f"Virtual environment already exists at {venv_dir}")
        return venv_dir
    
    print(f"Creating virtual environment at {venv_dir}")
    subprocess.run([sys.executable, "-m", "venv", str(venv_dir)], 
                   check=True)
    print("Virtual environment created successfully!")
    return venv_dir


def get_venv_python(venv_path: str = "venv") -> str:
    """หา Python executable ใน venv"""
    venv_dir = Path(venv_path)
    
    if os.name == 'nt':  # Windows
        return str(venv_dir / "Scripts" / "python.exe")
    else:  # Linux/Mac
        return str(venv_dir / "bin" / "python")


def is_in_venv() -> bool:
    """ตรวจสอบว่าอยู่ใน virtual environment หรือไม่"""
    return sys.prefix != sys.base_prefix


def get_venv_info() -> dict:
    """ข้อมูลของ virtual environment ปัจจุบัน"""
    return {
        "in_venv": is_in_venv(),
        "python_executable": sys.executable,
        "python_version": sys.version,
        "venv_prefix": sys.prefix if is_in_venv() else None,
        "site_packages": [p for p in sys.path if "site-packages" in p]
    }


# ตรวจสอบ
info = get_venv_info()
print(f"In virtual environment: {info['in_venv']}")
print(f"Python: {info['python_executable']}")
if info['in_venv']:
    print(f"Venv location: {info['venv_prefix']}")
```

---

## 2. pip - Package Installer

```bash
# === Basic pip Commands ===

# ติดตั้ง package
pip install requests
pip install "requests>=2.28"          # version constraint
pip install "requests>=2.28,<3.0"    # version range
pip install "requests==2.28.1"       # exact version
pip install requests[security]        # with extras
pip install requests[security,socks]  # multiple extras

# ติดตั้งหลาย packages พร้อมกัน
pip install requests flask sqlalchemy

# อัปเดต package
pip install --upgrade requests
pip install --upgrade pip  # อัปเดต pip เอง

# ลบ package
pip uninstall requests
pip uninstall requests flask  # ลบหลายตัว
pip uninstall -y requests     # ไม่ถาม confirm

# แสดง packages ที่ติดตั้ง
pip list
pip list --outdated      # แสดงที่มี update
pip list --format=freeze # รูปแบบ requirements.txt

# ข้อมูล package
pip show requests
pip show --verbose requests

# ค้นหา packages
pip search django  # deprecated ใน pip 21+

# ติดตั้งจาก source
pip install .           # จาก setup.py/pyproject.toml ในโฟลเดอร์นี้
pip install -e .        # editable install (development mode)
pip install -e ".[dev]" # พร้อม dev extras

# ติดตั้งจาก git
pip install git+https://github.com/user/repo.git
pip install git+https://github.com/user/repo.git@branch
pip install git+https://github.com/user/repo.git@v1.0

# ติดตั้งจาก local path
pip install /path/to/package
pip install /path/to/package.whl

# ===  Cache ===
pip cache list    # ดู cache
pip cache purge   # ล้าง cache
pip install --no-cache-dir requests  # ไม่ใช้ cache

# === Private Repository ===
pip install --index-url https://my-pypi.example.com/simple/ mypackage
pip install --extra-index-url https://my-pypi.example.com/simple/ mypackage
```

### pip Configuration

```bash
# pip.conf / pip.ini configuration

# Linux/Mac: ~/.config/pip/pip.conf
# Windows: %APPDATA%\pip\pip.ini

# เนื้อหา:
# [global]
# index-url = https://pypi.org/simple/
# trusted-host = pypi.org
# timeout = 60
# no-cache-dir = false

# สำหรับ corporate proxy
# [global]
# proxy = http://proxy.company.com:8080
# trusted-host = pypi.org
#                files.pythonhosted.org

# ตั้ง pip config จาก command line
pip config set global.index-url https://pypi.org/simple/
pip config list  # ดู config ปัจจุบัน
```

---

## 3. requirements.txt

```bash
# === สร้าง requirements.txt ===

# Export packages ที่ติดตั้งอยู่
pip freeze > requirements.txt

# ตัวอย่าง requirements.txt
```

```text
# requirements.txt
# Web Framework
Django>=4.2,<5.0
djangorestframework==3.14.0
django-cors-headers>=3.14

# Database
psycopg2-binary>=2.9
redis>=4.6

# HTTP Client
requests>=2.31
httpx>=0.25

# Utils
python-dotenv>=1.0
celery>=5.3
Pillow>=10.0

# Type hints (Python < 3.11)
typing_extensions>=4.7

# Testing
pytest>=7.4
pytest-django>=4.7
factory-boy>=3.3
coverage>=7.3
```

```bash
# === ติดตั้งจาก requirements.txt ===
pip install -r requirements.txt

# ติดตั้งในโหมด verbose
pip install -v -r requirements.txt

# ติดตั้งหลายไฟล์
pip install -r requirements.txt -r requirements-dev.txt
```

### Multiple Requirements Files

```text
# requirements/base.txt - dependencies ที่ใช้ทุก environment

Django>=4.2
psycopg2-binary>=2.9
celery>=5.3
```

```text
# requirements/development.txt - เพิ่มสำหรับ development

-r base.txt

# Testing
pytest>=7.4
pytest-cov>=4.1
factory-boy>=3.3

# Debugging
ipython>=8.14
django-debug-toolbar>=4.2
```

```text
# requirements/production.txt - เพิ่มสำหรับ production

-r base.txt

# Performance
gunicorn>=21.2
whitenoise>=6.6
```

```bash
# ใช้ตาม environment
pip install -r requirements/development.txt
pip install -r requirements/production.txt
```

---

## 4. pip-tools

pip-tools แก้ปัญหา requirements ที่ไม่ consistent

```bash
# ติดตั้ง pip-tools
pip install pip-tools
```

```text
# requirements.in - high-level dependencies เท่านั้น

Django
psycopg2-binary
celery[redis]
requests
```

```bash
# Generate requirements.txt จาก requirements.in
pip-compile requirements.in

# Update ทุก packages เป็น latest
pip-compile --upgrade requirements.in

# Update เฉพาะ package เดียว
pip-compile --upgrade-package Django requirements.in

# ผลลัพธ์ requirements.txt จะมี:
# Django==4.2.7
#   asgiref==3.7.2  # dependency ของ Django
#   sqlparse==0.4.4
# psycopg2-binary==2.9.9
# celery==5.3.4
#   ...

# Sync environment ให้ตรงกับ requirements.txt
pip-sync requirements.txt

# Sync หลายไฟล์
pip-sync requirements.txt requirements-dev.txt
```

```python
# Script สำหรับ automate pip-tools workflow
import subprocess
import sys
from pathlib import Path

def compile_requirements(input_file: str, output_file: str = None,
                          upgrade: bool = False):
    """Compile requirements"""
    cmd = ["pip-compile"]
    if upgrade:
        cmd.append("--upgrade")
    if output_file:
        cmd.extend(["--output-file", output_file])
    cmd.append(input_file)
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode == 0:
        print(f"Compiled {input_file} -> {output_file or 'auto'}")
    else:
        print(f"Error: {result.stderr}")
    return result.returncode == 0


def sync_environment(requirements_files: list):
    """Sync environment"""
    cmd = ["pip-sync"] + requirements_files
    result = subprocess.run(cmd, capture_output=True, text=True)
    print(result.stdout)
    if result.returncode != 0:
        print(f"Error: {result.stderr}")
    return result.returncode == 0
```

---

## 5. Poetry - Modern Dependency Management

```bash
# ติดตั้ง Poetry
curl -sSL https://install.python-poetry.org | python3 -
# หรือ
pip install poetry

# ตรวจสอบ
poetry --version
```

### pyproject.toml

```toml
# pyproject.toml - ไฟล์หลักของ Poetry

[tool.poetry]
name = "my-project"
version = "0.1.0"
description = "My awesome Python project"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
license = "MIT"
homepage = "https://example.com"
repository = "https://github.com/user/my-project"

[tool.poetry.dependencies]
python = "^3.10"
Django = "^4.2"
psycopg2-binary = "^2.9"
celery = {version = "^5.3", extras = ["redis"]}
requests = "^2.31"
python-dotenv = "^1.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4"
pytest-django = "^4.7"
black = "^23.0"
isort = "^5.12"
mypy = "^1.5"
factory-boy = "^3.3"

[tool.poetry.group.prod.dependencies]
gunicorn = "^21.2"
whitenoise = "^6.6"

[tool.poetry.scripts]
start = "myproject.main:main"
migrate = "myproject.manage:migrate"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry-core.masonry.api"

# Black configuration
[tool.black]
line-length = 88
target-version = ['py310', 'py311']
include = '\.pyi?$'

# isort configuration
[tool.isort]
profile = "black"
known_third_party = ["django", "rest_framework"]

# mypy configuration
[tool.mypy]
python_version = "3.10"
strict = true
ignore_missing_imports = true

# pytest configuration
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "myproject.settings"
python_files = ["test_*.py", "*_test.py"]
addopts = "-v --tb=short"
```

```bash
# === Poetry Commands ===

# เริ่ม project ใหม่
poetry new my-project
poetry init  # ใน project ที่มีอยู่แล้ว

# เพิ่ม dependencies
poetry add requests
poetry add "Django>=4.2,<5.0"
poetry add pytest --group dev   # dev dependency
poetry add gunicorn --group prod

# ลบ dependencies
poetry remove requests

# Install all dependencies
poetry install
poetry install --only main      # ติดตั้งแค่ main
poetry install --with dev       # รวม dev group
poetry install --without prod   # ยกเว้น prod

# Update
poetry update           # update ทั้งหมด
poetry update requests  # update เฉพาะ requests

# Show dependencies
poetry show
poetry show --tree      # แสดงเป็น tree

# Check for updates
poetry show --outdated

# Virtual environment
poetry env info         # ข้อมูล venv ปัจจุบัน
poetry env list         # list venv ทั้งหมด
poetry env use python3.11  # ใช้ Python version ต่างๆ
poetry env remove python3.10  # ลบ venv

# Run commands in venv
poetry run python script.py
poetry run pytest
poetry shell  # activate venv

# Build and publish
poetry build                    # build package
poetry publish                  # publish ไป PyPI
poetry publish --repository my-pypi  # private PyPI

# Export
poetry export -f requirements.txt --output requirements.txt
poetry export -f requirements.txt --with dev --output requirements-dev.txt
```

---

## 6. conda

```bash
# conda เหมาะสำหรับ Data Science และ package ที่ต้องการ C libraries

# ดาวน์โหลด Miniconda (แนะนำ)
# https://docs.conda.io/en/latest/miniconda.html

# === Basic Commands ===

# Create environment
conda create -n myenv python=3.11
conda create -n myenv python=3.11 numpy pandas

# Activate/Deactivate
conda activate myenv
conda deactivate

# List environments
conda env list
conda info --envs

# Install packages
conda install numpy
conda install -c conda-forge numpy  # จาก conda-forge channel
conda install numpy=1.24

# List packages
conda list

# Update packages
conda update numpy
conda update --all

# Remove package
conda remove numpy

# Remove environment
conda env remove -n myenv
conda remove --name myenv --all

# === Environment Files ===

# Export environment
conda env export > environment.yml
conda env export --no-builds > environment.yml  # portable

# Create from file
conda env create -f environment.yml
conda env update -f environment.yml  # update existing

# environment.yml ตัวอย่าง
```

```yaml
# environment.yml
name: data-science
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11
  - numpy>=1.24
  - pandas>=2.0
  - matplotlib>=3.7
  - scikit-learn>=1.3
  - jupyter>=1.0
  - pip
  - pip:
    # packages ที่ไม่มีใน conda
    - some-pypi-package>=1.0
```

```bash
# Conda กับ pip
# ควรใช้ conda ก่อน แล้วค่อย pip สำหรับที่ไม่มีใน conda
conda install numpy pandas
pip install some-package-not-in-conda

# === Mamba (Faster conda) ===
conda install mamba -n base -c conda-forge
mamba create -n myenv python=3.11 numpy  # เร็วกว่า conda มาก
```

---

## 7. pyenv - Python Version Manager

```bash
# pyenv ช่วยจัดการ Python หลาย version

# ติดตั้ง pyenv
# Mac
brew install pyenv

# Linux
curl https://pyenv.run | bash

# เพิ่มใน .bashrc / .zshrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

# === Commands ===

# ดู Python versions ที่มี
pyenv install --list
pyenv install --list | grep "3.11"

# ติดตั้ง Python version
pyenv install 3.11.6
pyenv install 3.12.0

# ดู installed versions
pyenv versions

# ตั้ง global default
pyenv global 3.11.6

# ตั้งสำหรับโฟลเดอร์นี้
pyenv local 3.11.6  # สร้าง .python-version file

# ตั้งสำหรับ session นี้
pyenv shell 3.10.0

# ตรวจสอบ version ที่ใช้อยู่
pyenv version
python --version

# ลบ version
pyenv uninstall 3.10.0

# === pyenv กับ venv ===
pyenv local 3.11.6
python -m venv venv  # ใช้ Python 3.11.6
```

### .python-version file

```text
# .python-version
3.11.6
```

---

## 8. Best Practices

### Project Structure

```
my-project/
├── .python-version          # pyenv version
├── .gitignore              
├── pyproject.toml           # Poetry หรือ build config
├── requirements.in          # (pip-tools) high-level deps
├── requirements.txt         # (pip-tools) locked deps
├── requirements-dev.txt     # (pip-tools) dev locked deps
├── src/
│   └── mypackage/
│       ├── __init__.py
│       └── ...
├── tests/
│   └── ...
├── .env.example             # template ของ .env
└── venv/                    # virtual environment (gitignored)
```

```text
# .gitignore
# Virtual Environments
venv/
.venv/
env/
ENV/
.env/

# Python
__pycache__/
*.py[cod]
*.pyo
*.pyd
*.so
*.egg
*.egg-info/
dist/
build/
.eggs/

# Environment variables
.env
.env.local
.env.*.local

# IDEs
.idea/
.vscode/
*.swp

# Test/Coverage
.coverage
htmlcov/
.pytest_cache/
.mypy_cache/
```

### Makefile สำหรับ Automation

```makefile
# Makefile

.PHONY: install install-dev test lint format clean

# ใช้ python จาก venv ถ้ามี
PYTHON := $(shell which python)
PIP := $(shell which pip)

install:
	$(PIP) install -r requirements.txt

install-dev:
	$(PIP) install -r requirements-dev.txt

test:
	pytest tests/ -v --cov=src --cov-report=html

lint:
	flake8 src/ tests/
	mypy src/

format:
	black src/ tests/
	isort src/ tests/

clean:
	find . -type f -name "*.pyc" -delete
	find . -type d -name "__pycache__" -delete
	find . -type d -name "*.egg-info" -exec rm -rf {} +
	rm -rf .coverage htmlcov/ .pytest_cache/

update-requirements:
	pip-compile requirements.in
	pip-compile requirements-dev.in

sync:
	pip-sync requirements.txt requirements-dev.txt
```

### setup.py vs pyproject.toml

```python
# setup.py (classic, ยังใช้ได้แต่ไม่แนะนำสำหรับ project ใหม่)
from setuptools import setup, find_packages

setup(
    name="my-package",
    version="0.1.0",
    author="Your Name",
    author_email="you@example.com",
    description="A short description",
    long_description=open("README.md").read(),
    long_description_content_type="text/markdown",
    url="https://github.com/user/my-package",
    packages=find_packages(where="src"),
    package_dir={"": "src"},
    python_requires=">=3.10",
    install_requires=[
        "requests>=2.28",
        "click>=8.0",
    ],
    extras_require={
        "dev": ["pytest>=7.0", "black>=23.0"],
        "full": ["pandas>=2.0", "numpy>=1.24"],
    },
    entry_points={
        "console_scripts": [
            "my-command=mypackage.cli:main",
        ],
    },
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
)
```

```toml
# pyproject.toml (modern, แนะนำ)
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "my-package"
version = "0.1.0"
authors = [
  {name = "Your Name", email = "you@example.com"},
]
description = "A short description"
readme = "README.md"
requires-python = ">=3.10"
license = {file = "LICENSE"}
classifiers = [
  "Programming Language :: Python :: 3",
  "License :: OSI Approved :: MIT License",
]
dependencies = [
  "requests>=2.28",
  "click>=8.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "black>=23.0"]
full = ["pandas>=2.0", "numpy>=1.24"]

[project.scripts]
my-command = "mypackage.cli:main"

[project.urls]
Homepage = "https://example.com"
Repository = "https://github.com/user/my-package"

[tool.setuptools.packages.find]
where = ["src"]
```

---

## 9. Dependency Security

```bash
# ตรวจสอบ security vulnerabilities
pip install pip-audit
pip-audit
pip-audit -r requirements.txt

# หรือใช้ safety
pip install safety
safety check
safety check -r requirements.txt

# ตรวจสอบ licenses
pip install pip-licenses
pip-licenses
pip-licenses --format=markdown

# pip-audit output ตัวอย่าง
# Found 1 known vulnerability in 1 package
# Name    Version ID             Fix Versions
# ------- ------- -------------- ------------
# pillow  9.0.0   GHSA-xxx-xxx   10.0.0
```

```python
# Python script สำหรับ check packages
import subprocess
import json
import sys

def audit_packages() -> list:
    """ตรวจสอบ security vulnerabilities"""
    try:
        result = subprocess.run(
            ["pip-audit", "--format", "json"],
            capture_output=True,
            text=True
        )
        vulnerabilities = json.loads(result.stdout)
        return vulnerabilities
    except (subprocess.CalledProcessError, json.JSONDecodeError, FileNotFoundError):
        return []


def check_outdated() -> list:
    """ตรวจสอบ packages ที่ outdated"""
    result = subprocess.run(
        ["pip", "list", "--outdated", "--format", "json"],
        capture_output=True,
        text=True
    )
    if result.returncode == 0:
        return json.loads(result.stdout)
    return []


def generate_report():
    """สร้างรายงาน dependencies"""
    print("=== Dependency Report ===\n")
    
    print("Outdated packages:")
    outdated = check_outdated()
    if outdated:
        for pkg in outdated:
            print(f"  {pkg['name']:30} {pkg['version']:15} -> {pkg['latest_version']}")
    else:
        print("  All packages up to date!")
    
    print("\nSecurity vulnerabilities:")
    vulns = audit_packages()
    if vulns:
        for vuln in vulns:
            print(f"  {vuln.get('name', 'Unknown')} {vuln.get('version', '')}: {vuln.get('id', '')}")
    else:
        print("  No vulnerabilities found!")


# generate_report()  # ต้องติดตั้ง pip-audit ก่อน
```

---

## 10. Environment Variables

```python
# python-dotenv - จัดการ environment variables

# pip install python-dotenv

from dotenv import load_dotenv
import os

# โหลด .env file
load_dotenv()                    # โหลดจาก .env ในโฟลเดอร์นี้
load_dotenv('.env.development')  # โหลดจากไฟล์เฉพาะ
load_dotenv(override=True)       # override environment ที่มีอยู่

# อ่าน environment variables
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///default.db")
SECRET_KEY = os.environ["SECRET_KEY"]  # raise KeyError ถ้าไม่มี
DEBUG = os.environ.get("DEBUG", "False").lower() == "true"

# ตัวอย่าง .env file
```

```bash
# .env
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
SECRET_KEY=your-secret-key-here
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
REDIS_URL=redis://localhost:6379/0
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
```

```python
# config.py - Structured configuration
from dataclasses import dataclass
from typing import Optional
import os
from dotenv import load_dotenv

load_dotenv()

@dataclass
class DatabaseConfig:
    url: str
    pool_size: int = 5
    max_overflow: int = 10
    
    @classmethod
    def from_env(cls) -> 'DatabaseConfig':
        return cls(
            url=os.environ.get("DATABASE_URL", "sqlite:///default.db"),
            pool_size=int(os.environ.get("DB_POOL_SIZE", "5")),
            max_overflow=int(os.environ.get("DB_MAX_OVERFLOW", "10")),
        )


@dataclass
class AppConfig:
    secret_key: str
    debug: bool
    allowed_hosts: list
    database: DatabaseConfig
    
    @classmethod
    def from_env(cls) -> 'AppConfig':
        secret_key = os.environ.get("SECRET_KEY")
        if not secret_key:
            raise ValueError("SECRET_KEY environment variable is required!")
        
        return cls(
            secret_key=secret_key,
            debug=os.environ.get("DEBUG", "False").lower() == "true",
            allowed_hosts=os.environ.get("ALLOWED_HOSTS", "localhost").split(","),
            database=DatabaseConfig.from_env()
        )


# ใช้งาน
try:
    config = AppConfig.from_env()
    print(f"Debug mode: {config.debug}")
    print(f"Allowed hosts: {config.allowed_hosts}")
    print(f"DB: {config.database.url[:20]}...")
except ValueError as e:
    print(f"Configuration error: {e}")
```

---

## Exercises

### Exercise 1: Project Setup Script
สร้าง script ที่ automate การตั้งค่า project ใหม่:
```bash
python setup_project.py my-new-project --type django --python 3.11
```
- สร้าง directory structure
- สร้าง venv
- ติดตั้ง dependencies
- สร้าง .gitignore
- สร้าง .env.example

### Exercise 2: Dependency Updater
สร้าง tool ที่:
- อ่าน requirements.txt
- ตรวจสอบ latest versions
- แสดง diff
- สร้าง requirements.txt ใหม่

### Exercise 3: Multi-environment Manager
สร้าง script จัดการ environments:
```bash
python envman.py create prod     # สร้าง production venv
python envman.py create dev      # สร้าง development venv  
python envman.py switch prod     # switch ไป production
python envman.py install prod    # ติดตั้ง production deps
```

---

## สรุป

| Tool | จุดเด่น | ใช้เมื่อ |
|------|---------|---------|
| `venv` | built-in, เบา | ส่วนใหญ่ใช้ได้ |
| `pip` | standard, รองรับ PyPI | ติดตั้ง packages ทั่วไป |
| `pip-tools` | deterministic builds | team projects |
| `poetry` | all-in-one, modern | new projects |
| `conda` | multi-language | Data Science, ML |
| `pyenv` | manage Python versions | ต้องการหลาย Python version |

**Best Practices:**
1. ใช้ virtual environment ทุกครั้ง
2. Pin versions ใน production (`==`)
3. ใช้ ranges ใน development (`>=`)
4. Lock dependencies ไว้ (pip freeze หรือ poetry.lock)
5. แยก dev/prod dependencies
6. อย่า commit `.env` ลง git
7. ตรวจสอบ security vulnerabilities สม่ำเสมอ
8. ใช้ `python-dotenv` สำหรับ config

---

## ต่อไป

[Part 026 - Testing](part-026.md) - pytest, unittest, mocking, TDD
