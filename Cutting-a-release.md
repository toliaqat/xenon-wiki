### Prerequisites

* Create a key pair, publish it to a key server (see [working with PGP signatures][working-with-pgp-signatures])
* Get credentials to Sonatype OSS Repository Hosting
* Observe [Semantic Versioning][semver]

[working-with-pgp-signatures]: https://central.sonatype.org/pages/working-with-pgp-signatures.html
[semver]: http://semver.org/

#### Notes

**Q**: Why not use the Maven Release Plugin?

**A**: It doesn't work well with Gerrit, since it can't directly push release commits to master. If you can find a way for it to work that improves on the process below, please modify accordingly.

### Marking a release

* Make sure the CHANGELOG is up to date
* Run `mvn versions:set -DnewVersion=0.3.0` (insert the new version here)
* Commit `Mark 0.3.0 release`
* Run `mvn versions:set -DnewVersion=0.3.1-SNAPSHOT`
* Commit `Mark 0.3.1-SNAPSHOT for development`
* Push commits to Gerrit
* Wait for +1/+2 for **both** commits
* Merge them at the **same time** (so no other commits can be interleaved)

### Deploying a release

**After** both commits have been merged you know the bits you're about to release won't change.

Checkout a new branch for the `0.3.0` release:

```
git checkout -b v0.3.0 <commit sha1>
```

Make sure you have the GPG agent running so you don't have to repeatedly enter your key's passphrase:

```
eval $(gpg-agent)
```

Make sure Maven can find credentials to Sonatype OSSRH (see `~/.m2/settings.xml`):

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
mvn clean deploy -P release
```

Done!