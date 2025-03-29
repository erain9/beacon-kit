# Logging System

## Overview

The logging package provides a structured logging system for the beacon node. It offers configurable log levels, formats, and output destinations, while maintaining a consistent interface across the application.

## Logger Interface

The core of the logging system is the Logger interface:

```go
type Logger interface {
    // Log methods for different levels
    Debug(msg string, keyvals ...interface{})
    Info(msg string, keyvals ...interface{})
    Warn(msg string, keyvals ...interface{})
    Error(msg string, keyvals ...interface{})
    
    // Create a new logger with additional context
    With(keyvals ...interface{}) Logger
    
    // Configure log level
    SetLevel(level Level)
}
```

This interface ensures consistent logging throughout the application.

## Log Levels

The system supports standard log levels:

```go
type Level int

const (
    DebugLevel Level = iota
    InfoLevel
    WarnLevel
    ErrorLevel
)
```

These levels control which messages are output:
- **Debug**: Detailed debugging information
- **Info**: General operational information
- **Warn**: Warning conditions that don't affect normal operation
- **Error**: Error conditions that affect operation

## Structured Logging

The logging system uses a structured approach with key-value pairs:

```go
// Unstructured
logger.Info("received request")

// Structured
logger.Info("received request",
    "method", "GET",
    "path", "/api/v1/beacon/blocks",
    "client", "127.0.0.1",
)
```

This structured approach:
- Makes logs easier to parse and analyze
- Provides consistent context
- Supports log aggregation and filtering

## Contextual Loggers

Loggers can be extended with additional context:

```go
// Base logger
baseLogger := log.NewLogger(config.LogLevel)

// Component-specific logger
apiLogger := baseLogger.With(
    "component", "api",
    "module", "server",
)

// Request-specific logger
requestLogger := apiLogger.With(
    "request_id", requestID,
    "client_ip", clientIP,
)
```

This allows for consistent context across related log entries.

## Logger Implementations

### Phuslu Logger

The primary logger implementation uses the phuslu logger:

```go
type Logger struct {
    logger *phuslu.Logger
    level  Level
}

func NewLogger(level Level) *Logger {
    // Create logger with specified level
    logger := phuslu.NewLogger()
    logger.Level = mapLevel(level)
    
    return &Logger{
        logger: logger,
        level:  level,
    }
}
```

This implementation provides:
- High-performance logging
- Support for various output formats
- Configurable output destinations

### No-op Logger

A no-op logger implementation is available for disabling logging:

```go
type NoopLogger struct{}

func (l *NoopLogger) Debug(msg string, keyvals ...interface{}) {}
func (l *NoopLogger) Info(msg string, keyvals ...interface{})  {}
func (l *NoopLogger) Warn(msg string, keyvals ...interface{})  {}
func (l *NoopLogger) Error(msg string, keyvals ...interface{}) {}

func (l *NoopLogger) With(keyvals ...interface{}) Logger {
    return l
}

func (l *NoopLogger) SetLevel(level Level) {}
```

This implementation is useful for:
- Performance testing
- Disabling logging in specific components
- Reducing noise in test output

## Log Formatting

The logging system supports different output formats:

### JSON Format

```json
{
  "level": "info",
  "ts": "2023-05-18T10:15:30.123Z",
  "msg": "received request",
  "method": "GET",
  "path": "/api/v1/beacon/blocks",
  "client": "127.0.0.1"
}
```

### Text Format

```
2023-05-18T10:15:30.123Z INFO received request method=GET path=/api/v1/beacon/blocks client=127.0.0.1
```

The format can be configured based on the application needs.

## Output Destinations

Logs can be directed to multiple destinations:

```go
func ConfigureLogger(config LogConfig) (*Logger, error) {
    logger := NewLogger(config.Level)
    
    // Configure console output
    if config.Console.Enabled {
        logger.SetConsoleOutput(config.Console.Format)
    }
    
    // Configure file output
    if config.File.Enabled {
        if err := logger.SetFileOutput(config.File.Path, config.File.Format); err != nil {
            return nil, err
        }
    }
    
    // Configure syslog output
    if config.Syslog.Enabled {
        if err := logger.SetSyslogOutput(config.Syslog.Facility); err != nil {
            return nil, err
        }
    }
    
    return logger, nil
}
```

Common destinations include:
- Console (stdout/stderr)
- Log files
- Syslog
- Remote logging services

## Configuration

Logging is configured through a configuration system:

```go
type LogConfig struct {
    // Global log level
    Level Level
    
    // Console output configuration
    Console struct {
        Enabled bool
        Format  string // "json" or "text"
    }
    
    // File output configuration
    File struct {
        Enabled bool
        Path    string
        Format  string // "json" or "text"
    }
    
    // Syslog output configuration
    Syslog struct {
        Enabled  bool
        Facility string
    }
}
```

This configuration can be loaded from:
- Configuration files
- Environment variables
- Command-line flags

## Integration with Handlers

The logging system integrates with API handlers:

```go
func (r *Route) DecorateWithLogs(logger log.Logger) {
    handler := r.Handler
    r.Handler = func(ctx Context) (any, error) {
        logger.Info("received request",
            "method", r.Method,
            "path", r.Path,
        )
        
        res, err := handler(ctx)
        
        if err != nil {
            logger.Error("error handling request",
                "method", r.Method,
                "path", r.Path,
                "error", err,
            )
        } else {
            logger.Info("request handled",
                "method", r.Method,
                "path", r.Path,
            )
        }
        
        return res, err
    }
}
```

This integration ensures consistent logging across all API endpoints.

## Example Usage

```go
// Create logger
logger := log.NewLogger(log.InfoLevel)

// Log at different levels
logger.Debug("Processing block", "slot", slot) // Won't be output at InfoLevel
logger.Info("Block processed", "slot", slot, "root", blockRoot)
logger.Warn("Missed attestation", "validator", validatorIndex)
logger.Error("Failed to process block", "error", err)

// Create contextual logger
validatorLogger := logger.With("validator", validatorIndex)
validatorLogger.Info("Validator activated")
validatorLogger.Info("Attestation submitted", "slot", slot)

// Configure logger
config := LogConfig{
    Level: log.InfoLevel,
    Console: {
        Enabled: true,
        Format:  "text",
    },
    File: {
        Enabled: true,
        Path:    "/var/log/beacon-node.log",
        Format:  "json",
    },
}

configuredLogger, err := ConfigureLogger(config)
if err != nil {
    // Handle error
}
```

This logging system provides a flexible and consistent way to log information throughout the beacon node. 