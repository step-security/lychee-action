[![StepSecurity Maintained Action](https://raw.githubusercontent.com/step-security/maintained-actions-assets/main/assets/maintained-action-banner.png)](https://docs.stepsecurity.io/actions/stepsecurity-maintained-actions)

# lychee link checking action


Quickly check links in Markdown, HTML, and text files using [lychee].

When used in conjunction with [Create Issue From File], issues will be
opened when the action finds link problems (make sure to specify the `issues: write` permission in the [workflow](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#permissions) or the [job](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idpermissions)).

## Usage

Here is a full example of a GitHub workflow file:

It will check all repository links once per day and create an issue in case of
errors. Save this under `.github/workflows/links.yml`:

```yaml
name: Links

on:
  repository_dispatch:
  workflow_dispatch:
  schedule:
    - cron: "00 18 * * *"

jobs:
  linkChecker:
    runs-on: ubuntu-latest
    permissions:
      issues: write # required for peter-evans/create-issue-from-file
    steps:
      - uses: actions/checkout@v7

      - name: Link Checker
        id: lychee
        uses: step-security/lychee-action@v2
        with:
          fail: false

      - name: Create Issue From File
        if: steps.lychee.outputs.exit_code != 0
        uses: step-security/create-issue-from-file@v6
        with:
          title: Link Checker Report
          content-filepath: ./lychee/out.md
          labels: report, automated issue
```

## Passing arguments

This action uses [lychee] for link checking.
lychee arguments can be passed to the action via the `args` parameter.

On top of that, the action also supports some additional arguments.

| Argument         | Examples                | Description                                                                             |
| ---------------- | ----------------------- | --------------------------------------------------------------------------------------- |
| args             | `--cache`, `--insecure` | See [lychee's documentation][lychee-args] for all arguments and values                  |
| debug            | `false`                 | Enable debug output in action (set -x). Helpful for troubleshooting                     |
| fail             | `false`                 | Fail workflow run on error (i.e. when [lychee exit code][lychee-exit] is not 0)         |
| failIfEmpty      | `false`                 | Fail entire pipeline if no links were found                                             |
| format           | `markdown`, `json`      | Summary output format                                                                   |
| jobSummary       | `false`                 | Write GitHub job summary (on Markdown output only)                                      |
| lycheeVersion    | `v0.15.0`, `nightly`    | Overwrite the lychee version to be used                                                 |
| output           | `lychee/results.md`     | Summary output file path                                                                |
| token            | `""`                    | Custom GitHub token to use for API calls                                                |
| workingDirectory | `.`, `path/to/subdir/`  | Custom working directory to run lychee in. This affects where `output.md` gets created. |

See [action.yml](./action.yml) for a full list of supported arguments and their default values.

### Passing arguments

Here is how to pass the arguments.

```yml
- name: Link Checker
  uses: step-security/lychee-action@v2
  with:
    # Check all markdown, html and reStructuredText files in repo (default)
    args: --root-dir "$(pwd)" --verbose --no-progress './**/*.md' './**/*.html' './**/*.rst'
    # Use json as output format (instead of markdown)
    format: json
    # Use different output file path
    output: /tmp/foo.txt
    # Use a custom GitHub token, which
    token: ${{ secrets.CUSTOM_TOKEN }}
    # Don't fail action on broken links
    fail: false
    # Run lychee in a different directory.
    # Note: This changes the lychee output directory as well.
    workingDirectory: website/subdir
```

(If you need a token that requires permissions that aren't available in the
default `GITHUB_TOKEN`, you can create a [personal access
token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token)
and pass it to the action via the `token` parameter.)

## Utilising the cache feature

In order to mitigate issues regarding rate limiting or to reduce stress on external resources, one can setup lychee's cache similar to this:

```yml
- name: Restore lychee cache
  uses: actions/cache@v6
  with:
    path: .lycheecache
    key: cache-lychee-${{ github.sha }}
    restore-keys: cache-lychee-

- name: Run lychee
  uses: step-security/lychee-action@v2
  with:
    args: "--root-dir "$(pwd)" --cache --max-cache-age 1d ."
```

It will compare and save the cache based on the given key.
So in this setup, as long as a user triggers the CI run from the same commit, it will be the same key. The first run will save the cache, subsequent runs will not update it (because it's the same commit hash).
For restoring the cache, the most recent available one is used (commit hash doesn't matter).

If you need more control over when caches are restored and saved, you can split the cache step and e.g. ensure to always save the cache (also when the link check step fails):

```yml
- name: Restore lychee cache
  id: restore-cache
  uses: actions/cache/restore@v6
  with:
    path: .lycheecache
    key: cache-lychee-${{ github.sha }}
    restore-keys: cache-lychee-

- name: Run lychee
  uses: step-security/lychee-action@v2
  with:
    args: "--root-dir "$(pwd)" --cache --max-cache-age 1d ."

- name: Save lychee cache
  uses: actions/cache/save@v6
  if: always()
  with:
    path: .lycheecache
    key: ${{ steps.restore-cache.outputs.cache-primary-key }}
```

## Excluding links from getting checked

Add a `.lycheeignore` file to the root of your repository to exclude links from
getting checked. It supports regular expressions. One expression per line.

You can also put your `.lycheeignore` into a subdirectory if you set that as the `workingDirectory`.

## Fancy badge

Pro tip: You can add a little badge to your repo to show the status of your
links. Just replace `org` with your organisation name and `repo` with the
repository name and put it into your `README.md`:

```
[![Check Links](https://github.com/org/repo/actions/workflows/links.yml/badge.svg)](https://github.com/org/repo/actions/workflows/links.yml)
```

## Troubleshooting and common problems

See [lychee's Troubleshooting Guide][troubleshooting] for solutions to common
link-checking problems.

## Performance

A full CI run to scan 576 links takes approximately 1 minute for the
[analysis-tools-dev/static-analysis](https://github.com/analysis-tools-dev/static-analysis)
repository.

## Security and Updates

It is recommended to pin lychee-action to a fixed version [for security
reasons][security]. You can use dependabot to automatically keep your GitHub
actions up-to-date. This is a great way to pin lychee-action, while still
receiving updates in the future. It's a relatively easy thing to do.

Create a file named `.github/dependabot.yml` with the following contents:

```yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: ".github/workflows"
    schedule:
      interval: "daily"
```

When you add or update the `dependabot.yml` file, this triggers an immediate check for version updates.
Please see [the documentation][dependabot] for all configuration options.

### Security tip

For additional security when relying on automation to update actions you can pin
the action to a SHA-256 rather than the semver version so as to avoid tag
spoofing Dependabot will still be able to automatically update this.

For example:

```yml
- name: Link Checker
  uses: step-security/lychee-action@7da8ec1fc4e01b5a12062ac6c589c10a4ce70d67 # for v2.0.0
```

## Credits

This action is based on the deprecated [peter-evans/link-checker] and uses
[lychee] (written in Rust) instead of liche (written in Go) for link checking.

## License

lychee is licensed under

- [MIT License] ([LICENSE-MIT](./LICENSE))


[lychee]: https://github.com/lycheeverse/lychee
[lychee-args]: https://github.com/lycheeverse/lychee#commandline-parameters
[lychee-exit]: https://github.com/lycheeverse/lychee#exit-codes
[troubleshooting]: https://github.com/lycheeverse/lychee/blob/master/docs/TROUBLESHOOTING.md
[security]: https://francoisbest.com/posts/2020/the-security-of-github-actions
[dependabot]: https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file
[peter-evans/link-checker]: https://github.com/peter-evans/link-checker
[create issue from file]: https://github.com/step-security/create-issue-from-file
[mit license]: https://choosealicense.com/licenses/mit
