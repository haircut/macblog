+++
title = "Use AutoPkg to package GitHub repositories with no binary assets"
date = 2023-11-04
description = "How to package GitHub repositories with AutoPkg, even when no binary assets are present."
path = "autopkg-github-no-assets"
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/25"
+++

GitHub is a popular channel for software distribution, and AutoPkg simplifies the task of packaging GitHub-released apps.
AutoPkg's built-in `GitHubReleasesInfoProvider` processor is the standard method to detect the latest version of an app provided using GitHub's [Releases][releases] feature.
However, the processor expects a binary asset attached to the release.

Not all software follows this convention, such as a repository containing a simple shell script or non-binary assets.

To work around the limitation of `GitHubReleasesInfoProvider`, you can use AutoPkg's `URLTextSearcher` and `URLDownloader` processors to download the release in the form of a zip archive.

<!-- more -->

Every GitHub release automatically provides compressed archives of repository contents in both zip and tar formats.
You can access the URLs of these archives via GitHub's REST API at the following endpoint:

```plaintext
https://github.com/api/v3/repos/{OWNER}/{REPOSITORY}/releases/latest
```

Within the JSON response, you'll find a key called "zipball_url" that points to the zip archive:

```plaintext
{
  ...
  "zipball_url": "https://api.github.com/repos/{OWNER}/{REPOSITORY}/zipball/v1.0.0"
  ...
}
```

Extracting this URL is simple using the `URLTextSearcher` processor and the regular expression `zipball_url\": \"(.*)\"`. Then, download the archive using the `URLDownloader` processor.

With an archive of the repository downloaded, you can run any additional processors required to package the repository contents as you need.

## Example

Consider an example repository containing a shell script named `example.sh`. The desired outcome is to produce an installer package that places an executable copy of this script in the default path.
Here is a complete example recipe:

```yaml
Process:
  - Processor: URLTextSearcher
    Arguments:
      url: https://github.com/api/v3/repos/OWNER/REPOSITORY/releases/latest
      re_pattern: "zipball_url\": \"(.*)\""
      result_output_var_name: "zipball_url"
      request_headers:
        Accept: application/vnd.github+json
        X-GitHub-Api-Version: "2022-11-28"

  - Processor: URLDownloader
    Arguments:
      url: "%zipball_url%"
      filename: "%NAME%.zip"

  - Processor: Unarchiver
    Arguments:
      archive_path: "%RECIPE_CACHE_DIR%/downloads/%NAME%.zip"

  - Processor: PkgRootCreator
    Arguments:
      pkgdirs: {}
      pkgroot: "%RECIPE_CACHE_DIR%/scripts"

  - Processor: PkgRootCreator
    Arguments:
      pkgdirs:
        usr: "0755"
        usr/local: "0755"
        usr/local/bin: "0755"
      pkgroot: "%RECIPE_CACHE_DIR%/pkgroot"

  - Processor: Copier
    Arguments:
      destination_path: "%RECIPE_CACHE_DIR%/pkgroot/usr/local/bin/example"
      overwrite: true
      source_path: "%RECIPE_CACHE_DIR%/%NAME%/*/example.sh"

  - Processor: FileCreator
    Arguments:
      file_path: "%RECIPE_CACHE_DIR%/scripts/postinstall"
      file_mode: "0755"
      file_content: |
        #!/bin/bash
        /bin/chmod +x /usr/local/bin/example

  - Processor: PkgCreator
    Arguments:
      pkg_request:
        pkgname: "%NAME%"
        pkgdir: "%RECIPE_CACHE_DIR%"
        id: "org.macblog.githubexample"
        options: purge_ds_store
        scripts: scripts
        version: "%version%"
        chown:
          - path: usr
            user: root
            group: admin
```

First, the recipe finds the latest release of the target repository and its associated `zipball_url` using `URLTextSearcher`.
Then it downloads that archive using `URLDownloader`.

Next, the recipes extracts the archive and creates a root directory for building an installer package.
It copies the target shell script `example.sh` from the extracted repository contents to the package root directory in the required location (e.g. `/usr/local/bin/example`), then creates a `postinstall` script that ensures that file is executable once installed.

Finally, it packages everything up into a `.pkg`.

## Authentication for private repos

You can also use this recipe pattern to access releases without assets in private repositories.
You'll need to create a [personal access token][pat], then [set the `GITHUB_TOKEN` environment variable][patvar] using that personal access token.
Then, simply add an authentication header to the `URLTextSearcher` and `URLDownloader` processors like so:

```yaml
- Processor: URLTextSearcher
  Arguments:
    url: https://github.com/api/v3/repos/OWNER/REPOSITORY/releases/latest
    re_pattern: "zipball_url\": \"(.*)\""
    result_output_var_name: "zipball_url"
    request_headers:
      Accept: application/vnd.github+json
      X-GitHub-Api-Version: "2022-11-28"
      Authorization: "Bearer %GITHUB_TOKEN%"

- Processor: URLDownloader
  Arguments:
    url: "%zipball_url%"
    filename: "%NAME%.zip"
    request_headers:
      Authorization: "Bearer %GITHUB_TOKEN%"
```

This method keeps your options open for packaging a variety of GitHub releases, whether they include binary assets or not.

[releases]: https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases
[pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
[patvar]: https://github.com/autopkg/autopkg/wiki/FAQ#how-do-i-provide-a-github-personal-access-token-to-autopkg
