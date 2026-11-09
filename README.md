# practical_03_all.py
"""
Практична робота №3: Метакласи, декоратори та паттерни проектування в Python
Виконано студенткою групи КБ-25: Григор'євська
"""

import hashlib
import functools
import time
from abc import ABC, abstractmethod
from datetime import datetime
import json
from dataclasses import dataclass, field
from typing import Callable, List, Dict, Optional, Tuple


# =====================================================================
# ЧАСТИНА 1. Завдання 1.1: Метаклас для плагінів
# =====================================================================
class PluginMeta(type):
    """Метаклас для автоматичної реєстрації та валідації плагінів."""

    _registry: dict[str, type] = {}
    REQUIRED_ATTRS = ("name", "version")

    def __new__(mcs, cls_name: str, bases: tuple, namespace: dict):
        cls = super().__new__(mcs, cls_name, bases, namespace)
        if bases:
            for attr in mcs.REQUIRED_ATTRS:
                if not hasattr(cls, attr):
                    raise TypeError(f"Клас '{cls_name}' повинен мати обов'язковий атрибут '{attr}'")
            plugin_name = getattr(cls, "name")
            mcs._registry[plugin_name] = cls
        return cls

    @classmethod
    def get_registry(mcs) -> dict[str, type]:
        return mcs._registry.copy()

    @classmethod
    def create_plugin(mcs, plugin_name: str):
        cls = mcs._registry.get(plugin_name)
        if not cls:
            raise ValueError(f"Плагін з іменем '{plugin_name}' не знайдено.")
        return cls()


class BasePlugin(metaclass=PluginMeta):
    """Базовий клас плагіна."""
    @abstractmethod
    def execute(self, data: str) -> str:
        pass


class UpperPlugin(BasePlugin):
    name = "upper"
    version = "1.0"
    def execute(self, data: str) -> str:
        return data.upper()


class ReversePlugin(BasePlugin):
    name = "reverse"
    version = "1.0"
    def execute(self, data: str) -> str:
        return data[::-1]


def hash_execute(self, data: str) -> str:
    return hashlib.sha256(data.encode('utf-8')).hexdigest()

HashPlugin = type(
    "HashPlugin",
    (BasePlugin,),
    {"name": "hash", "version": "2.0", "execute": hash_execute}
)


def task_1_1():
    print("\n--- Завдання 1.1: Метаклас плагінів ---")
    print("Зареєстровані плагіни:")
    for name, cls in PluginMeta.get_registry().items():
        print(f"  {name} (v{cls.version}): {cls.__name__}")

    test_data = "Привіт Світ"
    print(f"Тестові дані: '{test_data}'")
    for name in PluginMeta.get_registry():
        plugin = PluginMeta.create_plugin(name)
        print(f"  [{name}]: {plugin.execute(test_data)}")


# =====================================================================
# ЧАСТИНА 1. Завдання 1.2: Дескриптори та __init_subclass__
# =====================================================================
class ValidatedField:
    def __init__(self, expected_type: type, min_value=None, max_value=None):
        self.expected_type = expected_type
        self.min_value = min_value
        self.max_value = max_value
        self.name = None
        self.private_name = None

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(f"Поле '{self.name}' має бути типу {self.expected_type.__name__}")
        if self.min_value is not None and value < self.min_value:
            raise ValueError(f"Поле '{self.name}' менше за мінімум {self.min_value}")
        if self.max_value is not None and value > self.max_value:
            raise ValueError(f"Поле '{self.name}' перевищує максимум {self.max_value}")
        setattr(obj, self.private_name, value)


class ConfigSection:
    _sections: dict[str, type] = {}

    def __init_subclass__(cls, section_name: str = "", **kwargs):
        super().__init_subclass__(**kwargs)
        if section_name:
            cls._sections[section_name] = cls

    @classmethod
    def list_sections(cls) -> list[str]:
        return list(cls._sections.keys())


class DatabaseConfig(ConfigSection, section_name="database"):
    host = ValidatedField(str)
    port = ValidatedField(int, min_value=1, max_value=65535)

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port


def task_1_2():
    print("\n--- Завдання 1.2: Дескриптори та __init_subclass__ ---")
    print(f"Зареєстровані секції: {ConfigSection.list_sections()}")
    db = DatabaseConfig("localhost", 3306)
    print(f"DatabaseConfig створено: host={db.host}, port={db.port}")
    try:
        db.port = 80000
    except ValueError as e:
        print(f"Перехоплено помилку діапазону: {e}")


# =====================================================================
# ЧАСТИНА 2. Завдання 2.1: Декоратори функцій
# =====================================================================
def cache(max_size: int = 128):
    def decorator(func):
        _cache = {}
        _order = []
        @functools.wraps(func)
        def wrapper(*args):
            if args in _cache:
                return _cache[args]
            res = func(*args)
            if len(_cache) >= max_size:
                _cache.pop(_order.pop(0))
            _cache[args] = res
            _order.append(args)
            return res
        return wrapper
    return decorator


@cache(max_size=32)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)


def task_2_1():
    print("\n--- Завдання 2.1: Декоратори функцій (кешування) ---")
    print(f"fibonacci(15) = {fibonacci(15)}")


# =====================================================================
# ЧАСТИНА 2. Завдання 2.2: Декоратори класів
# =====================================================================
def auto_repr(cls):
    init_code = cls.__init__.__code__
    varnames = init_code.co_varnames[1:init_code.co_argcount]
    def __repr__(self):
        params = ", ".join(f"{name}={getattr(self, name, None)!r}" for name in varnames)
        return f"{cls.__name__}({params})"
    cls.__repr__ = __repr__
    return cls


@auto_repr
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y


def task_2_2():
    print("\n--- Завдання 2.2: Декоратори класів (@auto_repr) ---")
    p = Point(5.5, 10.2)
    print(f"repr(p) -> {p!r}")


# =====================================================================
# ЧАСТИНА 3. Завдання 3.1: Singleton та Factory Method
# =====================================================================
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]


class Logger(metaclass=SingletonMeta):
    def __init__(self):
        if getattr(self, "_initialized", False):
            return
        self.history = []
        self._initialized = True

    def info(self, msg: str):
        log_msg = f"[INFO] {datetime.now().strftime('%H:%M:%S')} - {msg}"
        print(log_msg)
        self.history.append(log_msg)


def task_3_1():
    print("\n--- Завдання 3.1: Singleton Logger ---")
    l1 = Logger()
    l2 = Logger()
    print(f"Singleton перевірка (l1 is l2): {l1 is l2}")
    l1.info("Тестове повідомлення логера")


# =====================================================================
# ЧАСТИНА 3. Завдання 3.2: Observer, Strategy, MVC
# =====================================================================
class EventEmitter:
    def __init__(self):
        self._listeners = {}
    def on(self, event: str, cb: Callable):
        self._listeners.setdefault(event, []).append(cb)
    def emit(self, event: str, *args, **kwargs):
        for cb in self._listeners.get(event, []):
            cb(*args, **kwargs)


class SortStrategy(ABC):
    @abstractmethod
    def sort(self, tasks: list) -> list: pass


class SortByPriority(SortStrategy):
    def sort(self, tasks: list) -> list:
        return sorted(tasks, key=lambda t: t.priority)


@dataclass
class Task:
    title: str
    priority: int
    deadline: datetime
    completed: bool = False


class TaskModel(EventEmitter):
    def __init__(self):
        super().__init__()
        self.tasks = []
    def add_task(self, title: str, priority: int, deadline: datetime):
        t = Task(title, priority, deadline)
        self.tasks.append(t)
        self.emit("task_added", t)


class TaskView:
    def show_message(self, msg: str):
        print(f"[VIEW INFO] {msg}")


class TaskController:
    def __init__(self, model: TaskModel, view: TaskView):
        self.model = model
        self.view = view
        self.strategy = SortByPriority()
        self.model.on("task_added", lambda t: self.view.show_message(f"Додано: {t.title}"))

    def add(self, title: str, priority: int, deadline: datetime):
        self.model.add_task(title, priority, deadline)


def task_3_2():
    print("\n--- Завдання 3.2: MVC, Observer та Strategy ---")
    model = TaskModel()
    view = TaskView()
    ctrl = TaskController(model, view)
    ctrl.add("Вивчити патерни", 1, datetime(2026, 6, 1))


# =====================================================================
# ГОЛОВНЕ МЕНЮ ПРОГРАМИ
# =====================================================================
def main():
    while True:
        print("\n" + "=" * 65)
        print(" ГОЛОВНЕ МЕНЮ — ПРАКТИЧНА РОБОТА №3 (Григор'євська, КБ-25)")
        print("=" * 65)
        print(" 1. Завдання 1.1 (Метаклас плагінів)")
        print(" 2. Завдання 1.2 (Типізовані дескриптори)")
        print(" 3. Завдання 2.1 (Декоратор cache)")
        print(" 4. Завдання 2.2 (Декоратор @auto_repr)")
        print(" 5. Завдання 3.1 (Singleton Logger)")
        print(" 6. Завдання 3.2 (MVC та Observer)")
        print(" 0. Вихід з програми")
        print("=" * 65)

        choice = input(" Оберіть номер завдання (0-6): ").strip()

        if choice == '1':
            task_1_1()
        elif choice == '2':
            task_1_2()
        elif choice == '3':
            task_2_1()
        elif choice == '4':
            task_2_2()
        elif choice == '5':
            task_3_1()
        elif choice == '6':
            task_3_2()
        elif choice == '0':
            print("\nРоботу програми завершено. До побачення!")
            break
        else:
            print("\nНевірний вибір! Введіть число від 0 до 6.")


if __name__ == "__main__":
    main()
