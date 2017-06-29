# Bash (or POSIX shell) process ops


## Question: [How can I display intermediate pipeline results for NUL-separated data?](https://stackoverflow.com/questions/44730184/how-can-i-display-intermediate-pipeline-results-for-nul-separated-data/)

How can I combine the following two commands:

    find . -print0 | grep -z pattern | tr '\0' '\n'
    find . -print0 | grep -z pattern | xargs -0 my_command

into a single pipeline?  If I don't need NUL separators then I can do:

    find . | grep pattern | tee /dev/tty | xargs my_command

I want to avoid using a temporary file like this:

    find . -print0 | grep -z pattern > tempfile
    cat tempfile | tr '\0' '\n'
    cat tempfile | xargs -0 my_command
    rm tempfile


## Answer

One of the possibilities:

    find . -print0 | grep -z pattern | { exec {fd}> >(tr '\0' '\n' >/dev/tty); tee "/dev/fd/$fd"; } | xargs -0 command

Where we create a temporary file descriptor ``fd`` with ``exec`` on fly which is connected to
``tr``'s stdin via standard process substitution. ``tee`` passes everything to stdout (ending on
``xargs``), and a duplicate to a ``tr`` subprocess that outputs to ``/dev/tty``.

This can be simplified to:

    find . -print0 | grep -z pattern | tee >(tr '\0' '\n' > /dev/tty) | xargs -0 command


---


## Question: [Kill Java app process tree run from Bash script with Ctrl+C](https://stackoverflow.com/questions/44830690/kill-java-app-process-tree-run-from-bash-script-with-ctrlc/)

I have a script that runs a java application and keeps running foreground. The java application, in turn, runs another java application.
What I want to achieve is to close all java applications when I issue Ctrl+c.

Suppose something like this:

bash-file-1:

    trap "stopit; exit" SIGINT
    echo "Going to run java app"
    source bash-file-2.sh

bash-file-2.sh:

    exec java -jar wrapper.jar

The stopit() function just looks for a .pid file an kills the process. It works fine when running the app in background.

What happens here is that, when I issue Ctrl+c, I exit the console mode, but the application keeps running in background. I'm guessing this is because the application is running in another shell because of the 'exec' command. Is this correct?

How can I trap ctrl+c and call the stopit function in this case?


## Answer

If your child process (Java app) spawns new processes, you'll need to kill the complete process tree rooted at your app.

First, define a (generic) helper [``killtree``](https://github.com/randomir/envie/blob/master/scripts/envie#L160) function:

    #!/bin/bash

    # Prints all descendant of a process `ppid`, level-wise, bottom-up.
    # Usage: _get_proc_descendants ppid
    function _get_proc_descendants() {
        local pid ppid="$1"
        local children=$(ps hopid --ppid "$ppid")
        for pid in $children; do
            echo "$pid"
            _get_proc_descendants "$pid"
        done
    }
    
    # Kills all process trees rooted at each of the `pid`s given,
    # along with all of their ancestors.
    # Usage: killtree [pid1 pid2 ...]
    function killtree() {
        while [ "$#" -gt 0 ]; do
            local pids=("$1" $(_get_proc_descendants "$1"))
            kill -TERM "${pids[@]}" &>/dev/null
            shift
        done
    }

Then, in your ``trap``, say:

    trap "killtree $java_app_pid; exit" SIGINT

where the ``$java_app_pid`` is the PID of your Java app (you said you have in a pid file).

Alternatively, you can define ``killtree`` to kill only descendants of the PIDs:

    function killtree() {
        while [ "$#" -gt 0 ]; do
            local pids=($(_get_proc_descendants "$1"))
            kill -TERM "${pids[@]}" &>/dev/null
            shift
        done
    }

and simplify your ``trap`` (kill script's children):

    trap "killtree $$; exit" SIGINT
