# Self-hosted Equinix Metal arm64 runner

This GitHub action is special: it's specific to the self-hosted runner on
which it's supposed to take place. Indeed on this self-hosted runner, the
action runs directly as a superuser without any containerization mechanism or
isolation in the filesystem.

Thus, the action must clean after itself and can assume that some tools are
already installed. It's better to install everything before because locking
mechanisms, like the one you have in apt-get can break the workflow if you
have multiple parallels runners on the same machine.

## Installation

Here are the steps to prepare the runner:

1. Spawn a https://deploy.equinix.com/product/servers/c3-large-arm64/ server
   and ssh into it.

1. Prepare the runner
   - Install LVH and GHA dependencies
     ```bash
     apt-get update
     apt-get install -y qemu-kvm jq docker.io
     ```
   - Install crane
     ```bash
     VERSION=v0.19.1
     ARCH=$([ "$(uname -m)" == "aarch64" ] && echo "arm64" || echo $(uname -m))
     URL="https://github.com/google/go-containerregistry/releases/download/$VERSION/go-containerregistry_Linux_$ARCH.tar.gz"
     curl -fSL $URL | sudo tar -xz -C /usr/local/bin crane
     crane version
     ```
   - Set up a basic firewall
     ```bash
     sudo ufw allow OpenSSH
     sudo ufw --force enable
     ```
   - Configure sshd
     ```bash
     vim /etc/ssh/sshd_config
     ```
     Put those two values
     ```bash
     PasswordAuthentication no
     PubkeyAuthentication yes
     ```
     Restart the ssh service
     ```bash
     systemctl restart ssh
     ```

1. Clone the repo
   ```bash
   git clone https://github.com/vbem/multi-runners.git
   cd multi-runners
   ```

1. Generate a PAT for the repository with the `administration:write` permission.

1. Create an env file with the PAT and correct URL for arm64 or better export
   those so that they are not
   ```bash
   tee .env <<EOF
   MR_RELEASE_URL='https://github.com/actions/runner/releases/download/v2.315.0/actions-runner-linux-arm64-2.315.0.tar.gz'
   EOF
   read -rs MR_GITHUB_PAT
   export MR_GITHUB_PAT
   ```

1. Setup org and repo
   ```bash
   export ORG=<org>
   export REPO=<repo>
   ```

1. Add one runner (for initial download, etc.)
   ```bash
    ./mr.bash add --org $ORG --repo $REPO --labels splendid-metal-enormous-arm64 --user runner-1
   ```

1. Continue prepare the runner:
   - Create shared folder
     ```bash
     mkdir -p /home/runners && chown :runners /home/runners && chmod 774 /home/runners
     ```

1. Add the rest of the runners
   ```bash
   for i in {2..10}; do ./mr.bash add --org $ORG --repo $REPO --labels splendid-metal-enormous-arm64 --user runner-$i & done
   ```


## Cleanup

1. Remove the runners
   ```bash
   for i in {1..10}; do ./mr.bash del --user runner-$i & done
   ```
   Or delete them on the UI on GitHub

1. Delete the server

