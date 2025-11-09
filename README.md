# setup-ruby-selfhosted-github-action

A lightweight composite GitHub Action that installs **Ruby** on self hosted runners using [`ruby-build`](https://github.com/rbenv/ruby-build), based on your project’s `.ruby-version` file.

This is designed as a **building block** for:

- GitHub Actions with self hosted runners  
- Self hosted CI platforms like **Forgejo**, **Gitea**, or similar that support GitHub style actions

You can use it directly in workflows or as the core of a wrapper action (for example, one that adds caching and `bundle install`).

---

## What this action does

Given a checkout that contains `.ruby-version`, this action:

1. Reads the Ruby version from `.ruby-version`
2. Installs system build dependencies with `apt-get`
3. Clones [`ruby-build`](https://github.com/rbenv/ruby-build)
4. Installs Ruby into `"$HOME/.rubies/<version>"`
5. Adds `"$HOME/.rubies/<version>/bin"` to `PATH` for subsequent steps
6. Ensures [`bundler`](https://bundler.io) is installed

It does **not**:

- Cache Ruby or gems  
- Run `bundle install`  
- Use the system toolcache (`/opt/hostedtoolcache`)

Those responsibilities are left for wrapper actions (see below).

---

## Requirements

- A Linux runner (Debian or Ubuntu based)  
- `apt-get` available inside the job container or VM  
- A `.ruby-version` file in your project root (or specified working directory)

No preinstalled Ruby or `rbenv` required.

---

## Basic usage on GitHub Actions

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Ruby from .ruby-version
        uses: andyklimczak/setup-ruby-selfhosted-github-action@v1

      - name: Install gems
        run: |
          bundle install --jobs 4 --retry 3

      - name: Run tests
        run: bundle exec rake
```

If your Ruby app lives in a subdirectory:

```yaml
- name: Install Ruby from .ruby-version
  uses: andyklimczak/setup-ruby-selfhosted-github-action@v1
  with:
    working-directory: backend
```

---

## Using this as a building block on Forgejo, Gitea, or other self hosted CI

When running on a self hosted instance, you often want:

- Automatic Ruby install  
- Caching for compiled Rubies and gems  
- A single, reusable step for all Ruby apps

A common pattern:

1. Keep this action on **GitHub** (installer only)  
2. Create a **wrapper action** in your self hosted Git instance that:
   - Caches Rubies (`~/.rubies`)
   - Calls this GitHub action to install Ruby
   - Caches `vendor/bundle`
   - Runs `bundle install`

Your app workflows then only need one line:

```yaml
- uses: yourorg/ruby-setup-build@v1
```

while `yourorg/ruby-setup-build` internally calls this GitHub action.

---

## Example wrapper action (self hosted Git like Forgejo)

Imagine your self hosted Git is at `https://git.example.com` and you create a repo  
`yourorg/ruby-setup-build` containing this `action.yml`:

```yaml
name: "Ruby setup build"
description: "Installs Ruby from .ruby-version, caches Ruby and gems, and runs bundle install"

inputs:
  working-directory:
    description: "Directory where .ruby-version, Gemfile, and Gemfile.lock live"
    required: false
    default: "."
  bundler-arguments:
    description: "Extra arguments for bundle install"
    required: false
    default: "--jobs 4 --retry 3"

runs:
  using: "composite"
  steps:
    - name: Cache Rubies
      uses: https://code.forgejo.org/actions/cache@v3
      with:
        path: ~/.rubies
        key: ruby-${{ runner.os }}-${{ hashFiles(format('{0}/.ruby-version', inputs.working-directory)) }}

    - name: Install Ruby via setup-ruby-selfhosted
      uses: https://github.com/andyklimczak/setup-ruby-selfhosted-github-action@v1
      with:
        working-directory: ${{ inputs.working-directory }}

    - name: Cache bundle
      uses: https://code.forgejo.org/actions/cache@v3
      with:
        path: ${{ inputs.working-directory }}/vendor/bundle
        key: bundle-${{ runner.os }}-${{ hashFiles(format('{0}/Gemfile.lock', inputs.working-directory)) }}

    - name: Install gems
      shell: bash
      run: |
        set -eux
        cd "${{ inputs.working-directory }}"
        bundle config set path 'vendor/bundle'
        bundle install ${{ inputs.bundler-arguments }}
```

Notes:

- Uses full URLs for all external actions  
  for example `https://github.com/...` and `https://code.forgejo.org/...`
- Caches Ruby per `.ruby-version`
- Caches gems per `Gemfile.lock`
- Runs on any self hosted runner with `apt-get`

You can adapt the cache action URL if your self hosted platform uses a different cache action.

---

## Example app CI on a self hosted Forgejo or Gitea

Once you have the wrapper action, your app workflow can be minimal:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: https://github.com/actions/checkout@v4

      - name: Ruby setup
        uses: yourorg/ruby-setup-build@v1

      - name: Run tests
        run: bundle exec rspec
```

If your app lives in a subdirectory:

```yaml
      - name: Ruby setup
        uses: yourorg/ruby-setup-build@v1
        with:
          working-directory: backend
```

Now you can change `.ruby-version` in the repo, and:

- The wrapper detects a cache miss and compiles the new Ruby version once  
- Later jobs reuse that cached Ruby  
- Your workflow YAML never changes

---

## Why not use `ruby/setup-ruby` here

`ruby/setup-ruby` is ideal on **GitHub hosted runners** because those have a pre populated `/opt/hostedtoolcache`.

On **self hosted** or **Forgejo or Gitea** runners, it often refuses to auto install Ruby and fails with messages like:

```
The current runner was detected as self-hosted... you should install Ruby in the $RUNNER_TOOL_CACHE yourself.
```

This action avoids that limitation by installing Ruby explicitly inside the job and not relying on the platform toolcache.

---

## Summary

| Platform                | Recommended setup                                     |
| ----------------------- | ----------------------------------------------------- |
| GitHub hosted runners   | Use [`ruby/setup-ruby`](https://github.com/ruby/setup-ruby) |
| GitHub self hosted      | Use this action                                      |
| Forgejo or Gitea        | Use this action inside a wrapper as shown above      |

---

### Maintainer

**Andy Klimczak**  
<https://github.com/andyklimczak>

Feel free to fork, customize, or mirror this action for your own self hosted CI setups.
