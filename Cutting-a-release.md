* ### Prerequisites

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

* Create a new branch in Gerrit, v0.3.0, using the hash from the corresponding commit for the release.
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

Perform the release:

```
$ mvn clean deploy -P release
```

Release the deployment to maven central by following [these instructions](http://central.sonatype.org/pages/releasing-the-deployment.html). You will do two steps: Closing the Staging repositories, and making the release public.

Done!