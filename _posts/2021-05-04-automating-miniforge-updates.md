---
layout: post
title: Automating miniforge updates using Github Actions
feature_image: "/images/c-m-zY8WM82id0s-unsplash.jpg"
---

[`miniforge` and its variants `miniforge-pypy` and `mambaforge-*`](https://github.com/conda-forge/miniforge) are the base installers for using `conda` with `conda-forge` as the default source for packages. They will provide you with a basic conda installation to get started. This means that as part of that, the newest installers should also bring the newest `conda` and `mamba` versions with them.

In addition to the `miniforge` installers, `conda-forge` also provides Docker images with `miniforge` installed inside them. The docker images are managed through the [`miniforge-images` repository](https://github.com/conda-forge/miniforge-images). The images are based upon a small Ubuntu 20.04 base container and have the respectively named `miniforge` version installed in `/opt/conda`. They are provided on both DockerHub and quay.io: 

* [`condaforge/miniforge3`](https://hub.docker.com/r/condaforge/miniforge3)
* [`condaforge/miniforge-pypy3`](https://hub.docker.com/repository/docker/condaforge/miniforge-pypy3)
* [`condaforge/mambaforge`](https://hub.docker.com/repository/docker/condaforge/mambaforge)
* [`condaforge/mambaforge-pypy3`](https://hub.docker.com/repository/docker/condaforge/mambaforge-pypy3)
* [`quay.io/condaforge/miniforge3`](https://quay.io/repository/condaforge/miniforge3)
* [`quay.io/condaforge/miniforge-pypy3`](https://quay.io/repository/condaforge/miniforge-pypy3)
* [`quay.io/condaforge/mambaforge`](https://quay.io/repository/condaforge/mambaforge)
* [`quay.io/condaforge/mambaforge-pypy3`](https://quay.io/repository/condaforge/mambaforge-pypy3)

Up until recently the version updates were all done manually when one of the maintainers noticed that they were out of sync. On the side of the container images, this led to the extreme situation that only one update had been done. While these updates are simple, they have not been getting that much attention as there was neither any critical bug fixed nor features people were eagerly looking forward. The changes were mostly small bit-by-bit improvements.

As the updates are simple to do in these repositories though, it was simple on the other side to automatise them with Github Actions.

## Updating `mamba` in Miniforge

As the first action, I have built [an automated Github Actions workflow to update the `mamba` version](https://github.com/conda-forge/miniforge/pull/117) in the [`miniforge` repository](https://github.com/conda-forge/miniforge). In this action, we execute a small Python script that asks the Anaconda API for the latest mamba version and if found updates the `Miniforge3/construct.yaml` file. If changed, it opens a new PR on the repository.

Getting the most recent version from `anaconda.org` can be done using the following Python snippet. We fetch therefore all the available versions from the Anaconda API and use `setuptools`'s `packaging.version` to parse and sort the version numbers.

```python
import requests
from packaging import version

def get_most_recent_version(name):
    request = requests.get(
        "https://api.anaconda.org/package/conda-forge/" + name
    )
    request.raise_for_status()

    pkg = max(
        request.json()["files"], key=lambda x: version.parse(x["version"])
    )
    return pkg["version"]
```

After we have received the latest version, we load and rewrite the `Miniforge3/construct.yaml` file with the latest version. We do this without checking what the current value is as the following step in the workflow can check for us whether there was a change. In the case of a change, the `peter-evans/create-pull-request` creates a new pull request (or updates the existing one if one is open).

Note that we have pinned the actions here with an exact commit id. This is an additional security measure to protect us from the compromise of this action. We have checked that the current version doesn't do anything we would be afraid of. If we would have only pinned this to a tag, an attacker could get access to a Github token with repository write access and modify the contents/releases of the `miniforge` repository.

```yaml
- name: Create Pull Request
  id: cpr
  # This is the v3 tag but for security purposes we pin to the exact commit.
  uses: peter-evans/create-pull-request@052fc72b4198ba9fbc81b818c6e1859f747d49a8
    with:
      commit-message: "Update mamba version"
      title: "Update mamba version"
      body: |
        This PR was created by the autoupdate action as it detected that
        the mamba version has changed and thus should be updated
        in the configuration.
      branch: autoupdate-action
      delete-branch: true
```

Normally, when a Github action opens a pull request, the CI checks are not automatically run. A user has to manually close/reopen the pull requests for the actions to start. This a measure to prevent actions from re-triggering themselves in a cyclic chain. You can though override this behaviour by using an (SSH) deploy key to make the commit for the PR. To integrate that in the workflow, you need to create such a deploy key, add it as a deploy key **and** a secret to the repository and then use it to check out the code:

```yaml
- uses: actions/checkout@v2
  with:
    ssh-key: ${{ secrets.MINIFORGE_AUTOUPDATE_SSH_PRIVATE_KEY }}
```

With this the action is complete and we set it up to run every 6 hours:

```yaml
on:
 schedule:
   - cron: "0 */6 * * *"
```

## Open an issue on new `conda` releases

The second workflow I have implemented was for raising an issue when a new conda version was released. We did opt here to only raise an issue as the conda version is pulled from the git tag. Thus for a new conda version, we don't need to change any code but simply push a new tag to the repository.

For this workflow, I decided to explore using JavaScript with the `actions/github-script@v3`. In contrast to Python, this is not my primary language of choice but Github Actions' language of choice and thus the startup time is near to non-existent.

With the `actions/github-script@v3` we already get an initialised Github client as `github` variable that we can use to query the Github API. We use this to get the latest release of the `miniforge` repository. Sadly, there is only an API endpoint for releases, not for tags. Thus we might get issues if `miniforge` is only tagged but not released.

With the following snippet, we can then extract the version of the latest release. We also have extracted the latest release of `conda` using the Anaconda API. 

```javascript
github.repos.getLatestRelease({
  owner: context.repo.owner,
  repo: context.repo.repo,
}).then((release) => {
  const current_version = release['data']['tag_name'].split("-")[0]
  …
});
```

In the case that the `conda` version is higher than the encoded version in the `miniforge` release, we are looking whether there is already an open issue asking for a new release. If there is none, we are using the `github` client to open a new one.

```javascript
github.issues.listForRepo({
  owner: context.repo.owner,
  repo: context.repo.repo,
  state: "open",
  labels: "[bot] conda release"
}).then((issues) => {
  if (issues.data.length === 0) {
    github.issues.create({
      owner: context.repo.owner,
      repo: context.repo.repo,
      title: "New conda release: please tag a miniforge release",
      body: "A new conda release was found, please tag a new miniforge release with `" + conda_version + "-0`",
      labels: ["[bot] conda release"]
    });
  }
});
```

Using these building blocks, we already have a fully working [workflow that opens an issue asking for a new tag of `miniforge` on a `conda` release](https://github.com/conda-forge/miniforge/pull/132).


## Updating `miniforge-images` on `miniforge` release

With `miniforge` now being fully automated, we can shift our focus to `miniforge-images`, the repository for which I initially started this effort. The final workflow to make automatic pull requests on `miniforge` releases is a combination of the ideas of both workflows from above.

We can use the `github` releases API again to retrieve the latest tag of `miniforge`:

```javascript
github.repos.getLatestRelease({
  owner: 'conda-forge',
  repo: 'miniforge',
}).then((release) => {
  const miniforge_version = release['data']['tag_name'];
```

We then revert to using `sed` to search and replace the version number in the files that contain it:

```javascript
exec("sed -i -e 's/MINIFORGE_VERSION: \"[0-9.\\-]*\"/MINIFORGE_VERSION: \"" + miniforge_version + "\"/' azure-pipelines.yml", (error, stdout, stderr) => { 
…
exec("sed -i -e 's/MINIFORGE_VERSION=[0-9.\\-]*/MINIFORGE_VERSION=" + miniforge_version + "/' ubuntu/Dockerfile", (error, stdout, stderr) => {
…
```

Once the files have been modified, we use the `peter-evans/create-pull-request` action again to create a new pull request in the case that there is a difference to master.

The workflow was added in [`miniforge-images#5`](https://github.com/conda-forge/miniforge-images/pull/5) and was fixed a little bit in [`miniforge-images#8`](https://github.com/conda-forge/miniforge-images/pull/8) because I accidentally hard-coded the miniforge version in the initial one.

## Tricks for developing these workflows

While developing these workflows, I learnt some tricks on how to effectively develop these kinds of automation workflows.

The most important part of developing these kinds of workflows is to do this on a separate development repository. As the final workflow will only be triggered from the `main` / `master` branch, you will push quite a number of commits (or amend-and-force-pushes) to that branch. This is something that the other collaborators on the project won't appreciate on the main repository.

Additionally, you could develop the workflow with the final cron trigger with just a shorter scheduling interval but that will still lead to quite long feedback cycles. The issue here is cron triggers are not immediately activated after the push but may be delayed by some unknown amount of time.

Finally, if you are automatically making pull requests, Github already by default disables automatic workflow runs on the created pull request. This prohibits that you build infinite CI loops. Once you have ensured that your workflow doesn't trigger that you can switch to using deploy keys for making the pull requests. Up until working on this, I expected that deploy keys would be read-only but they actually can be marked as write keys, too.o

*Title picture: Photo by [C M](https://unsplash.com/@ubahnverleih?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

