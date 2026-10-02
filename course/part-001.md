# Part 001: การติดตั้งและตั้งค่า Python
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ติดตั้ง Python บน Windows, macOS, และ Linux ได้
- ใช้ Python Interpreter ได้
- รู้จักกับ pip (Package Manager)
- สร้าง Virtual Environment ได้
- ตั้งค่า VS Code สำหรับพัฒนา Python ได้
- เขียนโปรแกรม Hello World แรกได้

---

## 1. Python คืออะไร?

Python เป็นภาษาโปรแกรมระดับสูง (High-level Programming Language) ที่:
- **อ่านง่าย** - syntax คล้ายภาษาอังกฤษ
- **เขียนง่าย** - โค้ดน้อยกว่าภาษาอื่น
- **ใช้งานได้หลายด้าน** - Web, AI/ML, Data Science, Automation, Script
- **Community ใหญ่** - มี library และ framework มากมาย
- **Open Source** - ใช้ฟรี

### ประวัติโดยย่อ
- สร้างโดย **Guido van Rossum** ในปี 1991
- ชื่อมาจากรายการทีวี "Monty Python's Flying Circus"
- Python 2 → Python 3 (ปัจจุบัน)
- **Python 3.11+** คือเวอร์ชันที่เราใช้ในหลักสูตรนี้

### Python ใช้ทำอะไรได้บ้าง?

```
✅ Web Development    → Django, Flask, FastAPI
✅ Data Science       → Pandas, NumPy
✅ Machine Learning   → TensorFlow, PyTorch, scikit-learn
✅ Automation         → Script งานประจำวัน
✅ DevOps             → Ansible, Fabric
✅ Network            → Socket, Paramiko
✅ Desktop App        → Tkinter, PyQt
✅ Game Development   → Pygame
✅ Cybersecurity      → Scapy, Nmap
```

---

## 2. การติดตั้ง Python

### 2.1 ติดตั้งบน Windows

#### วิธีที่ 1: ดาวน์โหลดจากเว็บไซต์ Official
1. ไปที่ **https://www.python.org/downloads/**
2. คลิก **"Download Python 3.x.x"** (เวอร์ชันล่าสุด)
3. เปิดไฟล์ `.exe` ที่ดาวน์โหลด
4. ✅ **สำคัญมาก!** เลือก **"Add Python to PATH"** ก่อน
5. คลิก **"Install Now"**
6. รอจนติดตั้งเสร็จ

#### วิธีที่ 2: ใช้ winget (Windows Package Manager)
```powershell
# เปิด PowerShell แบบ Administrator
winget install Python.Python.3.11
```

#### วิธีที่ 3: ใช้ Chocolatey
```powershell
# ติดตั้ง Chocolatey ก่อน (ถ้ายังไม่มี)
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง Python
choco install python
```

#### ตรวจสอบการติดตั้ง
```powershell
# เปิด Command Prompt หรือ PowerShell
python --version
# ผลลัพธ์: Python 3.11.x

python3 --version
# ผลลัพธ์: Python 3.11.x

pip --version
# ผลลัพธ์: pip 23.x.x from ...
```

---

### 2.2 ติดตั้งบน macOS

#### วิธีที่ 1: ใช้ Homebrew (แนะนำ)
```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Python
brew install python@3.11

# เพิ่ม PATH (สำหรับ bash)
echo 'export PATH="/usr/local/opt/python@3.11/bin:$PATH"' >> ~/.bash_profile
source ~/.bash_profile

# สำหรับ zsh
echo 'export PATH="/usr/local/opt/python@3.11/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

#### วิธีที่ 2: ดาวน์โหลดจากเว็บไซต์ Official
1. ไปที่ **https://www.python.org/downloads/macos/**
2. ดาวน์โหลด `.pkg` file
3. ดับเบิลคลิกและทำตามขั้นตอน

#### ตรวจสอบการติดตั้ง
```bash
python3 --version
# ผลลัพธ์: Python 3.11.x

pip3 --version
# ผลลัพธ์: pip 23.x.x from ...

which python3
# ผลลัพธ์: /usr/local/bin/python3
```

---

### 2.3 ติดตั้งบน Linux (Ubuntu/Debian)

```bash
# อัพเดท package list
sudo apt update

# ติดตั้ง Python 3.11
sudo apt install python3.11 python3.11-pip python3.11-venv

# หรือติดตั้งผ่าน deadsnakes PPA (สำหรับเวอร์ชันล่าสุด)
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.11 python3.11-venv python3.11-dev

# สร้าง alias
echo 'alias python=python3' >> ~/.bashrc
echo 'alias pip=pip3' >> ~/.bashrc
source ~/.bashrc
```

#### ติดตั้งบน CentOS/RHEL/Fedora
```bash
# Fedora
sudo dnf install python3 python3-pip

# CentOS 8+
sudo dnf install python3

# CentOS 7 (ต้องใช้ SCL)
sudo yum install centos-release-scl
sudo yum install rh-python38
scl enable rh-python38 bash
```

#### ตรวจสอบการติดตั้งบน Linux
```bash
python3 --version
pip3 --version
which python3
```

---

## 3. Python Interpreter

Python Interpreter คือโปรแกรมที่แปลงโค้ด Python ให้คอมพิวเตอร์เข้าใจและรัน

### 3.1 Interactive Mode (REPL)

REPL = **R**ead-**E**val-**P**rint-**L**oop

```bash
# เปิด Python Interactive Shell
python3
# หรือ
python
```

คุณจะเห็น prompt แบบนี้:
```
Python 3.11.x (main, ...) 
[GCC x.x.x] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

#### ทดลองใช้ Interactive Mode
```python
>>> print("สวัสดี Python!")
สวัสดี Python!

>>> 2 + 2
4

>>> "Hello" + " " + "World"
'Hello World'

>>> 10 * 5
50

>>> type(42)
<class 'int'>

>>> type("Hello")
<class 'str'>

>>> type(3.14)
<class 'float'>

>>> type(True)
<class 'bool'>

>>> help(print)
# แสดง documentation ของ print

>>> exit()  # หรือ quit() หรือ Ctrl+D
```

### 3.2 Script Mode

สร้างไฟล์ `hello.py` และรัน:
```bash
python3 hello.py
```

### 3.3 ตรวจสอบ Python Path
```python
>>> import sys
>>> print(sys.executable)
/usr/bin/python3

>>> print(sys.version)
3.11.x (main, ...) [GCC x.x.x]

>>> print(sys.path)
['', '/usr/lib/python3.11', ...]
```

---

## 4. pip - Python Package Manager

pip คือเครื่องมือจัดการ package (library) ใน Python

### 4.1 คำสั่ง pip พื้นฐาน

```bash
# ตรวจสอบเวอร์ชัน pip
pip --version
pip3 --version

# อัพเดท pip
pip install --upgrade pip

# ค้นหา package
pip search requests  # (บางเวอร์ชันอาจ disable แล้ว)

# ติดตั้ง package
pip install requests

# ติดตั้งเวอร์ชันที่ระบุ
pip install requests==2.28.0

# ติดตั้ง minimum version
pip install requests>=2.28.0

# ติดตั้งหลาย package พร้อมกัน
pip install requests flask django

# อัพเกรด package
pip install --upgrade requests

# ถอนการติดตั้ง package
pip uninstall requests

# แสดง package ที่ติดตั้งอยู่
pip list

# แสดงข้อมูล package
pip show requests

# ส่งออก list ของ package
pip freeze > requirements.txt

# ติดตั้งจาก requirements.txt
pip install -r requirements.txt
```

### 4.2 requirements.txt

ไฟล์ `requirements.txt` ใช้บันทึก dependencies ของโปรเจค:
```
# requirements.txt
Django==4.2.0
Flask==3.0.0
FastAPI==0.104.0
SQLAlchemy==2.0.0
requests==2.31.0
pytest==7.4.0
```

### 4.3 pip install จาก GitHub
```bash
# ติดตั้งจาก GitHub repository โดยตรง
pip install git+https://github.com/user/repo.git

# ติดตั้ง branch เฉพาะ
pip install git+https://github.com/user/repo.git@branch-name
```

---

## 5. Virtual Environment (venv)

Virtual Environment คือสภาพแวดล้อมแยกสำหรับแต่ละโปรเจค ทำให้ dependencies ไม่ปน

### 5.1 ทำไมต้องใช้ Virtual Environment?

```
ปัญหาที่อาจเกิดขึ้นโดยไม่ใช้ venv:
- Project A ต้องการ Django 3.2
- Project B ต้องการ Django 4.2
- ถ้าติดตั้งในระดับ system → เกิดการขัดแย้ง

แก้ปัญหาด้วย Virtual Environment:
- แต่ละโปรเจคมี Python environment ของตัวเอง
- Dependencies แยกจากกัน
- ไม่กระทบ system Python
```

### 5.2 สร้างและใช้ Virtual Environment

```bash
# สร้าง virtual environment
python3 -m venv myenv

# หรือระบุชื่อ (นิยมใช้ .venv)
python3 -m venv .venv

# หรือ venv
python3 -m venv venv
```

#### เปิดใช้ Virtual Environment

**บน Linux/macOS:**
```bash
source .venv/bin/activate
# หรือ
source venv/bin/activate

# เห็น (venv) นำหน้า prompt
(venv) $ 
```

**บน Windows (Command Prompt):**
```cmd
.venv\Scripts\activate.bat
```

**บน Windows (PowerShell):**
```powershell
.venv\Scripts\Activate.ps1

# ถ้า error เรื่อง execution policy
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

#### ปิด Virtual Environment
```bash
deactivate
```

### 5.3 Workflow การใช้ Virtual Environment

```bash
# 1. สร้างโฟลเดอร์โปรเจค
mkdir my_project
cd my_project

# 2. สร้าง virtual environment
python3 -m venv .venv

# 3. เปิดใช้ virtual environment
source .venv/bin/activate   # Linux/macOS
# หรือ
.venv\Scripts\activate      # Windows

# 4. ติดตั้ง packages
pip install django flask fastapi

# 5. บันทึก dependencies
pip freeze > requirements.txt

# 6. ทำงาน...

# 7. ปิด virtual environment เมื่อเสร็จ
deactivate
```

### 5.4 การแชร์โปรเจค

```bash
# คนอื่นรับโปรเจคต่อ:
git clone <repo-url>
cd my_project

# สร้าง virtual environment ใหม่
python3 -m venv .venv
source .venv/bin/activate

# ติดตั้ง dependencies ทั้งหมด
pip install -r requirements.txt
```

### 5.5 .gitignore สำหรับ Python

```
# .gitignore
.venv/
venv/
__pycache__/
*.pyc
*.pyo
.env
*.egg-info/
dist/
build/
.pytest_cache/
.coverage
*.log
```

---

## 6. ตั้งค่า VS Code

VS Code (Visual Studio Code) คือ code editor ที่แนะนำสำหรับ Python

### 6.1 ติดตั้ง VS Code

ดาวน์โหลดจาก **https://code.visualstudio.com/**

### 6.2 Extensions ที่ต้องติดตั้ง

เปิด VS Code แล้วกด `Ctrl+Shift+X` (หรือ `Cmd+Shift+X` บน Mac):

```
ที่ต้องมี:
1. Python (by Microsoft)              - Python language support
2. Pylance                            - Python language server
3. Python Debugger                    - Debug Python code

แนะนำ:
4. GitLens                            - Git integration
5. GitGraph                           - Git graph visualization
6. Thunder Client                     - HTTP client (เหมือน Postman)
7. SQLite Viewer                      - ดู SQLite database
8. Better Comments                    - Comments ที่สวยงาม
9. Indent Rainbow                     - ระบายสี indentation
10. Bracket Pair Colorizer            - ระบายสี brackets
11. Auto Close Tag                    - ปิด tag อัตโนมัติ
12. Error Lens                        - แสดง error inline
13. Code Runner                       - รันโค้ดได้เลย
14. Prettier                          - Code formatter
15. Django (by Baptiste Darthenay)    - Django support
```

### 6.3 ตั้งค่า Python Interpreter ใน VS Code

1. กด `Ctrl+Shift+P` (Command Palette)
2. พิมพ์ **"Python: Select Interpreter"**
3. เลือก Python ที่อยู่ใน virtual environment `.venv`

### 6.4 ตั้งค่า settings.json

กด `Ctrl+Shift+P` → "Open User Settings (JSON)":

```json
{
    "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
    "python.formatting.provider": "black",
    "editor.formatOnSave": true,
    "python.linting.enabled": true,
    "python.linting.flake8Enabled": true,
    "python.linting.pylintEnabled": false,
    "editor.tabSize": 4,
    "editor.insertSpaces": true,
    "files.autoSave": "afterDelay",
    "files.autoSaveDelay": 1000,
    "terminal.integrated.defaultProfile.linux": "bash",
    "python.testing.pytestEnabled": true,
    "python.testing.pytestArgs": ["tests"]
}
```

### 6.5 Keyboard Shortcuts ที่ใช้บ่อย

```
Ctrl+S          - บันทึกไฟล์
Ctrl+Z          - Undo
Ctrl+Shift+Z    - Redo
Ctrl+/          - Toggle comment
Ctrl+D          - เลือก word ถัดไปที่เหมือนกัน
Ctrl+F          - ค้นหา
Ctrl+H          - ค้นหาและแทนที่
Ctrl+Shift+P    - Command Palette
Ctrl+`          - เปิด Terminal
F5              - Debug
F9              - Toggle Breakpoint
Ctrl+Space      - เปิด autocomplete
```

---

## 7. เครื่องมืออื่นที่แนะนำ

### 7.1 pyenv - จัดการหลายเวอร์ชัน Python

```bash
# ติดตั้ง pyenv (Linux/macOS)
curl https://pyenv.run | bash

# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

# ดู Python เวอร์ชันที่มี
pyenv install --list

# ติดตั้ง Python เวอร์ชันที่ต้องการ
pyenv install 3.11.6
pyenv install 3.12.0

# ตั้งค่า global version
pyenv global 3.11.6

# ตั้งค่า local version (เฉพาะโฟลเดอร์นี้)
pyenv local 3.11.6

# ดู version ที่ใช้อยู่
pyenv version
```

### 7.2 Poetry - Alternative Package Manager

```bash
# ติดตั้ง Poetry
curl -sSL https://install.python-poetry.org | python3 -

# สร้างโปรเจคใหม่
poetry new my-project

# เพิ่ม dependency
poetry add requests
poetry add --dev pytest

# ติดตั้ง dependencies จาก pyproject.toml
poetry install

# เปิด shell ใน virtual environment
poetry shell

# รัน script
poetry run python main.py
```

### 7.3 Black - Code Formatter

```bash
# ติดตั้ง
pip install black

# Format ไฟล์
black myfile.py

# Format ทุกไฟล์ใน directory
black .

# ตรวจสอบโดยไม่แก้ไข
black --check .
```

### 7.4 flake8 - Linter

```bash
# ติดตั้ง
pip install flake8

# ตรวจสอบ
flake8 myfile.py
flake8 .
```

### 7.5 isort - Import Sorter

```bash
# ติดตั้ง
pip install isort

# จัดเรียง imports
isort myfile.py
isort .
```

---

## 8. โปรแกรมแรก: Hello World

### 8.1 สร้างไฟล์แรก

สร้างไฟล์ `hello.py`:

```python
# hello.py

# โปรแกรม Hello World แรก
print("Hello, World!")
print("สวัสดี โลก!")
print("ยินดีต้อนรับสู่การเรียน Python!")
```

รันโปรแกรม:
```bash
python3 hello.py
```

ผลลัพธ์:
```
Hello, World!
สวัสดี โลก!
ยินดีต้อนรับสู่การเรียน Python!
```

### 8.2 ทำความเข้าใจ print()

```python
# print() พื้นฐาน
print("Hello")          # ข้อความ
print(42)               # ตัวเลข
print(3.14)             # ทศนิยม
print(True)             # Boolean

# print() แบบหลายค่า
print("Name:", "Alice")
print("Age:", 25)
print("Score:", 95.5)

# print() กับ separator
print("A", "B", "C")                   # A B C
print("A", "B", "C", sep="-")          # A-B-C
print("A", "B", "C", sep="")           # ABC
print("A", "B", "C", sep=", ")         # A, B, C

# print() กับ end
print("Hello", end="")      # ไม่ขึ้นบรรทัดใหม่
print("World")               # HelloWorld
print("Hello", end="\n\n")   # ขึ้น 2 บรรทัด

# print() แบบ f-string (Python 3.6+)
name = "Alice"
age = 25
print(f"My name is {name} and I am {age} years old.")

# print() แบบ format()
print("My name is {} and I am {} years old.".format(name, age))

# print() แบบ % formatting (แบบเก่า)
print("My name is %s and I am %d years old." % (name, age))
```

### 8.3 Comments ใน Python

```python
# นี่คือ single-line comment

# Python ใช้ # สำหรับ comment
x = 10  # comment หลังโค้ดก็ได้

"""
นี่คือ multi-line string
ที่มักใช้เป็น documentation string (docstring)
หรือ comment หลายบรรทัด
"""

'''
แบบนี้ก็ได้เช่นกัน
ใช้ single quote 3 ตัว
'''

def greet(name):
    """
    ฟังก์ชันนี้รับชื่อและส่งคืนคำทักทาย
    
    Args:
        name (str): ชื่อของบุคคล
    
    Returns:
        str: ข้อความทักทาย
    """
    return f"Hello, {name}!"
```

### 8.4 โปรแกรมแรกที่ครบครัน

```python
# first_program.py
"""
โปรแกรมแรกของฉัน
เรียนรู้ Python ขั้นพื้นฐาน
"""

# แสดงข้อความต้อนรับ
print("=" * 50)
print("   ยินดีต้อนรับสู่โลกของ Python!")
print("=" * 50)
print()

# ข้อมูลพื้นฐาน
name = "Python Learner"
version = "3.11"

print(f"ชื่อ: {name}")
print(f"Python Version: {version}")
print()

# คำนวณง่ายๆ
a = 10
b = 5
print(f"การคำนวณ:")
print(f"  {a} + {b} = {a + b}")
print(f"  {a} - {b} = {a - b}")
print(f"  {a} * {b} = {a * b}")
print(f"  {a} / {b} = {a / b}")
print(f"  {a} % {b} = {a % b}  (remainder)")
print(f"  {a} ** {b} = {a ** b} (power)")
print()

print("เริ่มต้น Python journey ของคุณแล้ว! 🚀")
print("=" * 50)
```

ผลลัพธ์:
```
==================================================
   ยินดีต้อนรับสู่โลกของ Python!
==================================================

ชื่อ: Python Learner
Python Version: 3.11

การคำนวณ:
  10 + 5 = 15
  10 - 5 = 5
  10 * 5 = 50
  10 / 5 = 2.0
  10 % 5 = 0  (remainder)
  10 ** 5 = 100000 (power)

เริ่มต้น Python journey ของคุณแล้ว! 🚀
==================================================
```

---

## 9. โครงสร้างโปรเจค Python มาตรฐาน

```
my_project/
│
├── .venv/                  # Virtual environment (ไม่ commit git)
├── .git/                   # Git repository
│
├── src/                    # Source code หลัก
│   └── my_project/
│       ├── __init__.py
│       ├── main.py
│       └── utils.py
│
├── tests/                  # Unit tests
│   ├── __init__.py
│   └── test_main.py
│
├── docs/                   # Documentation
│
├── requirements.txt        # Production dependencies
├── requirements-dev.txt    # Development dependencies
├── README.md               # Project description
├── .gitignore              # Git ignore rules
├── .env                    # Environment variables (ไม่ commit git)
├── .env.example            # Example env file (commit ได้)
├── setup.py                # Package setup (ถ้าทำ package)
└── pyproject.toml          # Modern package config
```

---

## 10. ตั้งค่า Git สำหรับโปรเจค

```bash
# ตั้งค่า Git global
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.editor "code --wait"  # ใช้ VS Code

# เริ่มต้น repository ใหม่
cd my_project
git init

# สร้าง .gitignore
cat > .gitignore << 'EOF'
.venv/
venv/
__pycache__/
*.pyc
*.pyo
.env
*.egg-info/
dist/
build/
.pytest_cache/
.coverage
htmlcov/
*.log
.DS_Store
EOF

# Add และ commit ครั้งแรก
git add .
git commit -m "Initial commit"
```

---

## 11. การ Debug โปรแกรม Python

### 11.1 ใช้ print() debug (วิธีง่าย)

```python
def calculate_average(numbers):
    print(f"Debug: numbers = {numbers}")  # debug print
    total = sum(numbers)
    print(f"Debug: total = {total}")      # debug print
    count = len(numbers)
    print(f"Debug: count = {count}")      # debug print
    return total / count

result = calculate_average([1, 2, 3, 4, 5])
print(f"Average: {result}")
```

### 11.2 ใช้ VS Code Debugger

1. คลิกซ้ายที่ขอบซ้ายของบรรทัด → เพิ่ม **Breakpoint** (จุดสีแดง)
2. กด **F5** เพื่อเริ่ม Debug
3. ใช้ปุ่ม:
   - **F10** - Step Over (ข้ามบรรทัด)
   - **F11** - Step Into (เข้าไปใน function)
   - **Shift+F11** - Step Out (ออกจาก function)
   - **F5** - Continue (รันต่อจนถึง breakpoint ถัดไป)

### 11.3 ใช้ pdb (Python Debugger)

```python
import pdb

def buggy_function(x, y):
    pdb.set_trace()  # จุดหยุด
    result = x + y
    return result

buggy_function(10, 20)
```

คำสั่งใน pdb:
```
n    - next line
s    - step into function
c    - continue
q    - quit
p x  - print variable x
l    - list source code
h    - help
```

---

## 12. Python Shell และ IPython

### 12.1 IPython (ดีกว่า Python REPL ปกติ)

```bash
# ติดตั้ง IPython
pip install ipython

# เปิดใช้งาน
ipython
```

คุณสมบัติ IPython:
```python
In [1]: ?print          # ดู documentation
In [2]: ??print         # ดู source code
In [3]: %timeit sum(range(1000))    # วัดเวลา
In [4]: %run script.py  # รัน script
In [5]: %history        # ดูประวัติคำสั่ง
In [6]: name = "Alice"
In [7]: na<Tab>         # auto-complete
```

### 12.2 Jupyter Notebook

```bash
# ติดตั้ง Jupyter
pip install jupyter

# เปิด Jupyter Notebook
jupyter notebook

# เปิด Jupyter Lab (แนะนำ)
pip install jupyterlab
jupyter lab
```

---

## 13. Python PEP 8 - Style Guide

PEP 8 คือ style guide ของ Python ที่ทุกคนควรรู้:

```python
# ✅ ถูกต้อง - ชื่อตัวแปร (snake_case)
my_variable = 10
user_name = "Alice"
total_count = 0

# ❌ ไม่ควร
myVariable = 10     # camelCase
MyVariable = 10     # PascalCase (ใช้สำหรับ class)

# ✅ ถูกต้อง - ชื่อ function (snake_case)
def calculate_total():
    pass

def get_user_name():
    pass

# ✅ ถูกต้อง - ชื่อ class (PascalCase/UpperCamelCase)
class UserAccount:
    pass

class DatabaseConnection:
    pass

# ✅ ถูกต้อง - ชื่อ constants (UPPER_SNAKE_CASE)
MAX_RETRY = 3
DATABASE_URL = "postgresql://..."

# ✅ การเว้นวรรค
x = 10              # ก่อนและหลัง =
y = x + 5           # ก่อนและหลัง operator
if x > 0:           # หลัง if
    print(x)

# ✅ บรรทัดว่าง
class MyClass:
    def method_one(self):
        pass
    
    def method_two(self):  # 1 บรรทัดว่างระหว่าง methods
        pass


def function_one():    # 2 บรรทัดว่างระหว่าง functions
    pass


def function_two():
    pass

# ✅ Import ที่ถูกต้อง (แยกบรรทัด)
import os
import sys
from pathlib import Path
from typing import List, Dict

# ❌ ไม่ควร
import os, sys         # อย่า import หลายอันในบรรทัดเดียว

# ✅ ความยาวบรรทัดไม่เกิน 79 ตัวอักษร (PEP 8)
# หรือ 88-120 ตัวอักษร (สมัยใหม่ยอมรับกว่า)
```

---

## 14. Exercises (แบบฝึกหัด)

### Exercise 1: Hello World หลายรูปแบบ
```python
# สร้างไฟล์ exercise_01.py
# แสดงข้อความต่อไปนี้:
# 1. "Hello, World!"
# 2. ชื่อของคุณ
# 3. อายุของคุณ
# 4. ภาษาโปรแกรมที่คุณกำลังเรียน
# 5. วันที่ปัจจุบัน

from datetime import date

print("Hello, World!")
print("ชื่อ: สมชาย ใจดี")
print("อายุ: 25 ปี")
print("กำลังเรียน: Python")
print(f"วันที่: {date.today()}")
```

### Exercise 2: ข้อมูลโปรเจค
```python
# สร้างไฟล์ exercise_02.py
# แสดงข้อมูลเกี่ยวกับโปรเจค Python ที่คุณต้องการทำ

project_name = "My First Python App"
description = "แอพพลิเคชันแรกของฉันด้วย Python"
tech_stack = ["Python", "Flask", "PostgreSQL"]
start_date = "2024-01-01"

print("=" * 60)
print(f"โปรเจค: {project_name}")
print(f"คำอธิบาย: {description}")
print(f"เทคโนโลยี: {', '.join(tech_stack)}")
print(f"วันเริ่มต้น: {start_date}")
print("=" * 60)
```

### Exercise 3: Calculator พื้นฐาน
```python
# สร้างไฟล์ exercise_03.py
# สร้าง calculator พื้นฐาน

a = 100
b = 37

print(f"การคำนวณระหว่าง {a} และ {b}:")
print(f"  บวก:   {a} + {b} = {a + b}")
print(f"  ลบ:    {a} - {b} = {a - b}")
print(f"  คูณ:   {a} * {b} = {a * b}")
print(f"  หาร:   {a} / {b} = {a / b:.2f}")
print(f"  หารเต็ม: {a} // {b} = {a // b}")
print(f"  เศษ:   {a} % {b} = {a % b}")
print(f"  ยกกำลัง: {a} ** 2 = {a ** 2}")
```

---

## 15. สรุป Part 001

### สิ่งที่เรียนรู้ในวันนี้:

✅ **Python คืออะไร** - ภาษาโปรแกรมระดับสูง ใช้ได้หลากหลาย  
✅ **ติดตั้ง Python** - บน Windows, macOS, Linux  
✅ **Python Interpreter** - Interactive mode และ Script mode  
✅ **pip** - Package manager สำหรับจัดการ libraries  
✅ **Virtual Environment** - สภาพแวดล้อมแยกสำหรับแต่ละโปรเจค  
✅ **VS Code** - Editor และ extensions ที่ต้องมี  
✅ **Hello World** - โปรแกรมแรก  
✅ **PEP 8** - Style guide มาตรฐาน  

### คำสั่งที่ต้องจำ:

```bash
# ตรวจสอบ Python
python3 --version
pip --version

# Virtual Environment
python3 -m venv .venv
source .venv/bin/activate    # Linux/Mac
.venv\Scripts\activate       # Windows
deactivate

# pip
pip install package-name
pip freeze > requirements.txt
pip install -r requirements.txt
```

---

## ➡️ ถัดไป: Part 002 - ตัวแปรและชนิดข้อมูลพื้นฐาน

ใน Part ถัดไป เราจะเรียนรู้:
- ตัวแปร (Variables) และการตั้งชื่อ
- ชนิดข้อมูล (Data Types) ทุกประเภท
- การแปลงชนิดข้อมูล (Type Conversion)
- Input จาก user

---

*Part 001/100+ | Python Course - Beginner to World-Class*
