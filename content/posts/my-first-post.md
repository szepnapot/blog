---
title: "Bash best practices"
date: 2018-11-05T11:41:54+01:00
description: "Some best practices to help write complex bash scripts."
categories: [ "bash", "best practices", "unix", "shell"]
tags: ["bash", "unix", "shell", "linux"]
---

# Best practices and few useful snippets for writing Bash scripts 

## Intro

I've recently got a task to create a bash script to automatize the installation of a huge custom 'service' we use internally.

Without violating my NDA, it's a VoIP server with a few additional backend servers, databases, periodic jobs running alongside on the same instance. Common stuff I would say, but the the requirements was scary long and detailed, so I started to read a bit before, how to make my bash script writing days a little less painful :)

___

## Best practices

First the list, than I'll explain in more details.

1. `sh` != `bash` [details](#sh--bash)
* use `#!/usr/bin/env bash` [details](#shebang)
* use `set -o errexit` (a.k.a `set -e`) to make your script exit when a command fails [details](#set--e)
* add `|| true` to commands that you allow to fail
* use `set -u` to treat unset variables and parameters other than the special parameters ‘@’ or ‘*’ as an error when performing parameter expansion [details](#set--u)
* use `set -o pipefail` to prevent errors in a pipeline from being masked [details](#set--o-pipefail)
* set `IFS=$'\n\t'` (a.k.a Internal Field Separator) [details](#ifs)
* use `set -o xtrace` (a.k.a `set -x`) to trace what gets executed [details](#set--x)
* set magic variables for current file, basename, and directory at the top of your script [details](#magic-variables)
* use `readonly` and `local` to narrow down variable scope, mutability [details](#bash-variables)
* use exit traps [details](#exit-trap)


[top](#TableOfContents)

___

## sh != bash

`sh` (or the Shell Command Language) is a programming language described by the POSIX standard. It has many implementations, bash can also be considered an implementation of sh.

Because sh is a specification, not an implementation, `/bin/sh` is a symlink (or a hard link) to an actual implementation on most POSIX systems.

`bash` started as an sh-compatible implementation, but as time passed it has acquired many extensions. Many of these extensions may change the behavior of valid POSIX shell scripts, so by itself bash is not a valid POSIX shell. Rather, it is a dialect of the POSIX shell language.

bash supports a `--posix` switch, which makes it more POSIX-compliant. It also tries to mimic POSIX if invoked as sh. But there are a few differences:

* `[[` is not available in `sh` only `[`
* `sh` **does not** have arrays
* no keywords like `local`, `function`, and `select` are not portable to sh
* no `*.{png,jpg}` and `{0..9}` brace expansion
* no process substitution with `<(cmd)` and `>(cmd)`

For the complete list of differences check [this](https://www.gnu.org/software/bash/manual/html_node/Major-Differences-From-The-Bourne-Shell.html).

[top](#TableOfContents)
___

## shebang

`#!/usr/bin/env bash` should be used for portability.
Different *nixes put bash in different places, and using `/usr/bin/env` is a workaround to run the first bash found on the PATH.

`#!/bin/bash` is not 100% portable, some systems place bash in a location other than `/bin`.

[top](#TableOfContents)
___

## set -e

Probably the most useful one. This option instructs bash to immediately exit if any command has a non-zero exit status. You wouldn't want to set this for your command-line shell (or use with `|| true` or `set +e` before the problematic line than `set -e`), but in a script it's massively helpful.
By default, bash **does not** do this.

[top](#TableOfContents)
___

## set -u

When set, a reference to any variable you haven't previously defined - with the exceptions of $* and $@ - is an error, and causes the program to immediately exit.

```bash
#!/usr/bin/env bash
firstName="Peter"
secondName="Parker"
fullName="$firstname $secondName"
echo "$fullName"
```

Take a look, try to find the typo and guess what the result be.
If you execute the snippet above it will print `Parker` without any error thrown indicating that `firstname` variable was not defined.

If you however put `set -u` at the top of the script, it will fail, return code 1 and printing the message "firstname: unbound variable" to stderr.

Never fail silently.

[top](#TableOfContents)
___

## set -o pipefail


This setting prevents errors in a pipeline from being masked. If any command in a pipeline fails, that return code will be used as the return code of the whole pipeline. By default, the pipeline's return code is that of the last command - even if previous command failed. Take a look:

```bash
~|⇒ grep 'random text' /etc/php.ini.default | sort
~|⇒ echo $?
0
~|⇒ set -o pipefail
~|⇒ grep 'random text' /etc/php.ini.default | sort
~|⇒ echo $?
1
```

[top](#TableOfContents)
___

## IFS

It is used by the shell to determine how to do word splitting, i. e. how to recognize word boundaries.

```bash
items="a b c"
for x in $items; do
    echo "$x"
done
```
will print:

```bash
a
b
c
```

The default value for IFS consists of whitespace characters (to be precise: space, tab and newline).
However if you set `IFS=$'\n\t'`:

```bash
IFS=$'\n\t'
items="a b c"
for x in $items; do
    echo "$x"
done
```
will print:

```bash
a b c
```

Take a look how this looks if we're iterating over an array of names:

```bash
#!/usr/bin/env bash
names=(
  "Peter Parker"
  "Bruce Wayne"
  "Dr. Robert Bruce Banner"
  "Scott Lang"
)

echo "[*] default IFS value..."
for name in ${names[@]}; do
  echo "$name"
done

echo ""
echo "[*] strict-mode IFS value..."
IFS=$'\n\t'
for name in ${names[@]}; do
  echo "$name"
done
```

will print:

```bash
[*] default IFS value...
Peter
Parker
Bruce
Wayne
Dr.
Robert
Bruce
Banner
Scott
Lang

[*] strict-mode IFS value...
Peter Parker
Bruce Wayne
Dr. Robert Bruce Banner
Scott Lang
```

Setting IFS to `$'\n\t'` means that word splitting will happen **only** on newlines and tab characters. This very often produces useful splitting behavior (like in popular programming languages). By default, bash sets this to `$' \n\t'`, space, newline, tab.

[top](#TableOfContents)
___

## set -x

Use `set -o xtrace` (a.k.a `set -x`) to trace what gets executed. Useful for debugging. 

Commonly it's used with `PS4` to help debugging, like this:

```bash
#!/usr/bin/env bash
set -x
PS4='+${LINENO}: '

sleep 1m
sleep 1d
```

will print:

```bash
+ PS4='+${LINENO}: '
+5: sleep 1m
+6: sleep 1d
```

You can see that by setting `PS4='+${LINENO}: '` the currently executed line number also printed, followed by the executed command, thanks to `set -x`.

[top](#TableOfContents)
___

## magic variables

Set magic variables for current file, basename, and directory at the top of your script for convenience.

```bash
readonly __dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly __file="${__dir}/$(basename "${BASH_SOURCE[0]}")"
readonly __base="$(basename ${__file} .sh)"
readonly __root="$(cd "$(dirname "${__dir}")" && pwd)" # depends on your app

echo $__dir
echo $__file
echo $__base
echo $__root
```

[top](#TableOfContents)
___

## Bash variables

In Bash, constants are created by making a variable read-only. The `readonly` built-in marks each specified variable as unchangeable.

```bash
~/workspace|⇒ readonly a=1
~/workspace|⇒ echo $a
1
~/workspace|⇒ a=2
zsh: read-only variable: a
```

A variable declared as `local` is one that is visible only within the block of code in which it appears. It has local scope. In a function, a local variable has meaning only within that function block.

```bash
func() {
   nonlocal="Non local variable"
   local onlyhere="Local variable"
}
func
echo $nonlocal
echo $onlyhere
```

will print:

```bash
Non local variable
 
```

if you use the previously shown 'unused' contraint:

```bash
set -u
func() {
   nonlocal="Non local variable"
   local onlyhere="Local variable"
}
func
echo $nonlocal
echo $onlyhere
```

will result:

```bash
Non local variable
try.sh: line 9: onlyhere: unbound variable
```

[top](#TableOfContents)
___

## exit trap

Suppose your script is structured like my installer:

* Spin up some expensive resource
* install or build dependencies
* do something with them
* release that resource so it doesn't keep running and generate a giant bill, delete no longer needed stuffs

"Expensive resource" can be something like an EC2 instance, that cost real money, or it could be "just" a tmp directory you use during the installation but after it can be deleted.

If you use `set -e`, it's possible that an error will cause your script to exit before it can reach your cleanup block at the end of the script, which is not ideal. The solution is to use bash exit traps.

```
cleanup {
    INFO "Cleaning up..."
    rm -f *.tar.*
    apt-get autoclean
    apt-get install deborphan -y
    deborphan | xargs sudo apt-get -y remove --purge
    apt-get remove deborphan -y
}
trap cleanup EXIT
```

___

## Summary

I came up with this bash template, I'll use it pretty much for all my bash scripts since.

```bash
#!/usr/bin/env bash

set -o errexit
set -o pipefail
set -o nounset

cleanup {
    echo "Cleaning up..."
    apt-get autoclean
    apt-get install deborphan -y
    deborphan | xargs sudo apt-get -y remove --purge
    apt-get remove deborphan -y
}
trap cleanup EXIT

```


Took me a while, and a lot of iterations to finish my first bash installer, but it definitly worth it. Seeing that random co-workers were able to *just use* it, and it does what it should, was very uplifting.

[top](#TableOfContents)
