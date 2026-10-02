# Part 047: CLI Applications
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ argparse สร้าง CLI tools
- ใช้ click library สำหรับ CLI ที่ซับซ้อน
- ใช้ typer (Click + Type Hints)
- สร้าง command groups
- ตั้งค่า options และ arguments
- เพิ่มสี output ด้วย rich
- แสดง progress bars

---

## 1. argparse - Built-in Python

```python
import argparse
import sys
from pathlib import Path

# === Basic argparse ===
def create_basic_parser():
    parser = argparse.ArgumentParser(
        description="File processing tool",
        epilog="Example: python script.py input.txt --output out.txt --verbose"
    )
    
    # Positional argument (required)
    parser.add_argument(
        "input_file",
        type=str,
        help="Input file path"
    )
    
    # Optional argument
    parser.add_argument(
        "--output", "-o",
        type=str,
        default=None,
        help="Output file path (default: stdout)"
    )
    
    # Flag (boolean)
    parser.add_argument(
        "--verbose", "-v",
        action="store_true",
        help="Enable verbose output"
    )
    
    # Numeric argument
    parser.add_argument(
        "--lines", "-n",
        type=int,
        default=10,
        help="Number of lines to process (default: 10)"
    )
    
    # Choices
    parser.add_argument(
        "--format",
        choices=["json", "csv", "xml"],
        default="json",
        help="Output format"
    )
    
    # Multiple values
    parser.add_argument(
        "--tags",
        nargs="+",  # หนึ่งหรือมากกว่า
        help="Tags to apply"
    )
    
    # Multiple values แบบ fixed count
    parser.add_argument(
        "--range",
        nargs=2,
        type=int,
        metavar=("START", "END"),
        help="Range: START END"
    )
    
    return parser


# ทดสอบ
def demo_argparse():
    parser = create_basic_parser()
    
    # Simulate command line args
    test_args = [
        "input.txt",
        "--output", "output.txt",
        "--verbose",
        "--lines", "20",
        "--format", "csv",
        "--tags", "python", "tutorial",
        "--range", "1", "100"
    ]
    
    args = parser.parse_args(test_args)
    
    print(f"Input: {args.input_file}")
    print(f"Output: {args.output}")
    print(f"Verbose: {args.verbose}")
    print(f"Lines: {args.lines}")
    print(f"Format: {args.format}")
    print(f"Tags: {args.tags}")
    print(f"Range: {args.range}")


# === Subcommands (subparsers) ===
def create_cli_with_subcommands():
    parser = argparse.ArgumentParser(description="Task Manager CLI")
    
    subparsers = parser.add_subparsers(
        dest="command",
        help="Commands"
    )
    
    # === add command ===
    add_parser = subparsers.add_parser("add", help="Add a task")
    add_parser.add_argument("title", help="Task title")
    add_parser.add_argument("--priority", choices=["low", "medium", "high"], default="medium")
    add_parser.add_argument("--due", help="Due date (YYYY-MM-DD)")
    
    # === list command ===
    list_parser = subparsers.add_parser("list", help="List tasks")
    list_parser.add_argument("--status", choices=["all", "pending", "done"], default="all")
    list_parser.add_argument("--priority", choices=["low", "medium", "high"])
    
    # === complete command ===
    complete_parser = subparsers.add_parser("complete", help="Mark task as done")
    complete_parser.add_argument("task_id", type=int, help="Task ID")
    
    # === delete command ===
    delete_parser = subparsers.add_parser("delete", help="Delete a task")
    delete_parser.add_argument("task_id", type=int, help="Task ID")
    delete_parser.add_argument("--force", "-f", action="store_true", help="Skip confirmation")
    
    return parser


def handle_task_command(args):
    """Handle task commands"""
    if args.command == "add":
        print(f"Adding task: '{args.title}' (priority: {args.priority})")
    elif args.command == "list":
        print(f"Listing tasks (status: {args.status})")
    elif args.command == "complete":
        print(f"Marking task {args.task_id} as done")
    elif args.command == "delete":
        if not args.force:
            confirm = input(f"Delete task {args.task_id}? [y/N]: ")
            if confirm.lower() != "y":
                print("Cancelled")
                return
        print(f"Deleted task {args.task_id}")
    else:
        print("Please specify a command")


# ทดสอบ
demo_argparse()

task_parser = create_cli_with_subcommands()
args = task_parser.parse_args(["add", "Buy groceries", "--priority", "high"])
handle_task_command(args)
```

---

## 2. Click Library

```bash
pip install click
```

```python
import click
import json
import sys
from pathlib import Path

# === Basic Click Commands ===
@click.command()
@click.argument("name")
@click.option("--greeting", "-g", default="Hello", help="Greeting to use")
@click.option("--count", "-n", default=1, type=int, help="Number of times")
@click.option("--upper/--no-upper", default=False, help="Uppercase output")
def greet(name: str, greeting: str, count: int, upper: bool):
    """Greet someone. NAME is the person to greet."""
    for _ in range(count):
        message = f"{greeting}, {name}!"
        if upper:
            message = message.upper()
        click.echo(message)


# === Click with Types ===
@click.command()
@click.argument("input_file", type=click.Path(exists=True, readable=True))
@click.argument("output_file", type=click.Path(writable=True))
@click.option(
    "--format", "-f",
    type=click.Choice(["json", "csv", "xml"], case_sensitive=False),
    default="json",
    show_default=True
)
@click.option("--lines", "-n", default=-1, type=int, help="Limit lines (-1 for all)")
@click.option("--verbose", "-v", is_flag=True, help="Verbose output")
def convert(input_file: str, output_file: str, format: str, lines: int, verbose: bool):
    """Convert input file to specified format."""
    if verbose:
        click.echo(f"Converting {input_file} to {output_file} as {format}")
    
    # Read input
    with open(input_file) as f:
        content = f.readlines()
    
    if lines > 0:
        content = content[:lines]
    
    click.echo(f"Processed {len(content)} lines")


# === Click Groups (Multiple Commands) ===
@click.group()
@click.option("--debug/--no-debug", default=False, envvar="DEBUG")
@click.pass_context
def cli(ctx, debug: bool):
    """Task Manager - manage your tasks from CLI"""
    # ctx.obj เก็บ shared state ระหว่าง commands
    ctx.ensure_object(dict)
    ctx.obj["DEBUG"] = debug
    ctx.obj["tasks"] = []  # In-memory storage
    
    if debug:
        click.echo("Debug mode enabled", err=True)


@cli.command()
@click.argument("title")
@click.option("--priority", 
              type=click.Choice(["low", "medium", "high"]),
              default="medium")
@click.option("--due", type=click.DateTime(formats=["%Y-%m-%d"]), default=None)
@click.pass_context
def add(ctx, title: str, priority: str, due):
    """Add a new task"""
    task = {
        "id": len(ctx.obj["tasks"]) + 1,
        "title": title,
        "priority": priority,
        "due": due.strftime("%Y-%m-%d") if due else None,
        "done": False
    }
    ctx.obj["tasks"].append(task)
    click.echo(f"✅ Added task #{task['id']}: {title}")


@cli.command()
@click.option("--status", 
              type=click.Choice(["all", "pending", "done"]),
              default="all")
@click.pass_context
def ls(ctx, status: str):
    """List tasks"""
    tasks = ctx.obj["tasks"]
    
    if status == "pending":
        tasks = [t for t in tasks if not t["done"]]
    elif status == "done":
        tasks = [t for t in tasks if t["done"]]
    
    if not tasks:
        click.echo("No tasks found")
        return
    
    for task in tasks:
        status_icon = "✓" if task["done"] else "○"
        priority_color = {
            "high": "red",
            "medium": "yellow",
            "low": "green"
        }.get(task["priority"], "white")
        
        click.echo(
            f"[{status_icon}] #{task['id']} "
            f"{click.style(task['priority'].upper(), fg=priority_color)}: "
            f"{task['title']}"
        )


@cli.command()
@click.argument("task_id", type=int)
@click.pass_context
def done(ctx, task_id: int):
    """Mark task as complete"""
    tasks = ctx.obj["tasks"]
    task = next((t for t in tasks if t["id"] == task_id), None)
    
    if not task:
        click.echo(f"Task #{task_id} not found", err=True)
        sys.exit(1)
    
    task["done"] = True
    click.echo(f"✅ Task #{task_id} marked as done!")


@cli.command()
@click.argument("task_id", type=int)
@click.confirmation_option(prompt="Are you sure you want to delete this task?")
@click.pass_context
def delete(ctx, task_id: int):
    """Delete a task"""
    tasks = ctx.obj["tasks"]
    original_count = len(tasks)
    ctx.obj["tasks"] = [t for t in tasks if t["id"] != task_id]
    
    if len(ctx.obj["tasks"]) < original_count:
        click.echo(f"Deleted task #{task_id}")
    else:
        click.echo(f"Task #{task_id} not found", err=True)


# ทดสอบ Click CLI
def demo_click():
    from click.testing import CliRunner
    
    runner = CliRunner()
    
    # Test add command
    result = runner.invoke(cli, ["add", "Buy milk", "--priority", "low"])
    print(f"Add result: {result.output}")
    
    # Test list
    result = runner.invoke(cli, ["ls"])
    print(f"List result: {result.output}")
    
    # Test with context
    result = runner.invoke(cli, ["--debug", "add", "Do laundry"])
    print(f"Debug add: {result.output}")


demo_click()
```

---

## 3. Typer - Click + Type Hints

```bash
pip install typer[all]
```

```python
import typer
from typing import Optional, List
from enum import Enum
from pathlib import Path
from datetime import date

app = typer.Typer(
    name="task-manager",
    help="Modern task manager CLI",
    add_completion=True
)

# === Enums สำหรับ choices ===
class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

class OutputFormat(str, Enum):
    TABLE = "table"
    JSON = "json"
    CSV = "csv"


# === Typer Commands ===
@app.command()
def add(
    title: str = typer.Argument(..., help="Task title"),
    priority: Priority = typer.Option(Priority.MEDIUM, "--priority", "-p"),
    due_date: Optional[date] = typer.Option(None, "--due", "-d", formats=["%Y-%m-%d"]),
    tags: Optional[List[str]] = typer.Option(None, "--tag", "-t", help="Tags"),
    verbose: bool = typer.Option(False, "--verbose", "-v"),
):
    """Add a new task to the task list."""
    
    if verbose:
        typer.echo(f"Adding task with priority: {priority}")
    
    task_info = f"📝 Added: {title}"
    if priority == Priority.HIGH:
        # Colorize high priority
        task_info = typer.style(task_info, fg=typer.colors.RED, bold=True)
    elif priority == Priority.MEDIUM:
        task_info = typer.style(task_info, fg=typer.colors.YELLOW)
    else:
        task_info = typer.style(task_info, fg=typer.colors.GREEN)
    
    typer.echo(task_info)
    
    if due_date:
        typer.echo(f"  Due: {due_date}")
    if tags:
        typer.echo(f"  Tags: {', '.join(tags)}")


@app.command()
def list_tasks(
    status: str = typer.Option("all", "--status", "-s",
                               help="Filter by status: all/pending/done"),
    priority: Optional[Priority] = typer.Option(None, "--priority", "-p"),
    format: OutputFormat = typer.Option(OutputFormat.TABLE, "--format", "-f"),
    limit: int = typer.Option(10, "--limit", "-n", min=1, max=1000),
):
    """List all tasks."""
    typer.echo(f"Showing {status} tasks (format: {format})")


@app.command()
def complete(
    task_id: int = typer.Argument(..., help="Task ID to mark complete"),
):
    """Mark a task as complete."""
    typer.echo(f"✅ Task #{task_id} marked as complete!")


@app.command()
def delete(
    task_id: int = typer.Argument(..., help="Task ID to delete"),
    force: bool = typer.Option(False, "--force", "-f", help="Skip confirmation"),
):
    """Delete a task."""
    if not force:
        confirmed = typer.confirm(f"Delete task #{task_id}?")
        if not confirmed:
            typer.echo("Cancelled")
            raise typer.Abort()
    
    typer.echo(f"🗑️  Deleted task #{task_id}")


@app.command()
def export(
    output: Path = typer.Option(
        Path("tasks.json"),
        "--output", "-o",
        help="Output file path"
    ),
    format: OutputFormat = typer.Option(OutputFormat.JSON),
):
    """Export tasks to file."""
    typer.echo(f"Exporting to {output} as {format}")


# === Nested App Groups ===
admin_app = typer.Typer(name="admin", help="Admin commands")
app.add_typer(admin_app, name="admin")


@admin_app.command("stats")
def admin_stats():
    """Show admin statistics."""
    typer.echo("Admin stats...")


@admin_app.command("clear")
def admin_clear(
    confirm: bool = typer.Option(False, "--confirm", help="Confirm clearing all tasks")
):
    """Clear all tasks (admin only)."""
    if not confirm:
        typer.echo("Use --confirm to clear all tasks")
        return
    typer.echo("Cleared all tasks!")


# ทดสอบ
def demo_typer():
    from typer.testing import CliRunner
    
    runner = CliRunner()
    
    result = runner.invoke(app, ["add", "Buy groceries", "--priority", "high"])
    print(f"Typer add: {result.output}")
    
    result = runner.invoke(app, ["list-tasks", "--format", "json"])
    print(f"Typer list: {result.output}")
    
    result = runner.invoke(app, ["--help"])
    print(f"Help: {result.output[:200]}")


demo_typer()
```

---

## 4. Rich - สวยงามใน Terminal

```bash
pip install rich
```

```python
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.progress import Progress, TaskID
from rich.prompt import Prompt, Confirm, IntPrompt
from rich.syntax import Syntax
from rich.tree import Tree
from rich.columns import Columns
from rich.text import Text
from rich import print as rprint
import time

console = Console()


# === Rich Print ===
def demo_rich_print():
    # Markup
    console.print("[bold blue]Hello[/bold blue], [italic green]World![/italic green]")
    console.print("Status: [red]ERROR[/red] | [yellow]WARNING[/yellow] | [green]OK[/green]")
    
    # Emoji
    console.print(":white_check_mark: Task completed!")
    console.print(":fire: Important message!")
    
    # Rule (divider)
    console.rule("[bold]Section Header[/bold]")
    
    # Pretty print objects
    data = {"name": "Alice", "tasks": [1, 2, 3], "active": True}
    console.print(data)


# === Tables ===
def demo_table():
    table = Table(
        title="Task List",
        show_header=True,
        header_style="bold magenta",
        border_style="blue",
        expand=True
    )
    
    # Columns
    table.add_column("ID", style="dim", width=5, justify="right")
    table.add_column("Title", style="bold", min_width=20)
    table.add_column("Priority", justify="center", width=10)
    table.add_column("Status", justify="center", width=10)
    table.add_column("Due Date", justify="right", width=12)
    
    # Rows
    tasks = [
        (1, "Buy groceries", "HIGH", "pending", "2024-01-20"),
        (2, "Do laundry", "MEDIUM", "done", "2024-01-18"),
        (3, "Read book", "LOW", "pending", "2024-01-25"),
        (4, "Exercise", "HIGH", "pending", "2024-01-19"),
    ]
    
    for task_id, title, priority, status, due in tasks:
        # แต่ง style ตาม priority
        priority_style = {
            "HIGH": "[bold red]HIGH[/bold red]",
            "MEDIUM": "[yellow]MEDIUM[/yellow]",
            "LOW": "[green]LOW[/green]",
        }.get(priority, priority)
        
        status_icon = "✅" if status == "done" else "⏳"
        
        table.add_row(
            str(task_id),
            title,
            priority_style,
            f"{status_icon} {status}",
            due
        )
    
    console.print(table)


# === Panels ===
def demo_panels():
    # Simple panel
    console.print(Panel("Hello, World!", title="Greeting"))
    
    # Panel with markup
    console.print(Panel(
        "[green]Operation successful![/green]\n"
        "Files processed: 42\n"
        "Time elapsed: 3.2s",
        title="[bold]Result[/bold]",
        subtitle="Press Enter to continue",
        border_style="green"
    ))
    
    # Error panel
    console.print(Panel(
        "[red]Connection failed![/red]\n"
        "Host: db.example.com\n"
        "Error: Connection refused",
        title="❌ Error",
        border_style="red"
    ))


# === Progress Bars ===
def demo_progress():
    tasks = [
        ("Download data", 50),
        ("Process records", 200),
        ("Generate report", 30),
    ]
    
    with Progress(
        "[progress.description]{task.description}",
        "[progress.percentage]{task.percentage:>3.0f}%",
        "•",
        "[progress.bar]{task.completed}/{task.total}",
        transient=False  # Keep progress after completion
    ) as progress:
        
        for task_name, total in tasks:
            task_id = progress.add_task(f"[cyan]{task_name}[/cyan]", total=total)
            
            for i in range(total):
                progress.advance(task_id)
                time.sleep(0.01)  # Simulate work


# === Syntax Highlighting ===
def demo_syntax():
    code = '''
def fibonacci(n: int) -> list[int]:
    """Generate Fibonacci sequence up to n."""
    if n <= 0:
        return []
    elif n == 1:
        return [0]
    
    seq = [0, 1]
    while len(seq) < n:
        seq.append(seq[-1] + seq[-2])
    return seq

result = fibonacci(10)
print(f"Fibonacci: {result}")
'''
    
    syntax = Syntax(code, "python", theme="monokai", line_numbers=True)
    console.print(syntax)


# === Tree Structure ===
def demo_tree():
    tree = Tree("🗂️  [bold]Project Structure[/bold]")
    
    src = tree.add("📁 [blue]src[/blue]")
    src.add("📄 main.py")
    src.add("📄 models.py")
    api = src.add("📁 [blue]api[/blue]")
    api.add("📄 routes.py")
    api.add("📄 schemas.py")
    
    tests = tree.add("📁 [blue]tests[/blue]")
    tests.add("📄 test_main.py")
    tests.add("📄 conftest.py")
    
    tree.add("📄 requirements.txt")
    tree.add("📄 Dockerfile")
    tree.add("📄 .env.example")
    
    console.print(tree)


# === Prompt ===
def demo_prompts():
    # Text prompt
    name = Prompt.ask("[green]What's your name?[/green]", default="World")
    console.print(f"Hello, [bold]{name}[/bold]!")
    
    # Integer prompt
    age = IntPrompt.ask("How old are you?", default=25)
    console.print(f"You are {age} years old")
    
    # Confirmation
    proceed = Confirm.ask("Do you want to continue?", default=True)
    if proceed:
        console.print("[green]Continuing...[/green]")
    else:
        console.print("[yellow]Aborted[/yellow]")
    
    # Choice prompt
    priority = Prompt.ask(
        "Select priority",
        choices=["low", "medium", "high"],
        default="medium"
    )
    console.print(f"Priority: {priority}")


# === Spinner (Loading Indicator) ===
def demo_spinner():
    from rich.spinner import Spinner
    from rich.live import Live
    
    steps = [
        ("Connecting to database", 1),
        ("Running migrations", 2),
        ("Seeding data", 1.5),
        ("Starting server", 0.5),
    ]
    
    for step_name, duration in steps:
        with console.status(f"[bold green]{step_name}..."):
            time.sleep(duration)
        console.print(f"  ✅ {step_name}")
    
    console.print("\n[bold green]Application started successfully![/bold green]")


# Run demos
print("=== Rich Demo ===")
demo_rich_print()
demo_table()
demo_panels()
demo_tree()
# demo_progress()  # Uncomment เพื่อดู progress bars
# demo_syntax()    # Uncomment เพื่อดู syntax highlighting
```

---

## 5. Complete CLI Application

```python
import typer
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.progress import Progress
from pathlib import Path
from typing import Optional, List
from enum import Enum
import json
import datetime

# App setup
app = typer.Typer(
    name="tasks",
    help="Modern task manager with rich UI",
    rich_markup_mode="rich",
    add_completion=True
)
console = Console()

# Storage
TASKS_FILE = Path.home() / ".tasks.json"


class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


def load_tasks() -> list:
    if TASKS_FILE.exists():
        return json.loads(TASKS_FILE.read_text())
    return []


def save_tasks(tasks: list):
    TASKS_FILE.write_text(json.dumps(tasks, indent=2, default=str))


def get_next_id(tasks: list) -> int:
    if not tasks:
        return 1
    return max(t["id"] for t in tasks) + 1


# === Commands ===
@app.command()
def add(
    title: str = typer.Argument(..., help="Task title"),
    priority: Priority = typer.Option(Priority.MEDIUM, "--priority", "-p"),
    tags: Optional[List[str]] = typer.Option(None, "--tag", "-t"),
):
    """[green]Add[/green] a new task."""
    tasks = load_tasks()
    
    task = {
        "id": get_next_id(tasks),
        "title": title,
        "priority": priority.value,
        "tags": tags or [],
        "done": False,
        "created_at": datetime.datetime.now().isoformat(),
    }
    
    tasks.append(task)
    save_tasks(tasks)
    
    priority_style = {
        "high": "[red]HIGH[/red]",
        "medium": "[yellow]MEDIUM[/yellow]",
        "low": "[green]LOW[/green]",
    }[priority]
    
    console.print(
        Panel(
            f"[bold]#{task['id']}[/bold] {title}\n"
            f"Priority: {priority_style}"
            + (f"\nTags: {', '.join(tags)}" if tags else ""),
            title="[green]✅ Task Added[/green]",
            border_style="green"
        )
    )


@app.command("list")
def list_tasks(
    status: str = typer.Option("pending", "--status", "-s",
                               help="all/pending/done"),
    priority: Optional[Priority] = typer.Option(None, "--priority", "-p"),
):
    """[blue]List[/blue] tasks."""
    tasks = load_tasks()
    
    # Filter
    if status == "pending":
        tasks = [t for t in tasks if not t["done"]]
    elif status == "done":
        tasks = [t for t in tasks if t["done"]]
    
    if priority:
        tasks = [t for t in tasks if t["priority"] == priority.value]
    
    if not tasks:
        console.print("[yellow]No tasks found.[/yellow]")
        return
    
    table = Table(
        title=f"Tasks ({status})",
        border_style="blue",
        header_style="bold blue"
    )
    
    table.add_column("ID", width=5, justify="right")
    table.add_column("Title", min_width=20)
    table.add_column("Priority", width=10, justify="center")
    table.add_column("Tags", width=20)
    table.add_column("Status", width=10, justify="center")
    
    for task in tasks:
        priority_text = {
            "high": "[bold red]🔴 HIGH[/bold red]",
            "medium": "[yellow]🟡 MED[/yellow]",
            "low": "[green]🟢 LOW[/green]",
        }.get(task["priority"], task["priority"])
        
        status_text = "✅ Done" if task["done"] else "⏳ Pending"
        tags_text = ", ".join(task.get("tags", [])) or "-"
        
        table.add_row(
            str(task["id"]),
            task["title"],
            priority_text,
            tags_text,
            status_text
        )
    
    console.print(table)
    console.print(f"Total: [bold]{len(tasks)}[/bold] tasks")


@app.command()
def done(
    task_id: int = typer.Argument(..., help="Task ID"),
):
    """Mark task as [green]done[/green]."""
    tasks = load_tasks()
    
    task = next((t for t in tasks if t["id"] == task_id), None)
    if not task:
        console.print(f"[red]Task #{task_id} not found[/red]")
        raise typer.Exit(1)
    
    if task["done"]:
        console.print(f"[yellow]Task #{task_id} is already done[/yellow]")
        return
    
    task["done"] = True
    task["completed_at"] = datetime.datetime.now().isoformat()
    save_tasks(tasks)
    
    console.print(f"[green]✅ Task #{task_id} '{task['title']}' marked as done![/green]")


@app.command()
def delete(
    task_id: int = typer.Argument(..., help="Task ID"),
    force: bool = typer.Option(False, "--force", "-f"),
):
    """[red]Delete[/red] a task."""
    tasks = load_tasks()
    
    task = next((t for t in tasks if t["id"] == task_id), None)
    if not task:
        console.print(f"[red]Task #{task_id} not found[/red]")
        raise typer.Exit(1)
    
    if not force:
        confirmed = typer.confirm(
            f"Delete task #{task_id} '{task['title']}'?"
        )
        if not confirmed:
            console.print("[yellow]Cancelled[/yellow]")
            return
    
    tasks = [t for t in tasks if t["id"] != task_id]
    save_tasks(tasks)
    console.print(f"[red]🗑️  Task #{task_id} deleted[/red]")


@app.command()
def stats():
    """Show [cyan]statistics[/cyan]."""
    tasks = load_tasks()
    
    if not tasks:
        console.print("[yellow]No tasks yet[/yellow]")
        return
    
    total = len(tasks)
    done = sum(1 for t in tasks if t["done"])
    pending = total - done
    
    by_priority = {}
    for t in tasks:
        p = t["priority"]
        by_priority[p] = by_priority.get(p, 0) + 1
    
    console.print(Panel(
        f"[bold]Total:[/bold] {total}\n"
        f"[green]Done:[/green] {done} ({done/total*100:.0f}%)\n"
        f"[yellow]Pending:[/yellow] {pending}\n\n"
        f"By Priority:\n"
        f"  [red]High:[/red] {by_priority.get('high', 0)}\n"
        f"  [yellow]Medium:[/yellow] {by_priority.get('medium', 0)}\n"
        f"  [green]Low:[/green] {by_priority.get('low', 0)}",
        title="[cyan]📊 Statistics[/cyan]",
        border_style="cyan"
    ))


# Entry point
if __name__ == "__main__":
    app()
```

---

## 6. สรุป Part 047

✅ **argparse** - Built-in, subcommands, ทุก argument types  
✅ **click** - Groups, callbacks, context passing  
✅ **typer** - Type hints + Click, modern Python CLI  
✅ **rich** - Colors, tables, panels, progress bars  
✅ **Complete App** - Task manager ใช้งานได้จริง  

**เปรียบเทียบ Libraries:**
- `argparse`: ไม่ต้อง install, พื้นฐาน, verbose
- `click`: Popular, decorators, groups
- `typer`: Type hints, auto-complete, รองรับ rich

**Best Practices:**
- ใส่ `--help` ที่ดีใน commands ทั้งหมด
- ใช้ error codes (0=success, 1+=error)
- Confirm destructive operations
- แสดง progress สำหรับ long operations

## ➡️ ถัดไป: Part 048 - Performance Optimization
*Part 047/100+ | Python Course - Beginner to World-Class*
