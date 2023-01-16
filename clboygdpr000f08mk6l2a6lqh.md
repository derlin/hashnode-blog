# helmfile: understand (and visualize !) the order in which releases are deployed

This article looks at [helmfile](https://github.com/roboll/helmfile), and how it determines the order in which releases are deployed. More importantly, it gives you a bash script you can use to discover (i.e. print to the console) this order, without having to do a full `helmfile sync`/`helmfile apply`.

Skip the first sections if you are already familiar with helmfile and only want the script !

Please, like [this feature request](https://github.com/roboll/helmfile/issues/1916) if, like me, you would like a command to show the order !

## 🐙 ⇨ **Recap' on helmfile** ⇦ 🐙

If you work with Kubernetes and Helm charts, chances are you are already familiar with [helmfile](https://github.com/roboll/helmfile). It is

> *a declarative spec that brings additional functionality to Helm by allowing you to compose several charts together to create a comprehensive deployment artifact*. In other words, it allows you to bundle charts together.

In short, you declare *releases*, that is instances of specific Helm charts that need to be deployed. By default, all releases are deployed in parallel. To change this default, you declare *needs* (`releases.needs`): when a release A *needs* another release B, A is guaranteed to be deployed before B. This is very similar to the `depends_on` in docker-compose.

The final order will be computed (and applied) when you run `helmfile sync` or `helmfile apply`, which installs/update those releases in a given kubernetes namespace.

## Introduction

Mastering the ordering of releases in big helmfiles is difficult, and it is easy to miss some `needs`. Unfortunately, with helmfile the only way to get the final deployment order it to install everything (`sync`/`apply`) and see how it goes... Or is it ? (See the hack below)

## How helmfile plans the deployment of releases

Internally, helmfile generates a DAG (**D**irect **A**cyclic **G**raph) based on the `needs`. From there, it groups releases so that each release in a group can be deployed in parallel, and each subsequent group depends only on the groups coming before. In other words, releases in group 3 can only *need* releases in group 1-2, etc.

When those groups are formed, helmfile knows it can go one group at a time, executing in parallel `helm install` for all the releases it contains.

If you run `helmfile sync` without any constraint on parallelism, it is exactly what it does. However, it is possible to provide a threshold to the concurrency, using the `--concurrency N` flag. When `N` is smaller than the number of releases in a group, helmfile picks **randomly** N releases at a time, group by group.

This randomness comes from the way `range` over maps is implemented in go:

> *When iterating over a map with a range loop, the iteration order is not specified and is not guaranteed to be the same from one iteration to the next. Since Go 1 the runtime randomizes map iteration order, as programmers relied on the stable iteration order of the previous implementation*. [src](https://blog.golang.org/go-maps-in-action#TOC_7.)

## How to get the deployment order without installing

When running `helmfile sync` or `helmfile apply` with a log level `DEBUG`, the groups are printed at the beginning. For example:

```plaintext
helmfile -log-level debug sync

...
processing 3 groups of releases in this order:
GROUP RELEASES
1     orwell, freud, bradburry, wells, weber 
2     foo, bar
3     buzz
...
```

The trick is thus to trigger a `sync`/`apply` in debug mode, and stop (`ctrl+c`) the command right after the groups are printed, but before any release gets deployed.

As it is a pain to do manually, here is a script to do the job. Note that you will need to be connected to a kubernetes cluster for it to succeed, even though **nothing** will be deployed (rest easy). If you are in doubt, run `kubectl login` before running the script:

```bash
#!/bin/bash

file=$(mktemp -t hf)

trap "exit 0" INT # ctrl+c is somewhat a regular exit
trap "exit" TERM ERR
trap "rm $file 2>/dev/null || true" EXIT

# ensure a helmfile is present
helmfile $@ list > $file

# read helmfile output and stop when the info is displayed
start=0
helmfile --log-level debug $@ sync 2>&1 | tee $file | while IFS='' read -r line
do
  [[ $line =~ ^"processing "[0-9+]" groups of releases in this order:"$ ]] && start=1
  if [[ "$line" == "processing releases in group 1"* ]]; then
    start=0
    rm $file # all went fine
    kill -INT -$$ # send ctrl+c to the process, the only way to kill everything right away
  fi
  [[ "$start" == 1 ]] && echo "$line"
done

if [ -f $file ]; then
  # if the file is still present, it means the process
  # didn't complete as expected. Show the logs
  echo -e "An error occurred. Here is the full log:"
  cat $file
fi
```

Assuming the script is saved as `helmfile-dag.sh` in the user directory and is executable, the following:

```bash
cd folder-with-helmfile
~/helmfile-dag.sh # will use helmfile.yaml in CWD
```

outputs:

```plaintext
processing 3 groups of releases in this order:
GROUP RELEASES
1     orwell, freud, bradburry, wells, weber 
2     foo, bar
3     buzz
```

Note that it is possible to pass extra *global* options to helmfile, for example:

```bash
# process a helmfile not in CWD
~/helmfile-dag.sh --file folder-with-helmfile/helmfile.yaml
```

---

Written with ❤ by [derlin](https://github.com/derlin), thank you for reading !