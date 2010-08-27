# Jelly - A Dynamic ELPA Server

Jelly is an Emacs Lisp package server that allows authors to easily upload their
packages. It follows the protocol expected by `package.el`, the standard Emacs
package manager, and so can be used in conjunction with the [official GNU
repository](http://elpa.gnu.org/) or the original [ELPA
repository](http://tromey.com/elpa).

## Running Jelly

Jelly is designed to be easy to get up and running on your own server. It only
takes five steps:

1. Install [node.js](http://nodejs.org/#download).
2. Install the [Node package manager](http://github.com/isaacs/npm#readme)
3. `curl http://github.com/nex3/jelly/tarball/master | tar xz`
4. `npm install jelly`
5. `jelly`

## Installing Packages from a Jelly Archive

### Get `package.el`

To use Jelly, you first need `package.el`. If you're using Emacs 24 or later,
you've already got it. Otherwise, download it from
[here](http://github.com/technomancy/package.el/raw/master/package.el)
(currently the official ELPA `package.el` doesn't support multiple archives).

### Enable the Archive

Add this line to your `.emacs`:

    (add-to-list 'package-archives '("jelly" . "http://your.domain/packages/"))

### That's It!

Your archive is now active! Run `M-x package-list` and see all the new packages,
and run `M-x package-install` to install them.


## The Jelly API

Jelly supports a simple HTTP interface for uploading packages.

### Response Format

Jelly can send responses either as JSON objects or Emacs Lisp assoc lists. If
the user agent sends `application/json` in its `Accept` header, it will be
served JSON. If it sends `text/x-script.elisp`, it will be served Elisp.
Otherwise, if nothing is specified, Jelly will default to JSON.

All responses, including error responses, will have a `message` key. The value
of this key will be a human-readable description of the server event (or the
error).

All requests will return a 400 status if any required parameters are missing.

An example JSON response to a user registration might look like this:

    {
      "message": "Successfully registered nex3",
      "name": "nex3",
      "token": "some base64 token"
    }

The same response as Elisp might look like this:

    (
      (message . "Successfully registered nex3")
      (name . "nex3")
      (token , "some base64 token")
    )


### Authentication

Every user has a randomly-generated 256-bit authentication token. This token and
the user's username are required for any request that needs authentication. The
username is case-insensitive, while the token is not.

All requests that require authentication will return a 400 status if the
authentication fails.

#### `POST /v1/users/login`

*Parameters*: `name`, `password`.

*Response*: `name`, `token`.

*Error Codes*: 400

Gets the authentication token for a user. This is the only time the API requires
password authentication. The token can also be obtained from the web interface
(TODO: actually it can't yet).

This will have a 400 status if the username or password is wrong.

#### `POST /v1/users`

*Parameters*: `name`, `password`

*Response*: `name`, `token`

*Error Codes*: 400

Registers a new user, and returns the authentication token for that user.

This will have a 400 status if the username was already taken, or the password
is invalid.


### Packages

Currently, the API only supports uploading packages.

Packages are represented as objects with the following fields:

* `name`: The string name of the package.
* `description`: A single-line description of the package, taken from the
    header line for Elisp packages.
* `commentary`: An optional longer description of the package, taken from
    the Commentary section for Elisp packages and the README file for
    tarballs.
* `requires`: An array of name/version pairs describing the dependencies of
    the package. The format for the versions is the same as the `version`
    field.
* `version`: An array of numbers representing the dot-separated version.
* `type`: Either "single" (for an Elisp file) or "tar" (for a tarball).
* `owners`: A set of names of users who have the right to post updates for
    the package.

For example, the package for `sass-mode` version 3.0.13 might look like:

    {
      name: "sass-mode",
      description: "Major mode for editing Sass files",
      commentary: "Blah blah blah",
      requires: [["haml-mode", [3, 0, 13]]],
      version: [3, 0, 13],
      type: "single",
      owners: {"nex3": true}
    }


#### `POST /v1/packages`

*Parameters*: `name`, `token`, `package`

*Response*: `package`

*Error Codes*: 400, 403

Uploads a new package, or a new version of an existing package. This request
should use the `multipart/form-data` content type, with `name` and `token` as
fields and `package` as a file.

`package` can be either a single Elisp file or a tarball containing multiple
files. In either case, it must conform to the [ELPA packaging
standards](http://tromey.com/elpa/upload.html).

This returns the package object that has been parsed out of the uploaded
package.

The response will have a 400 status if the package is improperly formatted.

The response will have a 403 status if the username and token are valid, but the
user in question doesn't have permission to upload the given package.
