# go-getter

[![CircleCI](https://circleci.com/gh/hashicorp/go-getter/tree/master.svg?style=svg)][circleci]
[![Build status](https://ci.appveyor.com/api/projects/status/ulq3qr43n62croyq/branch/master?svg=true)][appveyor]
[![Go Documentation](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)][godocs]

[circleci]: https://circleci.com/gh/hashicorp/go-getter/tree/master
[godocs]: https://pkg.go.dev/github.com/hashicorp/go-getter/v2
[appveyor]: https://ci.appveyor.com/project/hashicorp/go-getter/branch/master

go-getter is a library for Go (golang) for downloading files or directories
from various sources using a URL as the primary form of input.

The power of this library is being flexible in being able to download
from a number of different sources (file paths, Git, HTTP, Mercurial, etc.)
using a single string as input. This removes the burden of knowing how to
download from a variety of sources from the implementer.

The concept of a _detector_ automatically turns invalid URLs into proper
URLs. For example: "github.com/hashicorp/go-getter" would turn into a
Git URL. Or "./foo" would turn into a file URL. These are extensible.

This library is used by [Terraform](https://terraform.io) for
downloading modules, [Packer](https://packer.io) for downloading binaries, and
[Nomad](https://nomadproject.io) for downloading binaries.

## Installation and Usage

Package documentation can be found on
[GoDoc](https://pkg.go.dev/github.com/hashicorp/go-getter/v2).

Installation can be done with a normal `go get`:

```
$ go get github.com/hashicorp/go-getter/v2
```

go-getter also has a command you can use to test URL strings:

```
$ go install github.com/hashicorp/go-getter/cmd/go-getter
...

$ go-getter github.com/foo/bar ./foo
...
```

The command is useful for verifying URL structures.

## Security
Fetching resources from user-supplied URLs is an inherently dangerous operation and may
leave your application vulnerable to [server side request forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery),
[path traversal](https://owasp.org/www-community/attacks/Path_Traversal), [denial of service](https://owasp.org/www-community/attacks/Denial_of_Service)
or other security flaws.

go-getter contains mitigations for some of these security issues, but should still be used with
caution in security-critical contexts. See the available [security options](#Security-Options) that
can be configured to mitigate some of these risks.

go-getter may return values that contain caller-provided query parameters that can contain sensitive data.
Context around what parameters are and are not sensitive is known only by the caller of go-getter, and specific to each use case.
We recommend the caller ensure that go-getter's return values (e.g., error messages) are properly handled and sanitized to ensure
sensitive data is not persisted to logs.
## URL Format

go-getter uses a single string URL as input to download from a variety of
protocols. go-getter has various "tricks" with this URL to do certain things.
This section documents the URL format.

### Supported Protocols and Detectors

**Protocols** are used to download files/directories using a specific
mechanism. Example protocols are Git and HTTP.

**Detectors** are used to transform a valid or invalid URL into another
URL if it matches a certain pattern. Example: "github.com/user/repo" is
automatically transformed into a fully valid Git URL. This allows go-getter
to be very user friendly.

go-getter out of the box supports the following protocols. Additional protocols
can be augmented at runtime by implementing the `Getter` interface.

  * Local files
  * Git
  * Mercurial
  * HTTP
  * Amazon S3
  * Google GCP
  * SMB

In addition to the above protocols, go-getter has what are called "detectors."
These take a URL and attempt to automatically choose the best protocol for
it, which might involve even changing the protocol. The following detection
is built-in by default:

  * File paths such as "./foo" are automatically changed to absolute
    file URLs.
  * GitHub URLs, such as "github.com/mitchellh/vagrant" are automatically
    changed to Git protocol over HTTP.
  * GitLab URLs, such as "gitlab.com/inkscape/inkscape" are automatically
    changed to Git protocol over HTTP.
  * BitBucket URLs, such as "bitbucket.org/mitchellh/vagrant" are automatically
    changed to a Git or mercurial protocol using the BitBucket API.

### Forced Protocol

In some cases, the protocol to use is ambiguous depending on the source
URL. For example, "http://github.com/mitchellh/vagrant.git" could reference
an HTTP URL or a Git URL. Forced protocol syntax is used to disambiguate this
URL.

Forced protocol can be done by prefixing the URL with the protocol followed
by double colons. For example: `git::http://github.com/mitchellh/vagrant.git`
would download the given HTTP URL using the Git protocol.

Forced protocols will also override any detectors.

In the absence of a forced protocol, detectors may be run on the URL, transforming
the protocol anyways. The above example would've used the Git protocol either
way since the Git detector would've detected it was a GitHub URL.

### Protocol-Specific Options

Each protocol can support protocol-specific options to configure that
protocol. For example, the `git` protocol supports specifying a `ref`
query parameter that tells it what ref to checkout for that Git
repository.

The options are specified as query parameters on the URL (or URL-like string)
given to go-getter. Using the Git example above, the URL below is a valid
input to go-getter:

    github.com/hashicorp/go-getter?ref=abcd1234

The protocol-specific options are documented below the URL format
section. But because they are part of the URL, we point it out here so
you know they exist.

### Subdirectories

If you want to download only a specific subdirectory from a downloaded
directory, you can specify a subdirectory after a double-slash `//`.
go-getter will first download the URL specified _before_ the double-slash
(as if you didn't specify a double-slash), but will then copy the
path after the double slash into the target directory.

For example, if you're downloading this GitHub repository, but you only
want to download the `testdata` directory, you can do the following:

```
https://github.com/hashicorp/go-getter.git//testdata
```

If you downloaded this to the `/tmp` directory, then the file
`/tmp/archive.gz` would exist. Notice that this file is in the `testdata`
directory in this repository, but because we specified a subdirectory,
go-getter automatically copied only that directory contents.

Subdirectory paths may also use filesystem glob patterns. The path must
match _exactly one_ entry or go-getter will return an error.
This is useful if you're not sure the exact directory name but it follows
a predictable naming structure.

For example, the following URL would also work:

```
https://github.com/hashicorp/go-getter.git//test-*
```

### Checksumming

For file downloads of any protocol, go-getter can automatically verify
a checksum for you. Note that checksumming only works for downloading files,
not directories, but checksumming will work for any protocol.

To checksum a file, append a `checksum` query parameter to the URL. go-getter
will parse out this query parameter automatically and use it to verify the
checksum. The parameter value can be in the format of `type:value` or just
`value`, where type is "md5", "sha1", "sha256", "sha512" or "file" . The
"value" should be the actual checksum value or download URL for "file". When
`type` part is omitted, type will be guessed based on the length of the
checksum string. Examples:

```
./foo.txt?checksum=md5:b7d96c89d09d9e204f5fedc4d5d55b21
```

```
./foo.txt?checksum=b7d96c89d09d9e204f5fedc4d5d55b21
```

```
./foo.txt?checksum=file:./foo.txt.sha256sum
```

When checksumming from a file - ex: with `checksum=file:url` - go-getter will
get the file linked in the URL after `file:` using the same configuration. For
example, in `file:http://releases.ubuntu.com/cosmic/MD5SUMS` go-getter will
download a checksum file under the aforementioned url using the http protocol.
All protocols supported by go-getter can be used. The checksum file will be
downloaded in a temporary file then parsed. The destination of the temporary
file can be changed by setting system specific environment variables: `TMPDIR`
for unix; `TMP`, `TEMP` or `USERPROFILE` on windows. Read godoc of
[os.TempDir](https://golang.org/pkg/os/#TempDir) for more information on the
temporary directory selection. Content of files are expected to be BSD or GNU
style. Once go-getter is done with the checksum file; it is deleted.

The checksum query parameter is never sent to the backend protocol
implementation. It is used at a higher level by go-getter itself.

If the destination file exists and the checksums match: download
will be skipped.

### Unarchiving

go-getter will automatically unarchive files into a file or directory
based on the extension of the file being requested (over any protocol).
This works for both file and directory downloads.

go-getter looks for an `archive` query parameter to specify the format of
the archive. If this isn't specified, go-getter will use the extension of
the path to see if it appears archived. Unarchiving can be explicitly
disabled by setting the `archive` query parameter to `false`.

The following archive formats are supported:

  * `tar.gz` and `tgz`
  * `tar.bz2` and `tbz2`
  * `tar.xz` and `txz`
  * `zip`
  * `gz`
  * `bz2`
  * `xz`

For example, an example URL is shown below:

```
./foo.zip
```

This will automatically be inferred to be a ZIP file and will be extracted.
You can also be explicit about the archive type:

```
./some/other/path?archive=zip
```

And finally, you can disable archiving completely:

```
./some/path?archive=false
```

You can combine unarchiving with the other features of go-getter such
as checksumming. The special `archive` query parameter will be removed
from the URL before going to the final protocol downloader.

## Protocol-Specific Options

This section documents the protocol-specific options that can be specified for
go-getter. These options should be appended to the input as normal query
parameters ([HTTP headers](#headers) are an exception to this, however).
Depending on the usage of go-getter, applications may provide alternate ways of
inputting options. For example, [Nomad](https://www.nomadproject.io) provides a
nice options block for specifying options rather than in the URL.

## General (All Protocols)

The options below are available to all protocols:

  * `archive` - The archive format to use to unarchive this file, or "" (empty
    string) to disable unarchiving. For more details, see the complete section
    on archive support above.

  * `checksum` - Checksum to verify the downloaded file or archive. See
    the entire section on checksumming above for format and more details.

  * `filename` - When in file download mode, allows specifying the name of the
    downloaded file on disk. Has no effect in directory mode.

### Local Files (`file`)

None

### Git (`git`)

  * `ref` - The Git ref to checkout. This is a ref, so it can point to
    a commit SHA, a branch name, etc. If it is a named ref such as a branch
    name, go-getter will update it to the latest on each get.

  * `sshkey` - An SSH private key to use during clones. The provided key must
    be a base64-encoded string. For example, to generate a suitable `sshkey`
    from a private key file on disk, you would run `base64 -w0 <file>`.

    **Note**: Git 2.3+ is required to use this feature.

  * `depth` - The Git clone depth. The provided number specifies the last `n`
    revisions to clone from the repository.


The `git` getter accepts both URL-style SSH addresses like
`git::ssh://git@example.com/foo/bar`, and "scp-style" addresses like
`git::git@example.com/foo/bar`. In the latter case, omitting the `git::`
force prefix is allowed if the username prefix is exactly `git@`.

The "scp-style" addresses _cannot_ be used in conjunction with the `ssh://`
scheme prefix, because in that case the colon is used to mark an optional
port number to connect on, rather than to delimit the path from the host.

### Mercurial (`hg`)

  * `rev` - The Mercurial revision to checkout.

### HTTP (`http`)

#### Basic Authentication

To use HTTP basic authentication with go-getter, simply prepend `username:password@` to the
hostname in the URL such as `https://Aladdin:OpenSesame@www.example.com/index.html`. All special
characters, including the username and password, must be URL encoded.

#### Headers

Optional request headers can be added by supplying them in a custom
[`HttpGetter`](https://godoc.org/github.com/hashicorp/go-getter#HttpGetter)
(_not_ as query parameters like most other options). These headers will be sent
out on every request the getter in question makes.

### S3 (`s3`)

S3 takes various access configurations in the URL. Note that it will also
read these from standard AWS environment variables if they're set. S3 compliant servers like Minio
are also supported. If the query parameters are present, these take priority.

  * `aws_access_key_id` - AWS access key.
  * `aws_access_key_secret` - AWS access key secret.
  * `aws_access_token` - AWS access token if this is being used.
  * `aws_profile` - Use this profile from local ~/.aws/ config. Takes priority over the other three.

#### Using IAM Instance Profiles with S3

If you use go-getter and want to use an EC2 IAM Instance Profile to avoid
using credentials, then just omit these and the profile, if available will
be used automatically.

### Using S3 with Minio
 If you use go-gitter for Minio support, you must consider the following:

  * `aws_access_key_id` (required) - Minio access key.
  * `aws_access_key_secret` (required) - Minio access key secret.
  * `region` (optional - defaults to us-east-1) - Region identifier to use.
  * `version` (optional - defaults to Minio default) - Configuration file format.

#### S3 Bucket Examples

S3 has several addressing schemes used to reference your bucket. These are
listed here: http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html#access-bucket-intro

Some examples for these addressing schemes:
- s3::https://s3.amazonaws.com/bucket/foo
- s3::https://s3-eu-west-1.amazonaws.com/bucket/foo
- bucket.s3.amazonaws.com/foo
- bucket.s3-eu-west-1.amazonaws.com/foo/bar
- "s3::http://127.0.0.1:9000/test-bucket/hello.txt?aws_access_key_id=KEYID&aws_access_key_secret=SECRETKEY&region=us-east-2"

### GCS (`gcs`)

#### GCS Authentication

In order to access to GCS, authentication credentials should be provided. More information can be found [here](https://cloud.google.com/docs/authentication/getting-started)

#### GCS Bucket Examples

- gcs::https://www.googleapis.com/storage/v1/bucket
- gcs::https://www.googleapis.com/storage/v1/bucket/foo.zip
- www.googleapis.com/storage/v1/bucket/foo

#### GCS Testing

The tests for `get_gcs.go` require you to have GCP credentials set in your environment.  These credentials can have any level of permissions to any project, they just need to exist.  This means setting `GOOGLE_APPLICATION_CREDENTIALS="~/path/to/credentials.json"` or `GOOGLE_CREDENTIALS="{stringified-credentials-json}"`.  Due to this configuration, `get_gcs_test.go` will fail for external contributors in CircleCI.

### SMB (smb)

There are two options that go-getter will use to download a file in a smb shared folder. The first option uses
[`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) and the second one uses the file system
to look for a file in a local mount of the shared folder in the OS specific volume folder. go-getter will try to download
files from a smb shared folder whenever the url is prefixed with `smb://`.

⚠️ The [`smbclient`](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) command is available only for Linux.
This is the ONLY option for a Linux user and therefore the client must be installed.

The `smbclient` cli is not available for Windows and MacOS. The go-getter
will try to get files using the file system, when this happens the getter uses the FileGetter implementation.

When connecting to a smb server, the OS creates a local mount in a system specific volume folder, and go-getter will
try to access the following folders when looking for local mounts.

- MacOS: /Volumes/<shared_path>
- Windows: \\\\\<host>\\\<shared_path>

The following examples work for all the OSes:
- smb://host/shared/dir (downloads directory content)
- smb://host/shared/dir/file (downloads file)

The following examples work for Linux:
- smb://username:password@host/shared/dir (downloads directory content)
- smb://username@host/shared/dir
- smb://username:password@host/shared/dir/file (downloads file)
- smb://username@host/shared/dir/file

⚠️ The above examples also work on the other OSes but the authentication is not used to access the file system.



#### SMB Testing
The test for `get_smb.go` requires a smb server running which can be started inside a docker container by
running `make start-smb`. Once the container is up the shared folder can be accessed via `smb://<ip|name>/public/<dir|file>` or
`smb://user:password@<ip|name>/private/<dir|file>` by another container or machine in the same network.

To run the tests inside `get_smb_test.go` and `client_test.go`, prepare the environment with `make smbtests-prepare`. On prepare some
mock files and directories will be added to the shared folder and a go-getter container will start together with the samba server.
Once the environment for testing is prepared, run `make smbtests` to run the tests.

### Security Options

**Disable Symlinks**

In your getter client config, we recommend using the `DisableSymlinks` option,
which prevents writing through or copying from symlinks (which may point outside the directory).

```go
client := getter.Client{
    // This will prevent copying or writing files through symlinks
    DisableSymlinks: true,
}
```

**Disable or Limit `X-Terraform-Get`**

Go-Getter supports arbitrary redirects via the `X-Terraform-Get` header. This functionality
exists to support [Terraform use cases](https://www.terraform.io/language/modules/sources#http-urls),
but is likely not needed in most applications.

For code that uses the `HttpGetter`, add the following configuration options:

```go
var httpGetter = &getter.HttpGetter{
    // Most clients should disable X-Terraform-Get
    // See the note below
    XTerraformGetDisabled: true,
    // Your software probably doesn’t rely on X-Terraform-Get, but
    // if it does, you should set the above field to false, plus
    // set XTerraformGet Limit to prevent endless redirects
    // XTerraformGetLimit: 10,
}
```

**Enforce Timeouts**

The `HttpGetter` supports timeouts and other resource-constraining configuration options. The `GitGetter` and `HgGetter`
only support timeouts.

Configuration for the `HttpGetter`:

```go
var httpGetter = &getter.HttpGetter{
    // Disable pre-fetch HEAD requests
    DoNotCheckHeadFirst: true,

    // As an alternative to the above setting, you can
    // set a reasonable timeout for HEAD requests
    // HeadFirstTimeout: 10 * time.Second,
    // Read timeout for HTTP operations
    ReadTimeout: 30 * time.Second,
    // Set the maximum number of bytes
    // that can be read by the getter
    MaxBytes: 500000000, // 500 MB
}
```

For code that uses the `GitGetter` or `HgGetter`, set the `Timeout` option:
```go
var gitGetter = &getter.GitGetter{
    // Set a reasonable timeout for git operations
    Timeout: 5 * time.Minute,
}
```

```go
var hgGetter = &getter.HgGetter{
    // Set a reasonable timeout for hg operations
    Timeout: 5 * time.Minute,
}
```
