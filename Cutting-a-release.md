### Prerequisites

* Several of these steps are only available to VMware employees.
* Create a key pair, publish it to a key server (see [working with PGP signatures][working-with-pgp-signatures])
* Set up access to Sonatype OSS Repository Hosting: they manage [Maven Central](http://search.maven.org/):
  * Make an account at [Sonatype Issues](https://issues.sonatype.org/)
  * Get permission to push artifacts (e.g. through a ticket to Sonatype)
* Observe [Semantic Versioning][semver]

[working-with-pgp-signatures]: http://central.sonatype.org/pages/working-with-pgp-signatures.html
[semver]: http://semver.org/

#### Notes

**Q**: Why not use the Maven Release Plugin?

**A**: It doesn't work well with Gerrit, since it can't directly push release commits to master. If you can find a way for it to work that improves on the process below, please modify accordingly.

### Marking a release

(Update version numbers appropriately)

* Make sure the CHANGELOG is up to date
* Run `mvn versions:set -DnewVersion=0.3.0` (insert new version)
* Commit `Mark 0.3.0 release`
* Add a line to the CHANGELOG with the new version, e.g. 0.3.1.
* Run `mvn versions:set -DnewVersion=0.3.1-SNAPSHOT` (insert new snapshot version)
* Commit `Mark 0.3.1-SNAPSHOT for development`
* Push commits to Gerrit
* Wait for +1/+2 for **both** commits
* Merge them at the **same time** (so no other commits can be interleaved)

### Deploying a release

**After** both commits have been merged you know the bits you're about to release won't change.

* Create a new branch in Gerrit, v0.3.0, using the hash from the corresponding commit for the release (**note**: the commit may be different if Gerrit rebased it upon submitting, so always fetch new changes after submitting the commits!)
* Checkout the new branch: 

```
$ git checkout v0.3.0
```

Make sure you have the GPG agent running so you don't have to repeatedly enter your key's passphrase:

```
$ eval $(gpg-agent)
```
(Warning: the gpg-agent once interfered with a `git pull --rebase`. Turn it off as necessary)

Make sure Maven can find credentials to Sonatype OSSRH by modifying `~/.m2/settings.xml`

```xml
<settings>
  <servers>
    <server>
      <id>ossrh</id>
      <username>myself</username>
      <password>secret</password>
    </server>
  </servers>
</settings>
```

Double check you don't have a dirty tree:

```
$ ( [ -z "$(git status --porcelain)" ] && echo "OK" ) || echo "Dirty..."
OK
```

Double check you're actually releasing `0.3.0`:

```
$ git show --oneline HEAD | head -1
6bb595a Mark 0.3.0 release
```

Tell git about the release: (It will show up as a release on github, but it takes a little while):

```
git tag -am "Tagging v.0.3.0" v0.3.0-release
git push origin v0.3.0-release HEAD:refs/heads/v0.3.0
```

Perform the release:

```
$ mvn clean deploy -P release
```

(If you get an error like "Could not find artifact com.vmware.xenon:xenon-loader-test:jar:0.3.1 in central (https://repo.maven.apache.org/maven2)", then run "mvn install". This is a dependency failure that we'll fix soon.)

Release the deployment to maven central: [Full instructions](http://central.sonatype.org/pages/releasing-the-deployment.html) are available on the sonatype web site, but we'll summarize them here. 

1. First, login and navigate to staging repo at the bottom that starts with "comvmware":
1. Next, verify that the correct binaries have been installed:
1. Then click the Close button (at the top) and wait for it to finish. You can hit the Refresh button to update the progress of the close operation. It shouldn't take long.
1. Finally, click on the Release button:

At this point, you'll be able to see the release on the sonatype web site
  * [Xenon Release Artifacts on Sonatype](https://oss.sonatype.org/content/groups/public/com/vmware/xenon/)

It will take a while for it to make it to Maven Central, but they'll eventually be there too
  * [Xenon Release Artifacts on Maven Central](https://repo1.maven.org/maven2/com/vmware/xenon/)

Done!

Links:
* [Manage Sonatype Staging Repositories via Web UI](https://oss.sonatype.org/#stagingRepositories)
* [Deploy snapshot artifacts into repository](https://oss.sonatype.org/content/repositories/snapshots)
* [Download snapshot, release and staged artifacts from staging group](https://oss.sonatype.org/content/groups/staging)
* [Download snapshot and release artifacts from group](https://oss.sonatype.org/content/groups/public)
* [See the released artifacts on Maven Central](https://repo1.maven.org/maven2/com/vmware/xenon/)

