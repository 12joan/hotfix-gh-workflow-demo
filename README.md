# GitHub workflow hotfix demo

This repo demonstrates how GitHub workflows can be configured such that, by default, a particular workflow job runs in a queue, but commits tagged `[Hotfix]` can skip this queue and cause all previous pending or in-progress jobs to be cancelled.

## How it works

https://github.com/12joan/hotfix-gh-workflow-demo/blob/b30e569e2bb817480b110fb5a579abf89fcbd6ef/.github/workflows/ci.yml#L12-L22

The important part is the `concurrency` group:

- `group` - This option uniquely identifies the group. Since this particular job runs only on the main branch, I've used a hardcoded string here, but you'll often see `group: ${{ github.workflow }}-${{ github.ref }}` for jobs that should run independently on multiple branches.
- `queue` - This controls what happens when there are multiple pending jobs. By default, GitHub only allows one running and one pending job; if a new pending job is added, then the existing pending job is cancelled.
  - For most commits, we're setting `queue` to `max`, which means that up to 100 jobs can be pending at once. Typically, this means that the job will run once for every pushed group of commits, one at a time and without skipping any, in chronological order. If you want to ensure that the job runs for each commit individually, make sure you push them one at a time.
  - If the commit message contains the string `[hotfix]` (case insensitive), we instead set the `queue` to its default value of `single`. This allows us to set `cancel-in-progress` to true, since GitHub doesn't allow both `cancel-in-progress` and `queue: max` to be enabled at once.
- `cancel-in-progress` - If this option is true, GitHub will cancel any already running job when a new job becomes pending. We set this to true for `[hotfix]` commits only, meaning that these commits will jump the queue and cancel anything that's already in progress.

In this particular workflow, the `concurrency` group is added only to the `deploy` job. The `test` job has no concurrency group, which means that the tests run immediately for each push, without waiting for the previous run to finish. This happens for both regular commits and `[hotfix]` commits.

## Try it out

To try it yourself, fork this repo and try pushing commits one at a time. You can use `git commit --allow-empty` to create a commit without changing any files. Go to the Actions tab to see the currently running workflows for each commit.

Remember to set the upstream to your fork before running any of the following commands; otherwise they may not work correctly.

### Push empty commits

Run this command to push three empty commits with a five-second delay between each one.

```bash
for i in 1 2 3; do
  git commit --allow-empty -m "Empty commit $i" && git push && sleep 5
done
```

| <img src=".github/assets/empty-1.png" alt="All three commits running" /> | <img src=".github/assets/empty-2.png" alt="Only the first commit running; the other two pending" /> | <img src=".github/assets/empty-3.png" alt="The first commit passed, the second running and the third pending" /> |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Initially, the `test` job runs for all commits, so they all show as in progress. | When the `test` job finishes, the second and third commits become pending. | When the first commit finishes, the second commit begins its `deploy` job. |

### Push a hotfix commit

Run this command to push three empty commits, followed by a hotfix commit.

```bash
for i in 1 2 3; do
  git commit --allow-empty -m "Empty commit $i" && git push && sleep 5
done
git commit --allow-empty -m "[HOTFIX] Empty hotfix commit" && git push
```

| <img src=".github/assets/hotfix-1.png" alt="The first commit running, the second and third pending, and the hotfix commit running" /> | <img src=".github/assets/hotfix-2.png" alt="The first three commits cancelled; the hotfix commit running" /> |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Initially, the empty commits are queued up as before, while the hotfix commit's `test` job runs. | When the hotfix commit's `deploy` job is ready, it cancels all other commits' jobs. |