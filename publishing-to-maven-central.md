# What is sonatype and why is it needed?

Sonatype provides a staging repository, which performs validation and allows to push the builds that pass all checks to the Central repo. Without it, basically, you can't easily publish anything to the Central easily, unless you're an Apache project or similar.

# Getting started

Follow their getting started [guide](http://central.sonatype.org/pages/ossrh-guide.html) to set up the needed credentials. This should be easy - you create a JIRA account and you create a ticket in JIRA to claim your namespace (groupId in Maven terms). If you have a github account, for example, __[http://github.com/chhh](http://github.com/chhh)__, you'll want to claim **com.github.chhh**.

## GPG signing
You'll need to set up and publish your GPG key for signing artifacts. This is described [here](http://central.sonatype.org/pages/working-with-pgp-signatures.html).  
In short you'll need to install gpg or gpg2. I did it on Windows and already had a working gpg that came with git installation. So I happily used that to generate my key with (create it with a passphrase!):
`gpg --gen-key`  
__Make sure to check__ that the generated key does not have sub keys for signing. First issue `gpg --list-keys`, the output should be like:
```
$ gpg --list-keys
/c/Users/<username>/.gnupg/pubring.gpg
---------------------------------
pub   2048R/DA123C12 2012-01-24
uid                  Dmitry Avtonomov (chhh) <email@gmail.com>
sub   2048R/3E123123 2012-01-24
```
Notice, that there is one _pub_ key and one _sub_ key. You want this _sub_ key to not be _Usage: C_ type.
`gpg --edit-key DA123C12`
```
gpg (GnuPG) 1.4.20; Copyright (C) 2015 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

pub  2048R/DA100C23  created: 2012-01-24  expires: never       usage: SC
                     trust: ultimate      validity: ultimate
sub  2048R/3E123123  created: 2012-01-24  expires: never       usage: E
[ultimate] (1). Dmitry Avtonomov (chhh) <email@gmail.com>
```
In this case the _sub_ key is _usage: E_, it's used for encryption only, so we're good to go, otherwise you'd need to delete or revoke it.
Published the key with:
`gpg --keyserver hkp://pool.sks-keyservers.net --send-keys <key-id>`  

### Windows caveat
The previous steps created the keychain file in `c:/Users/<username>/.gnupg`. However, when I later installed the native windows gpg from [https://www.gnupg.org/download/](https://www.gnupg.org/download/) I've found that it used a different default path and I could not list the key anymore. Addind a new environment variable `GNUPGHOME` and set it to `C:\Users\<username>\.gnupg`. Now the gpg that was installed in windows could read the old keychain, which meant maven could now use that key to sign files.

## Configuring Maven to know where to get the signing key
Check out your `<maven-install-apth>/conf` directory. There should be a `settings.xml` file. Copy that over to your `<user-home>/.m2`, unless you already have it there. Add the following to `<profiles>`:
```xml
<profiles>
    <profile>
      <id>ossrh</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <gpg.executable>gpg</gpg.executable>
        <gpg.passphrase>passphrase-you-used-when-created-gpg-key</gpg.passphrase>
      </properties>
    </profile>
</profiles>
```
It's ok to have your passphrase set here as this is your user-specific configuration file. If you don't want to specify that, however, there will be an option for you to provide that passphrase every time you publish.

## Configuring Maven to know the credentials for Sonatype servers
You'll provide the log-in credentials in the same `settings.xml` maven file in `~/.m2` directory. If you don't want to provide the actual username and password, log in to your account at [https://oss.sonatype.org](https://oss.sonatype.org). In the top right corner click `Log-In`, then click your username and select `Profile`. On the new screen there's a dropdown with two choices: _Summary_ and _User Token_. Select the user token, it will give you the info. In the `settings.xml` file add:
```xml
  <servers>
    <server>
      <id>ossrh</id>
      <username>user-name-token</username>
      <password>password-for-token</password>
    </server>
  </servers>
 ```

## Satisfying requirements to pass all checks upon submission to Sonatype
There's a lot of meta-info required to satisfy all the requirements. As you will be using the same `groupId` for all your artifacts, it's easier to put all the extra information to a parent POM. You can find an example parent project here: [https://github.com/chhh/sonatype-ossrh-parent](https://github.com/chhh/sonatype-ossrh-parent). This project consists only of the POM file, specifying the credentials, basic info and publishing locations. It adds some to the `release` target as well.
It's ok to just clone that repo and change the information to what you like.
__You will set this POM as the `<parent>` of the projects you wish to publish to Central.__ As it will be the parent POM, anyone who will want to build your artifacts will need to have that POM, so the first thing is to publish this project to Central by itself.

## Publishing parent POM project to Central
We'll be using maven-release plugin. Make sure that:
 - You have SCM information configured.
 - In this parent POM you set the `version` to something like `0.1-SNAPSHOT`.

The release plugin will use that information to create the build. It will remove the `SNAPSHOT` part, build the project, create a new tag in SCM, push everything to remote, bump up the version in POM and re-add SNAPSHOT back to it.
Now execute:
`mvn release:prepare`
`mvn release:perform`
If you encounter any problems with `release:perform` you can always do
`mvn release:rollback` to undo any changes done by `release:prepare`.

## Publishing the project to Central
In your actual project set the `parent`:
```xml
<parent>
    <groupId>com.github.chhh</groupId>
    <artifactId>sonatype-ossrh-parent</artifactId>
    <version>0.1</version>
    <relativePath>../sonatype-ossrh</relativePath>
</parent>
```
Notice how we used `relativePath` to give maven a hint at where to search for this POM. The parent project was resiging in a sibling directory next to the project directory in this case. Otherwise the POM would have to be in the parent directory of your project.

## Releasing the project 'more' manulally
We'll use just the `nexus-staging-maven-plugin` plugin. If it sees that the `<version>` in your
project's pom doesn't end with `-SNAPSHOT` it will deploy the artifact to staging area,
which will run the checks, and if all is fine will auto-close the staging repo and release it.
We only need to tell maven to add javadocs and sources as is required by Sonatype OSSRH
and sign all the files. We already have everything set up for the `maven-release-plugin` in `<profiles>`.
To tell maven to use the profile add `-P` switch like this:  
`mvn -Prelease clean deploy`   
_release_ here is the name of the profile
