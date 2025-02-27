# `dvc.yaml`

You can construct data science or machine learning pipelines by defining
individual [stages](/doc/command-reference/run) in one or more `dvc.yaml` files.
Stages form a pipeline when they connect with each other (forming a _dependency
graph_, see `dvc dag`). Refer to
[Get Started: Data Pipelines](/doc/start/data-management/data-pipelines).

<admon type="tip">

A helper command, `dvc stage`, is available to create and list stages.

</admon>

`dvc.yaml` files can be versioned with Git.

These files use the [YAML 1.2](https://yaml.org/) file format, and a
human-friendly schema explained below. We encourage you to get familiar with it
so you may modify, write, or generate stages and pipelines on your own.

<admon type="info">

We use [GNU/Linux](https://www.gnu.org/software/software.html) in these
examples, but Windows or other shells can be used too.

</admon>

## Stages

The list of `stages` contains one or more user-defined stages. Here's a simple
one named `transpose`:

```yaml
stages:
  transpose:
    cmd: ./trans.r rows.txt > columns.txt
    deps:
      - rows.txt
    outs:
      - columns.txt
```

<admon type="tip">

See also `dvc stage add`, a helper command to write stages in `dvc.yaml`.

</admon>

The most important part of a stage is the terminal command(s) it executes (`cmd`
field). This is what DVC runs when the stage is reproduced (see `dvc repro`).

If a [stage command](#stage-commands) reads input files, these (or their
directory locations) can be defined as <abbr>dependencies</abbr> (`deps`). DVC
will check whether they have changed to decide whether the stage requires
re-execution (see `dvc status`).

If it writes files or dirs, they can be defined as <abbr>outputs</abbr>
(`outs`). DVC will track them going forward (similar to using `dvc add`).

<admon type="tip">

Output files may be viable data sources for [top-level plots](#top-level-plots).

</admon>

<admon type="info">

See the full stage entry [specification](#stage-entries).

</admon>

### Stage commands

The command(s) defined in the `stages` (`cmd` field) can be anything your system
terminal would accept and run, for example a shell built-in, an expression, or a
binary found in `PATH`.

Surround the command with double quotes `"` if it includes special characters
like `|` or `<`, `>`. Use single quotes `'` instead if there are environment
variables in it that should be evaluated dynamically.

The same applies to the `command` argument for helper commands (`dvc stage add`,
`dvc exp init`), otherwise they would apply to the DVC call itself:

```cli
$ dvc stage add -n a_stage "./a_script.sh > /dev/null 2>&1"
$ dvc exp init './another_script.sh $MYENVVAR'
```

<admon type="tip">

See also [Templating](#templating) (and **Dict Unpacking**) for useful ways to
parametrize `cmd` strings.

</admon>

<details>

### 💡 Avoiding unexpected behavior

We don't want to tell anyone how to write their code or what programs to use!
However, please be aware that in order to prevent unexpected results when DVC
reproduces pipeline stages, the underlying code should ideally follow these
rules:

- Read/write exclusively from/to the specified <abbr>dependencies</abbr> and
  <abbr>outputs</abbr> (including parameters files, metrics, and plots).
- Completely rewrite outputs. Do not append or edit.
- Stop reading and writing files when the `command` exits.

Also, if your pipeline reproducibility goals include consistent output data, its
code should be
[deterministic](https://en.wikipedia.org/wiki/Deterministic_algorithm) (produce
the same output for any given input): avoid code that increases
[entropy](https://en.wikipedia.org/wiki/Software_entropy) (e.g. random numbers,
time functions, hardware dependencies, etc.).

</details>

### Parameters

<abbr>Parameters</abbr> are simple key/value pairs consumed by the `command`
code from a structured [parameters file](#parameters-files). They are defined
per-stage in the `params` field of `dvc.yaml` and should contain one of these:

1. A param name that can be found in `params.yaml` (default params file);
2. A dictionary named by the file path to a custom params file, and with a list
   of param key/value pairs to find in it;
3. An empty set (give no value or use `null`) named by the file path to a params
   file: to track all the params in it dynamically.

<admon type="info">

Dot-separated param names become tree paths to locate values in the params file.

</admon>

```yaml
stages:
  preprocess:
    cmd: bin/cleanup raw.txt clean.txt
    deps:
      - raw.txt
    params:
      - threshold # track specific param (from params.yaml)
      - nn.batch_size
      - myparams.yaml: # track specific params from custom file
          - epochs
      - config.json: # track all parameters in this file
    outs:
      - clean.txt
```

<admon type="tip">

Params are a more granular type of stage dependency: multiple `stages` can use
the same params file, but only certain values will affect their state (see
`dvc status`).

</admon>

#### Parameters files

The supported params file formats are YAML 1.2, JSON, TOML 1.0, [and Python].
[Parameter](#parameters) key/value pairs should be organized in tree-like
hierarchies inside. Supported value types are: string, integer, float, boolean,
and arrays (groups of params).

These files are typically written manually (or generated) and they can be
versioned directly with Git along with other <abbr>workspace</abbr> files.

[and python]: /doc/command-reference/params#examples-python-parameters-file

<admon type="tip">

See also `dvc params diff` to compare params across project version.

</admon>

### Metrics and Plots outputs

Like common output files, <abbr>metrics</abbr> and <abbr>plots</abbr> files are
produced by the stage `cmd`. However, their purpose is different. Typically they
contain metadata to evaluate pipeline processes. Example:

```yaml
stages:
  build:
    cmd: python train.py
    deps:
      - features.csv
    outs:
      - model.pt
    metrics:
      - accuracy.json:
          cache: false
    plots:
      - auc.json:
          cache: false
```

> `cache: false` is typical here, since they're small enough for Git to version
> directly.

The commands in `dvc metrics` and `dvc plots` help you display and compare
metrics and plots.

## Templating

_New in DVC 2.0 (see `dvc version`)_

`dvc.yaml` supports a templating format to insert values from different sources
in the YAML structure itself. These sources can be
[parameters files](/doc/command-reference/params), or `vars` defined in
`dvc.yaml` instead.

Let's say we have `params.yaml` (default params file) with the following
contents:

```yaml
models:
  us:
    threshold: 10
    filename: 'model-us.hdf5'
```

Those values can be used anywhere in `dvc.yaml` with the `${}` _substitution
expression_, for example to pass parameters as command-line arguments to a
[stage command](#stage-command):

<!-- prettier-ignore-start -->
```yaml
stages:
  build-us:
    cmd: >-
      python train.py
      --thresh ${models.us.threshold}
      --out ${models.us.filename}
    outs:
      - ${models.us.filename}:
          cache: true
```
<!-- prettier-ignore-end -->

DVC will track simple param values (numbers, strings, etc.) used in `${}` (they
will be listed by `dvc params diff`).

### Dict Unpacking

Only inside the `cmd` entries, you can also reference a dictionary inside `${}`
and DVC will _unpack_ it. For example, given the following `params.yaml`:

```yaml
dict:
  foo: foo
  bar: 2
  bool: true
  nested:
    foo: bar
  list: [1, 2, 'foo']
```

You can reference `dict` in the `cmd` section of a `dvc.yaml`:

```yaml
stages:
  train:
    cmd: python train.py ${dict}
```

And DVC will _unpack_ the values inside `dict`, creating the following `cmd`
call:

```cli
$ python train.py --foo 'foo' --bar 2 --bool \
                  --nested.foo 'bar' --list 1 2 'foo'
```

This can be useful for avoiding to write every argument passed to the `cmd` or
having to modify the `dvc.yaml` when adding or removing arguments.

<admon type="tip">

The [parsing](/doc/command-reference/config#parsing) section of `dvc config` can
be used to customize the syntax used for some ambiguous types like booleans and
lists.

</admon>

### Vars

Alternatively, values for substitution can be listed as top-level `vars` like
this:

```yaml
vars:
  - models:
      us:
        threshold: 10
  - desc: 'Reusable description'

stages:
  build-us:
    desc: ${desc}
    cmd: python train.py --thresh ${models.us.threshold}
```

<admon type="warn">

Values from `vars` are not tracked like parameters.

</admon>

To load additional params files, list them in the top `vars`, in the desired
order, e.g.:

> Params file paths will be evaluated based on [`wdir`](#stage-entries), if
> specified.

```yaml
vars:
  - params.json
  - myvar: 'value'
  - config/myapp.yaml
```

<admon type="info">

Note that the default `params.yaml` file is always loaded first, if present.

</admon>

It's also possible to specify what to include from additional params files, with
a `:` colon:

```yaml
vars:
  - params.json:clean,feats

stages:
  featurize:
    cmd: ${feats.exec}
    deps:
      - ${clean.filename}
    outs:
      - ${feats.dirname}
```

Stage-specific values are also supported, with inner `vars`. You may also load
additional params files locally. For example:

```yaml
stages:
  build-us:
    vars:
      - params.json:build
      - model:
          filename: 'model-us.hdf5'
    cmd: python train.py ${build.epochs} --out ${model.filename}
    outs:
      - ${model.filename}
```

DVC merges values from params files and `vars` in each scope when possible. For
example, `{"grp": {"a": 1}}` merges with `{"grp": {"b": 2}}`, but not with
`{"grp": {"a": 7}}`.

⚠️ Known limitations of local `vars`:

- [`wdir`](#stage-entries) cannot use values from local `vars`, as DVC uses the
  working directory first (to load any values from params files listed in
  `vars`).
- `foreach` is also incompatible with local `vars` at the moment.

The substitution expression supports these forms:

```yaml
${param} # Simple
${param.key} # Nested values through . (period)
${param.list[0]} # List elements via index in [] (square brackets)
```

> To use the expression literally in `dvc.yaml` (so DVC does not replace it for
> a value), escape it with a backslash, e.g. `\${...`.

## `foreach` stages

_New in DVC 2.0 (see `dvc version`)_

You can define more than one stage in a single `dvc.yaml` entry with the
following syntax. A `foreach` element accepts a list or dictionary with values
to iterate on, while `do` contains the regular stage fields (`cmd`, `outs`,
etc.). Here's a simple example:

```yaml
stages:
  cleanups:
    foreach: # List of simple values
      - raw1
      - labels1
      - raw2
    do:
      cmd: clean.py "${item}"
      outs:
        - ${item}.cln
```

Upon `dvc repro`, each item in the list is expanded into its own stage by
substituting its value in expression `${item}`. The item's value is appended to
each stage name after a `@`. The final stages generated by the `foreach` syntax
are saved to `dvc.lock`:

```yaml
schema: '2.0'
stages:
  cleanups@labels1:
    cmd: clean.py "labels1"
    outs:
      - path: labels1.cln
  cleanups@raw1:
    cmd: clean.py "raw1"
    outs:
      - path: raw1.cln
  cleanups@raw2:
    cmd: clean.py "raw2"
    outs:
      - path: raw2.cln
```

For lists containing complex values (e.g. dictionaries), the substitution
expression can use the `${item.key}` form. Stage names will be appended with a
zero-based index. For example:

```yaml
stages:
  train:
    foreach:
      - epochs: 3
        thresh: 10
      - epochs: 10
        thresh: 15
    do:
      cmd: python train.py ${item.epochs} ${item.thresh}
```

```yaml
# dvc.lock
schema: '2.0'
stages:
  train@0:
    cmd: python train.py 3 10
  train@1:
    cmd: python train.py 10 15
```

DVC can also iterate on a dictionary given directly to `foreach`, resulting in
two substitution expressions being available: `${key}` and `${item}`. The former
is used for the stage names:

```yaml
stages:
  build:
    foreach:
      uk:
        epochs: 3
        thresh: 10
      us:
        epochs: 10
        thresh: 15
    do:
      cmd: python train.py '${key}' ${item.epochs} ${item.thresh}
      outs:
        - model-${key}.hdfs
```

```yaml
# dvc.lock
schema: '2.0'
stages:
  build@uk:
    cmd: python train.py 'uk' 3 10
    outs:
      - path: model-uk.hdfs
        md5: 17b3d1efc339b416c4b5615b1ce1b97e
  build@us: ...
```

Importantly, dictionaries from
[parameters files](/doc/command-reference/params#examples) can be used in
`foreach` stages as well:

```yaml
stages:
  mystages:
    foreach: ${myobject} # From params.yaml
    do:
      cmd: ./script.py ${key} ${item.prop1}
      outs:
        - ${item.prop2}
```

> Note that this feature is not compatible with [templating](#templating) at the
> moment.

## Stage entries

These are the fields that are accepted in each stage:

| Field            | Description                                                                                                                                                                                                                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cmd`            | (Required) One or more commands executed by the stage (may contain either a single value or a list). [Learn more](#stage-commands).                                                                                                                                                       |
| `wdir`           | Working directory for the stage command to run in (relative to the file's location). Any paths in other fields are also based on this. It defaults to `.` (the file's location).                                                                                                          |
| `deps`           | List of <abbr>dependency</abbr> paths of this stage (relative to `wdir`).                                                                                                                                                                                                                 |
| `outs`           | List of stage <abbr>output</abbr> paths (relative to `wdir`). These can contain optional [subfields](#output-subfields).                                                                                                                                                                  |
| `params`         | List of <abbr>parameter</abbr> dependency keys (field names) to track from `params.yaml` (in `wdir`). The list may also contain other parameters file names, with a sub-list of the param names to track in them.                                                                         |
| `metrics`        | List of [metrics files](/doc/command-reference/metrics), and optionally, whether or not this metrics file is <abbr>cached</abbr> (`true` by default). See the `--metrics-no-cache` (`-M`) option of `dvc run`.                                                                            |
| `plots`          | List of [plot metrics](/doc/command-reference/plots), and optionally, their default configuration (subfields matching the options of `dvc plots modify`), and whether or not this plots file is <abbr>cached</abbr> ( `true` by default). See the `--plots-no-cache` option of `dvc run`. |
| `frozen`         | Whether or not this stage is frozen from reproduction                                                                                                                                                                                                                                     |
| `always_changed` | Causes this stage to be always considered as [changed] by commands such as `dvc status` and `dvc repro`. `false` by default                                                                                                                                                               |
| `meta`           | (Optional) arbitrary metadata can be added manually with this field. Any YAML content is supported. `meta` contents are ignored by DVC, but they can be meaningful for user processes that read or write `.dvc` files directly.                                                           |
| `desc`           | (Optional) user description for this stage. This doesn't affect any DVC operations.                                                                                                                                                                                                       |

[changed]: /doc/command-reference/status#local-workspace-status

`dvc.yaml` files also support `# comments`.

Note that we maintain a `dvc.yaml`
[schema](https://github.com/iterative/dvcyaml-schema) that can be used by
editors like [VSCode](/doc/install/plugins#visual-studio-code) or
[PyCharm](/doc/install/plugins#pycharmintellij) to enable automatic syntax
validation and auto-completion.

> See also
> [How to Merge Conflicts](/doc/user-guide/how-to/resolve-merge-conflicts#dvcyaml).

<admon type="warn">

While DVC is platform-agnostic, commands defined in `dvc.yaml` (`cmd` field) may
only work on some operating systems and require certain software packages or
libraries in the environment.

</admon>

### Output subfields

> These include a subset of the fields in `.dvc` file
> [output entries](/doc/user-guide/project-structure/dvc-files#output-entries).

| Field        | Description                                                                                                                                                                                                                                                                       |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cache`      | Whether or not this file or directory is <abbr>cached</abbr> (`true` by default). See the `--no-commit` option of `dvc add`.                                                                                                                                                      |
| `remote`     | (Optional) Name of the remote to use for pushing/fetching                                                                                                                                                                                                                         |
| `persist`    | Whether the output file/dir should remain in place during `dvc repro` (`false` by default: outputs are deleted when `dvc repro` starts)                                                                                                                                           |
| `checkpoint` | (Optional) Set to `true` to let DVC know that this output is associated with [checkpoint experiments](/doc/user-guide/experiment-management/checkpoints). These outputs are reverted to their last cached version at `dvc exp run` and also `persist` during the stage execution. |
| `desc`       | (Optional) User description for this output. This doesn't affect any DVC operations.                                                                                                                                                                                              |

⚠️ Note that using the `checkpoint` field in `dvc.yaml` is not compatible with
`dvc repro`.

## Top-level plot definitions

The `plots` dictionary contains one or more user-defined `dvc plots`
configurations. Every plot needs a unique ID, which may be either a file path
(relative to the location of `dvc.yaml`) or an arbitrary string. Optional
configuration fields can be provided as well.

📖 Refer to [Visualizing Plots] and `dvc plots show` for examples.

[visualizing plots]: /doc/user-guide/visualizing-plots#top-level-plots

### Available configuration fields

- `x` (string) - column/field name from which the X axis data comes from. An
  auto-generated _step_ field is used by default.

- `y` - source from which the Y axis data comes from:

  - Top-level plots: Accepts string, list, or dictionary. For strings and lists,
    the plot ID is used as path to the data source. List elements will be the
    names of columns/fields within the source file. For dictionaries, the keys
    are used as paths to data sources. The values (strings or lists) are treated
    as the source column/field names.

  - Plot outputs: column/field name found in the source plots file.

- `x_label` (string) - X axis label. Defaults to the X field name.

- `y_label` (string) - Y axis label. If all `y` data sources have the same field
  name, that will be the default. Otherwise, it's "y".

- `title` (string) - header for the plot(s). Defaults:

  - Top-level plots: `path/to/dvc.yaml::plot_id`
  - Plot outputs: `path/to/data.csv`

- `template` (string) - [plot template]. Defaults to `linear`.

[plot template]:
  https://dvc.org/doc/user-guide/visualizing-plots#plot-templates-data-series-only

## dvc.lock file

> ⚠️ Avoid editing these files. DVC will create and update them for you.

To record the state of your pipeline(s) and help track its <abbr>outputs</abbr>,
DVC will maintain a `dvc.lock` file for each `dvc.yaml`. Their purposes include:

- Allow DVC to detect when stage definitions, or their <abbr>dependencies</abbr>
  have changed. Such conditions invalidate stages, requiring their reproduction
  (see `dvc status`).
- Tracking of intermediate and final outputs of a pipeline — similar to `.dvc`
  files.
- Needed for several DVC commands to operate, such as `dvc checkout` or
  `dvc get`.

Here's an example:

```yaml
schema: '2.0'
stages:
  features:
    cmd: jupyter nbconvert --execute featurize.ipynb
    deps:
      - path: data/clean
        md5: d8b874c5fa18c32b2d67f73606a1be60
    params:
      params.yaml:
        levels.no: 5
    outs:
      - path: features
        md5: 2119f7661d49546288b73b5730d76485
        size: 154683
      - path: performance.json
        md5: ea46c1139d771bfeba7942d1fbb5981e
        size: 975
      - path: logs.csv
        md5: f99aac37e383b422adc76f5f1fb45004
        size: 695947
```

Stages are listed again in `dvc.lock`, in order to know if their definitions
change in `dvc.yaml`.

Regular
[dependency entries](/doc/user-guide/project-structure/dvc-files#dependency-entries)
and all forms of
[output entries](/doc/user-guide/project-structure/dvc-files#output-entries)
(including [metrics](/doc/command-reference/metrics) and
[plots](/doc/command-reference/plots) files) are also listed (per stage) in
`dvc.lock`, including a content hash field (`md5`, `etag`, or `checksum`).

Full <abbr>parameter dependencies</abbr> (both key and value) are listed too
(under `params`), under each parameters file name.
[templated `dvc.yaml`](#templating) files, the actual values are written to
`dvc.lock` (no `${}` expression). As for [`foreach` stages](#foreach-stages),
individual stages are expanded (no `foreach` structures are preserved).
