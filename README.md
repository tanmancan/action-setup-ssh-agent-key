# Setup SSH Agent and Key

Configure SSH key and persist the ssh-agent across all steps in a job.

## Inputs

| name            | description                                                                      | default            |
| --------------- | -------------------------------------------------------------------------------- | ------------------ |
| ssh-auth-sock   | Use a custom unix socket to bind the ssh-agent.                                  | /tmp/ssh_auth.sock |
| ssh-private-key | Add a private key to the ssh-agent.                                              | -                  |
| ssh-public-key  | Add a public key to known_hosts. Format should be "{hostname} {key-type} {key}". | -                  |

## How to Use

### `ssh-auth-sock`

Set a custom path for the Unix socket. If none provided, a default path will be used. The ssh-agent will bind to this socket via `ssh-agent -a [SOCKET_PATH]`. You can only set this value once. After the agent is started, you can no longer modify the socket path.

```yaml
- uses: tanmancan/action-setup-ssh-agent-key@1.0.0
  with:
    # Optional path to the unix socket
    ssh-auth-sock: /tmp/my_custom.sock

- uses: tanmancan/action-setup-ssh-agent-key@1.0.0
  with:
    # This will not do anything since the agent is already running
    ssh-auth-sock: /tmp/a_different.sock
```

### `ssh-private-key`

Adds a private key to the agent via `ssh-add`. Highly recommended that you use a [secret to store this value](https://docs.github.com/en/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository). If you want to add multiple private keys, you have to do it via separate `uses` command:

```yaml
# Set first private key
- uses: tanmancan/action-setup-ssh-agent-key@1.0.0
  with:
    ssh-private-key: ${{ secrets.PRIVATE_KEY_ONE }}
  # Key for a first server
  run: ssh -T example@first.example.com "some_command"

# Set second private key
- uses: tanmancan/action-setup-ssh-agent-key@1.0.0
  with:
    ssh-private-key: ${{ secrets.PRIVATE_KEY_TWO }}
  # Key for a second server
  run: ssh -T example2@second.example.com "some_command"
```

### `ssh-public-key`

Adds a public key to the `~/.ssh/known_hosts` file. This helps verify the identity of the remote server. The format for this should be `{hostname} {key-type} {key}`:

```
server.example.com ssh-rsa AAAAabb1234abcd...
```

It is highly recommended you use a [secret to store this value](https://docs.github.com/en/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository). You may add multiple public key:

```yaml
# Set first public key
- uses: tanmancan/action-setup-ssh-agent-key@1.0.0
  with:
    ssh-public-key: ${{ secrets.PUBLIC_KEY_ONE }}
# Set second public key
- uses: tanmancan/action-setup-ssh-agent-key@1.0.0
  with:
    ssh-public-key: ${{ secrets.PUBLIC_KEY_TWO }}
```

### Example in an workflow.

```yaml
name: CI
on:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - uses: tanmancan/action-setup-ssh-agent-key@1.0.0
        with:
          ssh-auth-sock: /tmp/my_auth.sock
          ssh-private-key: ${{ secrets.PRIVATE_KEY_ONE }}
          ssh-public-key: ${{ secrets.PUBLIC_KEY_ONE }}

      - name: SSH Command Example
        run: ssh -T test@example.com
        ...

      # Adds new private and public key to use with a second remote system
      - uses: tanmancan/action-setup-ssh-agent-key@1.0.0
        with:
          ssh-private-key: ${{ secrets.PRIVATE_KEY_TWO }}
          ssh-public-key: ${{ secrets.PUBLIC_KEY_TWO }}

      - name: Another SSH Command Example
        run: ssh -T test2@second.example.com
        ...
```

## Why use this

### Summary

Each step runs in its own process and has its own environment variable. This action exports `SSH_AUTH_SOCK` from a step to a global variable, so any subsequent steps can communicate with the `ssh-agent`.

### How the `ssh-agent` communicates

When you start a `ssh-agent` via `` eval `ssh-agent` ``, it will bind to a unix socket located in `/tmp/ssh-XXXXXXXXXX/agent.pid`. This socket will be exported to a variable called `SSH_AUTH_SOCK`.

Other process will look for this socket in `SSH_AUTH_SOCK` and use it to communicate with the agent. For example, `ssh-add` uses the socket to add identities to the agent.

If the agent is not running, or if the `SSH_AUTH_SOCK` is empty, you may see the following error:

```bash
ssh-add
# If SSH_AUTH_SOCK is empty, then outputs
Could not open a connection to your authentication agent.
```

### Steps in a job do not share environment

Each step in a job runs in its own process and has is own environment. This means if you start the `ssh-agent`in one step, other steps won't be able to use the exported `SSH_AUTH_SOCK`.

```yaml
# Starts ssh-agent, adds keys, echo SSH_AUTH_SOCK, and
# ssh into github.
- name: Configure SSH
  run: |
    eval `ssh-agent -s`
    ssh-add - <<< "${{ secrets.PRIVATE_KEY }}"

    echo $SSH_AUTH_SOCK
    ssh -T git@github.com
  continue-on-error: true

# Echoes SSH_AUTH_SOCK and ssh into github
- name: Run SSH
  run: |
    echo $SSH_AUTH_SOCK
    ssh -T git@github.com
  continue-on-error: true
```

The first step will print the location of the `SSH_AUTH_SOCK` and a success message from Github:

```
...
/tmp/ssh-XXXXXXXXX/agent.XXXX
...
Hi user/testing! You've successfully authenticated, but GitHub does not provide shell access
```

But the second step will print an empty value for `SSH_AUTH_SOCK` and a permission denied message from Github.

```yaml
# Empty line from echo SSH_AUTH_SOCK

git@github.com: Permission denied (publickey).
```

### Using workflow environment to share data between steps.

To get around this, we can export our local `SSH_AUTH_SOCK` to a global workflow environment. This way all steps within a job will be able to access the `SSH_AUTH_SOCK`.

To export a local variable to the workflow environment, you can run the following command:

```bash
echo "{name}={value}" >> $GITHUB_ENV
```

```yaml
- name: Configure SSH
  run: |
    eval `ssh-agent -s`
    ssh-add - <<< "${{ secrets.PRIVATE_KEY }}"

    # Export SSH_AUTH_SOCK to the workflow env
    echo "SSH_AUTH_SOCK=${SSH_AUTH_SOCK}" >> $GITHUB_ENV
  ...

# Echoes SSH_AUTH_SOCK and ssh into github
- name: Run SSH
  run: |
    echo $SSH_AUTH_SOCK
    ssh -T git@github.com
  continue-on-error: true
```

The second will now be print the `SSH_AUTH_SOCK` and successfully ssh into Github.

```
/tmp/ssh-XXXXXXXXX/agent.XXXX
...
Hi user/testing! You've successfully authenticated, but GitHub does not provide shell access.
```
