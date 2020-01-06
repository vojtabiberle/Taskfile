# Taskfile

This repository contains Taskfile templates for various projects and use cases. This idea is not mine but came from [adriancooney](https://github.com/adriancooney) and his [Taskfile](https://github.com/adriancooney/Taskfile). I just improve thins to suite my ise cases.

# Install

To "install", add the following to your `.bashrc` or `.zshrc` (or `.whateverrc`):

```bash
# Quick start with the default Taskfile template
# Take first parameter as template name
# Without parameter use default template
function run-init {
    if [ -z "$1" ]; then
        TEMPLATE_NAME='default'
    else
        TEMPLATE_NAME=$1
    fi

    URL="https://raw.githubusercontent.com/vojtabiberle/Taskfile/master/Taskfile.$TEMPLATE_NAME.template"
    
    echo "Downloading template: $URL …"
    curl -so Taskfile $URL && chmod +x Taskfile                                                                                           
}

# Run your tasks like: run <task>
alias run=./Taskfile
```

You don't want to use this global alias? Newermind, just use `./Taskfile` in project root directory.

# Usage

Open your directory and run `run-init` to add the default `Taskfile` template to your project directory:

```bash
$ cd my-project
$ run-init
```

To use specific template, add name as a parameter to `run-init`

```bash
$ cd my-project
$ run-init docker-compose
```

Open the `Taskfile` and add your tasks. To run tasks, use `run` alias (or `./Taskfile`):

```bash
$ run help
./Taskfile <task> <args>
Tasks:
     1  build
     2  build-all
     3  help
Task completed in 0m0.005s
```

# Techniques

## Arguments

Let’s pass some arguments to a task. Arguments are accessible to the task via the `$1`, `$2`, `$n` … variables. Let’s allow us to specify the port of the HTTP server:

```bash
#!/usr/bin/env bash

function serve {
    php -S localhost:$1
}

"$@"
```

And now we can run the `serve` task with specific port:

```bash
$ run serve 9090
[Mon Jan  6 21:37:31 2020] PHP 7.4.1 Development Server (http://localhost:9090) started
```

## Using PHP Packages

Many popular PHP packages have runner in `vendor/bin` folder, we can use `Taskfile` to run those tools:

```bash
#!/usr/bin/env bash

PATH=./vendor/bin:$PATH

function serve {
    php -S localhost:$1
}

function lint {
    phpstan
}

"$@"

unset PATH
```

## Task Dependencies

Sometimes tasks depend on other tasks to be completed before they can start. To add another task as a dependency, simply call the task's function at the top of the dependant task's function.

```bash
#!/usr/bin/env bash

PATH=./vendor/bin:$PATH

function serve {
    php -S localhost:$1
}

function lint {
    phpstan
}

function test {
    codecept
}

function check {
    phpstan && test
}

"$@"

unset PATH
```

## Parallelisation

To run tasks in parallel, you can use Bash `&` operator in conjunction with `wait`. The following will build the two tasks at the same time and wait until they’re completed before exiting.

```bash
#!/usr/bin/env bash

function build {
    echo "beep $1 boop"
    sleep 1
    echo "built $1"
}

function build-all {
    build web & build mobile &
    wait
}

"$@"

unset PATH
```

And execute the `build-all` task:

```bash
$ run build-all
beep web boop
beep mobile boop
built web
built mobile
```

## Default task

To make a task the default task called when no arguments are passed, we can use bash’s default variable substitution `${VARNAME:-<default value>}` to return `default` if `$@` is empty.

```bash
#!/usr/bin/env bash

function build {
    echo "beep boop built"
}

function default {
    build
}

${@:-default}

unset PATH
```

Now when we run `./Taskfile`, the `default` function is called.

## Runtime Statistics

To add some nice runtime statistics like Gulp so you can keep an eye on build times, we use the built in time and pass if a formatter.

```bash
#!/usr/bin/env bash

function build {
    echo "beep boop built"
}

function default {
    build
}

TIMEFORMAT="Task completed in %3lR"
time ${@:-default}

unset PATH
```

And if we execute the `build` task:

```bash
$ ./Taskfile build 
beep boop built 
Task completed in 0m1.008s

```

## Help

The final addition I recommend adding to your base `Taskfile` is the task which emulates help. In a much more basic fashion, with no arguments. It prints out usage and the available tasks in the `Taskfile` to show us what tasks we have available to ourself.

```bash
function help {
    echo "$0 <task> <args>"
    echo "Tasks:"
    compgen -A function | cat -n
}
```

The `compgen -A function` is a [bash builtin](https://www.gnu.org/software/bash/manual/html_node/Programmable-Completion-Builtins.html) that will list the functions in our `Taskfile` (i.e. tasks). This is what it looks like when we run the task:

```bash
$ ./Taskfile help
./Taskfile <task> <args>
Tasks:
     1  build
     2  default
     3  help
Task completed in 0m0.005s
```

## `task:` namespace

If you find you need to breakout some code into reusable functions that aren't tasks by themselves and don't want them cluttering your `help` output, you can introduce a namespace to your task functions. Bash is pretty lenient with it's function names so you could, for example, prefix a task function with `task:`. Just remember to use that namespace when you're calling other tasks and in your `task:$@` entrypoint! 

```bash
#!/usr/bin/env bash

function task:build-web {
    build-target web
}

function task:build-desktop {
    build-target desktop
}

function build-target {
    BUILD_TARGET=$1 webpack --production
}

function task:default {
    task:help
}

function task:help {
    echo "$0 <task> <args>"
    echo "Tasks:"

    # We pick out the `task:*` functions
    compgen -A function | sed -En 's/task:(.*)/\1/p' | cat -n
}

TIMEFORMAT="Task completed in %3lR"
time "task:${@:-default}"
```

Or you can offcourse use `task:` just for separing comands into groups.

```bash
#!/usr/bin/env bash

function deploy:database {
    echo 'Deploying database'
}

function deploy:source {
    echo 'Deploying source'
}

function deploy {
    deploy:source && deploy:databse
}

function cache:clear {
    echo 'Clearing cache'
}

function cache:warmup {
    echo 'WarmUping cache'
}

function help {
    echo "$0 <task> <args>"
    echo "Tasks:"
    compgen -A function | cat -n
}

TIMEFORMAT="Task completed in %3lR"
time "task:${@:-default}"
```

## Free Features

- Conditions and loops. Bash and friends have support for conditions and loops so you can error if parameters aren’t passed or if your build fails.
- Streaming and piping. Don’t forget, we’re in a shell and you can use all your favourite redirections and piping techniques.
- All your standard tools like `rm` and `mkdir`.
- Globbing. Shells like `zsh` can expand globs like `**/*.js` for you automatically to pass to your tools.
- Environment variables like `NODE_ENV` are easily accessible in your Taskfiles.

### Considerations

When writing my Taskfile, these are some considerations I found useful:

- You should try to use tools that you know users will have installed and working on their system. I’m not saying you have to be POSIX.1 compliant but be weary of using tools that aren’t standard (or difficult to install).
- Keep it pretty. The reason for the `Taskfile` format is to keep your tasks organised and readable.
- Don’t completely ditch the `composer.json`. You should proxy the scripts to the Taskfile by calling the Taskfile directory in your `composer.json` like `"test": "./Taskfile test"`. You can still pass arguments to your scripts with the `--` special argument and `composer build -- --production` if necessary.

### Caveats

The only caveat with the Taskfile format is we forgo compatibility with Windows which sucks. Of course, users can install Cygwin but one of most attractive things about the Taskfile format is not having to install external software to run the tasks. Hopefully, [Microsoft’s native bash shell in Windows 10](https://www.howtogeek.com/249966/how-to-install-and-use-the-linux-bash-shell-on-windows-10/) can do work well for us in the future.