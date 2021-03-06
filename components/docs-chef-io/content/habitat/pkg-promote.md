+++
title = "Promote Packages"
description = "Best practices for promoting packages in channels for building and testing code changes as part of continuous deployment with channel tags"

[menu]
  [menu.habitat]
    title = "Promoting Packages"
    identifier = "habitat/packages/promote Promoting Packages"
    parent = "habitat/packages"
    weight = 20

+++

Continuous deployment is a well-known software development practice of building and testing code changes in preparation for a release to a production environment.

## Continuous Deployment Using Channels

Chef Habitat supports continuous deployment workflows through the use of channels. A channel is a tag for a package that the Supervisors in a service group can subscribe to. Channels are useful in CI/CD scenarios where you want to gate a package before making it the default version of the package that users should consume. You can think of this split as the difference between test and production, or nightly releases versus stable releases of products.

By default, every new package is placed in the `unstable` channel by Builder. Packages in the `unstable` channel cannot be started or installed unless you specify the `--channel` flag in the `hab` CLI, or set the `HAB_BLDR_CHANNEL` environment variable to a non-stable channel. This is because the default channel used by the `hab` CLI when starting, installing, or loading packages is the `stable` channel. The `stable` channel indicates a level of stability and functionality suitable for use in multi-service applications or as a dependency for your service.

To promote your package to a channel, you must either use Builder to build your package or upload it to Builder yourself, and then use the `hab pkg promote` subcommand to promote the package to the intended channel. To combine operations, the `hab` CLI allows you to do both in one command by using the `--channel` option when uploading your package. The following shows how to upload and promote a package to a custom channel named `test`.

```bash
$ hab pkg upload -z <TOKEN> results/<hart file> --channel test
```

In the example above, if you look up your package in the Builder UI, or using the `hab pkg channels` subcommand, you can see that your package is tagged for both the `test` and `unstable` channels.

> **Note** Custom channels like `test` are scoped to each package. Builder does not create channels scoped to an origin, so if you want to use custom channels for future releases of a package, you must promote to those channels for each release.

If you have already uploaded your package to a channel and wish to promote it to a different channel, use the `hab pkg promote` subcommand as shown below.

```bash
$ hab pkg promote -z <TOKEN> <origin>/<package>/<version>/<release> stable
```

### Combining an Update Strategy with Channels

By using both channels and either the `at-once` or `rolling` [update strategies](#using-updates), you can automatically update packages in a given channel as shown below:

![Promoting packages through channels](/images/infographics/habitat-promote-packages-through-channels.png)

Using the infographic as a guide, a typical continuous deployment scenario would be as follows:

1. You build a new version of your package either through Builder or through a local Studio environment and then upload it to Builder.
2. When you are ready to roll out a new version of the package, you promote that package to the channel corresponding to the intended environment (e.g. `dev`). You can have multiple service groups in the same environment pointing to different channels, or the same channel.
3. An existing set of running Supervisors in that service group see that the channel they are subscribed to has an update, so they update their underlying Chef Habitat package, coordinating with one another per their update strategy, and restart their service.
4. When you are ready, you then promote that version of your package to a new channel (e.g. `acceptance`). The running Supervisors in that group see the update and perform the same service update as in step 3. You repeat steps 3 and 4 until your package makes its way through your continuous deployment pipeline.

Configuring a Supervisor's update strategy to point to a channel ensures that new versions of the application do not get deployed until the channel is updated, thereby preventing unstable versions from reaching environments for which they are not intended.

To start a service with an update strategy and pointing to a channel, specify them as options when loading the service.

    $ hab svc load <origin>/<package> --strategy rolling --channel test

While that service is running, update your package, rebuild it, and then promote it to the same channel that the previous release of that service is currently running in (e.g. `test`). Those running instances should now update according to their update strategy.

### Demoting a Package from a Channel

If you need to un-associate a channel from a specific package release, you can do so using the `hab pkg demote` subcommand. Packages can be demoted from all channels except `unstable`.

```bash
$ hab pkg demote -z <TOKEN> <origin>/<package>/<version>/<release> test
```

The Builder UI for that package release and `hab pkg channels` will both reflect the removal of that channel.

If you are running a package in a specific channel and demote

> **Note** If you demote a package from the `stable` channel, then any other packages that depend on the stable version of that package will either fail to load or build depending on how that demoted packaged was used.

> Also, downgrading to another release for a specific channel is not available at this time. This means if you run the latest release of a package from a specific channel, and demote the channel for that release, the package will not downgrade to the next most recent release from that channel.
