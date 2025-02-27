# exp branch

Commit the results from an [experiment](/doc/command-reference/exp) in a new Git
branch.

## Synopsis

```usage
usage: dvc exp branch [-h] [-q | -v] experiment branch

positional arguments:
  experiment     Experiment to turn into a branch
  branch         Name for the new Git branch
```

## Description

Creates a new [Git branch] containing the target `experiment`. The given
`branch` name will be used. It will stem from the experiment's baseline (`HEAD`
at the time the experiment was run).

This turns the experiment into one or more [regular commits] (one per checkpoint
in the case of [checkpoint experiments]).

<admon type="info">

Note that DVC **does not** switch into the new `branch` automatically.

</admon>

`dvc exp branch` is useful to make an experiment persistent without modifying
the workspace so they can be continued, [stored and shared] in a normal Git +
DVC workflow.

To switch into the new branch, use `git checkout branch` and `dvc checkout`. Or
use `git merge branch` and `dvc repro` to combine it with your current project
version.

[git branch]:
  https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging
[regular commits]: /doc/user-guide/experiment-management/persisting-experiments
[checkpoint experiments]: /doc/command-reference/exp/run#checkpoints
[stored and shared]: /doc/start/data-management#storing-and-sharing

## Options

- `-h`, `--help` - shows the help message and exit.

- `-q`, `--quiet` - do not write anything to standard output. Exit with 0 if no
  problems arise, otherwise 1.

- `-v`, `--verbose` - displays detailed tracing information from executing the
  `dvc pull` command.

## Example: Make a persistent branch from an experiment

> This example is based on our [Get Started](/doc/start/experiments), where you
> can find the actual source code.

Let's say we have run 3 experiments in our project:

```dvc
$ dvc exp show
```

```dvctable
 ────────────────────────────────────────────────────────────────────────────────────────────
  neutral:**Experiment**              neutral:**Created**           metric: **auc**   param:**featurize.max_features**   param:**featurize.ngrams**
 ────────────────────────────────────────────────────────────────────────────────────────────
  workspace               -              0.61314   1500                     2
  10-bigrams-experiment   Jun 20, 2020   0.61314   1500                     2
  ├── exp-e6c97           Oct 21, 2020   0.61314   1500                     2
  ├── exp-1dad0           Oct 09, 2020   0.57756   2000                     2
  └── exp-1df77           Oct 09, 2020   0.51676   500                      2
 ────────────────────────────────────────────────────────────────────────────────────────────
```

We may want to branch-off `exp-1dad0` for a separate experimentation process
(based on 2000 `max_features`).

```dvc
$ dvc exp branch exp-1dad0 maxf-2000
Git branch 'maxf-2000' has been created from experiment 'exp-1dad0'.
To switch to the new branch run:
        git checkout my-branch
```

We can inspect the result with Git:

```dvc
$ git branch
* master
  maxf-2000
```

`maxf-2000` can now be checked out, merged, rebased, pushed, etc. like any other
Git branch.
