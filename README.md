# Runner Container Hooks for Podman

The GitHub Actions runner has a feature called Container Hooks that let you customise what actually happens when a workflow requests some work run in a container. The default behaviour without hooks is to run `docker`.

This repo is based on the [official container hooks repo](https://github.com/actions/runner-container-hooks) which has implementations for Kubernetes and Docker. The Docker one has been customised relatively lightly here to instead call Podman, and makes allowances for differences in Podman behaviour.

These hooks will run `podman` as the runner user, so it assumes a rootless setup. Generally this is best especially for a persistent server.

> ⚠️ These hooks seem to work fine for us but we haven't tested every action and every server. Some issues may be related to your specific Podman configuration. If there's a general bug, we may look at fixing it. If there is some extra behaviour that you need for your use case, please raise a PR.

## Summary of changes made for Podman

* The command being run is now `podman`, naturally
* Passing environment variables for the container through to `podman` was confusing it, especially for variables like `HOME`, so the values of those are now passed through in the arguments.
* Podman will not create missing volume sources, but the runners assumes this behaviour, so we will create it if required.
* If a volume mount is detected where the source is /var/lib/docker.sock, we will translate that to be a Podman socket so that Docker-in-Podman will possibly work. It's recommended to use `podman` directly, though.
* Volume mounts often need `:z` to be appended so that the correct SELinux labels are added to files. Without this, the container generally can't access a host mount at all.

## Other allowances you may need to make

These don't relate to the hook itself but may affect how you set up the system containing your runner compared to running Docker. Some of these aren't even to do with running jobs as Podman containers but simply running any containers using Podman instead of Docker.

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