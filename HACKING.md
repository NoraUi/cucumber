# Hacking

This document describes the technical details of hacking on Cucumber.

## Installing build tools

Because of the polyglot nature of the Cucumber project, a lot of different tools
are required to build it. Here is how:

### OS X

    brew install go cmake maven gpg gpg-agent

## Monorepo

This git repository is a monorepo. It contains several libraries that Cucumber
is built on, in many programming languages. The goal is to migrate all of the
Cucumber codebase into this repository.

You can learn more about monorepos here:
* http://www.thedotpost.com/2016/05/fabien-potencier-monolithic-repositories-vs-many-repositories
* https://developer.atlassian.com/blog/2015/10/monorepos-in-git/
* http://danluu.com/monorepo/
* https://medium.com/@bebraw/the-case-for-monorepos-907c1361708a

### Branching and CI

The CI build will synchronise from the monorepo to all the subrepos for the `master` branch.
For other branches, a naming convention is used to decide what subrepos to sync to. 
(This is to avoid an explosion of unrelated branches in every subrepo).

For example, if you're making a change to `gherkin` on a branch, prefix the branch
with `gherkin`, for example `gherkin-upgrade-dependencies`.

Occasionally you want to sync to multiple subrepos. For example, if you are making changes
in two modules (say `messages` and `gherkin`), prefix the branch with both module names, 
separated by an underscore. For example, `messages_gherkin-use-protobuf`.

### Local Testing / Development checks

To run a copy of the CI locally simply run the command below. **NB: This will require docker being installed.**

    ./scripts/docker-run Dockerfile make
    
This will create a localised version of the Docker image used on CI to build cucumber then run through
all of the associated make tasks in each of the sub-repos. This can take a while, but it saves you needing
to push each individual commit up to the repo and then wait for the CI tests to finish.

### Adding a new subrepo

Occasionally, a sub directory is promoted to a separate subrepo. The process for doing this is:

#### Create a new directory in the monorepo

For example:

    mkdir -p tag-expressions/go

#### Add initial files

In the new directory, create the following files:

`.rsync`, with the following sample contents (adapt to the programming language):

    ../../LICENSE LICENSE
    ../../.templates/github/ .github/
    ../../.templates/go/.travis.yml .travis.yml

`README.md` with a build badge for the new subrepo. For example:

    # Tag Expressions for Go

    [![Build Status](https://travis-ci.org/cucumber/tag-expressions-go.svg?branch=master)](https://travis-ci.org/cucumber/tag-expressions-go)

    [The docs are here](https://cucumber.io/docs/cucumber/api/#tag-expressions).

#### Sync files

    git add .
    source scripts/functions.sh && rsync_files
    git add .
    git commit -m "New project"

#### Create new subrepo.

Create a new, empty subrepo at GitHub. Check the box for initialising
with a README - it's needed to create an initial `master` branch to push to.

Log into Travis and set up build for the new (empty) subrepo.

Initialise the subrepo, for example:
    
    echo "cucumber/tag-expressions-go" > tag-expressions/go/.subrepo
    git add tag-expressions/go/.subrepo
    git commit -m "Add cucumber/tag-expressions-go subrepo"

Push to the subrepo:

    push_subrepo tag-expressions/go

Wait for Travis to build.

Write some code in the monorepo, and push whenever you want to sync to the subrepo.

#### Set up GPG and automatic deployment

The GPG key in use is cukebot@cucumber.io, which has id `E60E1F911B996560FFB135DAF4CABFB5B89B8BE6`.

The secret key is encrypted and added to git (`scripts/codesigning.asc.enc`). 
The secret key is decrypted to `scripts/codesigning.asc` during a CI build.

The secret key must be re-encrypted for each subrepo, as Travis encrypts it
differently for each subrepo. 

    gpg --export-secret-key E60E1F911B996560FFB135DAF4CABFB5B89B8BE6 > scripts/codesigning.asc
    # Replace SLUG with the name of the subrepo, for example cucumber-expressions-java
    travis encrypt-file scripts/codesigning.asc scripts/codesigning.asc.enc --repo cucumber/SLUG

Copy the bold text and create a new file `scripts/decrypt_signing_key.sh` with:

```bash
#!/usr/bin/env bash
set -euf -o pipefail
openssl aes-256-cbc -K $encrypted_1570928b04a6_key -iv $encrypted_1570928b04a6_iv -in scripts/codesigning.asc.enc -out scripts/codesigning.asc -d
```

(The `openssl` line should be copied from `travis encrypt` output).

Make it executable and add to git:

    chmod +x scripts/decrypt_signing_key.sh 
    git add scripts/codesigning.asc.enc scripts/decrypt_signing_key.sh
    git commit -m "Add encrypted signing key"

#### Maven-specific

IMPORTANT: Make sure you [escape](https://docs.travis-ci.com/user/encryption-keys/#Note-on-escaping-certain-symbols)
characters with `\` in the password/passphrase when running the commands below:

Add the password for Sonatype cukebot (in 1Password) 

    travis encrypt 'CI_SONATYPE_PASSWORD=secretvalue' --add --repo cucumber/SLUG

Add the passphrase for the GPG signing key (in 1Password) 

    travis encrypt 'CI_GPG_PASSPHRASE=secretvalue' --add --repo cucumber/SLUG

### Docker and CI

Our CI build uses Docker. We have our own docker images defined in `Dockerfile.*`
files. These need to be rebuilt and published manually whenever they change.

In order to publish new images, log in as `cukebot`. The password is in 1Password,
in the "Cucumber Open Source" vault.

   source ./scripts/functions.sh
   # Standard image (for all builds except .NET)
   docker_build Dockerfile
   docker_push Dockerfile
   # .NET image (for .NET builds only)
   docker_build Dockerfile-dotnet
   docker_push Dockerfile-dotnet

The images are published [in the cucumber repository section at
Docker Hub](https://hub.docker.com/r/cucumber/).

