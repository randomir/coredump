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


---


## Question: [Python logger per function or per module](https://stackoverflow.com/q/45063099/404556)

I am trying to start using logging in python and have read several blogs. One issue that is causing
confusion for me is whether to create the logger per function or per module. In this 
[Blog: Good logging practice inPython](https://fangpenlin.com/posts/2012/08/26/good-logging-practice-in-python/)
it is recommended to get a logger per function. For example:

    import logging
    
    def foo():
        logger = logging.getLogger(__name__)
        logger.info('Hi, foo') 

    class Bar(object):
        def __init__(self, logger=None):
            self.logger = logger or logging.getLogger(__name__)

        def bar(self):
            self.logger.info('Hi, bar')

The reasoning given is that 

> The logging.fileConfig and logging.dictConfig disables existing loggers by default. So, those
> setting in file will not be applied to your logger. It’s better to get the logger when you need
> it. It’s cheap to create or get a logger.

The recommended way I read everywhere else is like as shown below. The blog states that this
approach `"looks harmless, but actually, there is a pitfall"`.

    import logging
    logger = logging.getLogger(__name__)

    def foo():
        logger.info('Hi, foo') 

    class Bar(object):
        def bar(self):
            logger.info('Hi, bar')

I find the former approach to be tedious as I would have to remember to get the logger in each
function. Additionally getting the logger in each function is surely more expensive than once per
module. Is the author of the blog advocating a non-issue? Would following logging best practices
avoid this issue?


## Answer

I would agree with you; getting logger in each and every function you use creates too much
unnecessary cognitive overhead, to say the least.

The author of the blog is right about the fact that *you should be careful to properly initialize*
(configure) your logger(s) before using them.

But the approach he suggests makes sense only in the case you have no control over your application
loading and the application entry point (which usually you do).

To avoid premature (implicit) creation of loggers [that happens with a first call to any of the message logging functions](https://docs.python.org/3/library/logging.html#logging.basicConfig) (like `logging.info()`, `logging.error()`, etc.)
if a root logger hasn't been configured beforehand, **simply make sure you configure your logger before logging**.

Initializing the logger from the main thread before starting other threads is also recommended in
[Python docs](https://docs.python.org/2/library/logging.html#logging.basicConfig).

Python's logging tutorial ([basic](https://docs.python.org/3/howto/logging.html#logging-basic-tutorial) and
[advanced](https://docs.python.org/3/howto/logging.html#logging-advanced-tutorial)) can serve you as
a reference, but for a more concise overview, have a look at the [logging section of The Hitchhiker's Guide to Python](http://python-guide-pt-br.readthedocs.io/en/latest/writing/logging/)

For **a simple blueprint of logging from multiple modules**, have a look at this modified
[example from Python's logging tutorial](https://docs.python.org/3/howto/logging.html#logging-from-multiple-modules):

    # myapp.py
    import logging
    import mylib
 
    # get the fully-qualified logger (here: `root.__main__`)
    logger = logging.getLogger(__name__)    

    def main():
        logging.basicConfig(format='%(asctime)s %(name)-12s %(levelname)-8s %(message)s',
                            level=logging.DEBUG)
        # note the `logger` from above is now properly configured
        logger.debug("started")
        mylib.something()

    if __name__ == "__main__":
        main()

And

    # mylib.py
    import logging

    # get the fully-qualified logger (here: `root.mylib`)
    logger = logging.getLogger(__name__)

    def something():
        logger.info("something")

Producing this on `stdout` (note the correct `name`):

    $ python myapp.py
    2017-07-12 21:15:53,334 __main__     DEBUG    started
    2017-07-12 21:15:53,334 mylib        INFO     something

