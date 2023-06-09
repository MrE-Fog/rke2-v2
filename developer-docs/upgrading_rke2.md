# Upgrade RKE2 Process

The steps which the user should follow to automatically upgrade rke2 are very well described in the [docs](https://docs.rke2.io/upgrade/automated_upgrade/). This documents goes a bit deeper into what happens behind the scenes.

## System upgrade controller

As described in the upper link, a system-upgrade-controller is used to upgrade the cluster. The upgrade procedure can be adjusted based on the configuration plan, which is fed to the system using a CR. The controller will read that CR and create a job that will execute the upgrade. Normally, the job is using the image `rancher/rke2-upgrade`, which is generated by a project with the same name.

## rke2-upgrade project

The [rke2-upgrade project](https://github.com/rancher/rke2-upgrade) basically generates an image with three important pieces:
1 - The new rke2 binary based on the version set in the plan
2 - The new kubectl binary based on the version set in the plan
3 - The upgrade.sh script

That script replaces the rke2 binary and kills the current rke2 process. That way, systemd (or others) will restart the service now using the correct upgraded rke2 binary.

## rke2 binary restarting

The rke2 binary will do as it does on every startup of rke2. It extracts the manifests of the runtime image into the /var/lib/rancher/rke2/server/manifests dir, replacing the existing ones coming from an old rke2 version, with the new ones. The [k3s deploy controller](https://github.com/k3s-io/k3s/blob/master/pkg/deploy/controller.go) will loop over them and apply them to the cluster creating an Addon resource. These will happen to all manifests except for:

* Manifests that refer to a explicitly disabled component (e.g. via the --disable flag in the cli). In this case, the controller tries to uninstall it and removes the manifest
* Manifests that refer to a component that should be skipped. For a component to be skipped, we just need to add the suffix `.skip` to the file name of the manifest (e.g. rke2-canal.yaml.skip). In this case, the controller will ignore this component and will not try to upgrade it. If you are running several control-plane nodes, remember to add the `.skip` suffix to the file in all control-plane nodes.

The skips procedure can be very useful for cases such as a user switching to the enterprise version of a cni plugin. In this case, the cni plugin vendor will take care of upgrades.
