- [Memory tracing](#memory-tracing)
  * [Generator, lambda, collection](#generator--lambda--collection)
  * [Mutable as a default argument](#mutable-as-a-default-argument)
  * [File Reader Generator](#file-reader-generator)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# Memory tracing
```python
import multiprocessing
import time
import tracemalloc

from rich.console import Console

console = Console()
print_ts = time.time()


def print(*args, **kwargs) -> None:
    global print_ts
    now = time.time()
    proc = multiprocessing.current_process().name
    if proc == "MainProcess":
        proc = f"[bold]{proc:<16}[/bold]"
    else:
        proc = f"{proc:>16}"
    console.print(
        f"{proc} [[green bold]{now - print_ts:>5.2f}s[/]]",
        *args,
        **kwargs
    )
    print_ts = now


def mem_usage() -> str:
    current, peak = tracemalloc.get_traced_memory()
    return f"Memory usage: {current//1024//1024} MB; peak: {peak//1024//1024} MB"

```

## Generator, lambda, collection

```python
from memory_check import print, mem_usage

n = 20000000

generator_argument_result = sum(x ** 2 for x in range(n))
print("[Generator]", mem_usage())
lambda_func_result = sum(map(lambda x: x ** 2, range(n)))
print("[Lambda]", mem_usage())
collection_argument_result = sum([x ** 2 for x in range(n)])
print("[Collection]", mem_usage())

```
Every sum funciton was executed separately, results:
```commandline
MainProcess      [123.88s] [Generator] Memory usage: 7 MB; peak: 7 MB
MainProcess      [138.58s] [Lambda] Memory usage: 7 MB; peak: 7 MB
MainProcess      [144.71s] [Collection] Memory usage: 7 MB; peak: 7726 MB

```

## Mutable as a default argument
```python
import tracemalloc

from mem_trace import print, mem_usage


def foo(input_values: list[int], bar: list = []):
    bar.extend(input_values)
    print(f"bar-list elements: {len(bar)}", mem_usage())


if __name__ == "__main__":
    tracemalloc.start()
    foo([x for x in range(100000)])
    foo([x for x in range(100000, 200000)])
    foo([x for x in range(200000, 300000)])
```

```commandline
MainProcess      [ 0.01s] bar-list elements: 100000 Memory usage: 4 MB; peak: 4 MB
MainProcess      [ 0.02s] bar-list elements: 200000 Memory usage: 8 MB; peak: 8 MB
MainProcess      [ 0.02s] bar-list elements: 300000 Memory usage: 12 MB; peak: 12 MB
```

## File Reader Generator
```python
...

with open("foo.txt", "r") as file:
    lines = file.readlines()
    for line in lines:
        print(mem_usage(), line.strip())
```

```commandline
MainProcess      [ 0.04s] Memory usage: 34 MB; peak: 34 MB Lorem ipsum dolor sit amet, consectetur adipiscing elit...
```

Let's compare it with file reader generator:

```python
...

def yield_lines(file_path):
    with open(file_path, 'r') as file:
        for line in file:
            yield line.strip()

            
for line in yield_lines('foo.txt'):
    print(mem_usage(), line)
```

```commandline
MainProcess      [ 0.00s] Memory usage: 0 MB; peak: 0 MB Lorem ipsum dolor sit amet, consectetur adipiscing elit...
```