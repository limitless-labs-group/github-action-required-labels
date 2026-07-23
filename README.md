# github-action-required-labels

This action allows you to fail the build if/unless a certain combination of labels are applied to a pull request.

## Usage

This action has three required inputs; `labels`, `mode` and `count`

| Name          | Description                                                                                                                                                  | Required | Default             |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------- | ------------------- |
| `labels`      | New line separated list of labels to match (legacy support for a comma separated list remains when `use_regex=false`)                                        | true     |
| `mode`        | The mode of comparison to use. One of: exactly, minimum, maximum                                                                                             | true     |
| `count`       | The required number of labels to match                                                                                                                       | true     |
| `token`       | The GitHub token to use when calling the API                                                                                                                 | false    | ${{ github.token }} |
| `message`     | The message to log and to add to the PR (if `add_comment=true`). See the README for available placeholders                                                   | false    |
| `add_comment` | Add a comment to the PR if required labels are missing. If a comment already exists, it will be updated. When the action passes, the comment will be deleted | false    | false               |
| `exit_type`   | The exit type of the action. One of: failure, success                                                                                                        | false    |
| `use_regex`   | Evaluate the values in `labels` as regular expressions                                                                                                       | false    |
| `trim`        | A prefix to strip from the matched labels in the `labels` and `matrix` outputs. Does not affect matching                                                     | false    |                     |

This action calls the GitHub API to fetch labels for a PR rather than reading `event.json`. This allows the action to run as intended when an earlier step adds a label. It will use `github.token` by default, and you can set the `token` input to provide alternative authentication.

This action provides the following outputs:

| Name     | Description                                                                                                     |
| -------- | --------------------------------------------------------------------------------------------------------------- |
| `labels` | The matching labels as a comma separated string. Only set when the action passes                                |
| `matrix` | The matching labels as a JSON array, for use with `fromJSON` in a job matrix. Set to `[]` when the action fails |
| `status` | `success` or `failure`. Useful in combination with `exit_type: success`                                         |

> [!TIP]
> Label matching is **case-insensitive**.

## Examples

### Complete example

```yaml
name: Pull Request Labels
on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]
jobs:
  label:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    steps:
      - uses: mheap/github-action-required-labels@v5
        with:
          mode: exactly
          count: 1
          labels: |
            semver:patch
            semver:minor
            semver:major
```

### Prevent merging if a label exists

```yaml
- uses: mheap/github-action-required-labels@v5
  with:
    mode: exactly
    count: 0
    labels: "do not merge"
```

### Post a comment when the check fails

You can choose to add a comment to the PR when the action fails. The default format is:

> Label error. Requires {{ errorString }} {{ count }} of: {{ provided }}. Found: {{ applied }}

```yaml
- uses: mheap/github-action-required-labels@v5
  with:
    mode: exactly
    count: 1
    labels: |
      semver:patch
      semver:minor
      semver:major
    add_comment: true
```

If a comment already exists, it will be updated. When the action passes, the comment will be deleted.

### Customising the failure message / comment

You can also customise the message used by providing the `message` input:

```yaml
- uses: mheap/github-action-required-labels@v5
  with:
    mode: exactly
    count: 1
    labels: |
      semver:patch
      semver:minor
      semver:major
    add_comment: true
    message: "This PR is being prevented from merging because you have added one of our blocking labels: {{ provided }}. You'll need to remove it before this PR can be merged."
```

The following tokens are available for use in custom messages:

| Token       | Value                                    |
| ----------- | ---------------------------------------- |
| mode        | One of: `exactly`, `minimum`, `maximum`  |
| count       | The value of the `count` input           |
| errorString | One of: `exactly`, `at least`, `at most` |
| provided    | The value of the `labels` input          |
| applied     | The labels that are applied to the PR    |

### Require multiple labels

```yaml
- uses: mheap/github-action-required-labels@v5
  with:
    mode: minimum
    count: 2
    labels: |
      community-reviewed
      team-reviewed
      codeowner-reviewed
```

### Use regular expressions

```yaml
- uses: mheap/github-action-required-labels@v5
  with:
    mode: exactly
    count: 1
    labels: "semver:.*"
    use_regex: true
```

### Preventing comment collisions

This action uses a combination of the workflow name/path, job ID, and step ID to add an invisible "match token" to the beginning of any comments it creates. That way, it can later know which comments it owns when modifying them, while supporting multiple "instances" of this action to be run at the same time within a repo.

However, note that if any of those three identifiers change, any "in flight" comments on open PRs may be orphaned (since the final match token will have changed between runs). If you rename any of those identifiers, you will have to delete any orphaned comments manually.

### Controlling failure

You can set `exit_type` to success then inspect `outputs.status` to see if the action passed or failed. This is useful when you want to perform additional actions if a label is not present, but not fail the entire build.

```yaml
- uses: mheap/github-action-required-labels@v5
  with:
    mode: minimum
    count: 2
    labels: |
      community-reviewed
      team-reviewed
      codeowner-reviewed
    exit_type: success # Can be: success or failure (default: failure)
```

If the action passed, `outputs.status` will be `success`. If it failed, `outputs.status` will be `failure`.

Here is a complete workflow example for this use case:

```yaml
name: Pull Request Labels
on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]
jobs:
  label:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    outputs:
      status: ${{ steps.check-labels.outputs.status }}
    steps:
      - id: check-labels
        uses: mheap/github-action-required-labels@v5
        with:
          mode: exactly
          count: 1
          labels: |
            semver:patch
            semver:minor
            semver:major
          exit_type: success
  do-other:
    runs-on: ubuntu-latest
    needs: label
    steps:
      - run: echo SUCCESS
        if: needs.label.outputs.status == 'success'
      - run: echo FAILURE && exit 1
        if: needs.label.outputs.status == 'failure'
```

### Using Output Labels

If the action was successful you can access the matching labels via `outputs.labels`. This is useful if you want to use the labels in a later step.

```yaml
name: Pull Request Labels
on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]
jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - id: check-labels
        uses: mheap/github-action-required-labels@v5
        with:
          mode: minimum
          count: 1
          labels: |
            feature-1
            feature-2
            feature-3
      - run: |
          echo "Enabled Features:"
          for f in $(echo "{{steps.check-labels.outputs.labels}}" | sed "s/,/ /g")
          do
            echo "$f"
          done
```

### Fanning out with a job matrix

`outputs.matrix` contains the same matching labels as a JSON array, which can be passed to `fromJSON` to build a [job matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs). This runs one job per matching label:

```yaml
name: Deploy Environments
on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]
jobs:
  label:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.check-labels.outputs.matrix }}
    steps:
      - id: check-labels
        uses: mheap/github-action-required-labels@v5
        with:
          mode: minimum
          count: 1
          labels: |
            deploy:staging
            deploy:preview
            deploy:production
  deploy:
    runs-on: ubuntu-latest
    needs: label
    strategy:
      matrix:
        label: ${{ fromJSON(needs.label.outputs.matrix) }}
    steps:
      - run: echo "Deploying ${{ matrix.label }}"
```

If no labels matched, `outputs.matrix` is `[]` and GitHub skips the dependent job rather than failing it. This means you can pair it with `exit_type: success` to fan out over whichever labels are present without ever failing the build:

```yaml
- id: check-labels
  uses: mheap/github-action-required-labels@v5
  with:
    mode: minimum
    count: 1
    labels: |
      deploy:staging
      deploy:preview
    exit_type: success
```

> [!NOTE]
> When `use_regex` is true, `outputs.matrix` contains the labels **applied to the PR** that matched your patterns, not the patterns themselves. This is the same behaviour as `outputs.labels`.

### Trimming a prefix from the outputs

Labels are often namespaced (`deploy:staging`, `semver:minor`) but downstream steps usually only want the part after the prefix. Set `trim` to strip it from both the `labels` and `matrix` outputs:

```yaml
jobs:
  label:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.check-labels.outputs.matrix }}
    steps:
      - id: check-labels
        uses: mheap/github-action-required-labels@v5
        with:
          mode: minimum
          count: 1
          trim: "deploy:"
          labels: |
            deploy:staging
            deploy:preview
            deploy:production
  deploy:
    runs-on: ubuntu-latest
    needs: label
    strategy:
      matrix:
        environment: ${{ fromJSON(needs.label.outputs.matrix) }}
    environment: ${{ matrix.environment }}
    steps:
      - run: echo "Deploying to ${{ matrix.environment }}"
```

With `deploy:staging` and `deploy:production` applied, `outputs.matrix` is `["staging","production"]` and `outputs.labels` is `staging,production`.

Notes on the behaviour:

- `trim` only affects the outputs. Matching, the `count` comparison, and the `{{ provided }}` / `{{ applied }}` values in the failure message all use the untrimmed labels, so adding `trim` can never change whether the action passes or fails.
- Labels that don't start with the prefix are passed through unchanged.
- The prefix is matched case-insensitively, consistent with label matching. The remainder of the label keeps its original casing.
- A label that is _exactly_ the prefix would trim to an empty string, so it is dropped from the outputs rather than adding an empty matrix entry.
