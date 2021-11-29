---
title: プラグインでkubectlを拡張する
description: kubectlプラグインを作るまたはインストールすることによってkubectlを拡張する
content_type: task
---

<!-- overview -->

このガイドは、[kubectl](/docs/reference/kubectl/kubectl/) の拡張機能をインストールまたは書く方法を実演します。
Kubernetesクラスターと相互作用するための不可欠な要素として `kubectl` コマンドの中核部分を考えることによって、クラスター管理者はこれらの構成要素を利用してより複雑な動作を実現する手段としてプラグインを考えることができます。
プラグインは `kubectl` を新しいサブコマンドによって拡張し、`kubectl` の主要なディストリビューションに含まれていない新しいカスタム機能を提供します。


## {{% heading "prerequisites" %}}

動作する `kubectl` のバイナリがインストールされている必要があります。

<!-- steps -->

## kubectlプラグインをインストールする

プラグインは、名前が `kubectl-` から始まるスタンドアロンな実行ファイルです。
プラグインをインストールするためには、その実行ファイルを `PATH` のどこかに移動します。

[Krew](https://krew.dev/) を使ってオープンソースとして利用可能なkubectlプラグインを探してインストールできます。
Krewは、Kubernetes SIG CLIコミュニティによってメンテナンスされているプラグインマネージャです。

{{< caution >}}
[plugin index](https://krew.sigs.k8s.io/plugins/) 経由で利用可能なkubectlプラグインは、セキュリティに対して監査されていません。
任意のプログラムをマシン上で動作することになるため、自己責任でサードパーティのプラグインをインストールし実行すべきです。
{{< /caution >}}


### プラグインを見つける

`kubectl` は有効なプラグイン実行ファイルのために `PATH` を検索する `kubectl plugin list` コマンドを提供しています。
このコマンドを実行することは、`PATH` にあるすべてファイルを走査することになります。
`kubectl-` で始まる実行可能なファイルが、*`PATH` に存在する順番で* コマンドの出力として表示されます。
警告メッセージには、`kubectl-` で始まる *実行可能ではない* ファイルが含まれるでしょう。
また警告メッセージには、互いに他の名前と重複する有効なプラグインのファイルも含まれます。

コミュニティによって精選された [plugin index](https://krew.sigs.k8s.io/plugins/) から、`kubectl` プラグインを見つけたりインストールしたりするために [Krew](https://krew.dev/) を使うことができます。


#### 制限

既存の `kubectl` コマンドを上書きするプラグインを作ることは現時点で不可能です。
例えば、`kubectl-version` プラグインを作った場合、すでに存在している `kubectl version` が常に優先順位が高くなるため、プラグインは実行されないでしょう。
この制限により、`kubectl` コマンドに対してサブコマンドを新しく追加するプラグインを利用することも *できません*。
例えば、プラグインを `kubectl-create-foo` と名付けてサブコマンド `kubectl create foo` を追加しようとしても `kubectl-create-foo` はプラグインを無視するでしょう。

`kubectl plugin list` は、このようなことを試みる有効なプラグインに対して警告を表示します。



＠＠＠ここから


## Writing kubectl plugins

You can write a plugin in any programming language or script that allows you to write command-line commands.

There is no plugin installation or pre-loading required. Plugin executables receive
the inherited environment from the `kubectl` binary.
A plugin determines which command path it wishes to implement based on its name.
For example, a plugin named `kubectl-foo` provides a command `kubectl foo`. You must
install the plugin executable somewhere in your `PATH`.

### Example plugin

```bash
#!/bin/bash

# optional argument handling
if [[ "$1" == "version" ]]
then
    echo "1.0.0"
    exit 0
fi

# optional argument handling
if [[ "$1" == "config" ]]
then
    echo "$KUBECONFIG"
    exit 0
fi

echo "I am a plugin named kubectl-foo"
```

### Using a plugin

To use a plugin, make the plugin executable:

```shell
sudo chmod +x ./kubectl-foo
```

and place it anywhere in your `PATH`:

```shell
sudo mv ./kubectl-foo /usr/local/bin
```

You may now invoke your plugin as a `kubectl` command:

```shell
kubectl foo
```

```
I am a plugin named kubectl-foo
```

All args and flags are passed as-is to the executable:

```shell
kubectl foo version
```

```
1.0.0
```

All environment variables are also passed as-is to the executable:

```bash
export KUBECONFIG=~/.kube/config
kubectl foo config
```

```
/home/<user>/.kube/config
```

```shell
KUBECONFIG=/etc/kube/config kubectl foo config
```

```
/etc/kube/config
```

Additionally, the first argument that is passed to a plugin will always be the full path to the location where it was invoked (`$0` would equal `/usr/local/bin/kubectl-foo` in the example above).

### Naming a plugin

As seen in the example above, a plugin determines the command path that it will implement based on its filename. Every sub-command in the command path that a plugin targets, is separated by a dash (`-`).
For example, a plugin that wishes to be invoked whenever the command `kubectl foo bar baz` is invoked by the user, would have the filename of `kubectl-foo-bar-baz`.

#### Flags and argument handling

{{< note >}}
The plugin mechanism does _not_ create any custom, plugin-specific values or environment variables for a plugin process.

An older kubectl plugin mechanism provided environment variables such as `KUBECTL_PLUGINS_CURRENT_NAMESPACE`; that no longer happens.
{{< /note >}}

kubectl plugins must parse and validate all of the arguments passed to them.
See [using the command line runtime package](#using-the-command-line-runtime-package) for details of a Go library aimed at plugin authors.

Here are some additional cases where users invoke your plugin while providing additional flags and arguments. This builds upon the `kubectl-foo-bar-baz` plugin from the scenario above.

If you run `kubectl foo bar baz arg1 --flag=value arg2`, kubectl's plugin mechanism will first try to find the plugin with the longest possible name, which in this case
would be `kubectl-foo-bar-baz-arg1`. Upon not finding that plugin, kubectl then treats the last dash-separated value as an argument (`arg1` in this case), and attempts to find the next longest possible name, `kubectl-foo-bar-baz`.
Upon having found a plugin with this name, kubectl then invokes that plugin, passing all args and flags after the plugin's name as arguments to the plugin process.

Example:

```bash
# create a plugin
echo -e '#!/bin/bash\n\necho "My first command-line argument was $1"' > kubectl-foo-bar-baz
sudo chmod +x ./kubectl-foo-bar-baz

# "install" your plugin by moving it to a directory in your $PATH
sudo mv ./kubectl-foo-bar-baz /usr/local/bin

# check that kubectl recognizes your plugin
kubectl plugin list
```

```
The following kubectl-compatible plugins are available:

/usr/local/bin/kubectl-foo-bar-baz
```

```
# test that calling your plugin via a "kubectl" command works
# even when additional arguments and flags are passed to your
# plugin executable by the user.
kubectl foo bar baz arg1 --meaningless-flag=true
```

```
My first command-line argument was arg1
```

As you can see, your plugin was found based on the `kubectl` command specified by a user, and all extra arguments and flags were passed as-is to the plugin executable once it was found.

#### Names with dashes and underscores

Although the `kubectl` plugin mechanism uses the dash (`-`) in plugin filenames to separate the sequence of sub-commands processed by the plugin, it is still possible to create a plugin
command containing dashes in its commandline invocation by using underscores (`_`) in its filename.

Example:

```bash
# create a plugin containing an underscore in its filename
echo -e '#!/bin/bash\n\necho "I am a plugin with a dash in my name"' > ./kubectl-foo_bar
sudo chmod +x ./kubectl-foo_bar

# move the plugin into your $PATH
sudo mv ./kubectl-foo_bar /usr/local/bin

# You can now invoke your plugin via kubectl:
kubectl foo-bar
```

```
I am a plugin with a dash in my name
```

Note that the introduction of underscores to a plugin filename does not prevent you from having commands such as `kubectl foo_bar`.
The command from the above example, can be invoked using either a dash (`-`) or an underscore (`_`):

```bash
# You can invoke your custom command with a dash
kubectl foo-bar
```

```
I am a plugin with a dash in my name
```

```bash
# You can also invoke your custom command with an underscore
kubectl foo_bar
```

```
I am a plugin with a dash in my name
```

#### Name conflicts and overshadowing

It is possible to have multiple plugins with the same filename in different locations throughout your `PATH`.
For example, given a `PATH` with the following value: `PATH=/usr/local/bin/plugins:/usr/local/bin/moreplugins`, a copy of plugin `kubectl-foo` could exist in `/usr/local/bin/plugins` and `/usr/local/bin/moreplugins`,
such that the output of the `kubectl plugin list` command is:

```bash
PATH=/usr/local/bin/plugins:/usr/local/bin/moreplugins kubectl plugin list
```

```
The following kubectl-compatible plugins are available:

/usr/local/bin/plugins/kubectl-foo
/usr/local/bin/moreplugins/kubectl-foo
  - warning: /usr/local/bin/moreplugins/kubectl-foo is overshadowed by a similarly named plugin: /usr/local/bin/plugins/kubectl-foo

error: one plugin warning was found
```

In the above scenario, the warning under `/usr/local/bin/moreplugins/kubectl-foo` tells you that this plugin will never be executed. Instead, the executable that appears first in your `PATH`, `/usr/local/bin/plugins/kubectl-foo`, will always be found and executed first by the `kubectl` plugin mechanism.

A way to resolve this issue is to ensure that the location of the plugin that you wish to use with `kubectl` always comes first in your `PATH`. For example, if you want to always use `/usr/local/bin/moreplugins/kubectl-foo` anytime that the `kubectl` command `kubectl foo` was invoked, change the value of your `PATH` to be `/usr/local/bin/moreplugins:/usr/local/bin/plugins`.

#### Invocation of the longest executable filename

There is another kind of overshadowing that can occur with plugin filenames. Given two plugins present in a user's `PATH`: `kubectl-foo-bar` and `kubectl-foo-bar-baz`, the `kubectl` plugin mechanism will always choose the longest possible plugin name for a given user command. Some examples below, clarify this further:

```bash
# for a given kubectl command, the plugin with the longest possible filename will always be preferred
kubectl foo bar baz
```

```
Plugin kubectl-foo-bar-baz is executed
```

```bash
kubectl foo bar
```

```
Plugin kubectl-foo-bar is executed
```

```bash
kubectl foo bar baz buz
```

```
Plugin kubectl-foo-bar-baz is executed, with "buz" as its first argument
```

```bash
kubectl foo bar buz
```

```
Plugin kubectl-foo-bar is executed, with "buz" as its first argument
```

This design choice ensures that plugin sub-commands can be implemented across multiple files, if needed, and that these sub-commands can be nested under a "parent" plugin command:

```bash
ls ./plugin_command_tree
```

```
kubectl-parent
kubectl-parent-subcommand
kubectl-parent-subcommand-subsubcommand
```

### Checking for plugin warnings

You can use the aforementioned `kubectl plugin list` command to ensure that your plugin is visible by `kubectl`, and verify that there are no warnings preventing it from being called as a `kubectl` command.

```bash
kubectl plugin list
```

```
The following kubectl-compatible plugins are available:

test/fixtures/pkg/kubectl/plugins/kubectl-foo
/usr/local/bin/kubectl-foo
  - warning: /usr/local/bin/kubectl-foo is overshadowed by a similarly named plugin: test/fixtures/pkg/kubectl/plugins/kubectl-foo
plugins/kubectl-invalid
  - warning: plugins/kubectl-invalid identified as a kubectl plugin, but it is not executable

error: 2 plugin warnings were found
```

### Using the command line runtime package

If you're writing a plugin for kubectl and you're using Go, you can make use
of the
[cli-runtime](https://github.com/kubernetes/cli-runtime) utility libraries.

These libraries provide helpers for parsing or updating a user's
[kubeconfig](/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
file, for making REST-style requests to the API server, or to bind flags
associated with configuration and printing.

See the [Sample CLI Plugin](https://github.com/kubernetes/sample-cli-plugin) for
an example usage of the tools provided in the CLI Runtime repo.

## Distributing kubectl plugins

If you have developed a plugin for others to use, you should consider how you
package it, distribute it and deliver updates to your users.

### Krew {#distributing-krew}

[Krew](https://krew.dev/) offers a cross-platform way to package and
distribute your plugins. This way, you use a single packaging format for all
target platforms (Linux, Windows, macOS etc) and deliver updates to your users.
Krew also maintains a [plugin
index](https://krew.sigs.k8s.io/plugins/) so that other people can
discover your plugin and install it.


### Native / platform specific package management {#distributing-native}

Alternatively, you can use traditional package managers such as, `apt` or `yum`
on Linux, Chocolatey on Windows, and Homebrew on macOS. Any package
manager will be suitable if it can place new executables placed somewhere
in the user's `PATH`.
As a plugin author, if you pick this option then you also have the burden
of updating your kubectl plugin's distribution package across multiple
platforms for each release.

### Source code {#distributing-source-code}

You can publish the source code; for example, as a Git repository. If you
choose this option, someone who wants to use that plugin must fetch the code,
set up a build environment (if it needs compiling), and deploy the plugin.
If you also make compiled packages available, or use Krew, that will make
installs easier.

## {{% heading "whatsnext" %}}

* Check the Sample CLI Plugin repository for a
  [detailed example](https://github.com/kubernetes/sample-cli-plugin) of a
  plugin written in Go.
  In case of any questions, feel free to reach out to the
  [SIG CLI team](https://github.com/kubernetes/community/tree/master/sig-cli).
* Read about [Krew](https://krew.dev/), a package manager for kubectl plugins.
