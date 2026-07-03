# Runner Container Hooks for Podman

The GitHub Actions runner has a feature called Container Hooks that let you customise what actually happens when a workflow requests some work run in a container. The default behaviour without hooks is to run `docker`.

This repo is based on the [official container hooks repo](https://github.com/actions/runner-container-hooks) which has implementations for Kubernetes and Docker. The Docker one has been customised relatively lightly here to instead call Podman, and makes allowances for differences in Podman behaviour.

These hooks will run `podman` as the runner user, so it assumes a rootless setup. Generally this is best especially for a persistent server.

> ⚠️ These hooks seem to work fine for us but we haven't tested every action and every server. Some issues may be related to your specific Podman configuration. If there's a general bug, we may look at fixing it. If there is some extra behaviour that you need for your use case, please raise a PR.

## Summary of changes made for Podman

* The command being run is now `podman`, naturally
* Environment variables were being passed through by passing them into the executable and then listing the keys in the `-e` flags. This was confusing `podman` because it was interpreting the incorrect `HOME` and `PATH`, so the values of those are now passed through in the `-e` arguments.
* Podman will not create nonexistent volume sources, but the runners assumes this behaviour, so we will create it if required.
* If a volume mount is detected where the source is /var/lib/docker.sock, we will translate that to be the user-scoped Podman socket so that Docker-in-Podman should work. It's recommended to use `podman` directly, though.
* SELinux labelling is disabled, which is the second-least secure option but as the runner mounts host directories so extensively, it's probably the only reasonable one that doesn't get into complicated custom labels. Docker doesn't label at all so it's no less secure than using Docker (aside from the fact that you don't have to run as root)

## Other allowances you may need to make

These don't relate to the hook itself but may affect how you set up the system containing your runner compared to running Docker. Some of these are just generally related to using rootless Podman instead of rootful Docker.

* Add config to `/etc/containers/registries.conf.d` to teach it how to resolve short names, as it won't always assume that the registry is `docker.io` and will fail rather than guess. Alternatively, stop using short names.
* Podman may re-exec itself with a cut-down PATH and be unable to find tools such as `pasta`. Configure `~/.config/containers/containers.conf` to include locations for all your networking helpers. e.g.:

      [engine]
      helper_binaries_dir = [
        "/usr/libexec/podman",
        "/usr/bin"
      ]

## Building

```bash
npm run bootstrap
npm run build-all
```

This should create a file at `packages/podman/dist/index.js` which you can then pass to your runner.

## Usage through NPM

Install `@universityofwarwick/github-runner-podman-hooks` somewhere and update the `.env` file in your `actions-runner` directory to include:

```
ACTIONS_RUNNER_CONTAINER_HOOKS=/your/path/to/module/packages/podman/dist/index.js
```