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
