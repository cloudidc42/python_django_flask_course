# Part 047: CLI Applications with Python
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจการใช้ `argparse` สร้าง CLI tools แบบ built-in
- ใช้ `click` library สำหรับ CLI ที่มีโครงสร้างซับซ้อน
- ใช้ `typer` ที่ผสม Click กับ Type Hints
- แสดงผลแบบสวยงามด้วย `rich` library
- สร้าง CLI tool ครบวงจรสำหรับ Task Manager

---

## 1. argparse — Built-in Python CLI Parser

`argparse` เป็น module ที่มาพร้อม Python สำหรับรับ arguments จาก command line

### 1.1 ArgumentParser พื้นฐาน

```python
import argparse
import sys
from pathlib import Path

# === สร้าง ArgumentParser พื้นฐาน ===
def create_basic_parser():
    """สร้าง parser สำหรับ file processing tool"""
    parser = argparse.ArgumentParser(
        prog="filetool",                          # ชื่อโปรแกรม
        description="เครื่องมือจัดการไฟล์",        # คำอธิบายหลัก
        epilog="ตัวอย่าง: python script.py input.txt --output out.txt --verbose",
        formatter_class=argparse.RawDescriptionHelpFormatter
    )
    return parser

# === เพิ่ม Positional Arguments (บังคับ) ===
def add_positional_args(parser):
    """Positional arguments ไม่ต้องใช้ flag"""
    parser.add_argument(
        "input_file",
        type=str,
        help="ไฟล์ input ที่ต้องการประมวลผล"
    )
    parser.add_argument(
        "output_file",
        nargs="?",           # optional (0 หรือ 1 ค่า)
        default="output.txt",
        help="ไฟล์ output (default: output.txt)"
    )

# === เพิ่ม Optional Arguments ===
def add_optional_args(parser):
    """Optional arguments ใช้ -- นำหน้า"""
    # Boolean flag
    parser.add_argument(
        "-v", "--verbose",
        action="store_true",  # เก็บค่า True ถ้าใส่ flag นี้
        help="แสดงข้อมูล debug เพิ่มเติม"
    )
    
    # String option
    parser.add_argument(
        "--encoding",
        type=str,
        default="utf-8",
        choices=["utf-8", "utf-16", "ascii", "tis620"],  # ค่าที่อนุญาต
        help="encoding ของไฟล์ (default: utf-8)"
    )
    
    # Integer option
    parser.add_argument(
        "--lines",
        type=int,
        default=None,
        metavar="N",          # ชื่อที่แสดงใน help
        help="จำนวนบรรทัดที่ต้องการอ่าน"
    )
    
    # Multiple values
    parser.add_argument(
        "--tags",
        nargs="+",            # 1 ค่าขึ้นไป
        type=str,
        help="tags สำหรับไฟล์ เช่น --tags python cli tool"
    )
    
    # Store constant
    parser.add_argument(
        "--format",
        dest="output_format",
        choices=["json", "csv", "txt"],
        default="txt",
        help="รูปแบบ output"
    )

# === ตัวอย่างการใช้งาน ===
def main():
    parser = create_basic_parser()
    add_positional_args(parser)
    add_optional_args(parser)
    
    # Parse arguments
    args = parser.parse_args()
    
    # ใช้งาน arguments
    if args.verbose:
        print(f"[DEBUG] Input: {args.input_file}")
        print(f"[DEBUG] Output: {args.output_file}")
        print(f"[DEBUG] Encoding: {args.encoding}")
    
    # ตรวจสอบไฟล์ input
    input_path = Path(args.input_file)
    if not input_path.exists():
        print(f"Error: ไม่พบไฟล์ {args.input_file}", file=sys.stderr)
        sys.exit(1)
    
    print(f"กำลังประมวลผล {args.input_file}...")
    print(f"Format: {args.output_format}")
    if args.tags:
        print(f"Tags: {', '.join(args.tags)}")

if __name__ == "__main__":
    main()
```

### 1.2 Subparsers — คำสั่งย่อย

```python
import argparse

# === Subparsers สำหรับคำสั่งย่อย ===
def create_git_like_cli():
    """สร้าง CLI คล้าย git ที่มีคำสั่งย่อย"""
    parser = argparse.ArgumentParser(
        description="ระบบจัดการโปรเจกต์"
    )
    
    # สร้าง subparsers
    subparsers = parser.add_subparsers(
        title="คำสั่งที่ใช้ได้",
        dest="command",          # เก็บชื่อ subcommand ใน args.command
        description="เลือกคำสั่งที่ต้องการ",
        help="คำสั่งย่อยที่ใช้ได้"
    )
    
    # === subcommand: init ===
    init_parser = subparsers.add_parser(
        "init",
        help="สร้างโปรเจกต์ใหม่"
    )
    init_parser.add_argument("name", help="ชื่อโปรเจกต์")
    init_parser.add_argument(
        "--template",
        choices=["django", "flask", "fastapi", "basic"],
        default="basic",
        help="template ที่ใช้"
    )
    
    # === subcommand: build ===
    build_parser = subparsers.add_parser(
        "build",
        help="Build โปรเจกต์"
    )
    build_parser.add_argument(
        "--env",
        choices=["dev", "staging", "prod"],
        default="dev",
        help="environment ที่ build"
    )
    build_parser.add_argument(
        "--clean",
        action="store_true",
        help="ลบไฟล์เก่าก่อน build"
    )
    
    # === subcommand: deploy ===
    deploy_parser = subparsers.add_parser(
        "deploy",
        help="Deploy โปรเจกต์"
    )
    deploy_parser.add_argument("server", help="server ที่ deploy ไป")
    deploy_parser.add_argument(
        "--port",
        type=int,
        default=8000,
        help="port ที่ใช้ (default: 8000)"
    )
    deploy_parser.add_argument(
        "--force",
        action="store_true",
        help="deploy แม้มีข้อผิดพลาด"
    )
    
    return parser

# === Handler functions ===
def handle_init(args):
    print(f"สร้างโปรเจกต์ '{args.name}' ด้วย template '{args.template}'")

def handle_build(args):
    if args.clean:
        print("ลบไฟล์เก่า...")
    print(f"Building สำหรับ environment: {args.env}")

def handle_deploy(args):
    force_str = " (force)" if args.force else ""
    print(f"Deploying ไปยัง {args.server}:{args.port}{force_str}")

def main():
    parser = create_git_like_cli()
    args = parser.parse_args()
    
    # เรียก handler ตาม subcommand
    handlers = {
        "init": handle_init,
        "build": handle_build,
        "deploy": handle_deploy,
    }
    
    if args.command is None:
        parser.print_help()
        return
    
    handler = handlers.get(args.command)
    if handler:
        handler(args)
    else:
        print(f"ไม่รู้จักคำสั่ง: {args.command}")

if __name__ == "__main__":
    main()
```

### 1.3 Custom Types และ Validation

```python
import argparse
from pathlib import Path
import re

# === Custom Type Functions ===
def valid_port(value):
    """ตรวจสอบว่าเป็น port ที่ถูกต้อง (1-65535)"""
    try:
        port = int(value)
    except ValueError:
        raise argparse.ArgumentTypeError(f"'{value}' ไม่ใช่ตัวเลข")
    
    if not (1 <= port <= 65535):
        raise argparse.ArgumentTypeError(
            f"Port ต้องอยู่ระหว่าง 1-65535, ได้รับ: {port}"
        )
    return port

def valid_email(value):
    """ตรวจสอบ email format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, value):
        raise argparse.ArgumentTypeError(f"'{value}' ไม่ใช่ email ที่ถูกต้อง")
    return value

def existing_file(value):
    """ตรวจสอบว่าไฟล์มีอยู่จริง"""
    path = Path(value)
    if not path.exists():
        raise argparse.ArgumentTypeError(f"ไม่พบไฟล์: {value}")
    if not path.is_file():
        raise argparse.ArgumentTypeError(f"ไม่ใช่ไฟล์: {value}")
    return path

def writable_dir(value):
    """ตรวจสอบว่า directory เขียนได้"""
    path = Path(value)
    if not path.exists():
        raise argparse.ArgumentTypeError(f"ไม่พบ directory: {value}")
    if not path.is_dir():
        raise argparse.ArgumentTypeError(f"ไม่ใช่ directory: {value}")
    return path

# === ใช้ Custom Types ===
def create_server_parser():
    parser = argparse.ArgumentParser(description="Server Configuration")
    
    parser.add_argument(
        "--port",
        type=valid_port,       # ใช้ custom type function
        default=8000,
        help="Port number (1-65535)"
    )
    
    parser.add_argument(
        "--admin-email",
        type=valid_email,
        required=True,         # บังคับต้องใส่
        help="Admin email address"
    )
    
    parser.add_argument(
        "--config",
        type=existing_file,    # ไฟล์ต้องมีอยู่จริง
        help="Config file path"
    )
    
    parser.add_argument(
        "--log-dir",
        type=writable_dir,
        default=Path("."),
        help="Directory สำหรับ log files"
    )
    
    return parser

# ทดสอบ: python script.py --port 8080 --admin-email admin@example.com
```

---

## 2. Click Library — Elegant CLI Framework

Click เป็น library ที่ใช้ decorators สร้าง CLI ที่สวยงามและยืดหยุ่น

```bash
# ติดตั้ง click
pip install click
```

### 2.1 Click Commands พื้นฐาน

```python
import click
from pathlib import Path

# === คำสั่งพื้นฐาน ===
@click.command()
@click.argument("name")                          # Positional argument
@click.option("--count", "-c",                   # Optional ที่ใช้ได้ทั้ง --count และ -c
    default=1,
    show_default=True,                           # แสดงค่า default ใน help
    help="จำนวนครั้งที่ทักทาย"
)
@click.option("--language", "-l",
    type=click.Choice(["th", "en", "jp"]),       # ค่าที่อนุญาต
    default="th",
    show_default=True,
    help="ภาษาที่ใช้ทักทาย"
)
@click.option("--shout/--no-shout",              # Boolean flag
    default=False,
    help="แสดงเป็นตัวพิมพ์ใหญ่"
)
def greet(name, count, language, shout):
    """ทักทายผู้ใช้ตามจำนวนที่กำหนด"""
    greetings = {
        "th": f"สวัสดี {name}!",
        "en": f"Hello, {name}!",
        "jp": f"こんにちは {name}!",
    }
    message = greetings.get(language, f"Hello, {name}!")
    
    if shout:
        message = message.upper()
    
    for _ in range(count):
        click.echo(message)

# รัน: python script.py greet สมชาย --count 3 --language th
```

### 2.2 Click Options ขั้นสูง

```python
import click
import os

# === Password input (ซ่อน input) ===
@click.command()
@click.option("--username", prompt="ชื่อผู้ใช้", help="Username")
@click.option("--password",
    prompt="รหัสผ่าน",
    hide_input=True,          # ซ่อน input
    confirmation_prompt=True, # ให้ใส่ซ้ำเพื่อยืนยัน
    help="Password"
)
def login(username, password):
    """เข้าสู่ระบบ"""
    click.echo(f"กำลังเข้าสู่ระบบในฐานะ: {username}")

# === File options ===
@click.command()
@click.argument("input_file",
    type=click.Path(exists=True, readable=True, path_type=Path)
)
@click.option("--output", "-o",
    type=click.File("w", encoding="utf-8"),   # เปิดไฟล์เลย
    default="-",                               # "-" หมายถึง stdout
    help="Output file (default: stdout)"
)
@click.option("--format",
    type=click.Choice(["json", "csv", "yaml"]),
    default="json"
)
def convert(input_file, output, format):
    """แปลงไฟล์เป็น format ต่างๆ"""
    click.echo(f"Converting {input_file} to {format}", file=output)

# === Environment variable support ===
@click.command()
@click.option("--api-key",
    envvar="MY_API_KEY",      # อ่านจาก environment variable ได้
    required=True,
    help="API key (หรือ MY_API_KEY env var)"
)
@click.option("--base-url",
    envvar="MY_BASE_URL",
    default="https://api.example.com",
    help="Base URL"
)
def api_call(api_key, base_url):
    """เรียก API"""
    click.echo(f"Calling {base_url} with key: {api_key[:4]}****")

# === Multiple values ===
@click.command()
@click.option("--include", "-i",
    multiple=True,            # รับได้หลายค่า: --include *.py --include *.txt
    help="รูปแบบไฟล์ที่รวม"
)
@click.option("--exclude", "-e",
    multiple=True,
    help="รูปแบบไฟล์ที่ยกเว้น"
)
def scan(include, exclude):
    """สแกนไฟล์"""
    click.echo(f"Include patterns: {list(include)}")
    click.echo(f"Exclude patterns: {list(exclude)}")
```

### 2.3 Click Groups — คำสั่งกลุ่ม

```python
import click

# === Main CLI group ===
@click.group()
@click.option("--debug/--no-debug", default=False)
@click.option("--config",
    type=click.Path(exists=False),
    default="~/.myapp/config.yaml",
    help="Config file path"
)
@click.version_option("1.0.0")          # เพิ่ม --version flag
@click.pass_context                      # รับ context object
def cli(ctx, debug, config):
    """My Application — เครื่องมือจัดการระบบ"""
    # เก็บ shared state ใน context
    ctx.ensure_object(dict)
    ctx.obj["debug"] = debug
    ctx.obj["config"] = config
    
    if debug:
        click.echo(f"[DEBUG] Config: {config}", err=True)

# === Database commands group ===
@cli.group()
def db():
    """คำสั่งจัดการ Database"""
    pass

@db.command("migrate")
@click.option("--fake", is_flag=True, help="Fake migration")
@click.pass_context
def db_migrate(ctx, fake):
    """รัน database migrations"""
    debug = ctx.obj["debug"]
    if debug:
        click.echo("[DEBUG] Running migrations...")
    
    if fake:
        click.echo("Fake migration สำเร็จ")
    else:
        click.echo("Migration สำเร็จ")

@db.command("reset")
@click.confirmation_option(
    prompt="แน่ใจหรือไม่ที่จะ reset database?"
)
def db_reset():
    """Reset database ทั้งหมด"""
    click.echo("Database reset สำเร็จ")

@db.command("backup")
@click.argument("output_path", type=click.Path())
@click.option("--compress", is_flag=True, help="บีบอัดไฟล์ backup")
def db_backup(output_path, compress):
    """สำรองข้อมูล database"""
    click.echo(f"กำลัง backup ไปยัง {output_path}")
    if compress:
        click.echo("กำลังบีบอัด...")

# === User commands group ===
@cli.group()
def user():
    """คำสั่งจัดการ Users"""
    pass

@user.command("create")
@click.option("--name", required=True, help="ชื่อผู้ใช้")
@click.option("--email", required=True, help="Email")
@click.option("--role",
    type=click.Choice(["admin", "editor", "viewer"]),
    default="viewer",
    help="Role ของผู้ใช้"
)
def user_create(name, email, role):
    """สร้างผู้ใช้ใหม่"""
    click.echo(f"สร้างผู้ใช้: {name} ({email}) role: {role}")

@user.command("list")
@click.option("--role", type=click.Choice(["admin", "editor", "viewer", "all"]),
    default="all")
@click.option("--format",
    type=click.Choice(["table", "json", "csv"]),
    default="table"
)
def user_list(role, format):
    """แสดงรายชื่อผู้ใช้"""
    click.echo(f"แสดงผู้ใช้ role={role} format={format}")

# รัน:
# python script.py --debug db migrate
# python script.py db backup /tmp/backup.sql --compress
# python script.py user create --name สมชาย --email somchai@example.com --role admin
```

### 2.4 Click Colors และ Styling

```python
import click
import sys

# === Click echo พร้อมสี ===
def print_status(message, level="info"):
    """พิมพ์ message พร้อมสีตาม level"""
    colors = {
        "info":    ("blue",   "ℹ"),
        "success": ("green",  "✓"),
        "warning": ("yellow", "⚠"),
        "error":   ("red",    "✗"),
    }
    color, icon = colors.get(level, ("white", "·"))
    
    click.echo(
        click.style(f"{icon} ", fg=color, bold=True) +
        click.style(message, fg=color)
    )

# === Progress bar ===
@click.command()
@click.argument("items", nargs=-1)     # รับหลาย arguments
def process_items(items):
    """ประมวลผลหลาย items พร้อม progress bar"""
    if not items:
        click.echo("ไม่มี items ให้ประมวลผล")
        return
    
    # วิธีที่ 1: click.progressbar
    with click.progressbar(
        items,
        label="กำลังประมวลผล",
        fill_char=click.style("█", fg="green"),
        empty_char="░",
        width=50
    ) as bar:
        import time
        for item in bar:
            time.sleep(0.1)  # จำลองการทำงาน
    
    print_status("ประมวลผลเสร็จสิ้น", "success")

# === Confirmation และ Prompts ===
@click.command()
@click.argument("filename")
def delete_file(filename):
    """ลบไฟล์พร้อมขอยืนยัน"""
    if not click.confirm(
        click.style(f"แน่ใจหรือไม่ที่จะลบ '{filename}'?", fg="red")
    ):
        click.echo("ยกเลิกการลบ")
        return
    
    click.echo(f"ลบ '{filename}' แล้ว")

# === Pager สำหรับ output ยาว ===
@click.command()
def show_help():
    """แสดง help ยาวๆ ด้วย pager"""
    long_text = "\n".join([f"บรรทัดที่ {i}: ข้อมูลต่างๆ" for i in range(100)])
    
    click.echo_via_pager(long_text)  # เปิด less/more โดยอัตโนมัติ

# === Clear screen ===
@click.command()
def refresh():
    """ล้างหน้าจอและแสดงข้อมูลใหม่"""
    click.clear()
    click.echo("หน้าจอถูกล้างแล้ว")
```

---

## 3. Typer Library — Modern CLI with Type Hints

Typer สร้าง CLI จาก Python type hints โดยอัตโนมัติ

```bash
# ติดตั้ง typer พร้อม extras
pip install "typer[all]"
```

### 3.1 Typer พื้นฐาน

```python
import typer
from typing import Optional, Annotated
from pathlib import Path
from enum import Enum

# === สร้าง Typer app ===
app = typer.Typer(
    name="myapp",
    help="My CLI Application — เครื่องมือจัดการระบบ",
    add_completion=True     # เพิ่ม shell completion
)

# === Command พื้นฐาน ===
@app.command()
def hello(
    name: str,                              # Positional argument
    count: int = 1,                         # Option ที่มี default
    language: str = "th",                   # String option
    shout: bool = False,                    # Boolean flag
):
    """ทักทายผู้ใช้"""
    greetings = {"th": f"สวัสดี {name}!", "en": f"Hello, {name}!"}
    message = greetings.get(language, f"Hello, {name}!")
    
    if shout:
        message = message.upper()
    
    for _ in range(count):
        typer.echo(message)

# === Annotated — วิธีใหม่ที่แนะนำ ===
@app.command()
def greet(
    name: Annotated[str, typer.Argument(help="ชื่อที่ต้องการทักทาย")],
    count: Annotated[int, typer.Option(
        "--count", "-c",
        help="จำนวนครั้งที่ทักทาย",
        min=1,
        max=100
    )] = 1,
    uppercase: Annotated[bool, typer.Option(
        "--upper/--no-upper",
        help="แปลงเป็นตัวพิมพ์ใหญ่"
    )] = False,
):
    """ทักทายด้วย Annotated syntax"""
    message = f"สวัสดี {name}!"
    if uppercase:
        message = message.upper()
    for _ in range(count):
        typer.echo(message)

if __name__ == "__main__":
    app()
```

### 3.2 Typer ขั้นสูงด้วย Enum และ Callbacks

```python
import typer
from enum import Enum
from typing import Optional, Annotated, List
from pathlib import Path

app = typer.Typer(help="File Manager CLI")

# === Enum สำหรับ choices ===
class OutputFormat(str, Enum):
    json = "json"
    csv = "csv"
    yaml = "yaml"
    table = "table"

class LogLevel(str, Enum):
    debug = "debug"
    info = "info"
    warning = "warning"
    error = "error"

# === Callback สำหรับ app options ===
@app.callback()
def main_callback(
    verbose: Annotated[bool, typer.Option(
        "--verbose", "-v",
        help="แสดงข้อมูลละเอียด"
    )] = False,
    log_level: Annotated[LogLevel, typer.Option(
        help="ระดับ log"
    )] = LogLevel.info,
):
    """My File Manager — จัดการไฟล์ด้วย CLI"""
    if verbose:
        typer.echo(f"[DEBUG] Log level: {log_level.value}")

# === Command ที่ใช้ Enum ===
@app.command()
def list_files(
    directory: Annotated[Path, typer.Argument(
        help="Directory ที่ต้องการแสดง",
        exists=True,
        file_okay=False,
        dir_okay=True,
    )] = Path("."),
    format: Annotated[OutputFormat, typer.Option(
        "--format", "-f",
        help="รูปแบบ output"
    )] = OutputFormat.table,
    pattern: Annotated[str, typer.Option(
        "--pattern", "-p",
        help="รูปแบบชื่อไฟล์ (glob pattern)"
    )] = "*",
):
    """แสดงรายการไฟล์"""
    files = list(directory.glob(pattern))
    
    if format == OutputFormat.table:
        typer.echo(f"{'ชื่อไฟล์':<40} {'ขนาด':>10} {'ประเภท':>10}")
        typer.echo("-" * 62)
        for f in files:
            size = f.stat().st_size if f.is_file() else 0
            ftype = "ไฟล์" if f.is_file() else "โฟลเดอร์"
            typer.echo(f"{f.name:<40} {size:>10,} {ftype:>10}")
    elif format == OutputFormat.json:
        import json
        data = [{"name": f.name, "size": f.stat().st_size if f.is_file() else 0} 
                for f in files]
        typer.echo(json.dumps(data, ensure_ascii=False, indent=2))

# === Command ที่มีหลาย Arguments ===
@app.command()
def copy(
    sources: Annotated[List[Path], typer.Argument(
        help="ไฟล์ต้นทาง (หลายไฟล์ได้)"
    )],
    destination: Annotated[Path, typer.Option(
        "--dest", "-d",
        help="โฟลเดอร์ปลายทาง"
    )],
    overwrite: Annotated[bool, typer.Option(
        "--overwrite/--no-overwrite",
        help="เขียนทับไฟล์ที่มีอยู่แล้ว"
    )] = False,
    dry_run: Annotated[bool, typer.Option(
        "--dry-run",
        help="แสดงเฉพาะสิ่งที่จะทำ ไม่ทำจริง"
    )] = False,
):
    """คัดลอกไฟล์หลายไฟล์พร้อมกัน"""
    for source in sources:
        dest_file = destination / source.name
        if dry_run:
            typer.echo(f"[DRY RUN] จะคัดลอก {source} → {dest_file}")
        else:
            typer.echo(f"คัดลอก {source} → {dest_file}")

if __name__ == "__main__":
    app()
```

### 3.3 Typer Sub-Applications

```python
import typer

# === Main app ===
app = typer.Typer(help="Project Manager")

# === Sub apps ===
users_app = typer.Typer(help="จัดการผู้ใช้")
projects_app = typer.Typer(help="จัดการโปรเจกต์")
tasks_app = typer.Typer(help="จัดการงาน")

# เพิ่ม sub apps เข้า main app
app.add_typer(users_app, name="user", help="คำสั่งเกี่ยวกับผู้ใช้")
app.add_typer(projects_app, name="project", help="คำสั่งเกี่ยวกับโปรเจกต์")
app.add_typer(tasks_app, name="task", help="คำสั่งเกี่ยวกับงาน")

# === User commands ===
@users_app.command("create")
def user_create(
    name: str,
    email: str,
    admin: bool = False
):
    """สร้างผู้ใช้ใหม่"""
    role = "admin" if admin else "user"
    typer.echo(f"สร้างผู้ใช้: {name} ({email}) [{role}]")

@users_app.command("delete")
def user_delete(user_id: int, force: bool = False):
    """ลบผู้ใช้"""
    if not force:
        typer.confirm(f"แน่ใจหรือไม่ที่จะลบผู้ใช้ #{user_id}?", abort=True)
    typer.echo(f"ลบผู้ใช้ #{user_id} แล้ว")

# === Project commands ===
@projects_app.command("create")
def project_create(name: str, description: str = ""):
    """สร้างโปรเจกต์ใหม่"""
    typer.echo(f"สร้างโปรเจกต์: {name}")

@projects_app.command("list")
def project_list():
    """แสดงรายการโปรเจกต์"""
    typer.echo("รายการโปรเจกต์ทั้งหมด")

# รัน:
# python script.py user create สมชาย somchai@example.com --admin
# python script.py project create "My Project"
# python script.py task --help
```

---

## 4. Rich Library — Beautiful Terminal Output

Rich ช่วยสร้าง output ที่สวยงามในเทอร์มินัล

```bash
# ติดตั้ง rich
pip install rich
```

### 4.1 Console และ Basic Output

```python
from rich.console import Console
from rich.theme import Theme
from rich.style import Style

# === สร้าง Console ===
# Console หลักสำหรับ stdout
console = Console()

# Console สำหรับ stderr (errors)
error_console = Console(stderr=True, style="bold red")

# Console พร้อม custom theme
custom_theme = Theme({
    "info":    "cyan",
    "success": "bold green",
    "warning": "bold yellow",
    "danger":  "bold red",
    "path":    "underline blue",
    "code":    "bold magenta on grey23",
})
themed_console = Console(theme=custom_theme)

# === การแสดงผลพื้นฐาน ===
def demo_basic_output():
    # Markup ด้วย [style]text[/style]
    console.print("[bold]ข้อความหนา[/bold]")
    console.print("[italic]ข้อความเอียง[/italic]")
    console.print("[bold italic red]ข้อความหนา เอียง แดง[/bold italic red]")
    
    # Colors
    console.print("[red]แดง[/red] [green]เขียว[/green] [blue]น้ำเงิน[/blue]")
    console.print("[#FF6B6B]สีเฉพาะ hex[/#FF6B6B]")
    
    # Background colors
    console.print("[white on blue]ตัวอักษรขาว พื้นหลังน้ำเงิน[/white on blue]")
    
    # Style object
    my_style = Style(color="cyan", bold=True, underline=True)
    console.print("ข้อความ styled", style=my_style)
    
    # Emoji
    console.print(":rocket: เริ่มต้น :tada:")
    console.print(":checkmark: สำเร็จ :x: ล้มเหลว")

# === Log methods ===
def demo_log_methods():
    console.log("Log message พร้อม timestamp และ location")
    console.log("[bold green]Success[/bold green]")
    
    # Rule (เส้นคั่น)
    console.rule("[bold red]Section Header[/bold red]")
    console.rule()  # เส้นว่าง
    
    # Print JSON
    import json
    data = {"name": "สมชาย", "age": 30, "city": "กรุงเทพ"}
    console.print_json(json.dumps(data, ensure_ascii=False))

demo_basic_output()
demo_log_methods()
```

### 4.2 Tables

```python
from rich.console import Console
from rich.table import Table
from rich import box

console = Console()

# === Table พื้นฐาน ===
def create_user_table(users):
    """สร้างตารางแสดงข้อมูลผู้ใช้"""
    table = Table(
        title="รายชื่อผู้ใช้",
        title_style="bold magenta",
        show_header=True,
        header_style="bold cyan",
        box=box.ROUNDED,           # รูปแบบ border
        border_style="blue",
        row_styles=["", "dim"],    # สลับสีแถว
        show_lines=True,           # แสดงเส้นแบ่งแถว
        padding=(0, 1),            # padding ใน cells
    )
    
    # เพิ่ม columns
    table.add_column("#", style="dim", width=5, justify="right")
    table.add_column("ชื่อ", style="bold white", min_width=20)
    table.add_column("Email", style="cyan")
    table.add_column("Role", justify="center")
    table.add_column("สถานะ", justify="center")
    table.add_column("เข้าระบบล่าสุด", justify="right")
    
    # เพิ่ม rows
    for i, user in enumerate(users, 1):
        # สีตาม role
        role_style = {
            "admin": "[bold red]Admin[/bold red]",
            "editor": "[yellow]Editor[/yellow]",
            "viewer": "[green]Viewer[/green]",
        }.get(user["role"], user["role"])
        
        # Status icon
        status = ":green_circle: ออนไลน์" if user["active"] else ":red_circle: ออฟไลน์"
        
        table.add_row(
            str(i),
            user["name"],
            user["email"],
            role_style,
            status,
            user.get("last_login", "ไม่เคย")
        )
    
    return table

# ข้อมูลตัวอย่าง
users = [
    {"name": "สมชาย ใจดี", "email": "somchai@example.com", "role": "admin", "active": True, "last_login": "2024-01-15"},
    {"name": "สมหญิง รักเรียน", "email": "somying@example.com", "role": "editor", "active": True, "last_login": "2024-01-14"},
    {"name": "ประทีป สว่างใจ", "email": "prateep@example.com", "role": "viewer", "active": False, "last_login": "2024-01-10"},
    {"name": "มาลี ดอกไม้", "email": "malee@example.com", "role": "editor", "active": True, "last_login": "2024-01-15"},
]

table = create_user_table(users)
console.print(table)

# === Nested Table ===
def create_stats_table():
    outer = Table(show_header=False, box=box.SIMPLE)
    outer.add_column("Metric")
    outer.add_column("Value")
    
    # Inner table สำหรับ breakdown
    inner = Table(box=box.MINIMAL)
    inner.add_column("ประเภท")
    inner.add_column("จำนวน")
    inner.add_row("Admin", "5")
    inner.add_row("Editor", "20")
    inner.add_row("Viewer", "75")
    
    outer.add_row("ผู้ใช้ทั้งหมด", "100")
    outer.add_row("การแบ่งตาม Role", inner)  # ใส่ตารางใน cell
    
    return outer
```

### 4.3 Progress Bars และ Spinners

```python
from rich.console import Console
from rich.progress import (
    Progress, BarColumn, TimeRemainingColumn,
    TimeElapsedColumn, SpinnerColumn, TextColumn,
    DownloadColumn, TransferSpeedColumn, MofNCompleteColumn,
    track
)
import time

console = Console()

# === วิธีที่ 1: ใช้ track() (ง่ายที่สุด) ===
def simple_progress():
    """Progress bar แบบง่าย"""
    items = list(range(100))
    
    for item in track(items, description="กำลังประมวลผล..."):
        time.sleep(0.02)  # จำลองการทำงาน
    
    console.print("[green]เสร็จสิ้น![/green]")

# === วิธีที่ 2: Progress context manager ===
def advanced_progress():
    """Progress bar ที่ปรับแต่งได้"""
    with Progress(
        SpinnerColumn(),                          # Spinner ด้านซ้าย
        TextColumn("[bold blue]{task.description}"),  # คำอธิบาย
        BarColumn(bar_width=40),                  # Progress bar
        MofNCompleteColumn(),                     # X/Y
        "•",
        TimeElapsedColumn(),                      # เวลาที่ผ่านไป
        "•",
        TimeRemainingColumn(),                    # เวลาที่เหลือ
        console=console
    ) as progress:
        
        # Task 1: ดาวน์โหลดข้อมูล
        task1 = progress.add_task("ดาวน์โหลดข้อมูล...", total=100)
        # Task 2: ประมวลผล
        task2 = progress.add_task("ประมวลผล...", total=50)
        # Task 3: บันทึก
        task3 = progress.add_task("บันทึกผล...", total=200, visible=False)
        
        while not progress.finished:
            time.sleep(0.05)
            progress.update(task1, advance=1)
            
            if progress.tasks[0].percentage > 50:
                progress.update(task2, advance=1)
            
            if progress.tasks[0].completed:
                progress.update(task3, visible=True)
                progress.update(task3, advance=2)

# === Spinner สำหรับงานที่ไม่รู้ความคืบหน้า ===
def spinner_demo():
    """แสดง spinner ระหว่างรอ"""
    with console.status(
        "[bold green]กำลังเชื่อมต่อ database...",
        spinner="dots"          # รูปแบบ spinner: dots, line, bouncingBar ฯลฯ
    ):
        time.sleep(2)           # จำลองการเชื่อมต่อ
    
    console.print("[green]✓[/green] เชื่อมต่อสำเร็จ")
    
    # Spinner types: dots, line, bouncingBar, bouncingBall, arc, etc.
    spinners = ["dots", "line", "arc", "bouncingBar"]
    for spinner in spinners:
        with console.status(f"Spinner: {spinner}", spinner=spinner):
            time.sleep(1)
```

### 4.4 Syntax Highlighting และ Markdown

```python
from rich.console import Console
from rich.syntax import Syntax
from rich.markdown import Markdown
from rich.panel import Panel
from rich.columns import Columns
from rich.text import Text
from rich import inspect

console = Console()

# === Syntax Highlighting ===
def demo_syntax():
    """แสดง code พร้อม syntax highlighting"""
    python_code = '''
def fibonacci(n: int) -> list[int]:
    """สร้าง Fibonacci sequence"""
    if n <= 0:
        return []
    elif n == 1:
        return [0]
    
    fib = [0, 1]
    while len(fib) < n:
        fib.append(fib[-1] + fib[-2])
    return fib

# ทดสอบ
result = fibonacci(10)
print(result)  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
'''
    
    # สร้าง Syntax object
    syntax = Syntax(
        python_code,
        "python",                   # language
        theme="monokai",            # theme: monokai, github-dark, one-dark, etc.
        line_numbers=True,          # แสดงเลขบรรทัด
        start_line=1,               # เริ่มต้นจากบรรทัดไหน
        highlight_lines={4, 5, 6},  # ไฮไลต์บรรทัดที่ระบุ
        word_wrap=True,             # ตัดบรรทัดอัตโนมัติ
    )
    
    # แสดงใน Panel
    console.print(Panel(syntax, title="Python Code", border_style="green"))
    
    # อ่านจากไฟล์
    # syntax_from_file = Syntax.from_path("script.py", line_numbers=True)

# === Markdown ===
def demo_markdown():
    """แสดง Markdown"""
    md_text = """
# หัวข้อหลัก

## หัวข้อรอง

ข้อความปกติ **ตัวหนา** และ *ตัวเอียง* และ `code inline`

### รายการ
- รายการที่ 1
- รายการที่ 2
  - รายการย่อย
  - รายการย่อยอีกรายการ
- รายการที่ 3

### Code Block
```python
print("สวัสดีโลก!")
```

### ตาราง
| ชื่อ | อายุ | เมือง |
|------|------|-------|
| สมชาย | 30 | กรุงเทพ |
| สมหญิง | 25 | เชียงใหม่ |

> blockquote: นี่คือข้อความอ้างอิง
"""
    
    markdown = Markdown(md_text)
    console.print(markdown)

# === Panel ===
def demo_panels():
    """แสดง Panels"""
    # Panel พื้นฐาน
    console.print(Panel("เนื้อหาใน panel", title="หัวข้อ", border_style="blue"))
    
    # Panel พร้อม subtitle
    console.print(Panel(
        "ข้อมูลสำคัญที่ต้องการเน้น",
        title="[bold red]Warning[/bold red]",
        subtitle="[dim]คลิกเพื่อปิด[/dim]",
        border_style="red",
        padding=(1, 2)   # padding (vertical, horizontal)
    ))
    
    # Panel แสดง nested
    inner = Panel("ข้อมูลภายใน", border_style="green")
    outer = Panel(inner, title="กล่องนอก", border_style="blue")
    console.print(outer)

# === Columns Layout ===
def demo_columns():
    """แสดงหลาย items แบบหลายคอลัมน์"""
    items = [
        Panel(f"Item {i}\nข้อมูล {i}", border_style="cyan")
        for i in range(6)
    ]
    
    console.print(Columns(items, equal=True, expand=True))

# === inspect() สำหรับ debugging ===
class MyClass:
    """คลาสตัวอย่าง"""
    name = "ทดสอบ"
    value = 42
    
    def my_method(self):
        """เมธอดตัวอย่าง"""
        return self.value

# inspect(MyClass(), methods=True)  # แสดง attributes และ methods ทั้งหมด
```

---

## 5. สร้าง CLI Tool ครบวงจร: Task Manager

ตัวอย่างการสร้าง Task Manager แบบ CLI ที่ใช้ทุกอย่างที่เรียนมา

```python
# task_manager.py — CLI Task Manager

import typer
import json
from pathlib import Path
from datetime import datetime, date
from enum import Enum
from typing import Optional, Annotated, List
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich import box
from rich.prompt import Prompt, Confirm
from rich.progress import track

# === App Setup ===
app = typer.Typer(
    name="tasks",
    help="Task Manager — จัดการงานด้วย CLI",
    add_completion=True,
    rich_markup_mode="rich"
)
console = Console()

# === Data Models ===
class Priority(str, Enum):
    low = "low"
    medium = "medium"
    high = "high"
    urgent = "urgent"

class Status(str, Enum):
    todo = "todo"
    in_progress = "in_progress"
    done = "done"
    cancelled = "cancelled"

# === Data Storage ===
DATA_FILE = Path.home() / ".tasks" / "tasks.json"

def load_tasks() -> list[dict]:
    """โหลด tasks จากไฟล์"""
    if not DATA_FILE.exists():
        return []
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except (json.JSONDecodeError, IOError):
        return []

def save_tasks(tasks: list[dict]) -> None:
    """บันทึก tasks ลงไฟล์"""
    DATA_FILE.parent.mkdir(parents=True, exist_ok=True)
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(tasks, f, ensure_ascii=False, indent=2, default=str)

def get_next_id(tasks: list[dict]) -> int:
    """หา ID ถัดไป"""
    if not tasks:
        return 1
    return max(t["id"] for t in tasks) + 1

# === Helper Functions ===
def priority_color(priority: str) -> str:
    colors = {
        "low":    "dim",
        "medium": "yellow",
        "high":   "red",
        "urgent": "bold red on dark_red"
    }
    return colors.get(priority, "white")

def status_color(status: str) -> str:
    colors = {
        "todo":        "white",
        "in_progress": "bold blue",
        "done":        "bold green",
        "cancelled":   "strikethrough dim"
    }
    return colors.get(status, "white")

def status_icon(status: str) -> str:
    icons = {
        "todo":        "⬜",
        "in_progress": "🔄",
        "done":        "✅",
        "cancelled":   "❌"
    }
    return icons.get(status, "❓")

# === Commands ===

@app.command("add")
def add_task(
    title: Annotated[str, typer.Argument(help="ชื่องาน")],
    description: Annotated[Optional[str], typer.Option(
        "--desc", "-d", help="คำอธิบายงาน"
    )] = None,
    priority: Annotated[Priority, typer.Option(
        "--priority", "-p", help="ความสำคัญ"
    )] = Priority.medium,
    due_date: Annotated[Optional[str], typer.Option(
        "--due", help="วันครบกำหนด (YYYY-MM-DD)"
    )] = None,
    tags: Annotated[Optional[List[str]], typer.Option(
        "--tag", "-t", help="Tags (ใส่ได้หลายอัน)"
    )] = None,
):
    """➕ เพิ่มงานใหม่"""
    tasks = load_tasks()
    
    task = {
        "id": get_next_id(tasks),
        "title": title,
        "description": description or "",
        "priority": priority.value,
        "status": Status.todo.value,
        "due_date": due_date,
        "tags": tags or [],
        "created_at": datetime.now().isoformat(),
        "updated_at": datetime.now().isoformat(),
    }
    
    tasks.append(task)
    save_tasks(tasks)
    
    console.print(Panel(
        f"[bold]#{task['id']}[/bold] {task['title']}\n"
        f"Priority: [{priority_color(priority.value)}]{priority.value}[/{priority_color(priority.value)}]\n"
        f"Due: {due_date or 'ไม่กำหนด'}\n"
        f"Tags: {', '.join(tags) if tags else 'ไม่มี'}",
        title="[green]✅ เพิ่มงานสำเร็จ[/green]",
        border_style="green"
    ))

@app.command("list")
def list_tasks(
    status: Annotated[Optional[Status], typer.Option(
        "--status", "-s", help="กรองตาม status"
    )] = None,
    priority: Annotated[Optional[Priority], typer.Option(
        "--priority", "-p", help="กรองตาม priority"
    )] = None,
    tag: Annotated[Optional[str], typer.Option(
        "--tag", "-t", help="กรองตาม tag"
    )] = None,
    show_all: Annotated[bool, typer.Option(
        "--all", "-a", help="แสดงทั้งหมดรวม cancelled"
    )] = False,
):
    """📋 แสดงรายการงานทั้งหมด"""
    tasks = load_tasks()
    
    # Filter
    filtered = tasks
    if not show_all:
        filtered = [t for t in filtered if t["status"] != "cancelled"]
    if status:
        filtered = [t for t in filtered if t["status"] == status.value]
    if priority:
        filtered = [t for t in filtered if t["priority"] == priority.value]
    if tag:
        filtered = [t for t in filtered if tag in t.get("tags", [])]
    
    if not filtered:
        console.print("[yellow]ไม่พบงานที่ตรงกับเงื่อนไข[/yellow]")
        return
    
    # Sort by priority then id
    priority_order = {"urgent": 0, "high": 1, "medium": 2, "low": 3}
    filtered.sort(key=lambda t: (t["status"] == "done", priority_order.get(t["priority"], 4), t["id"]))
    
    # สร้างตาราง
    table = Table(
        title=f"📋 Tasks ({len(filtered)} รายการ)",
        box=box.ROUNDED,
        border_style="blue",
        show_header=True,
        header_style="bold cyan",
    )
    
    table.add_column("ID", width=5, justify="right", style="dim")
    table.add_column("สถานะ", width=4, justify="center")
    table.add_column("งาน", min_width=30)
    table.add_column("Priority", width=10, justify="center")
    table.add_column("Due Date", width=12, justify="right")
    table.add_column("Tags", width=20)
    
    for task in filtered:
        p = task["priority"]
        s = task["status"]
        
        # ตรวจสอบ overdue
        due_str = task.get("due_date", "") or ""
        due_display = due_str
        if due_str:
            try:
                due_dt = date.fromisoformat(due_str)
                if due_dt < date.today() and s not in ("done", "cancelled"):
                    due_display = f"[bold red]{due_str} ⚠[/bold red]"
            except ValueError:
                pass
        
        title_style = status_color(s)
        
        table.add_row(
            str(task["id"]),
            status_icon(s),
            f"[{title_style}]{task['title']}[/{title_style}]",
            f"[{priority_color(p)}]{p}[/{priority_color(p)}]",
            due_display or "[dim]-[/dim]",
            ", ".join(task.get("tags", [])) or "[dim]-[/dim]"
        )
    
    console.print(table)
    
    # Summary
    done = sum(1 for t in filtered if t["status"] == "done")
    total = len(filtered)
    pct = (done / total * 100) if total > 0 else 0
    console.print(f"\n[dim]เสร็จแล้ว: {done}/{total} ({pct:.0f}%)[/dim]")

@app.command("done")
def complete_task(
    task_id: Annotated[int, typer.Argument(help="ID ของงาน")],
):
    """✅ ทำเครื่องหมายงานว่าเสร็จแล้ว"""
    tasks = load_tasks()
    
    task = next((t for t in tasks if t["id"] == task_id), None)
    if not task:
        console.print(f"[red]ไม่พบงาน #{task_id}[/red]")
        raise typer.Exit(1)
    
    task["status"] = Status.done.value
    task["updated_at"] = datetime.now().isoformat()
    task["completed_at"] = datetime.now().isoformat()
    save_tasks(tasks)
    
    console.print(f"[green]✅ งาน #{task_id} '{task['title']}' เสร็จสิ้นแล้ว![/green]")

@app.command("update")
def update_task(
    task_id: Annotated[int, typer.Argument(help="ID ของงาน")],
    title: Annotated[Optional[str], typer.Option("--title", help="ชื่องานใหม่")] = None,
    status: Annotated[Optional[Status], typer.Option("--status", "-s", help="Status ใหม่")] = None,
    priority: Annotated[Optional[Priority], typer.Option("--priority", "-p", help="Priority ใหม่")] = None,
    due_date: Annotated[Optional[str], typer.Option("--due", help="วันครบกำหนดใหม่")] = None,
):
    """✏️ แก้ไขงาน"""
    tasks = load_tasks()
    
    task = next((t for t in tasks if t["id"] == task_id), None)
    if not task:
        console.print(f"[red]ไม่พบงาน #{task_id}[/red]")
        raise typer.Exit(1)
    
    # อัพเดตเฉพาะที่ระบุ
    if title: task["title"] = title
    if status: task["status"] = status.value
    if priority: task["priority"] = priority.value
    if due_date: task["due_date"] = due_date
    task["updated_at"] = datetime.now().isoformat()
    
    save_tasks(tasks)
    console.print(f"[green]✅ อัพเดตงาน #{task_id} สำเร็จ[/green]")

@app.command("delete")
def delete_task(
    task_id: Annotated[int, typer.Argument(help="ID ของงาน")],
    force: Annotated[bool, typer.Option("--force", "-f", help="ลบโดยไม่ถามยืนยัน")] = False,
):
    """🗑️ ลบงาน"""
    tasks = load_tasks()
    
    task = next((t for t in tasks if t["id"] == task_id), None)
    if not task:
        console.print(f"[red]ไม่พบงาน #{task_id}[/red]")
        raise typer.Exit(1)
    
    if not force:
        if not Confirm.ask(f"แน่ใจหรือไม่ที่จะลบงาน #{task_id} '{task['title']}'?"):
            console.print("[yellow]ยกเลิกการลบ[/yellow]")
            return
    
    tasks = [t for t in tasks if t["id"] != task_id]
    save_tasks(tasks)
    console.print(f"[green]🗑️ ลบงาน #{task_id} '{task['title']}' แล้ว[/green]")

@app.command("show")
def show_task(
    task_id: Annotated[int, typer.Argument(help="ID ของงาน")],
):
    """🔍 แสดงรายละเอียดงาน"""
    tasks = load_tasks()
    
    task = next((t for t in tasks if t["id"] == task_id), None)
    if not task:
        console.print(f"[red]ไม่พบงาน #{task_id}[/red]")
        raise typer.Exit(1)
    
    s = task["status"]
    p = task["priority"]
    
    details = (
        f"{status_icon(s)} [bold]#{task['id']}[/bold] {task['title']}\n\n"
        f"[bold]คำอธิบาย:[/bold] {task['description'] or '[dim]ไม่มี[/dim]'}\n"
        f"[bold]Status:[/bold] [{status_color(s)}]{s}[/{status_color(s)}]\n"
        f"[bold]Priority:[/bold] [{priority_color(p)}]{p}[/{priority_color(p)}]\n"
        f"[bold]Due Date:[/bold] {task.get('due_date') or '[dim]ไม่กำหนด[/dim]'}\n"
        f"[bold]Tags:[/bold] {', '.join(task.get('tags', [])) or '[dim]ไม่มี[/dim]'}\n"
        f"[bold]สร้างเมื่อ:[/bold] {task['created_at']}\n"
        f"[bold]อัพเดตเมื่อ:[/bold] {task['updated_at']}\n"
    )
    
    if "completed_at" in task:
        details += f"[bold]เสร็จเมื่อ:[/bold] {task['completed_at']}\n"
    
    console.print(Panel(details, title=f"Task #{task_id}", border_style="blue"))

@app.command("stats")
def show_stats():
    """📊 แสดงสถิติงาน"""
    tasks = load_tasks()
    
    if not tasks:
        console.print("[yellow]ยังไม่มีงาน[/yellow]")
        return
    
    # คำนวณสถิติ
    total = len(tasks)
    by_status = {}
    by_priority = {}
    
    for task in tasks:
        s = task["status"]
        p = task["priority"]
        by_status[s] = by_status.get(s, 0) + 1
        by_priority[p] = by_priority.get(p, 0) + 1
    
    # สร้างตาราง
    status_table = Table(title="ตาม Status", box=box.SIMPLE)
    status_table.add_column("Status", style="bold")
    status_table.add_column("จำนวน", justify="right")
    status_table.add_column("เปอร์เซ็นต์", justify="right")
    
    for s, count in sorted(by_status.items()):
        pct = count / total * 100
        status_table.add_row(
            f"[{status_color(s)}]{status_icon(s)} {s}[/{status_color(s)}]",
            str(count),
            f"{pct:.1f}%"
        )
    
    priority_table = Table(title="ตาม Priority", box=box.SIMPLE)
    priority_table.add_column("Priority", style="bold")
    priority_table.add_column("จำนวน", justify="right")
    
    for p, count in sorted(by_priority.items(), key=lambda x: {"urgent":0,"high":1,"medium":2,"low":3}.get(x[0], 4)):
        priority_table.add_row(
            f"[{priority_color(p)}]{p}[/{priority_color(p)}]",
            str(count)
        )
    
    from rich.columns import Columns
    console.print(f"\n[bold]📊 สถิติงานทั้งหมด ({total} งาน)[/bold]\n")
    console.print(Columns([status_table, priority_table], expand=True))

@app.command("clear")
def clear_done(
    force: Annotated[bool, typer.Option("--force", "-f")] = False,
):
    """🧹 ลบงานที่เสร็จแล้วทั้งหมด"""
    tasks = load_tasks()
    done_tasks = [t for t in tasks if t["status"] == "done"]
    
    if not done_tasks:
        console.print("[yellow]ไม่มีงานที่เสร็จแล้ว[/yellow]")
        return
    
    if not force:
        console.print(f"พบงานที่เสร็จแล้ว {len(done_tasks)} งาน:")
        for t in done_tasks:
            console.print(f"  • #{t['id']} {t['title']}")
        
        if not Confirm.ask("ยืนยันการลบ?"):
            console.print("[yellow]ยกเลิก[/yellow]")
            return
    
    remaining = [t for t in tasks if t["status"] != "done"]
    save_tasks(remaining)
    console.print(f"[green]🧹 ลบงานที่เสร็จแล้ว {len(done_tasks)} งาน[/green]")

if __name__ == "__main__":
    app()
```

### 5.1 วิธีใช้งาน Task Manager

```bash
# เพิ่มงาน
python task_manager.py add "เขียน unit tests" --priority high --due 2024-02-01 --tag python --tag testing

# แสดงรายการ
python task_manager.py list
python task_manager.py list --priority high
python task_manager.py list --status todo
python task_manager.py list --tag python

# ดูรายละเอียด
python task_manager.py show 1

# อัพเดตงาน
python task_manager.py update 1 --status in_progress
python task_manager.py update 1 --priority urgent

# ทำเครื่องหมายเสร็จ
python task_manager.py done 1

# ลบงาน
python task_manager.py delete 2
python task_manager.py delete 3 --force

# สถิติ
python task_manager.py stats

# ลบงานที่เสร็จทั้งหมด
python task_manager.py clear
```

---

## 6. เทคนิคเพิ่มเติมสำหรับ CLI Tools

### 6.1 Configuration Management

```python
import typer
import json
from pathlib import Path
from rich.console import Console

console = Console()
app = typer.Typer()

# === Config file management ===
CONFIG_DIR = Path.home() / ".config" / "myapp"
CONFIG_FILE = CONFIG_DIR / "config.json"

def get_config() -> dict:
    """โหลด config"""
    if not CONFIG_FILE.exists():
        return {}
    with open(CONFIG_FILE) as f:
        return json.load(f)

def save_config(config: dict):
    """บันทึก config"""
    CONFIG_DIR.mkdir(parents=True, exist_ok=True)
    with open(CONFIG_FILE, "w") as f:
        json.dump(config, f, indent=2)

@app.command("config")
def manage_config(
    key: str = typer.Argument(None),
    value: str = typer.Argument(None),
    unset: bool = typer.Option(False, "--unset", help="ลบ config")
):
    """จัดการ configuration"""
    config = get_config()
    
    if key is None:
        # แสดง config ทั้งหมด
        if not config:
            console.print("[dim]ไม่มี config[/dim]")
        for k, v in config.items():
            console.print(f"{k} = {v}")
        return
    
    if unset:
        config.pop(key, None)
        save_config(config)
        console.print(f"[green]ลบ config '{key}' แล้ว[/green]")
        return
    
    if value is None:
        # แสดงค่าเดียว
        if key in config:
            console.print(f"{key} = {config[key]}")
        else:
            console.print(f"[yellow]ไม่พบ config '{key}'[/yellow]")
        return
    
    # ตั้งค่า
    config[key] = value
    save_config(config)
    console.print(f"[green]ตั้งค่า {key} = {value}[/green]")
```

### 6.2 Shell Completion

```python
# click completion
import click

@click.group()
def cli():
    pass

@cli.command()
@click.argument("environment",
    shell_complete=lambda ctx, param, incomplete: [
        click.shell_completion.CompletionItem(e)
        for e in ["development", "staging", "production"]
        if e.startswith(incomplete)
    ]
)
def deploy(environment):
    """Deploy to environment"""
    click.echo(f"Deploying to {environment}")

# typer completion (อัตโนมัติจาก type hints)
# python script.py --install-completion bash
# python script.py --install-completion zsh
# python script.py --show-completion bash
```

### 6.3 Testing CLI Tools

```python
# test_cli.py — ทดสอบ CLI tools

from click.testing import CliRunner
from typer.testing import CliRunner as TyperRunner
import pytest

# === ทดสอบ Click CLI ===
from my_click_app import cli

def test_click_hello():
    runner = CliRunner()
    result = runner.invoke(cli, ["hello", "สมชาย"])
    assert result.exit_code == 0
    assert "สมชาย" in result.output

def test_click_help():
    runner = CliRunner()
    result = runner.invoke(cli, ["--help"])
    assert result.exit_code == 0
    assert "help" in result.output.lower()

# === ทดสอบ Typer CLI ===
from task_manager import app

runner = TyperRunner()

def test_add_task():
    """ทดสอบการเพิ่มงาน"""
    result = runner.invoke(app, ["add", "ทดสอบงาน"])
    assert result.exit_code == 0
    assert "สำเร็จ" in result.stdout

def test_list_empty():
    """ทดสอบการแสดงรายการงานเมื่อไม่มีงาน"""
    # ใช้ temp directory สำหรับ test
    import tempfile
    import os
    
    with tempfile.TemporaryDirectory() as tmpdir:
        # Override data file path
        result = runner.invoke(app, ["list"])
        assert result.exit_code == 0

def test_invalid_priority():
    """ทดสอบ priority ที่ไม่ถูกต้อง"""
    result = runner.invoke(app, ["add", "งาน", "--priority", "invalid"])
    assert result.exit_code != 0
```

---

## 7. สรุป Part 047

✅ **argparse** — Built-in Python CLI parser พร้อม ArgumentParser, add_argument, subparsers, types, choices

✅ **click** — Decorator-based CLI framework พร้อม @click.command, @click.option, groups, context, colors

✅ **typer** — Modern CLI ด้วย type hints พร้อม Annotated, sub-apps, Enum choices

✅ **rich** — Beautiful terminal output พร้อม Console, Table, Progress, Panel, Syntax highlighting

✅ **Complete CLI Tool** — Task Manager ที่ใช้ทุกอย่างร่วมกัน พร้อม CRUD operations และ data persistence

## ➡️ ถัดไป: Part 048 - Performance Optimization

*Part 047/100+ | Python Course - Beginner to World-Class*
