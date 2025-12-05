# Forgejo Action for ORT

Run licensing, security, best practices checks and generate reports/Software Bill of Materials (SBOMs) using [ORT][ort]
within [Forgejo Actions][forgejo-action-docs].

## Usage

See [action.yml](action.yml)

By default, this action only works on Ubuntu-based runners and installs ORT by downloading the latest distribution archive, e.g., `ort-[version-number].tgz`, from [ORT releases](https://github.com/oss-review-toolkit/ort/releases). It requires you to have the necessary package managers installed for the types of projects you wish to analyze.

If you would like to use ORT to check a .NET project, be sure to run [setup-dotnet][gh-action-setup-dotnet] before executing the Forgejo Action for ORT.

For Java, you don't need to do anything—this action will automatically run [setup-java][gh-action-setup-java] if Java is not installed, as it is needed to run ORT from the distribution archive. Alternatively, you can also [run Forgejo Action for ORT using Docker](#run-forgejo-action-for-ort-using-docker) if you prefer not to deal with setting up various build and package managers within your CI runner.

### Basic

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
```

Alternatively, you can also use ORT to download the project sources using Git, Git-repo, Mercurial or Subversion.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          vcs-url: 'https://github.com/jshttp/mime-types.git'
```

### Scenarios

- [Run ORT and analyze only specified package managers](#run-ort-and-analyze-only-specified-package-managers)
- [Run ORT with labels](#run-ort-with-labels)
- [Run ORT and fail job on policy violations or security issues](#run-ort-and-fail-job-on-policy-violations-or-security-issues)
- [Run ORT on private repositories](#run-ort-on-private-repositories)
- [Run ORT on multiple repositories using a matrix](#run-ort-on-multiple-repositories-using-a-matrix)
- [Run ORT with a custom global configuration](#run-ort-with-a-custom-global-configuration)
- [Run ORT with PostgreSQL database](#run-ort-with-postgresql-database)
- [Run only parts of the Forgejo Action for ORT](#run-only-parts-of-the-forgejo-action-for-ort)
- [Install only ORT and ORT helper](#install-only-ort-and-ort-helper)
- [Run Forgejo Action for ORT using Docker](#run-forgejo-action-for-ort-using-docker)
- [Run ORT with a custom Docker image](#run-ort-with-a-custom-docker-image)

#### Run ORT and analyze only specified package managers

Want ORT to only check your repository for just certain package managers?
Just set the `ort-cli-args` argument to specify the ones you want.
Here’s how:

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
        with:
          repository: 'jshttp/mime-types'
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          allow-dynamic-versions: 'true'
          ort-cli-args: '-P ort.analyzer.enabledPackageManagers=Yarn,Maven'
```

#### Run ORT with labels

Use labels to track scan related info or execute policy rules
with a specific context e.g. product, delivery or organization.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          allow-dynamic-versions: 'true'
          ort-cli-analyze-args: >
            -l project=oss-project
            -l dist=external
            -l org=engineering-sdk-xyz-team-germany-berlin
```

#### Run ORT and fail job on policy violations or security issues

Set `fail-on` to fail the action if:
- policy violations reported by Evaluator exceed the `severeRuleViolationThreshold` level.
- security issues reported by the Advisor exceed the `severeIssueThreshold` level.

By default `severeRuleViolationThreshold` and `severeIssueThreshold` are set to `WARNING`
but you can change this to for example `ERROR` in your [config.yml][ort-config-yml].

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          allow-dynamic-versions: 'true'
          fail-on: 'violations'
```

#### Run ORT on private repositories

To run ORT on private Git repositories, we recommend to:
- Set up an account with read-only access rights
- Use a .netrc file, SSH keys or [Forgejo access tokens][forgejo-access-tokens] for authentication.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Add .netrc
        run: >
          default
          login ${{ secrets.NETRC_LOGIN }}
          password ${{ secrets.NETRC_PASSWORD }}" > ~/.netrc
      - name: Add SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_github
          echo "${{ secrets.SSH_PUBLIC_KEY }}" > ~/.ssh/id_github.pub
          chmod 600 ~/.ssh/id_github*
          cat >>~/.ssh/config <<END
          Host github.com
            HostName ssh.github.com
            User git
            Port 443
            IdentityFile ~/.ssh/id_github
            StrictHostKeyChecking no
          END
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          allow-dynamic-versions: 'true'
```

```yaml
jobs:
  ort:
    runs-on: [self-hosted, linux]
    name: Run ORT

    steps:
      - name: Configure proxy server
        run: |
          https_proxy="http://proxy.example.com:3128/"
          http_proxy="http://proxy.example.com:3128/"
          printenv >> "$GITHUB_ENV"
      - name: Ensure Git is installed on the CI runner, if not install it
        run: |
          command -v git >/dev/null 2>&1 || { echo -e "\e[1;34m Git not found, installing..."; apt-get update; apt-get install -y git; }
      - name: Use HTTPS with personal token always for Git cloning
        run: |
          git config --global url."https://oauth2:${{ secrets.PERSONAL_TOKEN_1 }}@github.com/".insteadOf "ssh://git@github.com/"
          git config --global url."https://oauth2:${{ secrets.PERSONAL_TOKEN_2 }}@git.example.com/".insteadOf "ssh://git@git.example.com/"
          git config --global url."https://oauth2:${{ secrets.PERSONAL_TOKEN_2 }}@git.example.com/".insteadOf "https://git.example.com/"
      - name: Checkout project
        uses: actions/checkout@v5
        with:
          repository: 'example-org/alpha'
          ref: 'master'
          github-server-url: 'https://git.example.com'
          token: ${{ secrets.PERSONAL_TOKEN_2 }}
      - name: Run Forgejo action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          ort-config-repository: 'https://oauth2:${{ secrets.PERSONAL_TOKEN_2 }}@git.example.com/ort-project/ort-config.git'
          run: >
            cache-dependencies,
            metadata-labels,
            analyzer,
            advisor,
            reporter,
            upload-results
```

#### Run ORT on multiple repositories using a matrix

Use [Forgejo's action matrix feature][forgejo-actions-matrix] to run
the Forgejo Action for ORT on multiple repositories.

```yaml
jobs:
  ort:
    strategy:
      fail-fast: false
      matrix:
        include:
          - repository: example-org/alpha
            sw-name: alpha
          - repository: example-org/beta
            sw-name: beta
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          repository: ${{ matrix.repository }}
      - uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          sw-name: ${{ matrix.sw-name }}
```

#### Run ORT with a custom global configuration

Use `ort-config-repository` to specify the location of your ORT global configuration repository.
If `ort-config-revision` is not automatically latest state of configuration repository will be used.

Alternatively, you can also place your ORT global configuration files in `~/.ort/config`
prior to running Forgejo Action for ORT.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          ort-config-repository: 'https://github.com/oss-review-toolkit/ort-config'
          ort-config-revision: 'e4ae8f0a2d0415e35d80df0f48dd95c90a992514'
```

#### Run ORT with PostgreSQL database

ORT supports using a PostgreSQL database to caching scan data to speed-up scans.

Use the following [action secrets at Forgejo org or repository level][forgejo-action-secrets] to specified the database to use:
- `POSTGRES_URL`: 'jdbc:postgresql://ort-db.example.com:5432/ort'
- `POSTGRES_USERNAME`: 'ort-db-username'
- `POSTGRES_PASSWORD`: 'ort-db-password'

Next, pass these secrets to Forgejo Action for ORT:

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
        with:
          repository: 'jshttp/mime-types'
          ref: '2.1.35'
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          db-url: ${{ secrets.POSTGRES_URL }}
          db-username: ${{ secrets.POSTGRES_USERNAME }}
          db-password: ${{ secrets.POSTGRES_PASSWORD }}
          sw-name: 'Mime Types'
          sw-version: '2.1.35'
```

#### Install only ORT and ORT helper

Want to just install `ort` and `orth` within your runner?
No problem! Just set up your Forgejo Actions workflow as shown below.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Setup ORT and ORT helper from release distribution archives
        with:
          run: >
            setup-ort
            setup-orth
      - name: Check ORT and ORT helper are installed, fail CI if not found
        shell: bash
        run: |
          if ! command -v ort &> /dev/null; then
            exit 1
          fi
          if ! command -v orth &> /dev/null; then
            exit 1
          fi
      - name: Print Forgejo Action for ORT outputs variables
        shell: bash
        run: |
          echo "Installed ORT version: ${{ steps.ort.outputs.orth-version }}"
          echo "URL of ORT dist archive: ${{ steps.ort.outputs.orth-dist-archive-url }}"
          echo "Installed ORT helper version: ${{ steps.ort.outputs.orth-version }}"
          echo "URL of ORT helper dist archive: ${{ steps.ort.outputs.orth-dist-archive-url }}"
```

#### Run Forgejo Action for ORT using Docker

Instead of using the latest distribution archive, e.g., `ort-[version-number].tgz`,
from [ORT releases](https://github.com/oss-review-toolkit/ort/releases),
you can also run this action using the [ORT Docker image][ort-docker-image].
While it might be a bit slower, it has the advantage of eliminating the need
to set up various build and package managers within your CI runner.
However, it does require a runner that supports docker-in-docker.

To configure the action in Docker mode, set the `mode` parameter to 'dnd' or 'docker-in-docker'.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          mode: dnd
```

#### Run ORT with a custom Docker image

Instead of the default latest released [ORT Docker image][ort-docker-image] you can set
the `image` parameter to use your own custom ORT Docker image.

```yaml
jobs:
  ort:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v5
      - name: Run Forgejo Action for ORT
        uses: https://codeberg.org/oss-review-toolkit/ort-ci-forgejo-action@v1
        with:
          image: 'my-org/ort-images/ort:latest'
          mode: dnd
```

# Want to Help or have Questions?

All contributions are welcome. If you are interested in contributing, please read our
[contributing guide][ort-contributing-md], and to get quick answers
to any of your questions we recommend you [join our Slack community][ort-slack].

# License

Copyright (C) 2020-2025 [The ORT Project Copyright Holders](./NOTICE).

See the [LICENSE](./LICENSE) file in the root of this project for license details.

OSS Review Toolkit (ORT) is a [Linux Foundation project][lf] and part of [ACT][act].

[act]: https://automatecompliance.org/
[forgejo-access-tokens]: https://forgejo.org/docs/latest/user/token-scope/
[forgejo-action-docs]: https://forgejo.org/docs/latest/user/actions/basic-concepts/
[forgejo-actions-matrix]: https://forgejo.org/docs/latest/user/actions/reference/#jobsjob_idstrategymatrix
[forgejo-action-secrets]: https://forgejo.org/docs/latest/user/actions/reference/#secrets
[gh-action-setup-dotnet]: https://github.com/actions/setup-dotnet
[gh-action-setup-java]: https://github.com/actions/setup-java
[ort]: https://github.com/oss-review-toolkit/ort
[ort-config-yml]: https://github.com/oss-review-toolkit/ort/blob/main/model/src/main/resources/reference.yml
[ort-contributing-md]: https://github.com/oss-review-toolkit/.github/blob/main/CONTRIBUTING.md
[ort-docker-image]: https://github.com/oss-review-toolkit/ort/pkgs/container/ort?tag=latest
[ort-releases]: https://github.com/oss-review-toolkit/ort/releases
[ort-slack]: http://slack.oss-review-toolkit.org
[lf]: https://www.linuxfoundation.org
