# Plugin uploader

## Description

```
Usage: plugin_uploader.py [OPTIONS] COMMAND [ARGS]...

CLI tool to upload/index goreleaser-built binaries to/in S3.

Options:
--help  Show this message and exit.

Commands:
upload-archives  Create tar.gz archives from binaries and upload to S3
upload-manifest  Create manifest.json and upload to S3

`plugin_uploader.py` is used to upload the binaries generated by goreleaser to S3 in a manner that is consumable by RPK as a plugin.

```

## Install

`pip install -r requirements.txt`

## How to use

Primary use case is in GitHub Actions in response to creation of a GitHub release.

See `.github/workflows/upload_plugin.yml` to see this in action.

It's expected that you have used goreleaser to build a set of binaries for a given release tag (such as following a
GitHub release tag creation).

Goreleaser creates a `$DIST` directory (`dist/` by default) at the project root dir containing all built binaries and
two JSON files:

* `$DIST/<build-name>-<os>-<arch>/<binary-filename>`
* ...
* `$DIST/artifacts.json`
* `$DIST/metadata.json`

### Create archives from binaries and upload them

Locate the `artifact.json` and `metadata.json` files produced by Goreleaser.
E.g. `$DIST/artifacts.json`, `$DIST/metadata.json`.

```shell
./plugin_uploader.py upload-archives \
                        --artifacts-file=$DIST/artifacts.json \
                        --metadata-file=$DIST/metadata.json \
                        --project-root-dir=<PROJECT_ROOT> \
                        --region=<AWS_REGION> \
                        --bucket=<AWS_S3_BUCKET> \
                        --plugin=<PLUGIN_NAME> \
                        --goos=<OS1,OS2,...> \
                        --goarch=<ARCH1,ARCH2,...>
```

`PROJECT_ROOT` should be the root directory of the Golang project (by default, where `.goreleaser.yml` lives)

`PLUGIN_NAME` should match the `<build-id>` as defined in goreleaser configs.

It's assumed that the output binary filename is `redpanda-<build-id>`. E.g. for the **connect** project:

* `build-id` is `connect`
* Binary is `redpanda-connect`

A binary is included for archival / upload only if it matches some `--goos` AND some `--goarch`.

`--dry-run` is available for skipping final S3 upload step.

AWS permissions are needed for these actions on the S3 bucket:

* `s3:PutObject`
* `s3:PutObjectTagging`
  You may also need permissions on any AWS KMS keys used for server side encryption of the S3 bucket.

### Create manifest.json and upload it

This lists all archives for the specific plugin and constructs a `manifest.json` from the listing.

This should be run after uploading any archives.

```shell
./plugin_uploader.py upload-manifest \
                        --region=<AWS_REGION> \
                        --bucket=<AWS_S3_BUCKET> \
                        --plugin=<PLUGIN_NAME> \
                        --repo-hostname=<REPO_HOSTNAME>
```

`--repo-hostname` is used for generating the right public facing download URLs for archives in the plugin repo. E.g.
`rpk-plugins.redpanda.com`

`--dry-run` is available for skipping the final S3 upload step.

AWS permissions are needed for these actions on the S3 bucket:

* `s3:PutObject`
* `s3:ListBucket`
* `s3:GetObjectTagging`
  You may also need permissions on any AWS KMS keys used for server side encryption of the S3 bucket.