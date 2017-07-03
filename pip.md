# On `pip`


## Question: [How to print warnings and errors when using setuptools (pip)](https://stackoverflow.com/questions/44616823/how-to-print-warnings-and-errors-when-using-setuptools-pip/)

I am using setuptools to package code such that it can be easily installed using 

    cd project_name && pip install .

During the setup process, I want to warn the user about pre-existing config files and print some post install instructions on the system. For example

    /etc/project_name/project.conf exists. Not copying default config file.

I have tried to use `print` and `logging.warning()` but still the warnings don't appear when installing using pip. I have a feeling I am missing something obvious.

We are trying to support 3.0 > python >= 2.6 on Redhat family >= el6 and Ubuntu >= 14.04LTS


## Answer

If you take a look at the pip source, in the function responsible for running the setup script, ``call_subprocess`` ([source here][1]), it says:

    def call_subprocess(cmd, show_stdout=True, cwd=None, ...
        ...
        # The obvious thing that affects output is the show_stdout=
        # kwarg. show_stdout=True means, let the subprocess write directly to our
        # stdout. Even though it is nominally the default, it is almost never used
        # inside pip (and should not be used in new code without a very good
        # reason); as of 2016-02-22 it is only used in a few places inside the VCS
        # wrapper code. Ideally we should get rid of it entirely, because it
        # creates a lot of complexity here for a rarely used feature.
        #
        # Most places in pip set show_stdout=False. What this means is:
        # - We connect the child stdout to a pipe, which we read.
        # - By default, we hide the output but show a spinner -- unless the
        #   subprocess exits with an error, in which case we show the output.
        # - If the --verbose option was passed (= loglevel is DEBUG), then we show
        #   the output unconditionally. (But in this case we don't want to show
        #   the output a second time if it turns out that there was an error.)

In short, you can see the output only if:

 - your setup exits with an error, or 
 - user calls pip with ``-v``, the verbose flag


  [1]: https://github.com/pypa/pip/blob/master/pip/utils/__init__.py#L624


---


## Question: [How to pip install a Python program such that it has a command line shortcut](https://stackoverflow.com/q/44888963/404556)

I'm trying to write a Python program which I can `pip install` so that it has a certain command-line alias, similar to the 'aws' command with the `awscli` package (cf. https://pypi.python.org/pypi/awscli/1.11.115).

The `setup.py` from the `awscli` seems to contain something like

    setuptools.setup(console='bin/aws')

which is presumably what makes the command line 'alias' work. However, at http://setuptools.readthedocs.io/en/latest/setuptools.html I was not able to easily find documentation or examples about how to use this `console` option.

My simplified use case is as follows. I have a directory `sayhello` which contains the following:

    .
    ├── sayhello.py
    └── setup.py

where `sayhello.py` has both a function (`say_hello`) defined and an `if __name__ == "__main__"` block to invoke this function:

    def say_hello():
        print("Hello, world!")
    
    if __name__ == "__main__":
        say_hello()

The `setup.py` I've tried is

    from setuptools import setup, find_packages
    
    setup(name="sayhello", version="1.0", packages=find_packages(), console="/bin/sayhello")

Then in the `sayhello` directory, I do

    pip install .

(after `virtualenv venv` and `source venv/bin/activate` to do it in a virtual environment). After this, I'm able to `import sayhello` and `sayhello.say_hello()` in a Python shell, for example, but the keyboard shortcut `sayhello` which I tried to define doesn't work.

How can I modify the `setup.py` such that the command `sayhello` triggers the `if __name__ == "__main__"` block in `sayhello.py`?


## Answer

Suppose you have a `sayhello.py` in the root of your package. Then just add [`scripts`](https://docs.python.org/2/distutils/setupscript.html#installing-scripts) in your `setup.py`:

    setup(
        ...
        scripts=['sayhello.py']
    )

It's customary to put all scripts inside `scripts/` dir, make them executable, and put a hashbang (`#!/usr/bin/env python`) in the first line.

The second approach is via [`console_scripts`](http://python-packaging.readthedocs.io/en/latest/command-line-scripts.html#the-console-scripts-entry-point):

    setup(
        ...
        entry_points = {
            'console_scripts': ['sayhello=sayhello:say_hello'],
        }
        ...
    )

### A minimal example ###

In `sayhello.py` put:

    #!/usr/bin/env python
    print "Hello!"

and in `setup.py`:

    from setuptools import setup
    setup(
        name="sayhello",
        version="0.0.1",
        scripts=['sayhello.py']
    )

After running `pip install .`, you should have your `sayhello.py` script copied to virtual environment's `bin` directory (which is in your `PATH`).

Test the script:

    $ sayhello.py
    Hello!
