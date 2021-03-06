sonatype-ossrh-parent
===============
Parent pom.xml to ease deployment to Maven Central

Getting started
----------------
Add this parent to your pom.xml:

```xml
<parent>
	<groupId>com.github.chhh</groupId>
    <artifactId>sonatype-ossrh-parent</artifactId>
    <version>0.1</version>
</parent>
```

Releasing
----------

Once you have your settings.xml sorted and your gpg keys as per http://central.sonatype.org/pages/apache-maven.html, you can release in one command line to Maven Central:

```bash
mvn release:prepare && mvn release:perform
```

You will be prompted for versions.

My preference is to use this bash function in `.bashrc`:

```bash
function release() {
  RELEASE_VERSION=$1
  GPG_PASSPHRASE=$2
  mvn --batch-mode release:prepare \
    -Dtag=$RELEASE_VERSION \
    -DreleaseVersion=$RELEASE_VERSION \
    -DdevelopmentVersion=$RELEASE_VERSION.1 \
    -DautoVersionSubmodules=true \
    -Darguments=-Dgpg.passphrase=$GPG_PASSPHRASE && \
  mvn --batch-mode release:perform \
    -Darguments=-Dgpg.passphrase=$GPG_PASSPHRASE
}
```
 which is called like this:
 ```bash
 release 0.4 <GPG_PASSPHRASE>
 ```
