Got it 👍 — you want the **standard console log format** (with timestamps, level, module, etc.) but still **colored output** per level — not the custom `"[LEVEL] [MODULE] message"` format.

Let’s keep the *regular console-style format*, like:

```
2025-10-27 18:31:04,822 - brocadeHelper - INFO - Message text
```

…but make it **colorized** by log level.
File logs will stay plain (no color).

---

### ✅ Updated `dvl/logHelper.py`

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

# === Custom Formatter (uses standard format) ===
class ColorFormatter(logging.Formatter):
    def format(self, record):
        color = LOG_COLORS.get(record.levelname, "")
        reset = Style.RESET_ALL

        # Get the module name of the original caller
        frame = inspect.stack()[8] if len(inspect.stack()) > 8 else inspect.stack()[1]
        module = os.path.splitext(os.path.basename(frame.filename))[0]
        record.module = module

        log_fmt = "%(asctime)s - %(module)s - %(levelname)s - %(message)s"
        formatter = logging.Formatter(log_fmt, "%Y-%m-%d %H:%M:%S")

        formatted = formatter.format(record)
        return f"{color}{formatted}{reset}"


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

        # Optional file handler (plain format)
        if log_dir:
            os.makedirs(log_dir, exist_ok=True)
            file_path = os.path.join(log_dir, f"{name}.log")
            file_handler = logging.FileHandler(file_path, encoding="utf-8")
            file_formatter = logging.Formatter(
                "%(asctime)s - %(module)s - %(levelname)s - %(message)s",
                "%Y-%m-%d %H:%M:%S",
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

### ✅ `dvl/__init__.py`

```python
from .logHelper import get_logger

# Initialize shared package logger
logger = get_logger(name="dvl", level="INFO", log_dir=None)
```

---

### ✅ Example (`brocadeHelper.py`)

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

### ✅ Example Output (console)

```
2025-10-27 19:10:12 - brocadeHelper - INFO - Information message
2025-10-27 19:10:12 - brocadeHelper - COMMAND - Command executed
2025-10-27 19:10:12 - brocadeHelper - SUCCESS - Operation successful
2025-10-27 19:10:12 - brocadeHelper - ERROR - Error occurred
```

Each line will be **colored** appropriately (green for success, magenta for command, etc.), while the **file logs remain uncolored** and use the same timestamped format.

---

Would you like the color mapping adjusted (for example, brighter blue for SYSTEM or bold green for SUCCESS)?
