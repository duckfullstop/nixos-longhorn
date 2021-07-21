# Longhorn on NixOS

These dockerfiles build some super hacky containers based on Longhorn v1.1.2 that should run on NixOS.

## Dear god why

Longhorn uses `nsenter` with a privileged Pod to access host resources and do storage / mounting stuff.
`nsenter` directly calls binaries based on the path of the calling process - because NixOS uses extremely non-standard paths, things like `mount` aren't available on the standard path of these containers.
This results in errors like `Failed to execute: nsenter [–mount=/host/proc/*/ns/mnt mount], output nsenter: failed to execute mount: No such file or directory` in the logs of the longhorn-manager pod during bringup.

### The Fix

These derived containers simply take the existing image and append the necessary Nix-specific paths so that nsenter works as expected. (Take a look for yourself)

### The Longer-Term Fix

~~I'll be submitting an issue to Longhorn in due course to bring this problem up~~ - my suggestion would probably be some kind of NixOS-specific workaround that could be enabled in the manifest where needed.

I've raised an issue upstream to discuss this problem: https://github.com/longhorn/longhorn/issues/2166

## Usage

Please don't.

### Usage if you really, really have to

First, ensure that `openiscsi` is included in your current configuration - use something like this:

```nix
config.environment.systemPackages = with pkgs; [
  openiscsi
];
```

Also, ensure that your `apiserver` is running with permission to create privileged pods (set `config.services.kubernetes.apiserver.allowPrivileged = true;`)

Once you've done that, change the images in the `longhorn.yaml` manifest to point to the ones built by this repository.

In the spec of the `longhorn-manager` daemonset:

```yaml
containers:
  - name: longhorn-manager
    image: ghcr.io/duckfullstop/nixos-longhorn-manager:v1.1.2
    command:
      - longhorn-manager
      - ...
      - --instance-manager-image
      - ghcr.io/duckfullstop/nixos-longhorn-instance-manager:v1_20210621
      - ...
      - --manager-image
      - ghcr.io/duckfullstop/nixos-longhorn-manager:v1.1.2
```

In the spec of the `longhorn-driver-deployment` daemonset (probably not necessary, but you might as well while you're here):
```yaml
initContainers:
  - name: wait-longhorn-manager
    image: ghcr.io/duckfullstop/nixos-longhorn-manager:v1.1.2
...
containers:
        - name: longhorn-driver-deployer
          # NixOS specific: oh dear god why
          image: ghcr.io/duckfullstop/nixos-longhorn-manager:v1.1.2
          ...
          command:
            - longhorn-manager
            - -d
            ...
            - ghcr.io/duckfullstop/nixos-longhorn-manager:v1.1.2
```

You should now be able to apply the manifest and continue with Longhorn setup as normal (in theory).

If you have any issues, please feel free to raise one, but please remember that this is **hacky intentionally** and there is **absolutely no guarantee it will work on your system**.
