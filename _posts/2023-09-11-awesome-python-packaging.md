---
layout: post
title: "Towards internet-grade python packaging for enterprises"
categories: [python, continuous_delivery, github_actions, semver]
---

*Maximizing enterprise DevOps productivity at scale, with a mature python packaging paradigm*


<!--excerpt-above-->

TL;DR - all code for this discussion is available [here](https://github.com/gsoertsz/cli-bumpversion-example)

Python has an incredibly low barrier to entry, and a broad range of applicability. You'll find it in almost every domain you are likely to encounter in a technical career, from embedded systems through to systems programming and infrastructure tooling; cloud API's, and DevOps, and into application development and data science, the language is indeed [ubiquitous](https://github.blog/2023-03-02-why-python-keeps-growing-explained/). As an aside, if you're not learning it now, please start. 

In the DevOps context, we often reach for python when automating the deployment of our systems to cloud environments, and for interacting with the API's of tools that support our production environment ecosystems, particularly when they create (or exist in different) tooling domains.

It is just so easy, and therefore compelling to just 'cut a python script' to create a necessary automation shim, or cross tool integration, to make progress on our automation objectives.

However, python scripts in a DevOps context often become an unmaintainable nightmare. For every iteration of the python script we wish to verify, we need to:

- push the change to the upstream git repo
- move over to the CI/CD tool, and trigger the pipeline
- wait for the pipeline stages and jobs to be allocated compute (a runner, or VM)
- wait for each task to load its execution container and resolve its dependencies
- wait for upstream tasks to execute

The above activities are all a necessary consequence of debugging python scripts in a CI/CD context, and they each involve latencies in the 10s of seconds to minutes. Putting aside observability frustrations, this is a well known, maddening and exhausting diagnostic loop that we've all suffered. For our purposes I'll call it the *"Python Script Debugging Cycle of Death"*, and obviously its something we wish to avoid. I confess to being a passionate developer productivity enthusiast / ergonomist, and the interpreter's approachability - `python -c ...` or `python my_script.py` - (its ability to execute arbitrary text as python code) is both a blessing and a curse. 

The positives of this approachability are obvious; you can turn python towards all sizes of problem, from big to small. This allows you to right size your approach, in proportion to the problem as it manifests today. You don't need to bootstrap an entire eco-system to perform a small action or transformation, particularly if you use a framework with a strong API, and your solution is 3 lines long. A usable python interpreter is also included in most if not all linux distributions out there today.

But in an enterprise context, problems don't usually stay the same size, particularly when platforms are under heavy development, or are evolving under maintenance. You may have started out with a small solution to a small problem, but if you've built something meaningful, the correlation between platform vitality and entropy, inevitably means you'll be revisiting 'small code' as you react to increasing demands. There'll be natural pressures to increase the complexity or responsibilities of fragments you've previously smacked together. Or more poignantly, as time and exposure continues, you'll perceive more of the problem space, or identify patterns that set your previous solution against a more expansive backdrop, putting it in a broader perspective. This perspective will force you to grapple with other software engineering principles such as DRY (don't repeat yourself), and SRP (single responsibility principle). Inevitably you'll look to re-architect your approach to exploit the discovered patterns and solve the problem more wholistically.

I'll refer to this as the *"Fallacy of Small Enterprise Code".* Somewhat ironically, this fallacy is an observable consequence of the natural tendency for code to approach cleaner software architectures and higher levels of quality. This emergent property, in turn, is a direct consequence of hiring smart, experienced, principled software developers, to maintain and develop code in production over non-trivial periods. Without going into further detail, let's just exploit this property as a [philosophical razor](https://en.wikipedia.org/wiki/Philosophical_razor) (*"Soertsz's Razor"* anyone?) to fast-forward past the *"Fallacy of Small Enterprise Code"*, the *"Python Script Debugging Cycle of Death"*, and beyond python's mischieviously accessible interpreter to a seemingly inevitable and necessarily greater level of sophistication.

Concurrently, we all use 3rd party and open source libraries from the internet all the time. The DRY (Don't Repeat Yourself) mantra is almost beaten into us throughout our careers, and in some cases rolling your own solution when a battle hardened internet-grade library is available, is at best [ill advised](https://www.schneier.com/blog/archives/2011/04/schneiers_law.html). It is true that when evaluating the programming languages one might choose for a project or platform, we also take into account the language's library and dependency eco-system. In many cases, within an enterprise context, in various application development situations, we may have developed a library, a tool or a compelling framework that wish to share across different teams. When we reach for a library dependency or framework from the open source world, we spend quite a bit of time browsing the [public repositories](https://pypi.org/) looking for the appropriate library. 

In evaluating good libraries, we would typically review and assess:

- project documentation
- vitality of the project - commit recency, maintenance activity
- popularity - downloads, forks etc
- software quality metrics - unit tests, coverage, build status, badges etc.

Not until we get through the above, would we even dream of installing the dependency with `pip install`, and using the dependency with an `import` statement, even in an experimental capacity. We have a strong sense of the qualitative criteria we are using when evaluating a good library for our purposes.

Given the discussion, why then would we treat internal code, and code we wish to share, any differently from a package published publicly, particularly, when we want to accelerate our maturity past that in which we would suffer the ills of *"Small Enterprise Code"*? 

If we can work with packages, published internally within an entrprise, in approximately the same way as we work with publicly available packages, this signals to the organization that your development team is serious in meeting the same standard to which public packages are held - the standard that makes them suitable for any one of us to choose, to support a workload that could be running in production tomorrow. This signal has the additional effect of attracting good engineers who are either interested in learning how to ship python code at a high standard, or are seasoned pythonistas who want to work in an environment that takes the language seriously. Interestingly, either way, the system of work produces high quality packages, and is self reinforcing. Who wouldn't want a system of work like this?

A convincing counter argument for not pursuing a system of work focussed on publishing python packages, is that it would absorb unaffordable engineering effort to achieve it. When set against the back drop of an unreceptive organisation, or one unappreciative or responsive to the inherent virtue of such a system, the situation can be likened to trying to *"perform Shakespeare to an empty theatre"* - folly, and a gross misallocation of skilled resources given the context.

But the alternative for developers is worse, and there are a number of tools I'll cover, that can be easily integrated into python repositories of all kinds, that reduce the effort needed to establish the foundations of a package oriented system of work. It's not a perfect situation, but at the very least we set off in the right direction.

So, when it comes to packaging python projects in an enterprise context, we can begin with the following hypothesis:

---

*It should be **easy** to setup a repository model that **publishes semantically versioned, well-documented, installable or deployable artefacts**. Other than the quantum leap in trust, usage safety, and the minimization of tech debt growth, this approach **signals** to the surrounding organisation a high standard for python software development, attracts pythonistas, and **reinforces** itself.*

---

### Producing packages in a continuous delivery context

Continuous delivery distinguishes strongly between software release and software deployment. For example, v1.0.2 can be created, and may not be deployed to production (or any environment) until days, weeks or months later. In the mean time the software is published to a Binary Artefact Management system, awaiting the moment its downloaded by a deployment pipeline and deployed into a target runtime environment for whatever purpose (testing, production etc). Obviously, the longer the software remains undeployed, the more likely it is to be obsolete, especially in busy development periods. While we might wish for high frequency deployment of small increments of software, in an enterprise context we can only deploy as fast as IT governance and change management ceremonies will allow.

Regardless, there may be sensible points at which functionality should be merged to a release line in SCM, and the software be made ready for release. Conceptually we want all our repositories to emit a stream of technology-specific, versioned, installable/deployable artefacts. The following requirements should also be met.

- The artefact is immutably published to an artefact management system, and downloadable individually by version
- For traceability, the main branch is tagged with the version of the release, consistent with the artefact naming and the contents of the artefact
- The artefact version should follow semantic versioning, reflecting the nature of the change, which should be determined by the committing developer
- A changelog should be produced (more on this later)

Let's put some of the above provisions in place, against a python repository that produces an installable custom CLI for demonstration purposes. All the code for a package oriented python repository is located [here](https://github.com/gsoertsz/cli-bumpversion-example)

#### Building a python package

It's relatively straight forward to build a python package, and a great overview can be found [here](https://packaging.python.org/en/latest/tutorials/packaging-projects/). You need to introduce a `pyproject.toml` file, a `Manifest.in` file, and install the python `build` package (`import build`). In our example we'll also use a Makefile to stay organised around the project commands.

Below is a tree listing of the project, calling out the relevant files

```
.
├── ...
├── MANIFEST.in
├── Makefile
├── ...
├── hello
│   ├── __init__.py
│   ├── ...
│   ├── 
├── pyproject.toml
├── ...

```
Below is an excerpt of the Makefile


```Makefile
.PHONY: install test lint clean build docs all
.ONESHELL:

install:
	pip install -r requirements-dev.txt
	pip install -r requirements.txt

... 

build:
	python -m build

all: clean test lint install build docs

```

Below is the `Manifest.in` file
```
include requirements.txt 
exclude *.egg-info

graft hello
```

And finally, the contents of the `pyproject.toml` file

```
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "cli-bumpversion-example"
version = "0.6.1"
authors = [
]
description = "Example CLI"
readme = "README.md"
license = { file="LICENSE" }
requires-python = ">=3.9.7"
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: LGPL License",
    "Operating System :: OS Independent",
]
dependencies = ["click"]

[project.urls]
"Homepage" = "https://github.com/gsoertsz/cli-bumpversion-example/"
"Bug Tracker" = "https://github.com/gsoertsz/cli-bumpversion-example/issues"

[tool.setuptools]
include-package-data = false

[tool.setuptools.packages.find]
# list of folders that contain the packages (["."] by default)
include = [ "hello", "hello.greeting", "hello.lib" ]
exclude = [ "hello.tests*" ]

[project.scripts]
hello = "hello.hello:main"
```

So producing an artifact for this project is as easy as typing in the following command:

```terminal
%> make clean build
```

Here is what happens when we execute this command:

![Build Package Screenplay](/assets/images/posts/awesome_python_packaging/ScreenPlay_Console_BuildPackages.gif){:class="img-fluid"}

The resulting packages are installable just like any 3rd party dependency we may consume via `pip`. Here's us installing the package into a python environment

![Build Package Screenplay](/assets/images/posts/awesome_python_packaging/ScreenPlay_Console_InstallPackages.gif){:class="img-fluid"}

Once built, the python package is directly installable, and as the package was configured to expose some command line tools, the `hello` CLI tool is invokable directly after install. The fact that the tool was built in python becomes an implementation detail. The user experience with the software now starts to approximate what we would go through consuming something similar from a public site. The `dependencies` clause in the `pyproject.toml` file ensures any dependencies that are needed in the environment are installed automatically, so for the most part we don't have to worry about packaging these dependencies ourselves. Therefore, there are very few touchpoints required to get the tool running. Just a `pip install`. Splendid.

By default, `build` will produce packages in a source distribution (sdist) format, and a more machine dependent wheel (whl) format. Source distributions are more portable across machine architectures in principal, and will build the source upon download. The wheel format avoids the need to build the package locally (and therefore the assumption that build tools such as `setuptools` and `venv`/`pyenv` and `pip` are available), but is machine dependent. It's constructive to publish both as part of a single publishing event. This allows consumers with the same machine architecture/OS as the publisher to consume an efficient packaging format. In a hetergenous OS/arch environment, the source distribution can be built where it is run, which means you don't have to support all the infra permutations out there, but the usage and installation takes a bit longer and requires some additional tools in the consumer python environment.

Let's publish these artefacts to a Binary Artifact Management System such as Azure Artifact, via a github actions workflow.

#### Publishing the package, and pushing the release tag upstream

Below is some of the github actions workflow for publishing the artefacts to Azure Artifact. As previously mentioned, the workflow has some key objectives.

- Build the packages (sdist, and wheel)
- Publish the packages to Azure Artifact
- Tag and commit to the local repo, the version established with this release (more on this later)
- Push the updated version information, and tags to the upstream branch

To the extent possible, in order to provide good traceability, all of the above activities should be performed together or not at all. Indeed, and in practice, any of the previous actions could fail, and if they do, there isn't likely to be a simple automated approach to recover. Despite this, if all of these occur you end up with superior traceability between built artefact and SCM commit via tag. From the point of view of auditing changes in production, this association is key.

In the workflow below, we focus on a simple feature branching strategy. Basically, the default main branch is the target branch for all features. The merge of each feature branch to main, governed by a reviewed and approved pull request, should result in a newly versioned artefact. Developers aren't allowed to push changes directly to main, and must take a non-main branch (feature branch) to which their code updates are committed. This causes two complications.

Firstly, if you wish to prevent developers from merging or committing directly to main without a pull request, you'll need to configure branch protection rules in github. This is fine, but your github workflow will also not be able to push upstream, any version related commits or tags due to those same restrictions. 

Secondly, a push event back to main will retrigger the github actions workflow, potentially bumping and producing another versioned artefact, and repeating the cycle. 

So, to resolve these issues:

1. ***Branch protections and using an application as the committer to support a narrow bypass*** - Register a Github App, and put the application on the bypass list. In the example below, we register an app called `Python Version Bumper`. In this way, you can define broad branch protections as suited to your environment, but allow the workflow to push changes back to main to update version strings and push tags and commits.
2. ***Retriggering github actions on the merge / push to main*** - ultimately, committing the tag and version information back to main will re-trigger this workflow. To avoid this, ensure the commit message for the version update contains a hint to skip a workflow run when the commit comes from your workflow, as per [this link](https://docs.github.com/en/actions/managing-workflow-runs/skipping-workflow-runs)

```yaml
name: Python application

on:
  push:
    branches:
      - "**"
  pull_request:
    branches:
      - main

permissions:
  contents: write

env:
  AZURE_DEVOPS_ORGURL: "<READ FROM REPO VARIABLES>"
  AZURE_ARTIFACTS_FEED_URL: "<READ FROM REPO VARIABLES>/cli-bumpversion-example-release/pypi/upload/"
  ARTIFACTS_KEYRING_NONINTERACTIVE_MODE: true
  NETRC: './.netrc'

jobs:
  build:
    runs-on: ubuntu-latest
    name: Build
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      ...

  bump-and-tag-version:
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-tags: true
          fetch-depth: 0
      - id: cz
        name: Create bump and changelog
        uses: commitizen-tools/commitizen-action@master
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          debug: true
      - name: Generate access token
        uses: tibdex/github-app-token@v1
        id: get_installation_token
        with:
          app_id: ${{ vars.VERSIONBUMP_APP_ID }}
          private_key: ${{ secrets.VERSION_BUMP_TOKEN_PRIVATE_KEY }}
      - name: "Push changes"
        env:
          GITHUB_TOKEN: ${{ steps.get_installation_token.outputs.token }}
        run: |
          git config --get http.${{ github.server_url }}/.extraheader && git config --unset-all http.${{ github.server_url }}/.extraheader
          git config user.name github-actions
          git config user.email github-actions@github.com
          git config credential.${{ github.server_url }}.helper '!f() { test "$1" = get && echo "password=$GITHUB_TOKEN"; }; f'
          git config credential.${{ github.server_url }}.username x-access-token
          echo "Add here your changes and a git commit if needed"
          git push origin ${{ github.ref }} --force-with-lease --tags
      - name: Setup Python
        uses: actions/setup-python@v3
      - name: Create .netrc file
        uses: 1arp/create-a-file-action@0.2
        with:
          path: '.'
          file: .netrc
          content: |
            machine pkgs.dev.azure.com
            login ${{ secrets.AZURE_ARTIFACTS_USERNAME }}
            password ${{ secrets.AZURE_ARTIFACTS_PAT }}
      - name: Create .pypirc file
        uses: 1arp/create-a-file-action@0.2
        with:
          path: '.'
          file: .pypirc
          content: |
            [distutils]
            Index-servers =
                cli-bumpversion-example-release
            
            [cli-bumpversion-example-release]
            Repository = "<READ FROM REPO VARIABLES>"
      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements-dev.txt ]; then pip install -r requirements-dev.txt; fi
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
      - name: Build package
        run: |
          make build
      - name: Install publishing dependencies
        run: |
          pip install twine
      - name: Publish distribution Azure Artifacts
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: <READ FROM REPO SECRETS>
          repository-url: <READ FROM REPO VARIABLES>
          user: <READ FROM REPO SECRETS>
          verbose: true

```

Let's see what happens when we merge a change in github actions, when the artefact is published to Azure Artifact.

Setting up a feed to receive your published artefact requires:

- Set up a feed, and assign yourself as a Contributor
- Create a personal access token and making that token available as the `password` field to the `pypa/gh-action-pypi-publish@release/v1` github action

Once the feed is set up, you can acquire the package url for the feed and configure it as a variable to the workflow.

Once the workflow is triggered, a versioned artefact should appear in the feed as per the below:

![Version Summary](/assets/images/posts/awesome_python_packaging/Screenshot_ADO_FeedSummary.png){:class="img-fluid"}

The list of versions that have been published can also be viewed:

![Version Summary](/assets/images/posts/awesome_python_packaging/Screenshot_ADO_VersionList.png){:class="img-fluid"}

... and if you select any one of the versions published, you can see greater detail:

![Version Summary](/assets/images/posts/awesome_python_packaging/Screenshot_ADO_VersionDetails.png){:class="img-fluid"}

Notably, you'll observe that the `Summary` field in the details screen above, is taken from the `pyproject.toml` file. Azure Artifact also extracts and displays the `dependencies`, and `classifiers` fields.

Also, the `Description` panel, is sourced from the artifacts `README.md` file. 

All the files that were uploaded as part of the publishing step are visible and available for download.



So to get good traceability for published artefacts:

---

*Regardless of how you determine the version, and update the project's version strings, **register a github app**, and use it to commit and push the versioning details back upstream. This allows you to maintain your development policies within Github, but achieve superior traceability through automation.*

---

... and:

---

*Given the **automatic** computing and updating of **version strings**, to **avoid retriggering** the github actions release workflow, ensure the **message** used to push the versioning and tag information back upstream **contains** a "skip workflow" **hint** as per [this link](https://docs.github.com/en/actions/managing-workflow-runs/skipping-workflow-runs)*

---

... also:


---

*Azure **Artifact** does a decent job of publishing metadata about a versioned python package. As a Python package catalog, it is **a good foundation for a mature package oriented system of work**.*

---

At this point, our repository automatically publishes a versioned artefact, and ensures traceability by updating the repository with the related version information as an addressable tag. This is fine if you ever want to produce only one artefact. But, we are obviously going to want to produce more than one artefact and change it in different ways as the software evolves. 

#### Semver controls and changelog generation with commitizen


[Semantic versioning](https://semver.org/), or 'semver' as its more commonly known, specifies a version string pattern that communicates the impactfulness of a change. It works on the basis that if you are going to version something, it's just as important to communicate what's changing between versions as it is to attribute a version to a known baseline. For example, an observer of an artifact that complies with semver, upon seeing a change from 1.0.1 to 1.0.2, noting a PATCH, would reason that they can rely on their client continuing to be compatible with 1.0.2. Contrariwise, a bump from 1.0.2 to 2.0.0 - MAJOR - implies breaking changes to the API. The observer is now informed that they would need to re-integrate their code to work with the new version.

Of course there are no guarantees. A developer can easily introduce bugs, or incompatibilities disguised as PATCHes, and break their consumers. Obviously this is not in anyones best interest, but what the version bump should be between any two changes is still at the discretion of the developer. This makes sense, as they are most familiar with the changes - the most authoritative. 

In python build parlance, the version information that get's included in the archive etc. is included in the `pyproject.toml` file, in the `version` directive. When you execute `python -m build` and `version = 1.0.5`, then the built artifact will be `artifact-name-1.0.5.tar.gz` etc. So, if we change the version, and wish to produce the corresponding artifact, we need to update this file as part of that process, generate a tag, commit the change in the `pyproject.toml` file, tag that commit with the version, push that commit (and its tags) to the release branch (then happily build the package subsequently). This is where it can get complicated to work with packages, and it obviously needs to be automated.

Additionally, reflecting on the earlier discussion, we considered a changelog as being a key qualitative element of a good python package. Changelogs are actually a fundamental way in which we reason about whether a specific version of a package is correct for our purposes. A changelog should describe the specific changes that were applied to a package between versions, and whether the fix you seek is included in a specific version can only be gleaned from the changelog.

Changelog information is available in the messages attributed to each commit between tagged or released versions, and so it makes sense that an update to a change log is a direct consequence of computing the new version, publishing an artefact and ensuring traceability.

[Commitizen](https://commitizen-tools.github.io/commitizen/) brings together commit quality, automatic version bumping, and changelog generation/update into one package.

From the github actions workflow, the following lines are critical:

```yaml
...
  bump-and-tag-version:
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-tags: true
          fetch-depth: 0
      - id: cz
        name: Create bump and changelog
        uses: commitizen-tools/commitizen-action@master
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          debug: true
...
```

Here is the commitizen configuration file, that's used by the github action:

```toml
[tool.commitizen]
name = "cz_conventional_commits"
version_scheme = "semver"
version = "0.6.1"
annotated_tag = true
update_changelog_on_bump = true
changelog_incremental = true
bump_message = "release $current_version → $new_version [no-ci]"
version_files = [
    "pyproject.toml"
]

```

The above configuration specifies the following:
- Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) as the commit format
- The versioning scheme is [Semantic Versioning](https://semver.org/)
- Update a changelog on every version bump, incrementally
- Use the specified commit message when commiting the changes to the version files to the local repo
- Update the version string in the `pyproject.toml` file

Notably, the above configuration is structured, such that it can be incorporated into `pyproject.toml` to further reduce the number of repository files.

We also lean on the commitizen configuration to customize the commit message, in order to avoid retriggering the github actions release workflow. 

Incorporating commitizen is really straightforward. It is important to note however, that you need to [checkout the tags](https://github.com/marketplace/actions/checkout#fetch-all-history-for-all-tags-and-branches) for the repo, because, given the configuration to incrementally update a change log, Commitizen determines how to update the change log by processing the git tags that have been created as refs in the repository.

Let's see what happens when commitizen detects a change and needs to bump the version. Here is the git log prior to 0.6.2:

```terminal
b139b2c fix(package): updated package includes and build (#6)
2c13e33 fix(manifest): removed unnecessary file (#5)
e220599 (tag: 0.6.1) release 0.6.0 → 0.6.1 [no-ci]
ae3a588 fix(build): move to pyproject.toml (#4)
46ce828 (tag: 0.6.0) release 0.5.0 → 0.6.0 [no-ci]
2ef56c9 feat(doco): added documentation (#3)
f5bd071 (tag: working, tag: 0.5.0) release 0.4.4 → 0.5.0 [no-ci]
```
Here is the execution log for the github action workflow after `b139b2c` was pushed upstream

```terminal
...
Commitizen version: 3.8.2
cz --debug --no-raise 21 bump --yes --changelog --check-consistency
release 0.6.1 → 0.6.2 [no-ci]
tag to create: 0.6.2
increment detected: PATCH

InvalidVersion GitTag('working', 'f5bd071a21182e5437c8991b9124f7ecfd413106', '2023-09-10')
[main 242c0a5] release 0.6.1 → 0.6.2 [no-ci]
 3 files changed, 9 insertions(+), 2 deletions(-)

Done!
Repository: gsoertsz/cli-bumpversion-example
Actor: gsoertsz
Pushing to branch...
From https://github.com/gsoertsz/cli-bumpversion-example
 * branch            main       -> FETCH_HEAD
Current branch main is up to date.
To https://github.com/gsoertsz/cli-bumpversion-example.git
   b139b2c..242c0a5  HEAD -> main
 * [new tag]         0.6.2 -> 0.6.2
Done.
...
```

Let's have a look at the git log after the change, and the generated change log update:

```terminal
242c0a5 (HEAD -> main, tag: 0.6.2, origin/main, origin/HEAD) release 0.6.1 → 0.6.2 [no-ci]
b139b2c fix(package): updated package includes and build (#6)
2c13e33 fix(manifest): removed unnecessary file (#5)
e220599 (tag: 0.6.1) release 0.6.0 → 0.6.1 [no-ci]
ae3a588 fix(build): move to pyproject.toml (#4)
...
```

Changelog:

```
## 0.6.2 (2023-09-12)

### Fix

- **package**: updated package includes and build (#6)
- **manifest**: removed unnecessary file (#5)

## 0.6.1 (2023-09-12)

### Fix

- **build**: move to pyproject.toml (#4)

## 0.6.0 (2023-09-11)

### Feat

- **doco**: added documentation (#3)

## 0.5.0 (2023-09-10)

### Feat

- **readme**: whitespace

## 0.4.4 (2023-09-10)

### Fix

- **actions**: checkout better
- **chore**: update readme

## 0.4.3 (2023-09-10)

### Fix

- **config**: fixed tag format

```

You can observe from the changelog above that when Commitizen has detected a commit pertaining to a `feat(scope)...` it has bumped the `MINOR` version; likewise when it has detected a `fix(scope)...` commit message it has bumped the `PATCH` component. Developers retain control over the versioning as long as they represent their changes in accordance with the convention. Automatically managing a changelog for a package can eliminate the drudgery of reading and shaping release notes, and Commitizen takes inspiration and structure from [Keep a changelog](https://keepachangelog.com/en/1.0.0/). 

Commitizen, by tying into some well known sensible standards for commits, changelogs and versioning, fulfils three of our objectives for a package-oriented python repository
- Computing the next version, while allowing the developer to retain control by convention
- Auto update and generation of a changelog
- Auto update of version carrying files, and ensuring consistency in the traceability between the generated artefact version in git and version files within the artefact

So:

---

*We add **Commitizen** to our release workflow, as a low effort, high yield option for automating the management of version information and repository traceability in the creation of publishable artefacts.*

---

### Documentation with README.md and Sphinx

Well built packages, that adhere to internet-grade standards for versioning and changelogs, are useless unless users are able to find details about what the package does and how to use it. This is where good package documentation comes in.

We've already seen that Azure Artifact understands the `pyproject.toml` and the `README.md` file when surfacing metadata about a published package. So in reality its possible to at least use the `README.md` file to provide overview information to help the user at an introductory level.

If we are looking for more details, say about the API, or how best to use the tool, framework or library, and specifically, considerations or caveats in relation to the implementation, the `README.md` is likely not going to be enough. The `README.md` is also something that's maintained separately from the code. Ideally, we'd like to write comments inline with the code, precisely where its relevant, and then produce user documentation from those comments.

This is where [Sphinx](https://docs.readthedocs.io/en/stable/intro/getting-started-with-sphinx.html) comes in.

Again, incorporating Sphinx into the build workflow is very straightforward. It starts by creating a `docs` directory in your project, installing sphinx with `pip install sphinx`, and then running `sphinx-quickstart`. This will walk you through creating the basic configuration. Be sure to specify the documentation source and build directories are `docs/source` and `docs/build` respectively. Along with a basic configuration in `docs/source/conf.py`, it will try to produce a Makefile, which you use to generate the documentation site with `make html`. If you already have Makefile, it will create a default Windows `make.bat`. In my case this wasn't useful, so I simply added a `docs` target to my existing Makefile as per the below:

```Makefile
.PHONY: install test lint clean build docs all
.ONESHELL:

...

docs:
	sphinx-apidoc --ext-autodoc -f -o docs/source hello "tests"
	sphinx-build -M html docs/source docs/build

docs-clean:
	rm -rf docs/build/*
...

all: clean test lint install build docs
```

The quickstart will also create a `conf.py` and a baseline set of `.rst` documents. The `conf.py` file defines how Sphinx will behave and the baseline templates provide an initial template for the documentation site.

Below is an extract of the `conf.py` file.

```
...
import os
import sys
sys.path.insert(0, os.path.abspath('../../hello'))
sys.path.insert(0, os.path.abspath('../..'))



# -- Project information -----------------------------------------------------

project = 'cli-bumpversion-example'
copyright = 'xxxx'
author = 'xxx'

# -- General configuration ---------------------------------------------------

# Add any Sphinx extension module names here, as strings. They can be
# extensions coming with Sphinx (named 'sphinx.ext.*') or your custom
# ones.
extensions = [
    'sphinx.ext.napoleon',
    'sphinx.ext.autodoc',
    'sphinxcontrib.plantuml',
    'sphinx_mdinclude',
    'sphinx_needs',
    'sphinxcontrib.test_reports'
]

# Add any paths that contain templates here, relative to this directory.
templates_path = ['_templates']

# List of patterns, relative to source directory, that match files and
# directories to ignore when looking for source files.
# This pattern also affects html_static_path and html_extra_path.
exclude_patterns = [
    "hello.tests.*",

]

# -- Options for HTML output -------------------------------------------------

# The theme to use for HTML and HTML Help pages.  See the documentation for
# a list of builtin themes.
#
html_theme = 'sphinx_rtd_theme'

# Add any paths that contain custom static files (such as style sheets) here,
# relative to this directory. They are copied after the builtin static files,
# so a file named "default.css" will overwrite the builtin "default.css".
html_static_path = ['_static']

```

Below is an extract from the `index.rst` source file:

```
.. cli-bumpversion-example documentation master file, created by
   sphinx-quickstart on Sun Sep 10 13:33:55 2023.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to cli-bumpversion-example's documentation!
===================================================

.. toctree::
   :maxdepth: 2
   :caption: Contents:

   modules

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

README
======

* :ref:`readme`

Test Results
============

.. test-results:: ../../unittest_output.xml
```

From our `Makefile` once you run `make docs`, you'll observe that we pre-process into the source directory, some apidocs by invoking `sphinx-apidoc` leveraging the `autodoc` extension. [Autodoc](https://www.sphinx-doc.org/en/master/usage/extensions/autodoc.html) parses docstrings into package documentation in the reStructuredText format, which is the documentation format Sphinx uses to create the documentation site. Below is a file listing demonstrating this:

```terminal
.
├── CHANGELOG.md
├── LICENSE
├── MANIFEST.in
├── Makefile
├── README.md
├── azure-pipelines.yml
├── docs
│   ├── build
│   └── source
│       ├── _static
│       ├── _templates
│       ├── conf.py
│       ├── hello.greeting.rst
│       ├── hello.rst
│       ├── hello.tests.rst
│       ├── index.rst
│       ├── modules.rst
│       └── readme.rst
├── hello
│   ├── __init__.py
│   ├- ...
├── pyproject.toml
├── requirements-dev.txt
└── requirements.txt

```

As you can see from the above, sphinx operates from source files stored in the `docs/source` directory. We use these files to customize the documentation site as required. These files are checked into source control. The files generated into the `docs/build` directory can be added to `.gitignore` and should not be checked in (generally).

Once this is complete, the `sphinx-build` command is executed to convert the rST templates into a documentation site. Below is a screenshot of the site

![Readthedocs index](/assets/images/posts/awesome_python_packaging/Screenshot_Sphinx_Index.png){:class="img-fluid"}

As you can see from the configuration, and the screenshot, I've included some additional information in the default site.

To avoid duplicating useful overview information, especially because both Github and Azure Artifact do a reasonable job of rendering the `README.md`, I've included the `mdinclude` directive, which inserts the contents of the `README.md` into the overview page. the `mdinclude` directive is available via the `sphinx-mdinclude` plugin we installed and added in the `extensions` list in `conf.py`. 

Additionally, while the software quality metrics (unit test and coverage reports) are available in the repo once built, unless they are explicitly published they are often hard to find and access. I've tried to include these details into the documentation site, and this is made possible with [Sphinx-Test-Reports](https://sphinx-test-reports.readthedocs.io/en/latest/).

The coverage report is also available in the repo, and i've exported it into html so it can be read from the console at least, however there doesn't seem to be a Sphinx plugin that helps you render the coverage details in user documentation. Bummer!

Sphinx is a static site generator, similar to [Jekyll](https://jekyllrb.com/), and [MkDocs](https://www.mkdocs.org/) is another compelling static site generator focused on project documentation. In this example I've focussed on Sphinx only because I found it to be quite capable, with a raft of extensions. Others can and should be explored, perhaps in another blog. Sphinx is capable, but reports are the learning curve can be quite steep. Your mileage may vary.

As these tools all produce a static HTML site, hosting the generated documentation is a challenge in enterprise environments. These packages are obviously private and shouldn't be accessible from the public internet. Pushing the documentation, created as a [workflow artefact](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts), to a [Github Pages](https://github.com/actions/deploy-pages) site, under a file path for the specific version is easy and feasible, however, Github Pages are public by default, and you'll need to explore self-hosted options such as Gitlab, or Github Enterprise. If you are already self hosting, and have a repository presence on Github Enterprise somewhere, then publishing this documentation to a Github Pages site is a compelling option. 

In the absence of somewhere well known to host project documentation, the alternative is to build the documentation and include it in the package source (sdist) archive. This is as simple as ensuring the `docs/build/html` directory is an entry in the `Manifest.in`, and that the site is built before the package is built. 

The documentation is then available as part of the source distribution. This makes sense, and is also a good place to include the coverage site as well. However, from my point of view, and likely that of any user of that package, the documentation is hidden in the source package, and its certainly not 'easy' to get at that documentation without downloading, decompressing and opening those pages manually. Other than expressing something in the Readme about where one might find the documentation, there's no way to communicate how one might access it. Stepping back, despite it being a reasonable starting point, its a bit of a tragedy when great end-user documentation is hidden in the source archive, and takes effort and pain to access. It also detracts from the perception that the python projects are a serious business for your development team. So, I think evaluating compliant, centralised and compelling hosting options for auto-generated documentation is advisable.

To sum up:

---

*Extensible and customizable **documentation generators for python packages** are available that can be **easily** installed and incorporated into a build workflow, producing **compelling documentation sites** alongside, and included in, other publishable artefacts. This documentation can be enriched by summary details in the `README.md` and from docstrings from within the code, **simplifying maintenance** and minimizing duplication.*

---

However:

---

*Between Github, Azure Artifact, and the generated documentation site, as to where package users should expect to find software quality metrics, it is possible to make **test reports and results available in the generated user documentation** (with Sphinx extensions such as Sphinx-Test-Reports). **This only goes so far, as pytest-cov coverage output is yet to be integrated.***

---

And finally:

---

*Investigate **hosting options** for project end-user documentation, as simply including the documentation in the package source distribution, is high friction, and counter-intuitive given the **internet-grade standard** for 3rd party projects.*

---

### Summary

We've demonstrated that with github actions, and a few tools and techniques, we can leverage a compelling post-merge workflow to publish semantically versioned, and well documented python packages, to Azure Artefact, in a manner, and to a standard that approximates internet-grade python packages and projects. 

It is easy to incorporate this workflow into python repositories, without loss of compliance in relation to repository standards and policies. The benefits of a package-centric system of work for your team and your enterprise are within reach, without expending a prohibitive amount of development effort. These benefits include establishing a high standard for python software development, attracting other budding and seasoned pythonistas, and creating a 'honey-pot' working environment that regulates and reinforces itself.

Below is a brief summary of the objectives for a package-centric python system of work, and the relevant tools that can be incorporated to bring it to life.

| Objective         | Tool              |
|---------------------------------------|
| Push Tags etc with branch protections | Github App    |
| Build Package                         | `build` + `setuptools` | 
| Version auto-compute and git traceability / consistency   | Commitizen |
| Change log                            | Commitizen |
| Package Publishing                    | Twine and Azure Artifact | 
| Package documentation                 | Sphinx + plugins |


### Key Takeaways

Overall, with github actions and by incorporating a few key tools, it is easy to set up python repositories to automatically produce internet-grade packages.

We can produce a workflow that automatically computes the next version, builds, publishes, ensures traceability, and generates a changelog along with high quality documentation.

Although we can automatically produce great package documentation, it is not straightforward to host the documentation within the enterprise, for consumption by artefact users. This is a key concern, because the alternative involves burying the documentation in a source package, which is not easy to access. There are also some gaps when it comes to capturing and displaying key software quality metrics in the documentation, such as coverage.

But, with a high-maturity, self-reinforcing, package-oriented python system of work established, producing packages at a quality that approximates that which we find in the public domain, signals to the surrounding organisation that our pythonistas mean business, and with trust, and safety in usage, encourage increased utility in our python artefacts.