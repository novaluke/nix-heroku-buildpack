#!/usr/bin/env bash

DIR=$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )
source $DIR/common

mkdir -p $CACHE_DIR/bin
mkdir -p $BUILD_DIR/bin

# We need proot for working with Nix and AWS for caching/restoring our Nix
# binaries.
topic "Downloading buildpack helpers"
download_and_install $CACHE_DIR $BUILD_DIR proot http://static.proot.me/proot-x86_64
download_and_install $CACHE_DIR $BUILD_DIR aws https://raw.githubusercontent.com/timkay/aws/master/aws

# If we don't copy at least the build-specific scripts we won't have access to
# them in one-off dynos. Copy them all to aid in debugging.
topic "Copying buildpack scripts"
cp -f $DIR/* $BUILD_DIR/bin/

# These scripts need to be created on the fly since they use proot which needs
# to know the current NIX_VERSION_FULL, preferably without having to source
# bin/common
cat <<EOF > $BUILD_DIR/bin/run_proot.sh
#!/usr/bin/env bash

/app/bin/proot -b /app/nix-mnt/$NIX_VERSION_FULL:/nix bash /app/bin/run_in_proot.sh "\$@"
EOF

cat <<EOF > $BUILD_DIR/bin/run_in_proot.sh
#!/bin/bash

set -e
test -d /nix && test -d /app/.nix-profile
source /app/.nix-profile/etc/profile.d/nix.sh

"\$@"
EOF

# Trying to give proot a command that uses `cd` prior to the real command
# doesn't seem to work, so we create a wrapper that lets proot commands be run
# in the result/ directory instead of the app/ directory, since that's where our
# build project will live.
cat <<EOF > $BUILD_DIR/bin/run_result.sh
#!/usr/bin/env bash

/app/bin/proot -b /app/nix-mnt/$NIX_VERSION_FULL:/nix bash /app/bin/run_in_result.sh "\$@"
EOF

topic "Creating run_in_result.sh"
cat <<EOF > $BUILD_DIR/bin/run_in_result.sh
#!/bin/bash

set -e
test -d /nix && test -d /app/.nix-profile
source /app/.nix-profile/etc/profile.d/nix.sh

cd /app/result
"\$@"
EOF

# During compile/build we get the Nix profile from /tmp/nix-mnt but we don't
# want to count on /tmp for running our app so we create our own link in
# BUILD_DIR
topic "Creating Nix and bash configurations"
ln -s /nix/var/nix/profiles/default $BUILD_DIR/.nix-profile
# Add our executable scripts etc. to the path for easy access from Procfile and
# one-off build dynos.
mkdir -p $BUILD_DIR/.profile.d
cat <<EOF > $BUILD_DIR/.profile.d/000_nix.sh
export PATH=/app/.nix-profile/bin:/app/bin:\$PATH
EOF
