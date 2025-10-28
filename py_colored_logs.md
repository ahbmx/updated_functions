Perfect — let’s build a **centralized logging system** in your `dvl/logHelper.py` that:

✅ Always logs to console (with colors).
✅ Optionally logs to file if `log_dir` is specified.
✅ Defines custom levels: `COMMAND (17)`, `SYSTEM (19)`, `SUCCESS (25)`.
✅ Uses `colorama` for coloring (no ANSI fallback, no typing module).
✅ Automatically initializes when importing the `dvl` package (in `__init__.py`).
✅ Includes the **calling module name** in the log format.

---

### 📁 Folder structure

```
stcol.py
dvl/
    __init__.py
    logHelper.py
    brocadeHelper.py
    pmaxHelper.py
    functionHelper.py
    reportHelper.py
```

---

## 🧩 logHelper.py

```python
import logging
import os
import sys
import inspect
from colorama import Fore, Style, init as colorama_init

colorama_init(autoreset=True)

# === Custom log levels ===
COMMAND_LEVEL = 17
SYSTEM_LEVEL = 19
SUCCESS_LEVEL = 25

logging.addLevelName(COMMAND_LEVEL, "COMMAND")
logging.addLevelName(SYSTEM_LEVEL, "SYSTEM")
logging.addLevelName(SUCCESS_LEVEL, "SUCCESS")

# === Color map ===
LOG_COLORS = {
    "DEBUG": Fore.CYAN,
    "INFO": Fore.WHITE,
    "COMMAND": Fore.MAGENTA,
    "SYSTEM": Fore.BLUE,
    "SUCCESS": Fore.GREEN,
    "WARNING": Fore.YELLOW,
    "ERROR": Fore.RED,
    "CRITICAL": Fore.RED + Style.BRIGHT,
}

# === Custom Formatter ===
class ColorFormatter(logging.Formatter):
    def format(self, record):
        color = LOG_COLORS.get(record.levelname, "")
        record.levelname = f"{color}{record.levelname}{Style.RESET_ALL}"

        # Determine caller module name (one frame up)
        frame = inspect.stack()[8] if len(inspect.stack()) > 8 else inspect.stack()[1]
        module = os.path.splitext(os.path.basename(frame.filename))[0]

        record.module_name = module
        return f"[{record.levelname}] [{record.module_name}] {record.getMessage()}"

# === Logger creator ===
def get_logger(name="dvl", level="INFO", log_dir=None):
    logger = logging.getLogger(name)

    if not logger.handlers:
        # Convert level string to numeric
        numeric_level = getattr(logging, level.upper(), logging.INFO)
        logger.setLevel(numeric_level)

        # Console handler (always)
        console_handler = logging.StreamHandler(sys.stdout)
        console_handler.setFormatter(ColorFormatter())
        logger.addHandler(console_handler)

        # Optional file handler
        if log_dir:
            os.makedirs(log_dir, exist_ok=True)
            file_path = os.path.join(log_dir, f"{name}.log")
            file_handler = logging.FileHandler(file_path, encoding="utf-8")
            file_formatter = logging.Formatter(
                "[%(levelname)s] [%(module)s] %(message)s"
            )
            file_handler.setFormatter(file_formatter)
            logger.addHandler(file_handler)

        # Add custom level methods
        def command(self, msg, *args, **kwargs):
            if self.isEnabledFor(COMMAND_LEVEL):
                self._log(COMMAND_LEVEL, msg, args, **kwargs)

        def system(self, msg, *args, **kwargs):
            if self.isEnabledFor(SYSTEM_LEVEL):
                self._log(SYSTEM_LEVEL, msg, args, **kwargs)

        def success(self, msg, *args, **kwargs):
            if self.isEnabledFor(SUCCESS_LEVEL):
                self._log(SUCCESS_LEVEL, msg, args, **kwargs)

        logging.Logger.command = command
        logging.Logger.system = system
        logging.Logger.success = success

    return logger
```

---

## 🧩 **init**.py

Initialize the logger here so it’s available everywhere in the `dvl` package.

```python
from .logHelper import get_logger

# Initialize global package logger
logger = get_logger(name="dvl", level="INFO", log_dir=None)
```

Now, in any other module (like `brocadeHelper.py`), you can simply do:

```python
from . import logger

logger.info("Starting Brocade operations...")
logger.command("Executing command on switch...")
logger.success("Operation completed successfully.")
logger.system("System-level message.")
logger.error("Something went wrong!")
```

---

## ✅ Features Recap

* **Custom levels** (`COMMAND`, `SYSTEM`, `SUCCESS`)
* **Colorized console output** (colorama)
* **Optional file output** (creates directory automatically)
* **Auto-initialized shared logger** via `dvl/__init__.py`
* **Includes calling module name** in logs
* **Simple, dependency-light design**

---

Would you like the log format to also include timestamps or just `[LEVEL] [MODULE] message` as shown?
