# Setup SSH Agent and Key

Configure SSH key and persist the ssh-agent across all steps in a job.

## Inputs

| name          | description                                             | required | default            |
| ------------- | ------------------------------------------------------- | -------- | ------------------ |
| ssh-auth-sock | A custom path to the unix socket to bind the ssh-agent. | no       | /tmp/ssh_auth.sock |
| ssh-key       | The private key to add.                                 | yes      | -                  |

## How to Use

Define a step and set the action via `uses`. You will need pass the private SSH key via the input parameter, `ssh-key`. It is highly recommended you use secrets to do this. You can optionally set a custom path to the unix socket to bind the ssh-agent.

```yaml
- uses: tanmancan/action-setup-ssh-agent-key@v1.0
  with:
    # Optional path to the unix socket
    ssh-auth-sock: /tmp/my_auth.sock
    # Pass in the private key from repository secrets.
    ssh-key: ${{ secrets.PRIVATE_KEY }}
```

Example in an workflow.

```yaml
name: CI
on:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - uses: tanmancan/action-setup-ssh-agent-key@v1.0
        with:
          # Optional path to the unix socket
          ssh-auth-sock: /tmp/my_auth.sock
          # Pass in the private key from repository secrets.
          ssh-key: ${{ secrets.PRIVATE_KEY }}

      - name: SSH Command Example
        run: ssh -T test@example.com
        ...

      - name: Another SSH Command Example
        run: ssh -T test2@example.com
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
