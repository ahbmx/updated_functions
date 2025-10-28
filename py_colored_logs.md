Good catch — the issue is that in your current formatter, you’re replacing `record.levelname` with a colored string *before* it’s formatted, but when Python’s logging module copies or reuses records, the color codes may be lost or stripped. Also, `colorama` needs `colorama.init()` called once (we already do that), but you must print the color codes **directly into the formatted message** — not into `record.levelname`.

Let’s fix that cleanly. 👇

---

### ✅ Corrected `logHelper.py`

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
        # Determine caller module name
        frame = inspect.stack()[8] if len(inspect.stack()) > 8 else inspect.stack()[1]
        module = os.path.splitext(os.path.basename(frame.filename))[0]

        color = LOG_COLORS.get(record.levelname, "")
        reset = Style.RESET_ALL

        # Build colored message directly
        formatted = f"{color}[{record.levelname}] [{module}] {record.getMessage()}{reset}"
        return formatted


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

        # Add custom level methods if not already added
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

### ✅ `__init__.py`

```python
from .logHelper import get_logger

# Global package logger
logger = get_logger(name="dvl", level="INFO", log_dir=None)
```

---

### ✅ Example usage (e.g., `brocadeHelper.py`)

```python
from . import logger

logger.debug("Debug message")
logger.info("Information message")
logger.command("Command executed")
logger.system("System check running")
logger.success("Operation successful")
logger.warning("Warning condition")
logger.error("Error occurred")
logger.critical("Critical failure")
```

---

### 💡 Why colors now appear:

* We **don’t overwrite** `record.levelname` (logging internals reuse it elsewhere).
* We **inject the color codes into the formatted message** instead.
* `colorama`’s `init(autoreset=True)` ensures colors reset correctly on each log.

---

Would you like the color scheme to be adjusted (e.g., darker magenta for COMMAND or bold green for SUCCESS)?
