---
layout: documentation
---

# Update Guide

Learn all about how to update your OpenDevStack repositories and the running
installation of it.

## How to update your OpenDevStack repositories

Updating repositories means that new refs from repositories under
`github.com/opendevstack` are pushed into the repositories in your BitBucket
instance. Note that only updates to the `production` branch in BitBucket will
have any effect on the OpenDevStack installation.

First, you need a clone of each repository in BitBucket which should be updated
on your local machine.

Once this has been done, you need to fetch new refs from
`github.com/opendevstack`. To do so, add a remote pointing to it like this:

```sh
git remote add ods https://github.com/opendevstack/<REPO_NAME>.git
```

Now you are ready to update the refs. It is recommended to update both the
`master` branch and, unless you want to live off the bleeding edge, a release
branch such as `1.0.x`. Use the steps shown below:

```sh
# Ensure you have the latest refs from ODS locally
git fetch ods
# Update master
git checkout master
git reset --hard ods/master
git push origin master
# Update 1.0.x
git checkout 1.0.x
git reset --hard ods/1.0.x
git push origin 1.0.x
```

So far, your OpenDevStack installation has not been affected yet because the
`production` branch has not been updated yet. To do that, it is recommended to
create a pull request on BitBucket from `1.0.x` into `production` This serves
as a gate to control changes coming in, and spread knowledge about what has
changed exactly.

Now that changes are "live", you need to update (parts of) the installation.


## How to update your OpenDevStack installation

Updating consists of two parts: following the general update procedure
(applicable to all version updates) and a version specific update procedure.


### General update procedure

### Backup

Before proceeding, it is advisable to make a backup of the existing OpenShift
configuration. This can be done easily with Tailor:

```sh
# backup CD project
tailor export -n cd > backup_CD.yml

# backup provision app projects
tailor export -n prov-cd > backup_PROV_CD.yml
tailor export -n prov-dev > backup_PROV_DEV.yml
tailor export -n prov-test > backup_PROV_TEST.yml
```

Note that the executing user needs to have permissions to access all resources
in the `cd` namespaces for this to work properly.

### Tailor

Next, update Tailor to the version corresponding to your new OpenDevStack
version. At the moment, this version is noted at the start of each version specific
update procedure.

### Configuration

Then, update the configuration parameters (located in `ods-configuration`)
according to what has changed in the `ods-configuration-sample` repository.

### Templates

Once configuration is up-to-date, the OpenShift templates stored within
OpenShift need to be updated:

```sh
# Within your local "ods-project-quickstarters" repository
cd ocp-templates/scripts
./upload-templates.sh
```

### OCP resources

Next, run `tailor update` in all `ocp-config` folders to bring OCP resources (such
as DC or BC) into sync. Review the diff produced by Tailor carefully, especially around
changes to PVCs. To find each `ocp-config` folder, you can use e.g.
`find . -name ocp-config -not -path "./ods-configuration*"` on Unix.

### Shared images

After all OCP resources have been updated, you need to start a build for all build configs
in the `cd` namespace to create new images.

### Provisioning App

Also, the provisioning app should be updated. To do that, run `tailor update`
in each `ocp-config` folder, and then trigger a build in Jenkins to redeploy the
service.

### Shared library

Finally, push the vanilla branch or tag of the shared library to your
BitBucket instance. E.g. if your `production` branch is based on `1.0.x`, push
`ods-jenkins-shared-library@1.0.x`. All quickstarters in the `1.0.x` branch point to this Git
reference in the `Jenkinsfile`. It is not recommended to point to `production` in your quickstarters,
otherwise consumers will encounter failing builds when breaking changes are introduced in the
`production` branch. You may also choose to create a custom tag from your `production` branch and
make all `Jenkinsfile`s point to this tag.

Now that the general procedure has been completed, you need to apply all the
update notes below which apply to your version change.

### 0.1.0 to 1.0.x

1.0.x requires Tailor [0.9.3](https://github.com/opendevstack/tailor/releases/tag/v0.9.3).

#### Update `xyz-cd` projects

There is a new webhook proxy now, which proxies webhooks sent from BitBucket to
Jenkins. As well as proxying, this service creates and deletes pipelines on the
fly, allowing to have one pipeline per branch. To update:
* Setup the image in the `cd` project by running `tailor update` in
  `ods-core/jenkins/webhook-proxy/ocp-config`.
* Build the image.
* Setup the  webhook proxy next to each Jenkins instance. E.g., go to
  `ods-project-quickstarters/ocp-templates/templates` and run
  `oc process cd//cd-jenkins-webhook-proxy | oc create -f- -n xyz-cd`. Repeat for
  each project.


#### Update components

For each component, follow the following steps:

In `Jenkinsfile`:
1. Set the shared library version to `1.0.x`.
2. Replace `stageUpdateOpenshiftBuild` with `stageStartOpenshiftBuild`.
3. Remove `stageCreateOpenshiftEnvironment` and `stageTriggerAllBuilds`.
4. Adapt the build logic to match the latest state of the quickstarter
   boilerplates.
5. Remove `verbose: true` config (replace with `debug: true` if you want debug
   output).
6. Configure `branchToEnvironmentMapping`, see README.md. If you used
   environment cloning, also apply the instructions for that.

In `docker/Dockerfile`:

Adapt the content to match the latest state of the quickstarter boilerplates.
In BitBucket, remove the existing "Post Webhooks" and create a new "Webhook",
pointing to the new webhook proxy. The URL has to be of the form
`https://webhook-proxy-$PROJECT_ID-cd.$DOMAIN?trigger_secret=$SECRET`. As
events, select "Repository Push" and "Pull request Merged + Declined".


#### Update provisioning app

If you want to build the provisioning app automatically when commits are pushed
to BitBucket, add a webhook as described in the previous section.

#### Fix Jenkins master BUILD_URL

1.0.x makes use of the `BUILD_URL` env variable automatically set by Jenkins. This
env variable might be `null` in your Jenkins master. To fix this, copy
https://github.com/opendevstack/ods-core/blob/1.0.x/jenkins/master/configuration/init.groovy.d/url.groovy into each Jenins master to `/var/lib/jenkins/init.groovy.d/url.groovy`.
