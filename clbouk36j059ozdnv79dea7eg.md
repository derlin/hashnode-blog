# How to create nightly releases with Github Actions

In my GitHub projects, I like to have the artifact (e.g. a jar) of the latest successful build on a specific branch (e.g. `develop` or `main`) available for download. This is the equivalent of a *nightly release*, but is not supported out-of-the-box by Github.

GitHub Actions lets you attach artifacts to builds, but those artifacts do not have a stable URL. Worse, the URL is difficult to find automatically (you must navigate to the *Actions* tab of the repo, find the latest build, etc.).

My workaround is the following.

### 1) create a release (just once)

The idea here is to have a release linked to some tag, which will be used later in the Github workflow.

In your repo, navigate to *Releases* and click on *Draft a new release*:

*   on the *Choose a tag* dropdown, create a tag, for example `nightly`,
    
*   (optional) fill in the title and description as you see fit,
    
*   (optional) select *This is a pre-release*
    

The only important thing here is to have a tag. I usually use `nightly`, but the actual name is up to you.

The final, stable link will be:

```bash
https://github.com/${user}/${repo}/releases/tag/${tag}
```

### 2) Add a step in your workflow

Use the [eine/tip action](https://github.com/eine/tip) to update the release with whatever artifact(s) you want.

The parameters are:

*   `token`: your GitHub token ⟶ `${{ secrets.GITHUB_TOKEN }}`,
    
*   `tag`: the name of the tag created in 1,
    
*   `rm`: whether or not to remove existing artifacts on upload (I always set it to `true`),
    
*   `files`: either a single filename/pattern or a multi-line string with one file per line
    

Here is a simple example, uploading anything under `build/libs` to the `nightly` release, removing/replacing old artifacts in the process:

```yaml
- name: Update nightly release
  uses: eine/tip@master
  with:
    tag: nightly
    rm: true
    token: ${{ secrets.GITHUB_TOKEN }}
    files: build/libs/*.*
```

Another nice feature is having some `version.txt` or `info.txt` file in the release, that holds some metadata such as the version of the artifact, the github ref, etc. For that, you can simply add a step *before* `eine/tip`:

```yaml
- name: Create info file
  run: |
     echo -e "ref: $GITHUB_REF \ncommit: $GITHUB_SHA\nbuild: $(date +"%Y-%m-%dT%H:%M:%SZ")" \
     > build/libs/info.txt
```

## Examples

Here are some of my repos that use this trick:

*   https://github.com/big-building-data/bbdata-api ⟶ [workflow](https://github.com/big-building-data/bbdata-api/blob/master/.github/workflows/main.yml)
    
*   https://github.com/derlin/bitdowntoc ⟶ [workflow](https://github.com/derlin/bitdowntoc/blob/master/.github/workflows/deploy.yml)
    
*   https://github.com/derlin/docker-compose-viz-mermaid ⟶ [workflow](https://github.com/derlin/docker-compose-viz-mermaid/blob/main/.github/workflows/main.yaml)