---
title: "Building an efficient Logger in TypeScript"
categories:
  - Cloud
tags:
  - TypeScript
---

Just about every project needs a logging system.  In the early days of development, you need to output data to the console so you aren't setting breakpoints in your code all over the place (or you are running the code in the browser, where breakpoints are more difficult).  Later on, you want logging to let you know where people are spending their time, or how much usage a particular feature gets.  It also helps with diagnosing problems early so that you can get ahead of the issues.

There are a ton of logging frameworks out there.  A [search on npmjs.org](https://www.npmjs.com/search?q=logging%20framework) gives us over 150 results and the "logger" tag provides over 2700 packages to choose from, from the venerable ones ([Morgan](https://www.npmjs.com/package/morgan) and [Winston](https://www.npmjs.com/package/winston)) to newcomers ([Pino](https://www.npmjs.com/package/pino) and [consola](https://www.npmjs.com/package/consola)).  How does one choose with all this going on?

Fortunately, logging isn't hard.  However, if you want to offer a pluggable logging architecture, you need some requirements:

1. Logging should not interfere with the regular operation of your app.  If it does, you have failed.
2. You should be able to attach log destinations easily.
3. You should be able to log at different levels (trace, debug, info, warning, error).
4. You should be able to log at different levels in different areas of your app.

The levels come from POSIX logging and are granular enough to deal with just about everything. 

Why roll your own?  There are two reasons:

1. It's a good idea to reduce dependencies within your code.  If a downstream dependency makes a breaking change, it breaks your app.  Logging should not break things.
2. Writing your own reduces the size of the transpiled code, meaning it loads faster.  This is because the code is really not that complex.

I tend to err on writing my own code unless the library has some definable benefit that I won't take the time to replicate.  I'll use frameworks like React or Angular, since it's unlikely I will be able to replicate the functionality easily and there is no benefit to me doing so.  However, logging frameworks allow me to remove a dependency and control the size of the resulting code.

The way I produce this code is to have two parts linked by an event emitter.  The first part is a logger.  Your code calls this to send logs into the event emitter.  The second part is a receiver - it's job is to actually send the logs where they are going.  `EventEmitter` is standard in Node applications for a pub/sub type asynchronous message passing system.  If you are within a browser, you can use the `events` npm module to simulate the same thing.

> **Interesting Factoid**:  Most of the popular logging frameworks use event emitters under the covers!
> 
## The LogManager

The LogManager handles configuration and management of the logging system:

{% highlight js %}
import { EventEmitter } from 'events';

export class LogManager extends EventEmitter {
  private options: LogOptions = {
    minLevels: {
      '': 'info'
    }
  };

  // Prevent the console logger from being added twice
  private consoleLoggerRegistered: boolean = false;

  public configure(options: LogOptions): LogManager {
    this.options = Object.assign({}, this.options, options);
    return this;
  }

  public getLogger(module: string): Logger {
    let minLevel = 'none';
    let match = '';

    for (const key in this.options.minLevels) {
      if (module.startsWith(key) && key.length >= match.length) {
        minLevel = this.options.minLevels[key];
        match = key;
      }
    }

    return new Logger(this, module, minLevel);
  }

  public onLogEntry(listener: (logEntry: LogEntry) => void): LogManager {
    this.on('log', listener);
    return this;
  }

  // public registerConsoleLogger()
}

export interface LogEntry {
  level: string;
  module: string;
  location?: string;
  message: string;
}

export interface LogOptions {
  minLevels: { [module: string]: string }
}

export const logging = new LogManager();
{% endhighlight %}

I'll come on to the details of a logger and log listener in a bit.  First, let's dissect this small amount of code, because it does a lot.  Firstly, let's talk about modules.  This is a hierarchical string that describes where you are in your code.  I use it by swapping the slashes for periods.  Thus, if I am in file `lib/my-class` within `MyClass`, I will use `lib.my-class.MyClass` as the module name.

The `getLogger()` method returns a logger that you can use.  Part of the process here is to work out what the minLevel is for this module.  I have a hierarchy of minLevels.  For instance, I might configure the log manager like this:

{% highlight js %}
logManager.configure({ minLevels: {
  '': 'error',
  'lib': 'info',
  'lib.my-class': 'debug'
}});
{% endhighlight %}

The most specific module wins.  For my hypothetical `MyClass` class, I'd have a minLevel of debug.  However, if I registered `lib.other-class`, then that would have a minLevel of info.  This allows me to nicely control the amount that I log on a per logger basis.

Let's take a look at the Logger next:

{% highlight js %}
export class Logger {
    private logManager: EventEmitter;
    private minLevel: number;
    private module: string;
    private readonly levels: { [key: string]: number } = {
        'trace': 1,
        'debug': 2,
        'info': 3,
        'warn': 4,
        'error': 5
    };

    constructor(logManager: EventEmitter, module: string, minLevel: string) {
        this.logManager = logManager;
        this.module = module;
        this.minLevel = this.levelToInt(minLevel);
    }

    /**
     * Converts a string level (trace/debug/info/warn/error) into a number 
     * 
     * @param minLevel 
     */
    private levelToInt(minLevel: string): number {
        if (minLevel.toLowerCase() in this.levels)
            return this.levels[minLevel.toLowerCase()];
        else
            return 99;
    }

    /**
     * Central logging method.
     * @param logLevel 
     * @param message 
     */
    public log(logLevel: string, message: string): void {
        const level = this.levelToInt(logLevel);
        if (level < this.minLevel) return;

        const logEntry: LogEntry = { level: logLevel, module: this.module, message };

        // Obtain the line/file through a thoroughly hacky method
        // This creates a new stack trace and pulls the caller from it.  If the caller
        // if .trace()
        const error = new Error("");
        if (error.stack) {
            const cla = error.stack.split("\n");
            let idx = 1;
            while (idx < cla.length && cla[idx].includes("at Logger.Object.")) idx++;
            if (idx < cla.length) {
                logEntry.location = cla[idx].slice(cla[idx].indexOf("at ") + 3, cla[idx].length);
            }
        }

        this.logManager.emit('log', logEntry);
    }

    public trace(message: string): void { this.log('trace', message); }
    public debug(message: string): void { this.log('debug', message); }
    public info(message: string): void  { this.log('info', message); }
    public warn(message: string): void  { this.log('warn', message); }
    public error(message: string): void { this.log('error', message); }
}
{% endhighlight %}

The main function here is the `log()` method.  This constructs a `LogEntry` and emits it to the event emitter to be gathered by whatever listeners you have defined.  Note the curious code within the center.  It tries to find a location for the log message by constructing an Error object.  The Error object usually contains a stack trace that can be used to determine the location in your code.

Let's see how your code will call this.  At the entrypoint to your application, set up logging.  If you just want the defaults, nothing need be done.  However, you usually want to set up a log destination.  I provide a console log driver in my example code:

{% highlight js %}
    public registerConsoleLogger(): LogManager {
        if (this.consoleLoggerRegistered) return this;

        this.onLogEntry((logEntry) => {
            const msg = `${logEntry.location} [${logEntry.module}] ${logEntry.message}`;
            switch (logEntry.level) {
                case 'trace':
                    console.trace(msg);
                    break;
                case 'debug':
                    console.debug(msg);
                    break;
                case 'info':
                    console.info(msg);
                    break;
                case 'warn':
                    console.warn(msg);
                    break;
                case 'error':
                    console.error(msg);
                    break;
                default:
                    console.log(`{${logEntry.level}} ${msg}`);
            }
        });

        this.consoleLoggerRegistered = true;
        return this;
    }
{% endhighlight %}

You can see how to register a new log driver from this code.  The important thing to note here is that you are already in an async context from your main code, so the activities on the log driver don't affect your code.  The code, once transpiled and minified, is less than 8K, most of which is the EventEmitter code.

Back to setting up the code:

{% highlight js %}
import { logging, LogEntry } from './lib/logging';

logging
  .configure({
    minLevels: {
      '': 'info',
      'core': 'warn'
    }
  })
  .registerConsoleLogger();
{% endhighlight %}

You can do this at the entrypoint.  Then, in each file that you want to log:

{% highlight js %}
import { logging } from './lib/logging';

const logger = logging.getLogger('core.module-name');

// Later on
logger.info(`This is my log message`);
{% endhighlight %}

The log message will get dumped to the console.  By writing my own, I can also enforce semantic logging (where you structure the text output so it is easily parsed - for example, by using key=value pairs) or even structured logging (where you record the data as JSON for even easier parsing).


