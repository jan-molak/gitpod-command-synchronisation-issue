# Gitpod command synchronisation issue

This repository demonstrates what I believe could be an issue with `task.command` synchronisation in Gitpod.io.

The issue can be reproduced using the below Gitpod workspace:

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/jan-molak/gitpod-command-synchronisation-issue)

## Context

I'd like to install custom VSCode extensions when the workspace is launched.
Those extensions are private and not available in the Open VSX registry (yet), so can't be defined using `vscode.extensions` in `.gitpod.yml`

However, since the issue can be reproduced with both private and public extensions, in the rest of this article my examples will use publicly-available extensions to keep the number of variables to an absolute minimum sufficient to reproduce the issue

To install an extension, I followed the guidance in the [Gitpod.io manual](https://www.gitpod.io/docs/ides-and-editors/vscode-extensions#where-do-i-find-extensions)
and [VSCode manual](https://code.visualstudio.com/docs/editor/extension-marketplace#_command-line-extension-management) and concluded, that I need the following command:

```
code --install-extension EditorConfig.EditorConfig
```

## The issue

Since the `--install-extension` command requires VSCode to be running, I understand that the best way to invoke it on Gitpod is to use a [start task](https://www.gitpod.io/docs/config-start-tasks). For example:

```yaml
tasks:
- name: "Install VSCode Extension - BROKEN"
  command: |
    code --install-extension EditorConfig.EditorConfig
```

However, defining the task as per the above listing results in the below error displayed in Gitpod terminal:

```
 HISTFILE=/workspace/.gitpod/cmd-0 history -r; {
code --install-extension EditorConfig.EditorConfig

}
gitpod /workspace/gitpod-command-synchronisation-issue (main) $  HISTFILE=/workspace/.gitpod/cmd-0 history -r; {
> code --install-extension EditorConfig.EditorConfig
> 
> }
Unable to connect to VS Code server: Error in request.
Error: connect ECONNREFUSED 127.0.0.1:23000
    at TCPConnectWrap.afterConnect [as oncomplete] (node:net:1161:16) {
  errno: -111,
  code: 'ECONNREFUSED',
  syscall: 'connect',
  address: '127.0.0.1',
  port: 23000
}
```

I believe that the reason for this error is that VSCode is still booting up when the task is executed.

## Experiment

To verify this hypothesis I prepared two tasks: one invoked straight away, and the other invoked after a slight delay which I thought should be sufficient to allow for VSCode to launch:

```yaml
tasks:
  - name: "Install VSCode Extension - BROKEN"
    command: |
      echo "This command is invoked before Code is launched, so it fails:"
      code --install-extension EditorConfig.EditorConfig

  - name: "Install VSCode Extension - WORKS"
    command: |
      echo "This command waits for a couple of seconds for Code to launch, and succeeds:"
      sleep 10                                              # Wait for VSCode to boot up
      code --install-extension dbaeumer.vscode-eslint
```

While the first task has failed as expected, the second one succeeded with the following output:

```
 HISTFILE=/workspace/.gitpod/cmd-1 history -r; {
sleep 10                                              # Wait for VSCode to boot up
code --install-extension dbaeumer.vscode-eslint

}
mv: cannot remove '/home/gitpod/.pyenv/version': No such file or directory
gitpod /workspace/gitpod-command-synchronisation-issue (main) $  HISTFILE=/workspace/.gitpod/cmd-1 history -r; {
> sleep 10                                              # Wait for VSCode to boot up
> code --install-extension dbaeumer.vscode-eslint
> 
> }
"Installing extensions on janmolak-gitpodcommands-9bcrz57ywj1.ws-eu62.gitpod.io...\nInstalling extension 'dbaeumer.vscode-eslint'...\nExtension 'dbaeumer.vscode-eslint' v2.2.6 was successfully installed."
gitpod /workspace/gitpod-command-synchronisation-issue (main) $ 
```

## Conclusion

I believe that `task.command` does not wait for VSCode to launch, and might need to be synchronised to allow for custom extensions to be installed without having to introduce an artificial delay. 
