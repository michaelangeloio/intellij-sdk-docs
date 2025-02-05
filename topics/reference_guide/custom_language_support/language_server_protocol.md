<!-- Copyright 2000-2023 JetBrains s.r.o. and contributors. Use of this source code is governed by the Apache 2.0 license. -->

# Language Server Protocol (LSP)

<link-summary>Language Server Protocol (LSP) support in IntelliJ-based IDEs</link-summary>

The [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) is an open-standard protocol developed by Microsoft. It enables communication between development tools and Language Servers.
Language Servers can provide language-specific features such as code completion, documentation, and formatting, which is far easier than implementing language support from scratch.
It also reduces the need for constant maintenance and tracking of changes in relevant languages and tools, making it easier to bring consistent language support to various development environments.

However, the canonical [](custom_language_support.md) provided by IntelliJ Platform still offers a wider range of integration with IDE features than handling and presenting data provided by a Language Server.
Therefore, the LSP approach shouldn't be considered as a replacement for the existing language API, but rather as an added value.

> The integration with the Language Server Protocol is created as an extension to the paid IntelliJ-based IDEs.
> Therefore, plugins utilizing Language Server integration may not be available in Community releases of JetBrains products and Android Studio from Google.
>
{style="note"}

Starting with the 2023.2 release cycle, the LSP API is publicly available as part of the IntelliJ Platform in the following IDEs:
IntelliJ IDEA Ultimate, WebStorm, PhpStorm, PyCharm Professional, DataSpell, RubyMine, CLion, Aqua, DataGrip, GoLand, and Rider.

## Plugin Configuration

To fully utilize the LSP API in a third-party plugin based on the [](tools_gradle_intellij_plugin.md), it is recommended to upgrade the Gradle IntelliJ Plugin to the latest version available.
This plugin will attach the LSP API sources and code documentation to the project.

As LSP became available in the 2023.2 EAP7 of IntelliJ-based IDEs, the plugin must target IntelliJ IDEA Ultimate `2023.2` or later.

Example <path>build.gradle.kts</path> configuration:

```kotlin
plugins {
  // ...
  id("org.jetbrains.intellij") version "%gradle-intellij-plugin-version%"
}

intellij {
  version = "%ijPlatform%"
  type = "IU"
}
```

For projects based on the [](plugin_github_template.md), update the Gradle IntelliJ Plugin to the latest version, and amend the <path>gradle.properties</path> file as follows:

```
platformType = IU
platformVersion = %ijPlatform%
```

The <path>plugin.xml</path> configuration file needs to specify the dependency on the IntelliJ IDEA Ultimate module:

```xml
<idea-plugin>
   <!-- ... -->
   <depends>com.intellij.modules.ultimate</depends>
</idea-plugin>
```

The LSP API sources are bundled in IntelliJ IDEA Ultimate and can be found within the <path>[IDE]/lib/src/src_lsp-openapi.zip</path> archive.

## Supported Features

The LSP support within the IntelliJ Platform covers the following features:

- Since 2023.2:
  - Errors and warnings highlighting ([textDocument/publishDiagnostics](https://microsoft.github.io/language-server-protocol/specification/#textDocument_publishDiagnostics))
  - Quick fixes for errors and warnings ([textDocument/codeAction](https://microsoft.github.io/language-server-protocol/specification/#textDocument_codeAction))
  - Code completion ([textDocument/completion](https://microsoft.github.io/language-server-protocol/specification/#textDocument_completion))
  - Go to Declaration ([textDocument/definition](https://microsoft.github.io/language-server-protocol/specification/#textDocument_definition))
- Since 2023.3:
  - Intention actions ([textDocument/codeAction](https://microsoft.github.io/language-server-protocol/specification/#textDocument_codeAction))
  - Code formatting ([textDocument/formatting](https://microsoft.github.io/language-server-protocol/specification/#textDocument_formatting))
- Since 2023.3.1:
  - Quick documentation ([textDocument/hover](https://microsoft.github.io/language-server-protocol/specification#textDocument_hover))

## Basic Implementation

To implement a minimal LSP plugin, perform the following steps:
- Implement `LspServerSupportProvider` and within the `LspServerSupportProvider.fileOpened()` method, spin up the relevant LSP server descriptor, which can decide if the given file is supported by using the `LspServerDescriptor.isSupportedFile()` check method.
- [Register](plugin_extensions.md#declaring-extensions) it as a `com.intellij.platform.lsp.serverSupportProvider` [Extension Point (EP)](plugin_extension_points.md):
- Tell how to start the server by implementing `LspServerDescriptor.createCommandLine()`.

```kotlin
import com.intellij.platform.lsp.api.LspServerSupportProvider
import com.intellij.platform.lsp.api.ProjectWideLspServerDescriptor

internal class FooLspServerSupportProvider : LspServerSupportProvider {
  override fun fileOpened(project: Project, file: VirtualFile, serverStarter: LspServerStarter) {
    if (file.extension == "foo") {
      serverStarter.ensureServerStarted(FooLspServerDescriptor(project))
    }
  }
}

private class FooLspServerDescriptor(project: Project) : ProjectWideLspServerDescriptor(project, "Foo") {
  override fun isSupportedFile(file: VirtualFile) = file.extension == "foo"
  override fun createCommandLine() = GeneralCommandLine("foo", "--stdio")
}
```

As a reference, check out the [Prisma ORM](https://plugins.jetbrains.com/plugin/20686-prisma-orm) open-source plugin implementation: [Prisma ORM LSP](%gh-ij-plugins%/prisma/src/org/intellij/prisma/ide/lsp)

## Language Server Integration

Language Server is a separate process that analyzes source code and provides language-specific features to development tools.
When creating a plugin that utilizes LSP within the IDE, there are two possibilities for providing a Language Server to end-users:

- Bundle a [Language Server implementation](https://microsoft.github.io/language-server-protocol/implementors/servers/) binary as a resource delivered with a plugin.
- Provide a possibility for users to define the location of the Language Server binary in their environment.

The Prisma ORM plugin presents the first approach, which distributes the `prisma-language-server.js` script and uses a local Node.js interpreter to run it.

For more complex cases, the plugin may request to provide a detailed configuration with a dedicated [Settings](settings_guide.md) implementation.

## Customization

- To fine-tune or disable the implementation of LSP-based features, plugins may override the corresponding properties of the `LspServerDescriptor` class.
  See the properties documentation for more details:
  - Since 2023.2:
    - `lspGoToDefinitionSupport`
    - `lspCompletionSupport`
    - `lspDiagnosticsSupport`
    - `lspCodeActionsSupport`
    - `lspCommandsSupport`
  - Since 2023.3:
    - `lspFormattingSupport`
    - `lspHoverSupport`
- To handle custom (undocumented) requests and notifications from the LSP server, override `LspServerDescriptor.createLsp4jClient` property and the `Lsp4jClient` class according to their documentation.
- To send custom (undocumented) requests and notifications to the LSP server, override `LspServerDescriptor.lsp4jServerClass` property and implement the `LspClientNotification` and/or `LspRequest` classes.
  The documentation in the source code includes implementation examples.

See bundled LSP API source code and its documentation for more information.

## Troubleshooting

All the IDE and LSP server communication logs are passed to the IDE log file.

To include them for preview, add the following entry to the <control>Help | Diagnostic Tools | Debug Log Settings…</control> configuration dialog:

```
#com.intellij.platform.lsp
```

For more information, see the [](ide_infrastructure.md#logging) section.

## Limitations

- The current LSP API implementation assumes that the IDE <-> LSP server communication channel is `stdio`.

## Integration Overview

Integrating the Language Server Protocol (LSP) into a plugin for IntelliJ-based IDEs involves a trade-off between simple and fast language support and a complex custom language support plugin with IDE capabilities.

When considering the LSP-based approach, it is important to assess the following criteria for providing a Language Server to end users:

- OS dependency of the Language Server.
- Availability of the latest version online.
- Compatibility with breaking changes between versions.
- Feasibility of requesting the user to provide the Language Server binary path.

## Sample Plugins

The following bundled open-source plugins make (heavy) use of DOM:

- [Prisma ORM LSP](%gh-ij-plugins%/prisma)
- [Vue.js](%gh-ij-plugins%/vuejs)

Explore 3rd party plugins using LSP on [IntelliJ Platform Explorer](https://jb.gg/ipe?extensions=com.intellij.platform.lsp.serverSupportProvider).

<include from="snippets.md" element-id="missingContent"/>
