# python.build - Mirror and Builder for Python Wheels

[python.build](https://python.build) is a builder and mirror for Python packages that don't have [wheels](https://packaging.python.org/en/latest/guides/distributing-packages-using-setuptools/#wheels) for platforms such as Apple Silicon (aarch64/ARM64, if you have an M1/M2 processor) or Linux ARM64 (AWS Graviton, etc), and to also back-build wheels for older Python and package versions.

With the release of Python 3.11, a lot more packages now support these platforms, but there are still a lot of packages that don't; or have older versions that popular projects have pinned. For example, Dagster relies on grpcio, and [it's not a great install experience on Apple Silicon](https://github.com/grpc/grpc/issues/25082).

So this project hopes to provide reproducible builds using Github Actions, and then provide a mirror for them on [python.build](https://python.build) as well as on the [releases](https://github.com/kilnbuild/python.build/releases).

## Installation / Using python.build
### Poetry
Using Poetry is a straight forward experience.

You have a couple of options, either adding `python.build` as a source to your `pyproject.toml` file,
or just using the wheel directly as a vendored dependency.

#### Option 1: Adding python.build as a source repository

Add the following to the bottom of your `pyproject.toml`:

```toml
[[tool.poetry.source]]
name = "pythonbuild"
url = "https://python.build/simple/"
default = false
secondary = false
```

Now when you define a dependency, you can use the `pythonbuild` source, falling back to PyPI for other platforms:

```toml
grpcio = [
    {version="^1.47.2", markers="sys_platform == 'darwin' and platform_machine=='arm64'", source="pythonbuild"},
    {version="^1.47.2", markers="sys_platform != 'darwin' and platform_machine!='arm64'", source="pypi"}
]
```

Then running `poetry update grpcio` will update `poetry.lock` with the correct version for your platform.

See [Poetry's documentation on environment markers](https://python-poetry.org/docs/dependency-specification/#using-environment-markers) for more information about this feature.

python.build only [builds and mirrors certain packages](./PACKAGES.md) (thanks to [bandersnatch](https://github.com/pypa/bandersnatch)), and packages are mirrored from PyPI every hour. However our primary goal is to provide wheels for platforms that don't have them.

You can also just use python.build as your primary source, which is a little simpler, but you should prefer the method above.

```toml
grpcio = { version="^1.47.2", source="pythonbuild" }
```

This should download the correct wheel for your platform, or if it doesn't exist, it will fall back to the default.

Then running `poetry update grpcio` will update `poetry.lock` with the correct version for your platform.

#### Option 2: Vendored dependencies
1. Download and copy the `.whl file into your project (for example `vendored/grpcio-1.47.2-cp310-cp310-macosx_11_0_arm64.whl`), then edit the dependency to point to it:

```toml
grpcio = [
    {markers="sys_platform == 'darwin' and platform_machine=='arm64'", path='vendored/grpcio-1.47.2-cp310-cp310-macosx_11_0_arm64.whl'},
    {version="^1.47.2", markers="sys_platform != 'darwin' and platform_machine!='arm64'", source="pypi"}
]
```

This way you can use the vendored wheel on Apple Silicon. For other platforms such as Linux, x86 Macs, etc, it will fallback normally to using the package from PyPI. You will need to commit the wheel file to your repository if you want to share it with your team.

You can also use the `url` option to point to the `python.build` URL, this will download the wheel when you use poetry, but with the benefit of not committing the wheel to your repository.

```toml
grpcio = [
    {markers="sys_platform == 'darwin' and platform_machine=='arm64'", url='https://python.build/wheels/grpcio-1.47.2-cp310-cp310-macosx_11_0_arm64.whl'},
    {version="^1.47.2", markers="sys_platform != 'darwin' and platform_machine!='arm64'", source="pypi"}
]
```

## How it works
Packages are built using Github Actions, using [cibuildwheel](https://github.com/pypa/cibuildwheel) to build the wheels for the following platforms:
- macOS x86_64
- macOS arm64
- Linux x86_64
- Linux arm64

The MacOS and Linux runners are currently running as [self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) on dedicated machines; because it's much faster to build these wheels on a dedicated machine compared to Hosted Github Actions. For example, a build of grpcio takes around 70 seconds compared to 20-25 minutes using Github Actions hosted.

Github Actions also has [a limit](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) for the amount of minutes you can use per month depending on your plan. So this is a way to get around that.

The following operating system versions are used:
- MacOS Big Sur 11.7.2 (20G1020) x86_64
- ArchLinux (rolling release) x86_64

MacOS Big Sur is used to ensure that compatibility, as wheels built for MacOS 11.x work on all later versions of MacOS (such as Monterey and Ventura). [cibuildwheel](https://github.com/pypa/cibuildwheel) allows us to build both x86_64 and arm64 on the x86_64 architecture.

For Linux builds, ArchLinux is simply used as the host and cibuildwheel takes care of the actual building for the platforms/architectures. We try to target as many platforms and architectures as possible.

## Build script
To make it as simple as possible to maintain (and for others to use!), [python.build] uses Github Actions with `workflow_dispatch`, which allows calling the workflow with the following parameters:

```
package_name: The package to build, this is the name of the package on PyPI
package_version: The package version to build, this is the version of the package on PyPI
package_url: Source URL for the package, this is the URL of the .tar.gz file on PyPI.
python_version: The Python version to use for the build. ["3.7", "3.8", "3.9", "3.10", "3.11"]
environment_variables: Environment variables to set for the build in CIBW_ENVIRONMENT_XX. Should be a string like "FOO=bar BAZ=qux". Handy for grpcio. Optional.
```

If you want to build a package, you can use [Github's CLI](https://github.com/cli/cli) to run the following command:
```bash
gh workflow run mac_arm_build.yml -f package_name=grpcio -f package_version=x ... # rest of the parameters
```

We use the Github API to queue packages for building.

## Packages
Please see [PACKAGES](./PACKAGES.md) for a list of packages that are currently being built.

If you would like to see a package added, please open an issue here (and please also contribute building for that architecture to the package! <3).

## Hosting
Using [Wasabi](https://wasabi.com) to host the wheels as a static site, which is a memebr of the [Bandwidth Alliance](https://www.cloudflare.com/bandwidth-alliance/), which allows us to save on bandwidth costs. We then use Cloudflare as the CDN.

It's planned to provide multiple mirror options, so those who can't use Cloudflare can use a different mirror (for example those in China).

You can also download the wheel directly from either the [python.build](https://python.build) website, or the [releases page](https://github.com/kilnbuild/python.build/releases). Note that Github Actions only stores logs and artifacts for a limited amount of time, but we provide a full audit log of all builds, and you can always use the workflow files to build the wheels yourself.

## Auditing
The goal is to ensure all built wheels have a full auditable build history and should be as reproducible as possible. You should be able to run the Github Action on your own repository and get the same result (unless OpenSSL changes, Python changes... Nix in future maybe? :-)).

You can see the audit logs on the [audit branch](https://github.com/kilnbuild/python.build/tree/audit), which has a log of built packages from Github Actions, as Github Actions doesn't store this indefinitely (it is currently a [maximum of 90 days](https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization)). You can also see a log of the packages built on the [Github Actions page](https://github.com/kilnbuild/python.build).

You can also fork the repository and just change the runner to Github Actions, which should work fine. Just be aware that Github has limits on the amount of minutes you can use per month, so please check before running.

## Who is behind this?
This was started by [Josh Taylor](https://github.com/joshuataylor), having been annoyed at the wheel situation with a new M1 Macbook Air. It's a fantastic laptop (as someone who has been using, and will continue to use, Linux as their primary desktop since 2006). So I wanted to try and help out with the situation and have a fun sideproject to work on that helps the community.

Kiln is just a silly name I came up with for another project, and it kind of fits, as it's kind of like baking packages, I guess? :-) 🐍🧱

## Inspiration
Thanks to [pietrodn](https://github.com/pietrodn), who created a [Github Action](https://github.com/pietrodn/grpcio-mac-arm-build)
for grpcio. This gave me the idea to create a generic repository that also mirrored it, so people can either download
the wheels from the mirror in their `pyproject.toml` or vendor it themselves.

## License
The code and documentation (basically anything in this repository that isn't an actual wheel) are licensed under the MIT license. If there is a good reason to change the license (for example dual licensing Apache 2.0/MIT) please start a discussion. I want this project to be as open as possible and allow as many people to use it as possible without any restrictions.

The built wheels and artifacts are subject to the license of the package itself. Please refer to the license of the package for more information. I will try to ensure that a link to the package license is included on the package page and audit log.

## Python Trademark
Python® is a registered trademark of the Python Software Foundation.

Please see the [PSF trademark documentation](https://www.python.org/psf/trademarks/) for additional information.

I believe this project falls under fair use, due to being opensource, and not being used for commercial purposes. If this assumption is incorrect, please let me know via discussion or emailing me at `josh@python.build`.

> Use of the word "Python" in the names of freely distributed products like IronPython, wxPython, Python Extensions, etc. -- Allowed when referring to use with or suitability for the Python programming language. For commercial products, contact the PSF for permission.