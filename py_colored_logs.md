Here's a comprehensive Python logging function without using the typing module:

```python
import logging
import sys
import colorama

# Initialize colorama for cross-platform colored output
colorama.init()

# Custom log levels
SUCCESS_LEVEL_NUM = 25
SYSTEM_LEVEL_NUM = 27
COMMAND_LEVEL_NUM = 29

# Add custom levels to logging module
logging.addLevelName(SUCCESS_LEVEL_NUM, "SUCCESS")
logging.addLevelName(SYSTEM_LEVEL_NUM, "SYSTEM")
logging.addLevelName(COMMAND_LEVEL_NUM, "COMMAND")

class ColoredFormatter(logging.Formatter):
    """Custom formatter with colored output for console"""
    
    # ANSI color codes
    COLORS = {
        'DEBUG': colorama.Fore.CYAN,
        'SUCCESS': colorama.Fore.GREEN,
        'SYSTEM': colorama.Fore.MAGENTA,
        'COMMAND': colorama.Fore.BLUE,
        'INFO': colorama.Fore.WHITE,
        'WARNING': colorama.Fore.YELLOW,
        'ERROR': colorama.Fore.RED,
        'CRITICAL': colorama.Fore.RED + colorama.Style.BRIGHT,
        'RESET': colorama.Style.RESET_ALL
    }
    
    def __init__(self, fmt=None, datefmt=None, style='%', colors=None):
        super().__init__(fmt, datefmt, style)
        self.colors = colors or self.COLORS

    def format(self, record):
        # Store original format
        original_fmt = self._style._fmt
        
        # Add color to format based on level
        if record.levelname in self.colors:
            color = self.colors[record.levelname]
            reset = self.colors['RESET']
            self._style._fmt = f"{color}{original_fmt}{reset}"
        
        # Call original formatter
        result = super().format(record)
        
        # Restore original format
        self._style._fmt = original_fmt
        
        return result

class CustomLogger(logging.Logger):
    """Custom logger with additional log levels"""
    
    def success(self, msg, *args, **kwargs):
        if self.isEnabledFor(SUCCESS_LEVEL_NUM):
            self._log(SUCCESS_LEVEL_NUM, msg, args, **kwargs)
    
    def system(self, msg, *args, **kwargs):
        if self.isEnabledFor(SYSTEM_LEVEL_NUM):
            self._log(SYSTEM_LEVEL_NUM, msg, args, **kwargs)
    
    def command(self, msg, *args, **kwargs):
        if self.isEnabledFor(COMMAND_LEVEL_NUM):
            self._log(COMMAND_LEVEL_NUM, msg, args, **kwargs)

# Register the custom logger
logging.setLoggerClass(CustomLogger)

def setup_logging(
    name="CustomLogger",
    level=logging.INFO,
    log_to_file=False,
    log_file=None,
    log_to_console=True,
    console_colors=None,
    file_format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    console_format="%(asctime)s - %(levelname)s - %(message)s",
    date_format="%Y-%m-%d %H:%M:%S"
):
    """
    Setup and return a configured logger.
    
    Args:
        name: Logger name
        level: Logging level
        log_to_file: Whether to log to file
        log_file: File path for logging (required if log_to_file is True)
        log_to_console: Whether to log to console
        console_colors: Custom color mapping for console output
        file_format: Format for file logging
        console_format: Format for console logging
        date_format: Date format for timestamps
    
    Returns:
        Configured logger instance
    """
    
    # Validate arguments
    if log_to_file and not log_file:
        raise ValueError("log_file must be provided when log_to_file is True")
    
    # Create logger
    logger = logging.getLogger(name)
    logger.setLevel(level)
    
    # Clear any existing handlers
    logger.handlers.clear()
    
    # Console handler
    if log_to_console:
        console_handler = logging.StreamHandler(sys.stdout)
        console_handler.setLevel(level)
        
        # Create colored formatter for console
        console_formatter = ColoredFormatter(
            fmt=console_format,
            datefmt=date_format,
            colors=console_colors
        )
        console_handler.setFormatter(console_formatter)
        logger.addHandler(console_handler)
    
    # File handler
    if log_to_file:
        file_handler = logging.FileHandler(log_file)
        file_handler.setLevel(level)
        
        # Create plain formatter for file
        file_formatter = logging.Formatter(
            fmt=file_format,
            datefmt=date_format
        )
        file_handler.setFormatter(file_formatter)
        logger.addHandler(file_handler)
    
    return logger

# Alternative simpler function for quick setup
def create_logger(
    name="AppLogger",
    level="INFO",
    log_file=None,
    use_colors=True
):
    """
    Simplified logger creation function.
    
    Args:
        name: Logger name
        level: Log level as string ('DEBUG', 'INFO', etc.)
        log_file: Optional file path for logging
        use_colors: Whether to use colored console output
    
    Returns:
        Configured logger instance
    """
    
    # Convert string level to logging constant
    level_map = {
        'DEBUG': logging.DEBUG,
        'SUCCESS': SUCCESS_LEVEL_NUM,
        'SYSTEM': SYSTEM_LEVEL_NUM,
        'COMMAND': COMMAND_LEVEL_NUM,
        'INFO': logging.INFO,
        'WARNING': logging.WARNING,
        'ERROR': logging.ERROR,
        'CRITICAL': logging.CRITICAL
    }
    
    log_level = level_map.get(level.upper(), logging.INFO)
    
    return setup_logging(
        name=name,
        level=log_level,
        log_to_file=log_file is not None,
        log_file=log_file,
        log_to_console=True,
        console_colors=ColoredFormatter.COLORS if use_colors else None
    )

# Example usage and demonstration
if __name__ == "__main__":
    # Example 1: Basic setup with console logging only
    print("=== Example 1: Basic Console Logging ===")
    logger = setup_logging(
        name="MyApp",
        level=logging.DEBUG,
        log_to_console=True,
        log_to_file=False
    )
    
    # Test all log levels
    logger.debug("This is a debug message")
    logger.info("This is an info message")
    logger.success("This is a success message!")
    logger.system("This is a system message")
    logger.command("This is a command message")
    logger.warning("This is a warning message")
    logger.error("This is an error message")
    logger.critical("This is a critical message!")
    
    print("\n" + "="*50 + "\n")
    
    # Example 2: Custom colors and file logging
    print("=== Example 2: Custom Colors + File Logging ===")
    custom_colors = {
        'DEBUG': colorama.Fore.BLUE,
        'SUCCESS': colorama.Fore.GREEN + colorama.Style.BRIGHT,
        'SYSTEM': colorama.Fore.CYAN,
        'COMMAND': colorama.Fore.YELLOW,
        'INFO': colorama.Fore.WHITE,
        'WARNING': colorama.Fore.YELLOW + colorama.Style.BRIGHT,
        'ERROR': colorama.Fore.RED,
        'CRITICAL': colorama.Fore.RED + colorama.Style.BRIGHT,
        'RESET': colorama.Style.RESET_ALL
    }
    
    logger2 = setup_logging(
        name="CustomApp",
        level=logging.DEBUG,
        log_to_console=True,
        log_to_file=True,
        log_file="app.log",
        console_colors=custom_colors,
        console_format="[%(levelname)-8s] %(message)s",
        file_format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )
    
    logger2.debug("Debug with custom colors")
    logger2.success("Success with custom colors!")
    logger2.system("System message")
    logger2.command("Command executed")
    logger2.warning("Warning message")
    logger2.error("Error occurred")
    
    print("\n" + "="*50 + "\n")
    
    # Example 3: Using the simplified function
    print("=== Example 3: Simplified Function ===")
    logger3 = create_logger(
        name="SimpleApp",
        level="SUCCESS",
        log_file="simple.log",
        use_colors=True
    )
    
    logger3.debug("This debug won't show (level is SUCCESS)")
    logger3.info("This info won't show (level is SUCCESS)")
    logger3.success("This success will show!")
    logger3.error("This error will show!")
    
    print("\n" + "="*50 + "\n")
    
    # Example 4: Using the logger in a class
    print("=== Example 4: Class Usage ===")
    class MyApplication:
        def __init__(self):
            self.logger = create_logger("MyApplication", "INFO")
        
        def run(self):
            self.logger.info("Application starting...")
            self.logger.system("Initializing system components")
            self.logger.command("Executing main command")
            self.logger.success("Application started successfully!")
            
            try:
                # Simulate some work
                self.logger.debug("Debug information (won't show at INFO level)")
                # Simulate an error
                raise ValueError("Something went wrong")
            except Exception as e:
                self.logger.error("Error in application: %s", e)
    
    app = MyApplication()
    app.run()
    
    # Cleanup colorama
    colorama.deinit()
```

## Key Features:

1. **All Custom Log Levels**: DEBUG, SUCCESS, SYSTEM, COMMAND, INFO, WARNING, ERROR, CRITICAL
2. **Configurable Colors**: Each level has customizable console colors
3. **Flexible Output**: Console, file, or both
4. **Two Setup Functions**: 
   - `setup_logging()` - Full control over all parameters
   - `create_logger()` - Simplified version for quick setup
5. **No External Dependencies** (except colorama for colors)

## Installation:

```bash
pip install colorama
```

## Usage Examples:

### Basic Usage:
```python
logger = create_logger("MyApp", "DEBUG")
logger.success("Operation completed!")
```

### Advanced Configuration:
```python
logger = setup_logging(
    name="MyApp",
    level=logging.DEBUG,
    log_to_file=True,
    log_file="app.log",
    log_to_console=True
)
```

### Without Colors:
```python
logger = create_logger("MyApp", "INFO", use_colors=False)
```

The code is completely self-contained and provides all the functionality you requested without using the typing module.
