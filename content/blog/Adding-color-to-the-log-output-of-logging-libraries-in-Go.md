---
title: "Adding Color to the Log Output of Logging Libraries in Go"
date: 2024-02-17T21:46:28+05:30
draft: false
---
Logging is an integral part of software development, providing developers with valuable insights into the behaviour and performance of their applications. In the Go programming language, various logging libraries, such as the standard library's `log` package or third-party options like `logrus`, `zap` and `zerolog`, facilitate the generation of log output. While the primary goal of logging is to convey information, the traditional black-and-white log messages can sometimes make it challenging to quickly discern critical information amidst a sea of logs.

## Need for colouring logs

**Prioritisation and Highlighting**: Colour can be used to prioritise and highlight critical information. For example, error messages or warning logs can be displayed in attention-grabbing colours like red or yellow, making it immediately apparent when an issue requires urgent attention. This facilitates a faster response to potential problems.

**Enhanced Readability**: Colours can improve the overall readability of log messages by adding structure and visual hierarchy. Differentiating between log levels, timestamps, and contextual information becomes more intuitive, leading to a more user-friendly experience during log analysis and troubleshooting.

**User-Friendly Debugging**: Developers spend a considerable amount of time interacting with logs during debugging. colour logging contributes to a more user-friendly debugging experience by allowing developers to quickly spot relevant information, errors, or patterns in log outputs, thereby expediting the debugging process.

### Support for colouring log level keywords

Nearly all logging libraries offer the option to enable colorization for log level keywords such as info, debug, warning and error.

In the case of Zap, you can use the `CapitalColorLevelEncoder` function to achieve this effect.

Here's an example of how you might configure Zap to enable colourised log levels:

```go
logCfg := zap.NewDevelopmentConfig()
logCfg.EncoderConfig.EncodeLevel = zapcore.CapitalColorLevelEncoder
```

### Contextual Logs

In complicated systems and tricky debugging situations, it's crucial to add extra details to log messages. This additional information, like variable values, timestamps, and user IDs, helps developers grasp what's happening in the application at specific times. Contextual logging jazzes up log messages by connecting them to dynamic key-value pairs, giving more flexibility than sticking to fixed formats. This method is super helpful during debugging, making it quicker to find the main issues and understand how the program is running.

Here is an example of how you can display contextual logs in zerolog logging library:


```go
package main

import "github.com/rs/zerolog/log"

func main() {
    log.Debug()
        .Str("Scale", "833 cents")
        .Float64("Interval", 833.09)
        .Msg("Fibonacci is everywhere")
}
// output:
// {"level":"debug","Scale":"833 cents","Interval":833.09,"time":"2024-01-24T22:01:03+05:30","message":"Fibonacci is everywhere"}
```

### Lack of native support to colour contextual logs

While contextual logging provides a powerful means of enhancing information, the visual representation of this context often remains monochromatic. Unfortunately, many logging libraries do not inherently support the colorization of key and value components within log entries. This absence leaves developers with a missed opportunity to leverage visual cues for quick identification and differentiation of critical information, hindering the efficiency of log analysis and debugging processes.

lets see what will happen if we try to add colour to context log in zap :

```go
package main

import (
    "github.com/fatih/color"
    "go.uber.org/zap"
)

func main() {
    PlainLogger, _:= zap.NewDevelopment()
    var HighlightGreen = color.New(color.FgGreen).SprintFunc()
    ColorLogger.Info("test log", zap.String(HighlightGreen("key"), "value"))
}
```

The output generated would be:

```shell
2024-01-24T22:23:16.356+0530 INFO zap-logging/main.go:66 test log {"\u001b[32mkey\u001b[0m": "value"}
```

The reason why the ANSI escape code for colour formatting are not interpreted is because of the way the `EncoderEntry` function is implemented in `zapcore` package.

We will now discuss how to solve this misinterpretation.

### Colouring contextual logs

The primary cause of the mentioned problem lies with the default encoder, which can be either "json" or "console".

To address this, we'll begin by crafting our own encoder.

1. Lets create a custom encoder named `colorConsoleEncoder` will be initialized by `NewColorConsole`:
   
   ```go
    type colorConsoleEncoder struct {
        *zapcore.EncoderConfig
        zapcore.Encoder
    }
    
    func NewColorConsole(cfg zapcore.EncoderConfig) (enc zapcore.Encoder) {
        return colorConsoleEncoder{
            EncoderConfig: &cfg,
            // Using the default ConsoleEncoder can avoid rewriting interfaces such as ObjectEncoder
            Encoder: zapcore.NewConsoleEncoder(cfg),
        }
    }
    ```
    
2. We will then register our encoder using `RegisterEncoder` function
    ```go
    func init() {
        _ = zap.RegisterEncoder("colorConsole", func(config zapcore.EncoderConfig) (zapcore.Encoder, error) {
            return NewColorConsole(config), nil
        })
    }
    ```
3. 
    The misinterpretation issue discussed earlier is associated with the `EncodeEntry` function within the [`Encoder`]("https://pkg.go.dev/go.uber.org/zap/zapcore#Encoder") interface of the `zapcore` package. To address this, it is necessary to override this function.
    ```go
    // EncodeEntry overrides ConsoleEncoder's EncodeEntry
    func(c colorConsoleEncoder) EncodeEntry(ent zapcore.Entry, fields []zapcore.Field) (buf *buffer.Buffer, err error) {
        buff, err := c.Encoder.EncodeEntry(ent, fields) // Utilize the existing implementation of zap
        if err != nil {
            return nil, err
        }
        
        bytesArr := bytes.Replace(buff.Bytes(), []byte("\\u001b"), []byte("\u001b"), -1)
        buff.Reset()
        buff.AppendString(string(bytesArr))
        return buff, err
    }
    ```
    
    This function will utilize the existing `EncodeEntry` implementation from the embedded `Encoder` field but introduces a correction using `bytes.Replace`. The aim is to replace occurrences of "\\u001b" (which represent a literal backslash followed by the characters "u001b") with the ANSI escape code "\u001b" (representing the escape character). This adjustment ensures accurate handling of ANSI escape codes during log entry encoding, allowing for the intended colorization without disrupting the overall logging system functionality.
    
4. 
    we can now create a `setupLogger` function that utilizes the registered encoder and produces a logger with colorization.
    ```go
    func setupLogger() *zap.Logger {
        logCfg := zap.NewDevelopmentConfig()
        
        logCfg.Encoding = "colorConsole"
        
        logger, _ := logCfg.Build()
        return logger
    }
    ```

5. Write a Log to see the effect
    ```go
    func main() {     
        ColorLogger := setupLogger()
        var HighlightGreen = color.New(color.FgGreen).SprintFunc()
        var HighlightYellow = color.New(color.FgYellow).SprintFunc()
        ColorLogger.Info("test log", zap.String(HighlightGreen("key"), HighlightYellow("value")))
    }
    ```
    ![output-color](/ouput-colorlog.png)
    
Full Code: "https://github.com/AkashKumar7902/coloring-log-output

While we've specifically covered this aspect for the Zap logging library, comparable solutions can be identified for other logging libraries as well :)

Thank you and Happy colouring 🎨 !