# atomic

`atomic` is a command that runs atomically on commands received as arguments.

## Quick start

### Installation

```sh
git clone https://github.com/ajfAfg/atomic.git
sh atomic/install.sh
```

### Usa case

It is mainly intended to be used in combination with [`fswatch`](https://github.com/emcrisostomo/fswatch). Even if two file changes are observed in one file save due to automatic formatting, the command can be executed only once.

```sh
fswatch . | xargs -P0 -n1 -I{} atomic echo foo
```

## Background

[`fswatch`](https://github.com/emcrisostomo/fswatch) command monitors file changes and outputs the observed file changes. This output can be used to create a mechanism to execute build and test when file changes.

One of the problems in using `fswatch` is related to automatic formatting when saving files. With this mechanism in place, `fswatch` observes two file changes in one file save, so build and test are also executed twice. On the other hand, since the user saves a file once, the user expects the build and test to be executed only once. This discrepancy in perception is the problem I wish to resolve.

## Approach

Ignore file changes that occur during a command execution. In this way, even if file changes are observed twice due to automatic formatting, build and test will be executed only once.

To realize this idea, I use locking to control exclusivity. That is, a lock is acquired before the command is executed, and the lock is released after execution. If the lock has already been acquired, the command is terminated without execution. `atomic` command performs this exclusive control.

Note that this approach cannot be achieved simply by using `atomic`, but requires parallel execution of `xargs`. This is because the current approach assumes that `xargs` executes commands in parallel. Precisely, the following modifications need to be made to the original `fswatch` usage.

```sh
# Before
fswatch . | xargs -n1 -I{} echo foo

# After
fswatch . | xargs -P0 -n1 -I{} atomic echo foo
```

### vs Method using file change observation time

`fswatch` can output the time when a file change is observed. Using this time, it is possible to determine if file changes were observed during the execution of a command, thus solving this problem in the same way as the above method. The pros and cons of the two methods are as follows:

- Exclusion control using a lock
  - Pros: `fswatch` output is free
  - Cons: When the number of processes that can be executed in parallel by `xargs` reaches the limit, the command may be executed multiple times
- Method using file change observation time
  - Pros: No need to run `xargs` in parallel
  - Cons: `fswatch` output is limited (must output time)

The latter cons is fatal, limiting the actions possible by users. On the other hand, if the number of file changes observed in a file save is often two, the former cons is less of a problem. Therefore, the former approach is taken in `atomic`.

## Known issue

When execution of the command is interrupted, the lock remains acquired.

```sh
$ atomic exit 0
# No output
$ atomic echo foo # "foo" should be output.
# No output
```

A solution to this problem is being sought.

## LICENSE

MIT
