Repository Permissions Updater
==============================

About
-----

The Jenkins project hosts Maven artifacts such as core and plugin releases on [Artifactory](https://repo.jenkins-ci.org/).

Its permissions system is independent of GitHub's, and we limit which users (identified by the Jenkins LDAP account, same as wiki and JIRA) are allowed to upload which artifacts.

This repository contains both the definitions for Artifactory upload permissions in [YAML format](https://en.wikipedia.org/wiki/YAML), as well as the tool that synchronizes them to Artifactory.

**Note:** These permissions are specifically for _uploading_ artifacts to the Jenkins project's Maven repository. It is independent of GitHub repository permissions. You may have one without the other. Typically, you'll either have both, or just the GitHub repository access.

Requesting Permissions
----------------------

**Prerequisite**: You need to have logged in once to [Artifactory](https://repo.jenkins-ci.org/) with your Jenkins community account (this is the same as the account you would use to login to Jira) before you can be added to a permissions target.

To request upload permissions to an artifact (typically a plugin), [file a PR](https://help.github.com/articles/creating-a-pull-request/) editing the appropriate YAML file, and provide a reference that shows you have commit permissions, or have an existing committer to the plugin comment on your PR, approving it.
See [this page](https://jenkins.io/doc/developer/plugin-governance/managing-permissions/) for more information.

Managing Permissions
--------------------

The directory `permissions/` contains a set of files, one per plugin or artifact, that define the permissions for the respective artifacts. Files typically have a `component`, `plugin`, or `pom` prefix for organization purposes:

* `plugin` is used for Jenkins plugins.
* `pom` is used for parent POMs and everything else consisting of just a POM file.
* `component` is used for everything else, usually libraries.

These prefixes, like the rest of the file name, have no semantic meaning and just help in organizing these files. 

Each file contains the following in [YAML format](https://en.wikipedia.org/wiki/YAML):

- A `name` (typically mirrored in the file name), this is also the `artifactId` of the Maven artifact.
- A `github` field indicating the GitHub organization and repository which is expected to produce these artifacts.
- A set of paths, usually just one. These correspond to the full Maven coordinates (`groupId` and `artifactId`) used for the artifact. Since Jenkins plugins can change group IDs and are still considered the same artifact, multiple entries are possible.
- A set of user names (Jenkins community user accounts in LDAP, the same as used for wiki and JIRA) allowed to upload this artifact to Artifactory. This set can be empty, which means nobody is currently allowed to upload the plugin in question (except Artifactory admins). This can happen for plugins that haven't seen releases in several years, or permission cleanups.

Example file:

```yaml
---
name: "p4"
github: "jenkinsci/p4-plugin"
paths:
- "org/jenkins-ci/plugins/p4"
developers:
- "p4paul"
```

* `p4` (lines 2 and 5): `artifactId`
* `p4-plugin` (line 3): GitHub repository name
* `org/jenkins-ci` (line 5): `groupId` (with slashes replacing periods)
* `p4paul` (line 7): Jenkins community account user name

### Adding a new plugin

Create a new YAML file similar to existing files.

### Adding a new uploader to an existing plugin

Edit the `developers` list in the YAML file for the plugin.

### Deprecating a plugin

Remove the YAML file. The next synchronization will remove permissions for the plugin.

### Renaming a plugin

Rename and edit the existing permissions file, changing both `name` and the last `path` component.

### Changing a plugin's `groupId`

Change the `paths` to match the new Maven coordinates, or, if further uploads for the old coordinates are expected, add a new list entry.

Managing Security Process
-------------------------

The Jenkins project acts as a primary contact point for security researchers seeking to report security vulnerabilities in Jenkins and Jenkins plugins ([learn more](https://jenkins.io/security/)).

We generally assign reported issues to maintainers after a basic sanity check ([learn more](https://www.jenkins.io/security/for-maintainers/)).
Through additional metadata in the YAML file described above, you can define who else should be contacted in the event of a report being received and is authorized to make decisions regarding security updates, e.g. scheduling.
Add a section like the following to your plugin's YAML file:

```yaml
security:
  contacts:
    jira: some_user_name
    email: security@acme.org
```

Given the above example, we will primarily assign any security issue in Jira to `some_user_name` and send an email notification to `security@acme.org` to establish contact.
Regular maintainers are added to the issue as well to give visibility and allow participation/discussion.
This means that specifying a Jira security contact is only useful when it's an account not already listed as maintainer.
Either of `jira` and `email` is optional.

Please note that we generally reject email contacts due to the additional overhead in reaching out via email.
Unless you represent a large organization with dedicated security team that needs to be involved in the coordination of a release, but is not otherwise part of plugin maintenance, please refrain from requesting to be contacted via email.

Usage
-----

To see how to run this tool to synchronize Artifactory permission targets with the definitions in this repository, see `Jenkinsfile`.

The following Java system properties can be used to customize the tool's behavior:

* `dryRun` - Set to `true` to generate the API payloads without submitting them. No modifications will be executed.
* `development` - Set to `true` during tool development to ensure production data is not overridden. This will have the following effects:
  - Permissions are only granted to deploy to the `snapshots` repository (rather than both `snapshots` and `releases`)
  - A different, non-colliding set of prefixes (unless overridden, see below) is used.
* `definitionsDir` - Path to directory containing permission definitions YAML files, defaults to `./permissions`.
* `artifactoryUserNamesJsonListUrl` - URL to a list containing known Artifactory user names, any permissions assigned to a user not on that list will cause the tool to abort
* `artifactoryApiTempDir` - Path to directory (that will be created) where this tool stores Artifactory permissions API JSON payloads, defaults to `./json`.
* `artifactoryObjectPrefix` - Override the prefix for groups and permission targets managed (created, updated, removed) using the tool.
  If unspecified, the value will be `generatedv2-` by default, or `generateddev-` in _development mode_.
* `artifactoryUrl` - URL to Artifactory, defaults to `https://repo.jenkins-ci.org`
* `artifactoryTokenMinutesValid` - How long authentication tokens to Artifactory for CD enabled repos should be valid for, default `240` (4 hours).
  Regular script execution frequency needs to be aligned with this.
* `gitHubSecretNamePrefix` - Prefix for secrets sent to GitHub repos.
  If unspecified, the value will be `MAVEN_` by default, or `DEV_MAVEN_` in _development mode_.

It expected the following environment variables to be set:

- `ARTIFACTORY_USERNAME` - Admin user name for Artifactory
- `ARTIFACTORY_PASSWORD` - Corresponding admin password (or API key) for Artifactory admin user
- `GITHUB_USERNAME` - GitHub user name for a user with admin access to any CD enabled repos
- `GITHUB_TOKEN` - Corresponding token for the user with admin access to any CD enabled repos, [requires `repo` scope to create/update secrets](https://docs.github.com/en/free-pro-team@latest/rest/reference/actions#create-or-update-a-repository-secret)

### How It Works

The tool runs three steps in sequence:

1. Generate JSON payloads from YAML permission definition files.
2. Submit generated JSON payloads to Artifactory.
3. Remove all generated permission targets in Artifactory that have no corresponding generated JSON payload file.
