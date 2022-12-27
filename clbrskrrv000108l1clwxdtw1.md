# helmfile: a simply trick to handle values intuitively

[helmfile](https://helmfile.readthedocs.io) is a very nice and powerful tool to manage multiple Helm charts declaratively. However, there is one area in which I find it suboptimal: the handling of values / environment values.

Let's go over how it works, and see how we can make it better. If you don't like to read, skip to [My tip on using values in helmfile](#my-tip-on-using-values-in-helmfile) (or read the TL;DR in the repo linked below).

* * *

**For a full example, check out this code:**

👉 ✨ [**https://github.com/derlin/helmfile-intuitive-values-handling**](https://github.com/derlin/helmfile-intuitive-values-handling) ✨ 👈

* * *

## Values in umbrella charts (pure Helm)

Coming from the Helm world, I am used to using umbrella charts, where all the default values for my charts are defined in one single `values.yaml`: \`\`\`yaml

# globals are available to all sub-charts

# using .[Values.global](http://Values.global).\* global: domain: [dev.example.com](http://dev.example.com)

[helmfile](https://helmfile.readthedocs.io) is a very nice and powerful tool to manage multiple Helm charts declaratively. However, there is one area in which I find it suboptimal: the handling of values / environment values.

Let's go over how it works, and see how we can make it better. If you don't like to read, skip to [My tip on using values in helmfile](#my-tip-on-using-values-in-helmfile) (or read the TL;DR in the repo linked below).

* * *

**For a full example, check out this code !** 👉 ✨ [**https://github.com/derlin/helmfile-intuitive-values-handling**](https://github.com/derlin/helmfile-intuitive-values-handling) ✨ 👈

* * *

## Values in umbrella charts (pure Helm)

Coming from the Helm world, I am used to using umbrella charts, where all the default values for my charts are defined in one single `values.yaml`:

```yaml
# globals are available to all sub-charts 
# using .Values.global.*
global:
  domain: dev.example.com

foo:
  # default values passed to the sub-chart called 'foo'
  image: nginx
  tag: latest

bar:
  # default values passsed to the sub-chart called 'bar'
  mode: local
...
```

When some values need to be overridden per environment, I simply create a file `<env>.yaml` and pass it to helm using `--values`. For example:

```yaml
# in environments/prod.yaml
global:
  domain: prod.example.com # override the domain for all

foo:
  image:
    tag: 1.19 # use a stable docker image 
...
```

To deploy to prod:

```bash
helm install my-umbrella-name . --values environments/prod.yaml
```

With helmfile though, there is no easy way to reproduce this behavior (well, there is actually, keep reading 😉).

### Values in helmfile

In helmfile, one defines default values for a chart using the `releases.<name>.values`:

```yaml
releases:
  - name: foo
    ...
    values:
      - image:
          repository: nginx
          tag: latest
```

To add global values, there is an equivalent `environments.default.values`, but this only makes values available to the templates... It doesn't attach those values automatically. In other words, the following does nothing:

```yaml
releases:
  - name: foo
    ...
environments:
  prod:
    values:
      - prod: true
```

To make it work, we need to add some values template to release `foo` (and all other releases), for example:

```yaml
releases:
  - name: foo
    ...
    values: 
      - {{ toYaml .Values | nindent 8 }}
```

Now, `prod: true` will be passed to `foo` upon `helmfile -e prod` ...

The environment values are passed to all release *templates*, not releases! That is, they can be used inside `gotmpl` templates/files listed under `release.<name>.values`, but are not attached directly ...

This is already too complex to follow.

## My tip on using values in helmfile

Instead of trying to understand how all those values work (and creating specific `.gotmpl` files for each release), here is how I managed to **mimic the umbrella chart behavior** regarding values with helmfile (one default value file + one file per environment, with `global` section and `<release-name>` sections).

First, create a folder called `environments`. In it, create a `default.yaml` file, and specify the default values for each release and the globals using the "umbrella chart syntax":

```yaml
global:
  # ... values passed to all releases
foo:
  # ... values passed to release foo
bar:
  # ... values passed to release bar
```

Next, create as many files as you have environments (environment `prod` → `environments/prod.yaml`) and override only what needs to be overridden (compared to default).

In the helmfile, configure each environment to read from `default.yaml` and the specific environment values:

```yaml
# in helmfile.yaml
environments:
  default:
    values:
      - environments/default.yaml
  prod:
    values: # apply default first, then prod
      - environments/default.yaml 
      - environments/prod.yaml
```

Now, here is the trick. Create a *magic* `gotmpl` file that will extract both the `global` section and the release-specific section of the values:

```go
{{/* in env-magic.gotmpl */}}

{{/* 
extract both global and <release-name> sections from
.Values, and merge them (giving precedence to release
specific values.
Note: missing entries are fine.
*/}}
{{ merge (.Values | get .Release.Name  dict) (.Values | get "global"  dict) | toYaml }}
```

And attach this magic file to all releases in the helmfile:

```yaml
# in helmfile.yaml
releases:
  - name: foo
    ...
    values:
      - &env env-magic.gotmpl # use a YAML anchor for DRYness
  - name: bar
    ...
    values:
      - *env # reference the anchor
...
```

That's it! Now, you can simply edit the files in `environments/` and don't have to think about (or touch) values in helmfile anymore.

## Example

A complete example (and a different explanation) is available here:

%[https://github.com/derlin/helmfile-intuitive-values-handling] 

* * *

Written with ❤ by [derlin](https://derlin.ch)