+++
title = "Use AutoPkg to package GitHub repositories with no binary assets"
date = 2023-11-06
description = "How to package GitHub repositories with AutoPkg, even when no binary assets are present."
path = "autopkg-github-no-assets"
[extra]
author = "Matthew Warren"
github_discussion = "https://github.com/haircut/macblog/discussions/25"
+++

GitHub is a popular channel for software distribution, and AutoPkg simplifies the task of packaging GitHub-released apps.
AutoPkg's `GitHubReleasesInfoProvider` processor is the standard method for identifying the latest version of software released using GitHub's [Releases][releases] feature, but it has one key requirement - a binary asset must be attached to the release.

However, not all software adheres to this convention.
Some repositories might only contain simple shell scripts or other non-binary assets. 

To work around this requirement of `GitHubReleasesInfoProvider`, you can use AutoPkg's `URLTextSearcher` and `URLDownloader` processors to download the release in the form of a zip archive.

<!-- more -->

Every GitHub release automatically provides compressed archives of repository contents in both zip and tar formats.
You can access the URLs of these archives via GitHub's REST API at the following endpoint:

```plaintext
https://github.com/api/v3/repos/{OWNER}/{REPOSITORY}/releases/latest
```

The JSON response contains a key called "zipball_url" which points to the zip archive:

```plaintext
{
  ...
  "zipball_url": "https://api.github.com/repos/{OWNER}/{REPOSITORY}/zipball/v1.0.0"
  ...
}
```

Extracting this URL is straighforward using the `URLTextSearcher` processor and the regular expression `zipball_url\": \"(.*)\"`.
Once extracted, you can download the archive using the URLDownloader processor.

With an archive of the repository downloaded, you can proceed with additional processors as needed to package the repository contents.

## Example

Consider an example repository that contains a shell script named `example.sh`, and your goal is to create an installer package that places an executable copy of this script in the default path.
Below is a complete example recipe process:

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
        version: "1.0.0"
        chown:
          - path: usr
            user: root
            group: admin
```

First, the recipe will locate the latest release of the target repository and its associated `zipball_url` using `URLTextSearcher`.
Next, it will download the archive using `URLDownloader`.

Following that, the recipe will extract the archive and create a root directory for building an installer package.
It will then copy the target shell script `example.sh` from the extracted repository contents to the package root directory at the required location (e.g., `/usr/local/bin/example`). 
Finally, a `postinstall` script is created to ensure that the file is executable after installation, and everything is packaged into a `.pkg`.


## Authentication for private repos

This recipe pattern is also handy for accessing releases without assets in private repositories. 
To do this, you'll need to create a [personal access token][pat] and set the `GITHUB_TOKEN` environment variable [as described here][patvar].
Then, simply add an `Authorization` header using Bearer authentication to the `URLTextSearcher` and `URLDownloader` processors as follows:


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

This approach keeps your options open for packaging a variety of GitHub releases, whether they include binary assets or not.

[releases]: https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases
[pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
[patvar]: https://github.com/autopkg/autopkg/wiki/FAQ#how-do-i-provide-a-github-personal-access-token-to-autopkg
