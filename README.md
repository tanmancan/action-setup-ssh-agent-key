# Setup SSH Agent and Key

Configure SSH key and persist the ssh-agent across all steps in a job.

## Inputs

| name          | description                                                        | required | default            |
| ------------- | ------------------------------------------------------------------ | -------- | ------------------ |
| ssh-auth-sock | A custom path to the unix socket to bind the ssh-agent (optional). | no       | /tmp/ssh_auth.sock |
| ssh-key       | The private key to add.                                            | yes      | -                  |

## How to Use

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
          ssh-key: ${{ secrets.SSH_DEPLOY_GH }}

      - name: SSH Command Example
        run: ssh test@example.com

      - name: Another SSH Command Example
        run: ssh test2@example.com
```

### Why use this

Each step in a job runs in its own process. When you start a `ssh-agent` inside a step, the agent binds to a temporary communication socket within the current step's environment. This socket is exported to a variable named `SSH_AUTH_SOCK`

```bash
eval "$(ssh-agent -s)"
```

When you add a SSH key via `ssh-add`, it communicates with the `ssh-agent` via the `$SSH_AUTH_SOCK` variable. Within this same step, you will be able to run ssh command using the key you added. But if you try to run a ssh command in another step, it no longer has access to the ssh-agent socket from the previous step.

To get around this, we will bind our ssh-agent to a socket stored inside the global workflow environment. This way all steps within a job will be able to access the ssh-agent setup inside this action.
