# click 완벽 가이드

> **CLI 애플리케이션 개발 프레임워크**


---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://click.palletsprojects.com |
| **GitHub** | https://github.com/pallets/click |
| **PyPI** | https://pypi.org/project/click/ |

### 한 줄 요약

**데코레이터 기반의 우아한 CLI 도구 개발 라이브러리**

---

## 설치

```bash
$ uv add click
```

---

## 기본 사용법

### 최소 예제

```python
import click

@click.command()
@click.option('--name', default='World', help='Name to greet')
def hello(name):
    """Simple program that greets NAME."""
    click.echo(f'Hello {name}!')

if __name__ == '__main__':
    hello()
```

```bash
$ python hello.py
Hello World!

$ python hello.py --name Alice
Hello Alice!

$ python hello.py --help
Usage: hello.py [OPTIONS]

  Simple program that greets NAME.

Options:
  --name TEXT  Name to greet
  --help       Show this message and exit.
```

---

## 실전 패턴

### 패턴 1: 인자와 옵션

```python
@click.command()
@click.argument('filename')  # 필수 인자
@click.option('--count', default=1, help='Number of greetings')
@click.option('--name', prompt='Your name', help='The person to greet')
def hello(filename, count, name):
    for _ in range(count):
        click.echo(f'Hello {name}!')
```

### 패턴 2: 하위 명령어 (Git 스타일)

```python
@click.group()
def cli():
    """Database management tool."""
    pass

@cli.command()
def init():
    """Initialize database."""
    click.echo('Initializing database...')

@cli.command()
@click.argument('name')
def create(name):
    """Create a new table."""
    click.echo(f'Creating table: {name}')

if __name__ == '__main__':
    cli()
```

```bash
$ python db.py init
$ python db.py create users
```

### 패턴 3: 플래그

```python
@click.command()
@click.option('--verbose', '-v', is_flag=True, help='Verbose output')
@click.option('--quiet', '-q', is_flag=True, help='Quiet mode')
def process(verbose, quiet):
    if verbose:
        click.echo('Verbose mode on')
    if quiet:
        click.echo('Quiet mode on')
```

### 패턴 4: 진행 표시줄

```python
import time

@click.command()
def process():
    items = range(100)

    with click.progressbar(items, label='Processing') as bar:
        for item in bar:
            time.sleep(0.01)  # 시뮬레이션
```

### 패턴 5: 색상

```python
@click.command()
def colorful():
    click.secho('Success!', fg='green', bold=True)
    click.secho('Warning!', fg='yellow')
    click.secho('Error!', fg='red', bold=True)
    click.echo(click.style('Info', fg='blue'))
```

### 패턴 6: 파일 처리

```python
@click.command()
@click.argument('input', type=click.File('r'))
@click.argument('output', type=click.File('w'))
def process(input, output):
    for line in input:
        output.write(line.upper())
```

---

## 복잡한 CLI 예제

```python
import click

@click.group()
@click.option('--config', type=click.Path(), help='Config file path')
@click.pass_context
def cli(ctx, config):
    """My awesome CLI tool."""
    ctx.ensure_object(dict)
    ctx.obj['config'] = config

@cli.group()
def db():
    """Database commands."""
    pass

@db.command()
@click.option('--host', default='localhost')
@click.option('--port', default=5432)
@click.pass_context
def connect(ctx, host, port):
    """Connect to database."""
    config = ctx.obj['config']
    click.echo(f'Connecting to {host}:{port}')
    click.echo(f'Using config: {config}')

if __name__ == '__main__':
    cli()
```

```bash
$ python tool.py --config app.yml db connect --host prod.example.com
```

---

**[← CLI 도구](../../04-library-catalog/cli/README.md)** | **[다음: structlog →](../logging/structlog.md)**
