# 自定义修改记录

本文档记录了对上游仓库 [six2dez/burp-ai-agent](https://github.com/six2dez/burp-ai-agent) 的自定义修改，便于后续同步上游更新时快速重新应用这些改动。

---

## 修改概览

| 功能 | 描述 |
|------|------|
| Base64 编码修复 | 修复聊天会话持久化的 XML 特殊字符问题 |
| iFlow CLI 后端 | 新增 iFlow CLI 作为可选后端 |

---

## 修改 1: Base64 编码修复

### 问题
Java Preferences API 使用 XML 特殊字符 (`\u001D`, `\u001E`, `\u001F`) 导致序列化异常。

### 解决方案
使用 Base64 编码存储聊天消息和草稿，向后兼容旧格式。

### 修改的文件
- `src/main/kotlin/com/six2dez/burp/aiagent/ui/ChatPanel.kt`

### 具体改动

1. 添加 import:
```kotlin
import java.util.Base64
```

2. 修改消息存储逻辑 (saveSessions 方法):
```kotlin
// Save messages for each session (base64 encoded to avoid XML invalid chars)
for (s in sessionsById.values) {
    val msgData = s.messages.joinToString("\n") { msg ->
        "${msg.role}:${Base64.getEncoder().encodeToString(msg.content.toByteArray(Charsets.UTF_8))}"
    }
    prefs.setString(SESSION_MSG_KEY_PREFIX + s.id, msgData)
}
val draftsData = sessionsById.keys.map { id ->
    val draft = Base64.getEncoder().encodeToString(sessionDrafts[id].orEmpty().toByteArray(Charsets.UTF_8))
    "$id:$draft"
}
```

3. 修改草稿加载逻辑 (restoreSessions 方法):
```kotlin
val draftsRaw = prefs.getString(SESSION_DRAFTS_KEY).orEmpty()
if (draftsRaw.isNotBlank()) {
    draftsRaw.split('\n')
        .filter { it.isNotBlank() }
        .forEach { entry ->
            val colonIdx = entry.indexOf(':')
            if (colonIdx > 0) {
                val id = entry.substring(0, colonIdx)
                if (!sessionsById.containsKey(id)) return@forEach
                val payload = entry.substring(colonIdx + 1)
                sessionDrafts[id] = try {
                    String(Base64.getDecoder().decode(payload), Charsets.UTF_8)
                } catch (_: IllegalArgumentException) {
                    // Legacy format fallback
                    payload.replace("\u001D", "\n")
                }
            }
        }
}
```

4. 修改消息恢复逻辑 (restoreSessions 方法):
```kotlin
// Restore messages
val messages = mutableListOf<ChatMessage>()
val msgRaw = prefs.getString(SESSION_MSG_KEY_PREFIX + id)
if (!msgRaw.isNullOrBlank()) {
    for (msgEntry in msgRaw.split("\n")) {
        val colonIdx = msgEntry.indexOf(':')
        if (colonIdx > 0) {
            val role = msgEntry.substring(0, colonIdx)
            val payload = msgEntry.substring(colonIdx + 1)
            val text = try {
                String(Base64.getDecoder().decode(payload), Charsets.UTF_8)
            } catch (_: IllegalArgumentException) {
                // Legacy format fallback
                payload.replace("\u001D", "\n")
            }
            val cleaned = text.lines()
                .filterNot { it.lowercase().startsWith("hook registry initialized") }
                .joinToString("\n")
                .trim()
            if (cleaned.isNotBlank()) {
                messages.add(ChatMessage(role, cleaned))
            }
        }
    }
}
```

---

## 修改 2: iFlow CLI 后端支持

### 功能
将 iFlow CLI 添加为可选的 AI 后端，支持会话继续功能。

### 新增文件

1. `src/main/kotlin/com/six2dez/burp/aiagent/backends/cli/IflowcliBackendFactory.kt`:
```kotlin
package com.six2dez.burp.aiagent.backends.cli

import com.six2dez.burp.aiagent.backends.AiBackend
import com.six2dez.burp.aiagent.backends.AiBackendFactory

class IflowcliBackendFactory : AiBackendFactory {
    override fun create(): AiBackend = CliBackend(
        id = "iflowcli",
        displayName = "iFlow CLI"
    )
}
```

2. 在 `src/main/resources/META-INF/services/com.six2dez.burp.aiagent.backends.AiBackendFactory` 添加:
```
com.six2dez.burp.aiagent.backends.cli.IflowcliBackendFactory
```

### 修改的文件

#### 2.1 `src/main/kotlin/com/six2dez/burp/aiagent/backends/cli/CliBackend.kt`

1. 在 `isAvailable` 方法添加 iflowcli 命令获取:
```kotlin
"iflowcli" -> settings.iflowcliCmd
```

2. 在 `buildCommand` 方法添加 iflowcli 分支:
```kotlin
"iflowcli" -> {
    val cmd = buildIflowcliCommand(baseCommand)
    cmd to prompt
}
```

3. 添加 `buildIflowcliCommand` 方法:
```kotlin
private fun buildIflowcliCommand(cmd: List<String>): List<String> {
    val base = cmd.firstOrNull() ?: "iflow"
    val extras = cmd.drop(1)
    val args = mutableListOf<String>()
    args.add(base)
    // -c must come before -p
    val currentSessionFile = _cliSessionId.get()
    if (currentSessionFile != null) {
        args.add("-c")
    }
    args.addAll(extras)
    if (!args.contains("-p") && !args.contains("--prompt")) {
        args.add("-p")
    }
    // First message: no session flag needed, iFlow auto-creates session
    // Set session marker after first successful call
    if (currentSessionFile == null) {
        _cliSessionId.set("active")
    }
    return args
}
```

4. 在输出读取 switch 添加 iflowcli:
```kotlin
"iflowcli" -> readIflowcliOutput(stdoutText, text)
```

5. 添加 `readIflowcliOutput` 和 `isIflowcliNoiseLine` 方法:
```kotlin
private fun readIflowcliOutput(stdout: String, prompt: String): String {
    val inputLines = prompt.lines().map { it.trim() }.filter { it.isNotBlank() }.toSet()
    // Remove Execution Info block using regex
    var cleaned = stdout
    // Match <Execution Info> followed by JSON content until </Execution Info>
    val execInfoRegex = Regex(
        "<Execution Info>\\s*\\{[\\s\\S]*?\\s*\\}\\s*</Execution Info>",
        RegexOption.DOT_MATCHES_ALL
    )
    cleaned = execInfoRegex.replace(cleaned, "")
    // Also clean up any remaining JSON lines
    return cleaned.lineSequence()
        .map { it.trim() }
        .filter { it.isNotBlank() && !inputLines.contains(it) }
        .filterNot { isIflowcliNoiseLine(it) }
        .joinToString("\n")
        .trim()
}

private fun isIflowcliNoiseLine(line: String): Boolean {
    val trimmed = line.trim()
    return trimmed.startsWith("<Execution Info>") ||
        trimmed.startsWith("</Execution Info>") ||
        trimmed == "{" ||
        trimmed == "}" ||
        trimmed.startsWith("\"session-id\"") ||
        trimmed.startsWith("\"conversation-id\"") ||
        trimmed.startsWith("\"assistantRounds\"") ||
        trimmed.startsWith("\"executionTimeMs\"") ||
        trimmed.startsWith("\"tokenUsage\"") ||
        trimmed.startsWith("\"input\"") ||
        trimmed.startsWith("\"output\"") ||
        trimmed.startsWith("\"total\"")
}
```

#### 2.2 `src/main/kotlin/com/six2dez/burp/aiagent/backends/BackendRegistry.kt`

1. 添加 import:
```kotlin
import com.six2dez.burp.aiagent.backends.cli.IflowcliBackendFactory
```

2. 在 fallback 列表添加:
```kotlin
IflowcliBackendFactory()
```

#### 2.3 `src/main/kotlin/com/six2dez/burp/aiagent/supervisor/AgentSupervisor.kt`

在 `buildLaunchConfig` 方法添加 iflowcli 分支:
```kotlin
"iflowcli" -> {
    val cmd = (settings?.iflowcliCmd ?: prefs.getString("iflowcli.cmd") ?: "iflow").trim()
    val iflowEnv = embeddedCliEnv(baseEnv, embeddedMode) + mapOf("IFLOW_coreTools" to "")
    BackendLaunchConfig(
        backendId = backendId,
        displayName = "iFlow CLI",
        command = tokenizeCommand(cmd),
        embeddedMode = embeddedMode,
        sessionId = sessionId,
        determinismMode = determinism,
        env = iflowEnv,
        cliSessionId = cliSessionId
    )
}
```

#### 2.4 `src/main/kotlin/com/six2dez/burp/aiagent/config/AgentSettings.kt`

1. 在 `AgentSettings` data类添加字段:
```kotlin
val iflowcliCmd: String = "",
```

2. 在 `load` 方法添加:
```kotlin
iflowcliCmd = prefs.getString(KEY_IFLOWCLI_CMD).orEmpty().trim().ifBlank { defaultIflowcliCmd() },
```

3. 在 `defaultSettings` 添加:
```kotlin
iflowcliCmd = defaultIflowcliCmd(),
```

4. 在 `save` 方法添加:
```kotlin
prefs.setString(KEY_IFLOWCLI_CMD, settings.iflowcliCmd)
```

5. 添加常量和方法:
```kotlin
private const val KEY_IFLOWCLI_CMD = "iflowcli.cmd"

private fun defaultIflowcliCmd(): String {
    return "iflow"
}
```

#### 2.5 `src/main/kotlin/com/six2dez/burp/aiagent/ui/panels/BackendConfigPanel.kt`

1. 在 `BackendConfigState` data class 添加字段:
```kotlin
val iflowcliCmd: String = ""
```

2. 添加 UI 组件:
```kotlin
private val iflowcliCmd = JTextField(initialState.iflowcliCmd)
```

3. 在 init 坼添加样式和 tooltip:
```kotlin
applyFieldStyle(iflowcliCmd)
iflowcliCmd.toolTipText = "Command used to launch iFlow CLI (e.g., iflow). Core tools will be disabled."
```

4. 添加卡片:
```kotlin
cards.add(buildSingleFieldPanelWithCli("iFlow CLI command", iflowcliCmd, "iflowcli") { iflowcliCmd.text.trim() }, "iflowcli")
```

5. 在 `currentBackendSettings` 添加:
```kotlin
iflowcliCmd = iflowcliCmd.text.trim()
```

6. 在 `applyState` 添加:
```kotlin
iflowcliCmd.text = state.iflowcliCmd
```

#### 2.6 `src/main/kotlin/com/six2dez/burp/aiagent/ui/SettingsPanel.kt`

1. 在 `getBackendStateFromSettings` 添加:
```kotlin
iflowcliCmd = settings.iflowcliCmd
```

2. 在 `currentSettings` 添加:
```kotlin
iflowcliCmd = backendState.iflowcliCmd,
```

3. 在 `applySettings` 添加:
```kotlin
iflowcliCmd = updated.iflowcliCmd
```

#### 2.7 `src/main/kotlin/com/six2dez/burp/aiagent/ui/MainTab.kt`

在 `validateBackendSettings` 添加:
```kotlin
"iflowcli" -> if (settings.iflowcliCmd.isBlank()) "iFlow CLI command is empty." else null
```

---

## 同步上游更新的步骤

当上游仓库更新后，执行以下步骤重新应用这些修改：

1. 添加上游仓库（如果还没有）:
   ```bash
   git remote add upstream https://github.com/six2dez/burp-ai-agent.git
   ```

2. 拉取上游更新:
   ```bash
   git fetch upstream
   ```

3. 重置到上游版本:
   ```bash
   git reset --hard upstream/main
   ```

4. 按照本文档重新应用所有修改

5. 提交并推送:
   ```bash
   git add -A && git commit -m "feat: sync with upstream and apply custom modifications" && git push origin main --force
   ```

---

## 备注

- Base64 修复解决了 Java Preferences API 的 XML 序列化问题
- iFlow CLI 后端设置 `IFLOW_coreTools=""` 环境变量以禁用内置工具
- 会话继续使用 `-c` 参数，需要放在 `-p` 参数之前
- 输出过滤使用正则表达式移除整个 `<Execution Info>` 块
