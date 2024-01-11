# lichen: the sed build system

For sedists, or perhaps sadists.

# ABOUT

While C enjoys make, and Java her Maven, it has come to our attention that sed up to now, has had no build system of its own. Let us correct this oversight. Enter, the sed build system, designed for sed projects, and written itself in GNU sed.

# EXAMPLE

```console
$ cd example

$ ./lichen
help
usage:
<task>
<task>
<task>
...
<ctrl-d>

tasks:
* help
* test
* lint

test
info: pass

lint
./bad.sed
warn: strange permissions: bad.sed

(Control+D)

$
```

Here, we see our sed build system operating as a REPL. We invite the user to enter task names, one per line.

When you're finished, press `Control+D` to terminate the lichen session.

# REQUIREMENTS

* GNU or BSD [findutils](https://en.wikipedia.org/wiki/Find_(Unix))
* [GNU sed](https://www.gnu.org/software/sed/manual/sed.html) 4.2+
* POSIX compatible [sh](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/sh.html)

# CONFIGURATION

Let's open [example/lichen](example/lichen) for reading.

## Shebang!

At the top, we have a not strictly POSIX compliant, multi-argument shebang:

```sh
#!/usr/bin/env gsed -nE -f
```

It may break on some fringe UNIX implementations, but it is necessary for the system to work.

The `-n` sed flag in the shebang, hides some REPL input mirroring. Try temporarily removing `n` in `-nE`, and you'll quickly experience this noisy behavior.

The `-E` sed flag enables ERE syntax, a more modern and capable regular expression language.

The `-f` flag instructs sed to load the the file as a series of editing commands.

## Don't cross the streams!

Next, we have a warning against output formats matching input patterns.

```sh
# Warning: Ensure none of the output formats match any of the input patterns.
```

This is because sed will apply *all* the sed editing commands that eventually match the transformed user input line. If the name of a build task accidentally matches the *output* text block of a running task, then we may see the computer go crazy!

In order to avoid cross-contamination between tasks, we must tune the output format so that no output block will trigger further processing by another sed editing command.

The easiest way to do this is to introduce a uniquely spicy prefix for each task's output. For example, our unit tests feature an `info:` log level prefix, and we will take pains to avoid creating a task called `info`. Even if we did, the colon delimiter (`:`) in the output prefix, catches most of these kinds of accidents.

By the way, avoid colons in task names. Better yet, limit task names to alphanumerics, ideally lowercase for simplicity and typing speed.

## Validation

We have a beefy validation command:

```
/^(test|lint|help)$/!s/(.+)/error: no such task: \1/p
```

The validation pattern uses a hardcoded group `(test|lint|help)`, permitting any of three tasks `test`, `lint`, `help` through the build system.

If the user input line does not match exactly one of these entries, then lichen emits a validation error. In the event that the task names grow, shrink, or rename, then this list must be updated to reflect the change.

## Creating Simple Tasks

In lichen, a task is created by writing a sed editing command. For example, we have a rather droll, NO-OP unit test suite:

```
s/^test$/echo 'info: pass'/ep
```

Naturally, this test suite is bereft of meaning. It states that it passes, without performing any significant checks. But this is merely a placeholder. You will want to replace `echo 'info: pass'` with some shell commands that test your own sed application.

The input pattern `^test$` matches the user REPL line `test`. When the user invokes the test this way, the `e` flag in `/ep` instructs sed to execute the output pattern as a shell command: `echo 'info: pass'`.

The `-n` sed flag in the shebang aggressively disables too much output. So the `p` flag in `/ep` at the end of the editing command, counteracts `-n`, turning output back on, but only for the REPL lines that match our patterns. Optionally, remove a `p` to silence command results.

After the `test` task, we have a more intricate `lint` task:

## Creating Complex Taks

```
s/^lint$/find . -iname '*.sed' -not -perm 0644 -print -execdir echo 'warn: strange permissions:' "{}" +/ep
```

This rudimentary `lint` task uses the GNU or BSD `find` utility to look for sed script filenames matching `*.sed` in the current directory tree.

Any sed scripts that both feature a `.sed` file extension and have chmod file permissions other than `0644` (octal), will generate a linter warning.

That's just a good habit to follow in UNIX, reserving file extensions on scripts for libraries, and reserving extensionless, executable chmod bits for applications. The practice applies to shell scripts, sed files, awk, Perl, Python, Ruby, and essentially any interpreted programming language script. The *metadata* about the script should help to reinforce the *intent* of the script, in case the author is unavailable.

A shebang should also be present in executable application scripts and removed from library scripts for the same reason. We leave implementation of sed shebang linter checks to [other systems](https://github.com/mcandre/stank).

Getting back to sed, review the precise syntax for this example lint task. Many sed editing command characters such as forward slash (`/`), back slash (`\`), caret (`^`), and dollar (`$`), just to name a few of these, will require additional back slashes.

Throw in escapes for any quoting, and the commands can get messy in a hurry.

When a command gets too hairy to manage directly in a lichen script, then chop it up into short `.lichen.d/<script>` files, or even short shell scripts. You can call scripts by path in your output command patterns, like any other shell command.

## Usage Menu

```
s/^help$/printf 'usage\:\n<task>\n<task>\n<task>\n...\n<ctrl-d>\n\ntasks:\n'; gsed -nE 's\/s\\\/\\^([^$]+)\\$.*\$\/* \\1\/p' <.\/lichen/ep
```

Here's a fun one. This editing command triggers on the `help` task. It emits a usage message to clarify basic program interaction. And even extracts the task names automatically! sed reading sed code.

## Further Research

Triggering multiple tasks in rapid succession is not as easy as space delimiting, due to the cross-contamination problem. One workaround involves copying and pasting a text block with the desired tasks to execute together.

```txt
test
lint
(empty line here)
```

That should enable automation, for larger CI/CD systems that want to kick off preselected task lists without requiring a human REPL operator. However, this method of interaction is still awkward (sedward?) to use. Compared to traditional space delimited `test lint` subcommands in like classic `make test lint`. Perhaps an outer lichen script could transform spaces into lines a la `xargs`, passing an intermediate newline delimited text block onto an inner lichen script. Regardless, the choice of delimiter and validation logic should balance UX appearances with predictable, reliable semantics.

As an aside, worse still for any hypothetical CI/CD, 100% sed pipeline, is the fact that exit codes do not appear to persist oustide of the sed editing command, nor does an explicit `exit` non-zero trigger termination of sed script processing. Any error handling for failing tasks, is currently a manual coding exercise, using error log entries to bring attention to the problem. Something that tends to get drowned out in large CI/CD pipelines with a lot of useless noise. Imagine having to debug an application build, with the build script continuing to run past the error, and only the chance of a corresponding error message buried deep in the logs! That's why exit code semantics are so important for production grade software components. But treating failing editing command `e` flag commands as sed application termination events, is something that GNU, BSD, and POSIX could theoretically agree to begin doing someday. Who knows, could be a one line change!

In principle, task trees could be constructed by chaining together multiple `.lichen.d/<script>` files together, using `s/^...$/gsed.../ep` editing commands to form task aggregates. If a hellish DevOps team out there has to debug a sed based build system, might as well keep it as organized as possible.

For all the warts and fragility of common build systems like `make`, lichen does miss out on some surprisingly convenient features. A wrapping *awk* script could use associative arrays as logical sets, in order to deduplicate redundant task invocations from `test lint lint lint`... to just `test lint`. And if one does ever implement task trees, then deduplicating redundant task dependencies shared by different parent tasks, becomes a performance concern. Lulz.

# SEE ALSO

* Inspiration from [nobuild](https://github.com/tsoding/nobuild), a convention for C/C++ build systems
* [bashate](https://github.com/openstack/bashate), a shell script style linter
* [bb](https://github.com/mcandre/bb), a build system for (g)awk projects
* [beltaloada](https://github.com/mcandre/beltaloada), a guide to writing build systems for (POSIX) sh
* [Gradle](https://gradle.org/), a build system for JVM projects
* [jelly](https://github.com/mcandre/jelly), a JSON task runner
* [lake](https://luarocks.org/modules/steved/lake), a Lua task runner
* [Leiningen](https://leiningen.org/) + [lein-exec](https://github.com/kumarshantanu/lein-exec), a Clojure task runner
* [Mage](https://magefile.org/), a task runner for Go projects
* [mian](https://github.com/mcandre/mian), a task runner for (Chicken) Scheme Lisp
* [npm](https://www.npmjs.com/), [Grunt](https://gruntjs.com/), Node.js task runners
* [POSIX make](https://pubs.opengroup.org/onlinepubs/009695299/utilities/make.html), a task runner standard for C/C++ and various other software projects
* [Rake](https://ruby.github.io/rake/), a task runner for Ruby projects
* [Rebar3](https://www.rebar3.org/), a build system for Erlang projects
* [rez](https://github.com/mcandre/rez) builds C/C++ projects
* [sbt](https://www.scala-sbt.org/index.html), a build system for Scala projects
* [Shake](https://shakebuild.com/), a task runner for Haskell projects
* [ShellCheck](https://www.shellcheck.net/), a shell script linter with a rich collection of rules for promoting safer scripting
* [slick](https://github.com/mcandre/slick), a linter to enforce stricter, unextended POSIX sh syntax compliance
* [stank](https://github.com/mcandre/stank), a collection of POSIX-y shell script linters
* [tinyrick](https://github.com/mcandre/tinyrick) for Rust projects
* [yao](https://github.com/mcandre/yao), a task runner for Common LISP projects

🪨
