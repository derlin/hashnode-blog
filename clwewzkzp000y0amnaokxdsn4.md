---
title: "From Jar to Brew: distribute your Java programs easily with Homebrew and GitHub Actions"
seoDescription: "Learn how to distribute a program packaged as a Jar using homebrew and to keep a homebrew tap up-to-date using GitHub actions."
datePublished: Mon May 20 2024 12:00:15 GMT+0000 (Coordinated Universal Time)
cuid: clwewzkzp000y0amnaokxdsn4
slug: from-jar-to-brew
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715505069895/557e71f8-b0d7-464f-80d3-4f725ddf4fd1.jpeg
tags: java, homebrew, devops, github-actions-1

---

In this article, we'll explore how to make a program packaged as a Jar (Java, Kotlin, Scala, ...) easily installable by end users using homebrew and GitHub actions.

This is inspired by my work to distribute [bitdowntoc](https://github.com/derlin/bitdowntoc) with brew and aims at answering two questions:

1. how to create a homebrew tap to distribute a jar with brew?
    
2. how to automatically update the tap when a new version is available (using GitHub and GitHub actions)?
    

Let's get started!

---

* [Prerequisites](#heading-prerequisites)
    
* [Publish your JAR via homebrew](#heading-publish-your-jar-via-homebrew)
    
    * [Step 1: create a fat jar](#heading-step-1-create-a-fat-jar)
        
    * [Step 2: make your jar available from a public URL](#heading-step-2-make-your-jar-available-from-a-public-url)
        
    * [Step 3: create a homebrew tap](#heading-step-3-create-a-homebrew-tap)
        
    * [Step 4: write the formula](#heading-step-4-write-the-formula)
        
    * [Step 5: validate and test the formula](#heading-step-5-validate-and-test-the-formula)
        
* [Keep the homebrew tap in sync](#heading-keep-the-homebrew-tap-in-sync)
    
    * [Step 1: the tap workflow](#heading-step-1-the-tap-workflow)
        
    * [Step 2: the app workflow](#heading-step-2-the-app-workflow)
        
* [Conclusion](#heading-conclusion)
    

---

## Prerequisites

* you have standalone software written in a JVM-compatible language (Java, Scala, Kotlin, ...) that you want to distribute easily.
    
* you are hosting it on GitHub. For the second part, you use GitHub releases.
    
* you have homebrew installed and roughly know what it is about.
    

In this article, I will use [bitdowntoc](https://github.com/derlin/bitdowntoc), a command line (and online) tool to add a table of contents to your Markdown files. Written in Kotlin, it is available [on the web](https://derlin.github.io/bitdowntoc) and in the command line (using a [jar](https://github.com/derlin/bitdowntoc/releases/latest) or a [native executable](https://github.com/derlin/bitdowntoc/releases/latest)).

## Publish your JAR via homebrew

### Step 1: create a fat jar

The best way to ship JVM-compatible software to all architectures and OSes is through a fat jar. The advantage of a *jar* (Java ARchive) is it only requires a JRE (Java Runtime Environment) on the target machine to execute. The colloquial term "fat jar" designates a self-contained jar - an archive that includes all its necessary dependencies.

Depending on your language and build system, there are many ways to create a fat jar - too much to enumerate here. If you use Java + Gradle (groovy), baeldung's article "[Creating a Far Jar in Gradle](https://www.baeldung.com/gradle-fat-jar)" is a good start.

🔥 Ensure your app follows *semver* (semantic versioning) and the jar is called `<app>-<version>.jar`, where `<version>` matches `<major>.<minor>.<patch>`. This should be the default when using common tooling.

### Step 2: make your jar available from a public URL

If you are using GitHub for release management, this means attaching your jar to each release. This can be done manually, or completely automated using GitHub actions.

bitdowntoc uses [release-please](https://github.com/google-github-actions/release-please-action) for release automation, and [action-gh-release](https://github.com/softprops/action-gh-release) to attach build artifacts to the release. For more details, check out the `release-please` and `upload-jar` in the [release.yml](https://github.com/derlin/bitdowntoc/blob/main/.github/workflows/release.yml) workflow (I should write an article on release automation 😄).

However you do it, ensure you can download a properly named jar from a public URL. Here is the URL for bitdowntoc version 2.1.0:

```bash
https://github.com/derlin/bitdowntoc/releases/download/v2.1.0/bitdowntoc-jvm-2.1.0.jar
```

### Step 3: create a homebrew tap

Homebrew supports formulae and casks. *Formulae* build from upstream sources, while *casks* install macOS native applications. We could argue a JAR is a native application, but it still requires some fiddling during installation, so we'll use a formula.

*taps* are git repositories of formulae and casks that homebrew uses for discovery. When installing homebrew, it comes with some built-in taps such as [homebrew-core](https://github.com/Homebrew/homebrew-core).

Adding your software to built-in taps requires it to be "*notable enough*" (at least 30 forks, 30 watchers, and 75 stars). If it is your case, go ahead and create a pull request by following the [Formula Cookbook](https://docs.brew.sh/Formula-Cookbook#basic-instructions)! If like me you lack exposure, you'll have to create and maintain your own tap.

👉 The following steps are a condensed version of brew's [How to Create and Maintain a Tap](https://docs.brew.sh/How-to-Create-and-Maintain-a-Tap).

Brew is very developer-friendly. To bootstrap a new tap, run the following in a terminal:

```bash
GH_USERNAME=derlin
APP_NAME=bitdowntoc

brew tap-new $GH_USERNAME/$APP_NAME
cd $(brew --repository)/Library/Taps/$GH_USERNAME/homebrew-$APP_NAME
```

Replace `derlin` with your GitHub username, and `bitdowntoc` with the name of your application. This will initialize a git repository called `homebrew-<appname>` at the path of the taps on your system with a `README.md` and a `Formula` directory. Cd into it using:

```bash
cd $(brew --repository)/Library/Taps/derlin/homebrew-bitdowntoc
```

The tap is created, let's write the formula.

### Step 4: write the formula

In the tap repository created above, add a ruby file in the `Formula` repository. Here is the bitdowntoc formula, that you can adapt to your app:

```bash
class Bitdowntoc < Formula
  # ↓↓ ①
  desc "Markdown Table Of Content (TOC) generator"
  homepage "https://derlin.github.io/bitdowntoc"
  url "https://github.com/derlin/bitdowntoc/releases/download/v2.1.0/bitdowntoc-jvm-2.1.0.jar"
  sha256 "b7a21d2a793a2ecb6c20f27aa8fffc72702daa364b2a89bed32580e302194650"
  license "Apache-2.0"

  ## ↓↓ ②
  depends_on "openjdk"

  ## ↓↓ ③
  def install
    libexec.install "bitdowntoc-jvm-#{version}.jar"
    bin.write_jar_script libexec/"bitdowntoc-jvm-#{version}.jar", "bitdowntoc"
  end

  ## ↓↓ ④
  test do
    assert_match "BitDownToc Version: #{version}", shell_output("#{bin}/bitdowntoc --version")
  end
end
```

There are 4 parts to this Ruby file.

Section ① defines the metadata and the "source" URL (the jar). To get the correct sha, use `sha256sum` on Linux or `shasum -a 256` on Mac. Ensure your description is less than 80 characters.

Section ② specifies that the formula requires Java to be installed.

Section ③ copies the jar (downloaded and unpacked automatically by homebrew using the URL in ①) in the `libexec` folder and creates a bash executable to launch it in the `bin` folder. The `write_jar_script` method takes care of resolving to the proper JDK. In my system, for instance, the bin script looks like this after install:

```bash
cat /opt/homebrew/Cellar/bitdowntoc/2.1.0/bin/bitdowntoc
#!/bin/bash
export JAVA_HOME="${JAVA_HOME:-/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home}"
exec "${JAVA_HOME}/bin/java"  -jar "/opt/homebrew/Cellar/bitdowntoc/2.1.0/libexec/bitdowntoc-jvm-2.1.0.jar" "$@"
```

Section ④ is completely optional and defines some tests to check the installation worked properly.

### Step 5: validate and test the formula

Now, let's install the formula locally (note that it will download the world if you don't have openjdk already installed):

```bash
HOMEBREW_NO_INSTALL_FROM_API=1 brew install bitdowntoc
```

If this works, the next steps are to run the tests and audit the formula:

```bash
brew test bitdowntoc
brew audit --strict bitdowntoc
```

The last step is to commit and push everything to GitHub. Once done, anyone can install your software using either:

```bash
brew tap derlin/bitdowntoc
brew install bitdowntoc
```

Or in one line (will fetch the tap and run the install in one command):

```bash
brew install derlin/bitdowntoc/bitdowntoc
```

---

## Keep the homebrew tap in sync

The homebrew tap lives in a different repository and must be updated every time a new version is available. In the age of the "*\*\*\* as Code*", this manual step is frowned upon. This section shows how I automated it for bitdowntoc.

⚠ Note that this assumes you use GitHub Releases and have a GitHub Actions release workflow.

The idea is simple:

1. create a workflow on the homebrew tap repository able to check for new versions of the app and update the formula accordingly.
    
2. trigger this workflow from the release workflow of the app.
    

(Note that we could also schedule 1 to run e.g. every week, but this adds latency - imagine you release on Monday but the workflow runs on Sunday! GitHub may also disable scheduled workflows e.g. due to a lack of activity in the repo. So meh.)

### Step 1: the tap workflow

First, let's create a workflow on the homebrew tap repository. Its role is to:

1. check the latest version available upstream (the main repository),
    
2. if a new version is available, update the formula and commit the changes.
    

This is what it looks like for bitdowntoc ([homebrew-bitdowntoc/.github/workflows/update.yml](https://github.com/derlin/homebrew-bitdowntoc/blob/main/.github/workflows/update.yml)):

```yaml
name: update version from upstream

on:
  # Allow manual triggers
  workflow_dispatch:
permissions:
  # Necessary to commit the changes back
  contents: write

jobs:
  update_version:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.ref }}

      # ↓↓ ①
      - name: get upstream version
        id: upstream
        run: |
          res=$(curl -s https://api.github.com/repos/derlin/bitdowntoc/releases/latest)
          url=$(jq -r '.assets[] | select( .content_type == "application/java-archive" ) | .browser_download_url' <<< "$res")
          version=$(echo $url | sed -E 's/.*-([0-9]\.[0-9]\.[0-9]).*/\1/')
           
          curl -L -o /tmp/bitdowntoc.jar $url
          sha=$(sha256sum /tmp/bitdowntoc.jar | cut -d' ' -f1)
           
          echo "✅ Found version $version: $url" | tee -a $GITHUB_STEP_SUMMARY
          echo "url=$url" >> $GITHUB_OUTPUT
          echo "version=$version" >> $GITHUB_OUTPUT
          echo "sha=$sha" >> $GITHUB_OUTPUT

      ## ↓↓ ②
      - name: update bitdowntoc formula
        id: update
        run: |
          formula="Formula/bitdowntoc.rb"
          sed -i'' -E 's#^  url ".*"$#  url "${{ steps.upstream.outputs.url }}"#' $formula
          sed -i'' -E 's#^  sha256 ".*"$#  sha256 "${{ steps.upstream.outputs.sha }}"#' $formula

          if [[ -z "$(git status -s $formula)" ]]; then
            echo "✅ Already up-to-date" | tee -a $GITHUB_STEP_SUMMARY
            echo "changed=" >> $GITHUB_OUTPUT
          else
            echo "🟠 Updated $formula" | tee -a $GITHUB_STEP_SUMMARY
            git add $formula
            cat $formula
            echo "changed=true" >> $GITHUB_OUTPUT
          fi

      ## ↓↓ ③
      - name: commit changes
        if: steps.update.outputs.changed == 'true'
        run: |
          git config --global user.name 'Github Workflow'
          git config --global user.email 'workflow@bot.github.com'
          git commit -am "chore: update bitdowntoc to ${{ steps.upstream.outputs.version }}"
          git push

          echo "✅ Changes committed" | tee -a $GITHUB_STEP_SUMMARY
```

It may seem daunting, but this boils down to some basic bash. Let's break it down together.

In ①, I use the GitHub API to fetch the list of artifacts from the latest GitHub release of bitdowntoc, and find the URL of the one with a `application/java-archive` type. Next, I extract the version from the URL (looking for a `X.X.X` pattern), download the jar, and compute its SHA. The URL, SHA, and version are stored as a GitHub output of this step.

In ②, I update the `Formula/bitdowntoc.rb` file with the URL and SHA of step ① using a regex (`sed`). I can now ask `git` if there is any change, and store the result as an output of this step.

Step ③ only runs if a change was detected in step ②. I configure the git user and commit the changes back to the repo.

This workflow is *idempotent*: I can run it as often as I want, it will only update the tap if a new release is present upstream.

### Step 2: the app workflow

The last step is to trigger the tap workflow on every release of bitdowntoc. This means adding a new job to my release workflow that uses the GitHub API to trigger the tap workflow:

```yaml
  update_homebrew:
    runs-on: ubuntu-latest
    needs: [release-please, upload-jar]
    steps:
      - name: trigger homebrew update
        # The PAT should have actions:read-write
        run: |
          curl -L \
            -X POST \
            --fail-with-body \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer ${{ secrets.HOMEBREW_PAT }}" \
            -H "X-GitHub-Api-Version: 2022-11-28" \
            https://api.github.com/repos/derlin/homebrew-bitdowntoc/actions/workflows/${{ secrets.HOMEBREW_WORKFLOW_ID }}/dispatches \
            -d '{"ref":"main","inputs":{}}'
```

For the trigger to work, I had to create a PAT (Personal Access Token) with write permissions on the tap repository and add it as a secret (`HOMEBREW_PAT`) on the bitdowntoc repository (*Settings* &gt; *Secrets and variables* &gt; *Actions*: *Repository secrets*).

The `needs` here is specific to my workflow, and is necessary to ensure the GitHub release exists (with the jar!) *before* the job runs.

And that's it!

## Conclusion

I always loved homebrew (as a user), but never tried to publish anything. I was pleasantly surprised by the overall architecture and the great documentation. Setting up a personal tap was a breeze. My only reproach is the whole brew-related [terminology](https://docs.brew.sh/Formula-Cookbook#homebrew-terminology)... For a non-native speaker, *cask*, *tap*, *keg*, *cellar*, etc are not straightforward without a dictionary close by.

The syncing of the tap was more "experimental", but I am quite happy with the result. If you are a tap maintainer, let me know how *you* solved this challenge!

I hope this will be useful to some, and remember, [bitdowntoc](https://github.com/derlin/bitdowntoc/) can now be installed with homebrew, so give it a try 😉.

Happy coding!