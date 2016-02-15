# Heroku buildpack for Nix applications

This buildpack allows any Nix project to be deployed to Heroku. The resulting
app must be run within [PRoot](http://proot.me/), however, which does incur some
degree of performance penalty. The buildpack requires access to an S3 bucket to
save and restore the Nix closure. For some apps (such as Haskell apps using
large frameworks like Yesod) this may be as much as 2GB, so make sure that your
S3 bucket is in the same region as the dyno to take advantage of the free data
transfer.

## Usage

Create a Procfile for your processes. The `run_result.sh` script is made
available by the buildpack and will ensure that your process is run within the
build output directory (ie. `nix-build`'s `result/`).

```
web: run_result.sh bin/webserver
worker: run_result.sh foo bin/bar
```

Then create your application on Heroku, setting the buildpack and S3 access
variables. You'll also need to set the `NIX_APP_NAME` variable - this tells the
buildpack what name the closure file should have.

```bash
heroku create -b https://github.com/mayhewluke/nix-heroku-buildpack.git
heroku config:set NIX_S3_KEY=... \
                  NIX_S3_SECRET=... \
                  NIX_S3_BUCKET=... \
                  NIX_APP_NAME=my-awesome-app
```

### Building and installing

By default the buildpack does not build the app when you push to Heroku. Heroku
imposes a 15 minute time limit on builds - larger apps with lots of dependencies
and/or long compile times will often exceed this time limit. This is especially
true for the first push since Nix will have to pull in all dependencies by hand
since a closure has not been created yet.

When you push, the buildpack will set up the build environment. If a closure is
found it will attempt to import your app from the closure. Note that since this
is being restored, not built, it will actually be the *previous* state of your
app, not the one you just pushed.

Once you've successfully pushed you'll be able to access the build environment
from one-off dynos (eg. `heroku run bash`) which *have no time limit*. You can
now use a one-off dyno to build your app using `heroku run build`. You may also
choose to use a different size of dyno than your web dyno - `heroku run
--size=performance-l build`.

After the app has been built it will be stored in the closure on S3 and the next
time you deploy the app it will be restored and run. To trigger a re-deploy:

```bash
git commit --amend --no-edit
git push -f heroku master
```

If you want to have the buildpack attempt to build your app when you push, set
`NIX_BUILD_ON_PUSH`:

```bash
heroku config:set NIX_BUILD_ON_PUSH=1
```

Note that you have to actually *unset* it to turn it off, not just set it to 0.

### Nix build flags

If you've already built the app on a machine with the same architecture as your
Heroku machine (eg. 64-bit Linux) you may be able to speed up the build process
by serving up that machine's binary cache. This ensures that any packages that
don't exist in the Nix binary caches will be downloaded from your substitute
machine instead of built from scratch. This is only beneficial of course if your
network doesn't bottleneck your Heroku dyno and you use a significant number of
packages not in the Nix cache.

First you'll need to serve up the machine's cache using `nix-serve -p PORT`.
Then set the `NIX_BUILD_FLAGS` config variable in your Heroku app to make use of
it:

```bash
heroku config:set NIX_BUILD_FLAGS='--option extra-binary-caches http://url-to-cache-machine:8080'
```

The `NIX_BUILD_FLAGS` string will be passed to `nix-build` verbatim, so you can
use it to gain greater control over the build process. For example, if you
`default.nix` evaluates to a set of derivations, you can specify the one you
want to use:

```bash
heroku config:set NIX_BUILD_FLAGS='-A deploy'
```

### .slugexclude

Heroku limits slug sizes to 300MB. The buildpack uses the Nix garbage collector
to remove any packages from the slug that the app doesn't need. However, Nix
also keeps a lot of stuff that *isn't* needed at runtime, and this can cause
your app to exceed the slug size. To prevent this, create a `.slugexclude` file
to tell the buildpack what can be excluded from the final slug. It will be used
as an rsync exclude file - see the `INCLUDE/EXCLUDE PATTERN RULES` section of
the [rsync man page](https://rsync.samba.org/ftp/rsync/rsync.html) for details.

**Example:**

```
+ store/**/
+ store/*-user-environment/**
+ store/*-nix-1*/**
+ *[._]so*
+ store/*my-awesome-app*/**

- store/**
```

This `.slugexclude` tells rsync:

- Ignore everything under `store/` **except:**
  - Check every directory and subdirectory for anything that might match our
    include rules
  - Keep everything in the `user-environment` directories (***Make sure you do not
    exclude these directories - they are runtime requirements!***)
  - Keep everything in the Nix installation path (***Make sure you do not
    exclude this directory - it is a runtime requirement!***)
  - Keep all `*.so*` and `*_so*` files
  - Keep everything in our app's directory

See [here](https://silentorbit.com/notes/2013/08/rsync-by-extension/) for an
explanation of the need for the `+ store/**/` and `- store/**` lines.

## How it works

Nix requires `/nix` to be in `/nix`. Heroku doesn't let buildpacks write to `/`.
In order to get around this we use `proot` to make the system think that another
path is actually `/nix`. In our case we use
`$BUILD_DIR/nix-mnt/$NIX_VERSION_FULL`. This gets around the issue of not being
able to write to `/nix` at the expense of a drop in performance.

The next challenge is how Heroku works. The filesystem is ephemeral and we don't
have the option of bootstrapping from a base image like we might with EC2 for
example. Everything that our app needs to run needs to fit within the BUILD_DIR,
which will get compressed down to a "slug" that cannot exceed 300MB. However, we
don't know what the app does or doesn't need in order to run, and the Nix
garbage collector leaves a lot of things that won't be needed at runtime and
just take up space.

The solution: `.slugexclude`. By using rsync include/exclude rules we can give
the user fine-grained control over what to keep and what to leave out. This
allows for some very aggressive pruning when need be (see example above) and
allows slug sizes to be kept more reasonable.

Heroku also presents a challenge in that its time limit on deploys requires a
two-stage build process. It's not really set up this way, though, so we kind of
have to cheat the system a bit. The only way to get past the deploy time limit
is to run the build in a one-off dyno. One-off dynos don't have any access to
the "real" dynos though, nor do they have access to the CACHE_DIR. In order to
run the build in a one-off dyno we have to re-download Nix, re-download the
closure, build the project, then upload it and its closure back to S3.
Unfortunately the only way to get the running dynos to fetch this new version of
the app is to make a redundant push to them, though something slicker may be
achievable through the Heroku API in the future.  `nix-store --import` imports
the downloaded closure and `nix-store -qR` and `nix-store --export` export it.

Once all is said and done we copy everything from our closure (minus the
`.slugexclude`s) to the BUILD_DIR so that the running app can have access to the
packages that it needs. Although the packages in BUILD_DIR get pruned by
`.slugexclude` we don't filter the packages in the closure saved to S3 - if we
did, we'd have to redownload them all next time we built the app, since there's
a difference between what the app needs to run and what Nix needs to build it.
