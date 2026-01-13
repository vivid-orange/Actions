# Actions
### A repo for shared github actions

The **CI Dispatcher** is an easy-to-use, automated CI tool designed to run GitHub Actions workflows from events from your local repo.  

## Contents

- [What You Get](#what-you-get) – The benefits of the pipelines
- [What it does](#what-it-does) – What the pipeline does
- [How to use](#how-to-use) – See how to integrate and call the dispatcher from your repository.  
- [How it works](#how-it-works) – Dive into the internal logic and event flow of the dispatcher.



## What You Get

- **Build** validation on pull requests — prevents merging broken code.
- **Linting** for consistent code style enforcement - optionally.
- **Unit Testing** with coverage tracking:
- **Coverage reports** automatically uploaded to [codecov.io](https://app.codecov.io/gh/vivid-orange) (public) or as a PR comment (for private repos).
- **Package** libraries automatically and **draft release** creation with **semantic versioning** on merge to `main`.
- **Version tagging** and **NuGet publishing** on release  .
- **Private feed support** — seamlessly restore private NuGet package references from DevExpress and Azure Artifacts.
- **Unified CI logic** across all repositories — managed centrally, no duplication of `.github/workflows`.  



## What it does

#### Pull requests (PR):
- Lint (code style enforcement)
- Build the project in release configuration
- Run test with coverage collection
- Upload coverage to Codecov (public repos only) and post results as a PR comment (both public and private).
<img width="656" height="346" alt="image" src="https://github.com/user-attachments/assets/f1dd5dfa-59b5-4d19-991d-2d51b59a8e24" />




#### Merges to main
- Build and test in Release configuration  
- Upload coverage
- Package `.nupkg` and `.snupkg` files
- Auto-tag commit with semantic version number (1.2.3.4)
- Create or update a **draft release**
<img width="657" height="347" alt="image" src="https://github.com/user-attachments/assets/2cbd8d13-999c-40da-85ad-0008668fb9a5" />




#### Releases
- Strip build number from package (e.g., `1.2.3.4 → 1.2.3`)  
- Repackage and push to NuGet.org
- Clean up old draft releases
<img width="669" height="345" alt="image" src="https://github.com/user-attachments/assets/493f05a0-c614-4f5e-ab19-ff389a33e03f" />




## How to use

Add this single workflow file in your repo at:
.github/workflows/ci.yml

```yaml
name: CI using shared actions

on:
  push:
    branches: [ main ]
  pull_request:
    types: [opened, edited, synchronize]
  release:
    types: [published]

jobs:
  ci:
    uses: vivid-orange/Actions/.github/workflows/ci-dotnet.yml@main
    secrets: inherit
```

That’s it — this one file triggers the correct pipeline for:
- Pull Requests: lint, build, test, and upload coverage
- Merge to main: build, test, package, and create a draft release
- Published Releases: strip build numbers, retag, and push to NuGet

### Optional inputs
You can override default behaviour directly in the job call:

```yaml
jobs:
  ci:
    uses: vivid-orange/Actions/.github/workflows/ci-dotnet.yml@main
    secrets: inherit
jobs:
  ci:
    uses: vivid-orange/Actions/.github/workflows/ci-dotnet.yml@main
    secrets: inherit
    with:
      dotnet: '8.0.x'                        # Optional .NET version
      lint: false                            # Disable linting (default = true)
      lint-auto-fix: true                    # The linter will commit changes to your PR (default = false)
      codecov: false                         # Skip Codecov upload
      test-filter: 'Category!=Integration'   # Filter expression for dotnet test
      private: true                          # Enables private repo logic
```



## How it works
All logic runs through `ci-dotnet.yml`, which dispatches to specialised workflows depending on the event type.

ci-dotnet.yml
| Event                 | Workflow                                                                                                               | Description                                      |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `pull_request`        | [`dotnet-build-test.yml`](https://github.com/vivid-orange/Actions/blob/main/.github/workflows/dotnet-build-test.yml)     | Lints, builds, tests, and uploads coverage       |
| `push` (merge to main)         | [`dotnet-pack-release.yml`](https://github.com/vivid-orange/Actions/blob/main/.github/workflows/dotnet-pack-release.yml) | Packages the project and creates a draft release |
| `release` (published) | [`release-push-nuget.yml`](https://github.com/vivid-orange/Actions/blob/main/.github/workflows/release-push-nuget.yml)   | Republishes release assets to NuGet.org          |


### dotnet-build-test.yml

Runs on PRs and pushes.
Steps:
- Checkout repository
- Setup a specific .NET version (optional input)
- Add private NuGet feeds (if `private: true`)
- Lint using dotnet format (optional lint code styling check)
- Build project in Release configuration
- Run Tests with coverage collection
- Upload coverage artifact
- Outputs: Upload coverage and package artifacts (on merge to main)

### codecov-upload.yml

Uploads coverage reports to codecov.io.
Uses organisation secret `CODECOV_TOKEN` (without having it passed down).

Steps:
- For public repos: uploads to Codecov and posts a PR comment
- For private repos: merges coverage results, generates Markdown summary, and posts as PR comment

### dotnet-pack-release.yml

Triggered on push to main.
Builds and packages all .nupkg / .snupkg files, removes any -preview suffixes, and creates or updates a draft release tagged as Major.Minor.Patch.Build.

Steps:
- Downloads package artifacts
- Removes -preview suffix
- Extracts semantic version number
- Creates or updates a draft GitHub release with .nupkg and .snupkg files

### release-push-nuget.yml

Triggered on release publication.
Republishes the previously drafted packages to NuGet and cleans up old drafts.

Requires: `NUGET_API_KEY` secret which must be passed from the local repo stub. By passing `secrets: inherit` in the local repo stub we pass the global organisation secret.

Steps:
- Downloads release artifacts
- Strips build number from version
- Updates .nuspec and repackages
- Pushes tag to Git and publishes package to NuGet.org
- Cleans up old drafts


### Notes

- Supports .NET 8 and newer
- Reusable workflows can be called directly if fine-grained control is needed, or mixed with private repo requirements.
- Default behaviour should cover 99% of use cases: PR validation, release drafting, and NuGet publishing
- All jobs run on ubuntu-latest
- Only private repos will have access to organisation secrets and variables.
