# F-Droid prebuilts for custom ROMs

Fetch apps from F-Droid repositories and generate an AOSP prebuilts
repository with the APKs, an `Android.bp` and a `product.mk`.

The apps are listed in [repo/fdroid.txt](repo/fdroid.txt). No versions
are pinned: every run picks the version the repo currently suggests
(the stable version the F-Droid client would offer).

App list
--------

One app per line, columns separated by whitespace, `#` for comments:

```
# package           name     partition   arch  cert       type  repo
org.fdroid.fdroid   F-Droid  system_ext  all   presigned  app   https://f-droid.org/repo?fingerprint=43238D51...
```

| Column    | Values |
|-----------|--------|
| package   | Android package name |
| name      | Soong module name, also used for the APK file name |
| partition | `system_ext` or `product` |
| arch      | `all`, `arm64`, `arm`, `x86_64`, `x86` |
| cert      | `presigned` keeps the F-Droid signature, `default` uses the build's default key, anything else (`platform`, `shared`, `media`, ...) is passed as `certificate` |
| type      | `app` or `priv` (privileged, gets a generated privapp-permissions XML) |
| repo      | repo URL; append `?fingerprint=<sha256>` to verify the signed index |

`all` needs a universal APK. For apps that ship one APK per ABI, add one
line per arch with the same `name`; they become a single module with an
`arch` block.

Downloaded APKs are always checked against the SHA-256 in the repo index.

Output
------

```
Android.bp      android_app_import (+ prebuilt_etc for priv-app permissions)
product.mk      PRODUCT_PACKAGES for all apps
versions.txt    lock file: version, signer and sha256 of every APK
apps/<name>/    APKs and privapp-permissions XMLs
```

When a version is published under several signers, the signer recorded in
`versions.txt` is kept, so installed apps keep receiving updates. A
signer change on a `presigned` app is an error; delete the app's row from
`versions.txt` in the output repo to accept it.

Running locally
---------------

Needs Python 3, and `jarsigner`/`keytool` (a JDK) when using fingerprints.

```console
$ ./fdroid_sync.py --out ../android_vendor_fdroid_prebuilts
```

GitLab CI
---------

The `check` job runs the sync to check that every app in the list resolves.
On the default branch the `publish` job syncs into the prebuilts repo and pushes a
commit over SSH when anything changed.

1. Generate a key pair: `ssh-keygen -t ed25519 -N '' -C fdroid-ci -f fdroid-ci`
2. In the prebuilts repo, add `fdroid-ci.pub` as a deploy key with
   *Grant write permissions*, and allow that deploy key to push to the
   protected branch (Settings > Repository > Protected branches).
3. In this repo, set these CI/CD variables:

* `OUTPUT_REPO_URL`: SSH URL of the prebuilts repo, e.g.
  `git@gitlab.com:group/android_vendor_fdroid_prebuilts.git`
* `OUTPUT_REPO_SSH_KEY`: contents of `fdroid-ci` (the private key), type
  *File*, protected
* `OUTPUT_REPO_BRANCH` (optional): branch to push to, default `main`

Add a pipeline schedule on the default branch (e.g. daily) to follow new releases.

Using the prebuilts
-------------------

Add the prebuilts repo to your local manifest:

```xml
<project name="<group>/proprietary_vendor_fdroid-prebuilts"
         path="vendor/fdroid-prebuilts"
         remote="gitlab"
         revision="main" />
```

and include it from your device or product makefile:

```make
$(call inherit-product, vendor/fdroid-prebuilts/product.mk)
```
