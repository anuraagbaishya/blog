---
title: "Exploring Python Code Execution Patterns for Fun and Profit"
date: 2024-02-10T10:01:34-08:00
draft: false
toc: true
images:
tags:
  - python
---

Recently, I was looking at the Semgrep output for an open source project and I saw an interesting finding about potential arbitrary code execution. The code essentially ran an `eval` but the interesting part was converting Python code to an abstract syntax tree, compiling it to bytecode, and evaluating it. This made me curious about other unsafe ways to write Python code that could lead to code execution. In this article, I will discuss a few patterns that I learnt about through my limited research.

Notes:
* This is by no means a complete list of all patterns that can lead to arbitrary code execution in Python.
* Some of these examples are contrived, and might not make any sense in the real world. I merely use them to demonstrate vulnerable code patterns.

## Basic Patterns
Python provides many ways of running system commands or arbitrary code in its `os` and `subprocess` modules. Some examples include:
* [`os.system`, `os.popen`, `os.execve`](https://docs.python.org/3/library/os.html)
* [`subprocess.Popen`, `subprocess.run`, `subprocess.check_call`, `subprocess.check_output`](https://docs.python.org/3/library/subprocess.html)

While `subprocess` functions are designed to be safer than `os` functions, allowing code execution via untrusted input should always be avoided. Some examples of untrusted inputs leading to remote code execution can be seen in [these vulnerabilities](https://pentest.blog/advisory-roxy-wi-unauthenticated-remote-code-executions-cve-2022-31137/) reported against Roxy-WI. Here, user input is directly concatenated to commands used in calls to `os.system`, `subprocess.Popen` and `ssh.exec_command`.

It is also possible to run arbitrary Python code using [`eval`](https://docs.python.org/3/library/functions.html#eval) and [`exec`](https://docs.python.org/3/library/functions.html#exec). 

## `eval` and `exec` with Abstract Syntax Trees
`eval` and `exec` require a string, bytes or code object. There is also [`compile`](https://docs.python.org/3/library/functions.html#compile) which can compile code into bytecode. The compiled bytecode can then be used with `eval` or `exec`. However `compile` requires a string, bytes or AST object. Therefore, just by themselves one cannot `eval` or `exec` a function like this:
```Python
eval(my_function)
TypeError: eval arg 1 must be a string, bytes or code object
```

Although I can't think of any good reasons for evaluating a function like this, someone might have some use case. They can then convert the function to an AST first, compile it to bytecode, and evaluate or execute it as needed. If a malicious user can control what function is being evaluated or executed, they will achieve code execution.

### Code example
```Python
def eval_node(func):
    tree = ast.parse(textwrap.dedent(inspect.getsource(func)))

    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            try:
                return eval(compile(ast.Expression(node), "fl", "eval"))
            except Exception:
                pass

    return None

def my_function():
    subprocess.check_output(["id"]).decode("utf-8").strip

print(eval_node(my_function))
```

Here, `my_function` is parsed into an AST, which is then compiled to bytecode before being evaluated.

## Recursive attribute lookup
I learnt about this pattern while reading a [CVE report for Celery](https://snyk.io/blog/python-rce-vulnerability/) by Calum Hutton. I would highly recommend reading the original article, but I will summarize the finding here.

Python allows performing recursive lookups on objects to get their attributes or sub-attributes. This means it is possible to traverse from one module to another, and use functions present in the other module, if the first module imports the subsequent module(s). This also means if an application exposes functionality that allows user controlled object traversal, the user might be able to manipulate the traversal and get references to functions such as `os.system`, therefore acheiving arbitrary code execution. (I am using `os.system` as example, but this would apply to something like `subprocess.run` too.)

Not every module imports the `os` module, so this pattern only affects modules import `os` and thus have `os` as an attribute . Some examples of such modules are:
* [`random`](https://github.com/python/cpython/blob/main/Lib/random.py#L62) (`os` is imported as `_os`) 
* [`shutil`](https://github.com/python/cpython/blob/main/Lib/shutil.py#L7)
* [`pathlib`](https://github.com/python/cpython/blob/main/Lib/pathlib/__init__.py#L10) 
* [`posixpath`](https://github.com/python/cpython/blob/main/Lib/posixpath.py#L25)

If you are interested in finding more modules, you can look at the Python [sourcecode](https://github.com/python/cpython/tree/main/Lib) where the modules are defined and check if the module imports `os`.

```Python
getattr(pathlib, "os")
<module 'os' from '/Users/user/.pyenv/versions/3.9.5/lib/python3.9/os.py'>

getattr(shutil, "os")
<module 'os' from '/Users/user/.pyenv/versions/3.9.5/lib/python3.9/os.py'>

getattr(random, "_os")
<module 'os' from '/Users/user/.pyenv/versions/3.9.5/lib/python3.9/os.py'>

getattr(posixpath, "os")
<module 'os' from '/Users/user/.pyenv/versions/3.9.5/lib/python3.9/os.py'>
```

### Code example

This snippet is adapted from Celery source code presented in the original article.
```Python
def code_exec(d):
    _module = d["module"]
    _type = d["type"]
    try:
        cls = sys.modules[_module]
        for name in _type.split("."):
            cls = getattr(cls, name)
    except Exception:
        pass

    _msg = d["message"]
    try:
        if isinstance(_msg, (tuple, list)):
            d = cls(*_msg)
        else:
            d = cls(_msg)
    except Exception:
        pass


d = {
    "module": "os",
    "type": "system",
    "message": "id",
}
```
The above snippet will lead to running `os.system("id")`. First `cls` is instantiated as `os` using `sys.modules`. It is then updated to the `os.system` function via recursive lookup using `getattr`. Finally code is executed when `cls(_msg)` is evaluated.

### CVE-2023-33733
CVE-2023-3373 is a CVE in the `reportlab` Python library. This was reported by Elyas Damej, who has published a great [write-up](https://github.com/c53elyas/CVE-2023-33733) describing the vulnerabilties and its technical details. This is a very interesting CVE since it combines recursive attribute lookup and using `eval` to execute arbitrary code.

I would recommend reading the original write-up, but I'll summarize it here. The `reportlab` library allows creating PDFs using Python. In 2019, a [CVE](https://nvd.nist.gov/vuln/detail/CVE-2019-17626) was discovered which allowed remote code execution through the `color` HTML tag which is passed to `eval` without proper sanitization. To fix this  `rl_safe_eval` sandbox was implemented where all Python builtins functions are removed and many builtin functions are overriden to prevent access to dangerous functions that could lead to code execution. This sandbox was bypassed by creating a new class `Word` which is specifically crafted to bypass the checks in the sandbox. Following this, the `__globals__` attribute of `Word` is accessed and used to call `os.system`

## Pickle


In Python, the `pickle` module is used for serializing and deserializing objects. Serialization refers to the process of converting objects in memory to a byte stream. Deserialization is the reverse process, where the byte stream is converted back into objects in memory. Serialization is performed by `pickle.dump` / `pickle.dumps` and deserialization is performed by `pickle.load / pickle.loads`. 

Python allows specifying how an object should be pickled by using the `__reduce__` method. The `__reduce__` function can return a tuple which represents callable code along with arguments to the callable code. When this object is deserialized, Python will run the callable code in the object's `__reduce__` method. This gives one the ability to create an object that can lead to code execution when deserialized.

### Code example

```Python
import pickle
class Payload:
    def __reduce__(self):
        import os
        return (os.system, ('whoami',))

serialized = pickle.dumps(Payload())

pickle.loads(serialized)
```

When an object of the class `Payload` is deserialized using `pickle.loads()`, the `__reduce__` function is called. This function returns callable code `os.system('whoami')`, which is then executed.

This behavior of `pickle` has led to many CVEs. Some examples:
* [CVE-2023-23930](https://nvd.nist.gov/vuln/detail/CVE-2023-23930)
* [CVE-2022-34668](https://nvd.nist.gov/vuln/detail/CVE-2022-34668)
* [CVE-2021-33026](https://nvd.nist.gov/vuln/detail/CVE-2021-33026)
* [CVE-2020-22083](https://nvd.nist.gov/vuln/detail/CVE-2020-22083)

## Forward References and `typing.get_type_hints`
I learnt of this technique from a [Stack Overflow](https://stackoverflow.com/questions/25353753/python-can-i-safely-unpickle-untrusted-data) post about dangers of unpickling untrusted data.

A forward reference is a reference to a variable, function, or class that is defined later in the code. One place where forward references can be used is with `typing.get_type_hints`.

From [documentation](https://docs.python.org/3/library/typing.html#typing.get_type_hints): `typing.get_type_hints` returns a dictionary containing type hints for a function, method, module or class object. In addition, forward references encoded as string literals are handled by evaluating them in globals and locals namespaces. 

This behavior can be misused to run arbitrary code, by specifying the code as a forward reference. Setting the `__anotations__` attribute of an object to such a forward reference does the trick.

### Code example

```Python
class Payload(object):
    def __init__(self):
        self.__annotations__ = {"x": """eval('__import__("os").system("ls")')"""}


p = Payload()
print(typing.get_type_hints(p))
```

Here, we specify a forward reference which imports `os` and calls `os.system`

It is possible that `typing.get_type_hints` is used internally by other Python functions, which could make these functions also susceptible to similar misuse. The Stack Overflow post mentions `functools.singledispatch` which has inner function `register`. The `register` function calls `typing.get_type_hints`. I tried using the above `Payload` class with `functools.singledispatch` but was not able to achieve code execution. Therefore, I leave this as an exercise to the reader. 

***

While researching for this article, I learnt many things about Python internals. I found this quite interesting, and therefore, plan to look more into Python internals - and research other unsafe patterns that could lead to security issues. I also plan to use what I learn to look for security issues in open source code. Only time will tell how successful I am in these endeavors!

Thanks for reading!