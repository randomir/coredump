# On logging in Python


## Question: [Logging level randomly changing and not always writing to file](https://stackoverflow.com/q/44957262/404556)

I have this piece of code to set my logger in Python:

      #Configure logging
      logging.basicConfig(format = '%(asctime)s %(name)-12s %(levelname)-8s %(message)s',
                          datefmt='%m-%d %H:%M',
                          filename= "log.txt",
                          level = logging.getLevelName('DEBUG'))


      print(logging.getLogger().getEffectiveLevel())

But the output from the print statement sometimes is this:

    30

And other times is this (which is correct):
    
    10

But often even when the the logging level is set to the correct number, it is not logging anything
to the file, but other times it works. What do I need to do to make sure my logging level is set
correctly?


## Answer

The [`logging.basicConfig()`](https://docs.python.org/3/library/logging.html#logging.basicConfig)
creates a root logger for you application (that's a logger with the name `root`, you can get it with
`logging.getLogger('root')`, or `logging.getLogger()`).

The trick is that the `root` logger gets created with defaults (like `level=30`) on the first call
to any logging function (like `logging.info()`) if it doesn't exist already. So, make sure you call
your `basicConfig()` before any logging in any other part of the application.

You can do that by extracting your logger config to a separate module, like `logger.py`, and then
import that module in each of your modules. Or, if your application has a central entry point, just
do the configuration there. Note that the 3rd party functions you call will also create the `root`
logger if it doesn't exist.

Also note, if you application is multi-threaded:

> **Note** This function should be called from the main thread before other threads are started. In
versions of Python prior to 2.7.1 and 3.2, if this function is called from multiple threads, it is
possible (in rare circumstances) that a handler will be added to the root logger more than once,
leading to unexpected results such as messages being duplicated in the log.

