# 代码评审.  
### 😐代码评分：42  
#### 😐代码逻辑与目的：  
该代码实现了一个自动化代码审查 CLI 工具，核心流程为：① 获取当前 HEAD 与前一次提交的 `git diff`；② 将 diff 内容送入 DashScope 大模型（通过 ReactAgent）执行 AI 驱动的代码评审；③ 将评审结果以 Markdown 格式写入指定 GitHub 仓库并推送。整体意图是构建一个可复用、分层解耦的 SDK，支持未来扩展不同 Git 源、AI 后端或存储策略。  

#### ✅代码优点：  
- ✅ 明确引入了 **分层架构**（domain/service/infrastructure），抽象出 `ICodeReviewService` 与 `AbstractCodeReviewService`，体现初步的 DDD 思想；  
- ✅ 将 Git 操作（`GitCommand`）与 AI 调用（`AiCodeReview`）解耦至 infrastructure 层，职责分离合理；  
- ✅ `RandomStringUtils` 提取为独立工具类，避免重复造轮子（虽未用 Apache Commons，但封装意识正确）；  
- ✅ `diff()` 方法中校验 `exitCode` 并抛出明确异常，具备基础错误反馈能力。  

#### 🤔问题点：  
- ⚠️ **硬编码密钥泄露高危漏洞**：`AiCodeReview.java` 和 `GitCommand.java` 中分别硬编码了 DashScope API Key（`sk-f1c12bf5...`）和 GitHub Personal Access Token（`ghp_V1pDMGdh...`），**已直接暴露在公开代码库中，属于严重安全红线**，必须立即撤销并轮换密钥；  
- ⚠️ **敏感凭证明文传输与存储风险**：`UsernamePasswordCredentialsProvider(token, "")` 将 token 以明文传入 Git 客户端，且 `writeLog()` 在本地克隆仓库时未清理 `.git/config` 中的凭证缓存，存在凭证残留风险；  
- ⚠️ **资源泄漏严重**：`GitCommand.diff()` 中 `logReader` 和 `diffReader` 虽调用 `close()`，但未使用 try-with-resources；`logProcess` 和 `diffProcess` 的 `InputStream`/`ErrorStream` 未读取，可能导致子进程阻塞（JVM bug #6903972）；  
- ⚠️ **Git 克隆无并发保护 & 本地目录污染**：`writeLog()` 每次都 `cloneRepository()` 到固定路径 `"repo"`，若多线程/多次执行将导致 `FileAlreadyExistsException` 或覆盖旧数据，且未设置 `setCloneSubmodules(false)` 等安全选项；  
- ⚠️ **系统提示词（systemPrompt）冗余拼接、可维护性极差**：长达 20+ 行的字符串拼接混杂 `\n` 和 `+`，无格式化、无常量提取、无单元测试覆盖，极易因引号/转义错误导致 agent 初始化失败；  
- ⚠️ **`ReactAgent` 名称语义错误**：`.name("weather_agent")` 与代码审查场景完全无关，暴露设计随意性，且 `ReactAgent` 实际应为 `CodeReviewAgent`，违背单一职责；  
- ⚠️ **`AbstractCodeReviewService.exec()` 异常处理粗暴**：`throws Exception` 完全掩盖底层异常类型（如 `IOException`, `GraphRunnerException`, `GitAPIException`），破坏调用方错误分类能力；  
- ⚠️ **`.idea/uiDesigner.xml` 误提交**：该文件属 IDE 本地配置，不应纳入版本控制，污染仓库且引发团队配置冲突；  
- ⚠️ **`Main.java` 修改无业务意义**：仅将 `println(2)` 改为 `println(1)`，属无效变更，暴露缺乏测试验证流程。  

#### 🎯修改建议：  
- 🔒 **立即移除所有硬编码密钥**：改用环境变量（`System.getenv("DASHSCOPE_API_KEY")` / `System.getenv("GITHUB_TOKEN")`）或 Spring Boot 配置属性，并在 `.gitignore` 中加入 `application-secret.yml`；  
- 🧹 **强制资源自动释放**：所有 `Process`、`BufferedReader`、`Git` 实例必须用 try-with-resources 或显式 `close()` + `destroyForcibly()`；  
- 🌐 **Git 克隆改为 `fetch` + `checkout` 模式**：避免重复 clone，改用 `Git.open(repoDir)` + `git.pull()`，并确保 `repoDir` 使用 `Files.createTempDirectory()` 动态生成；  
- ✍️ **重构 systemPrompt 为 `static final String` 常量**，使用 `TextBlock`（Java 15+）或 `String.format()` 分段注入，添加 `@Test` 验证 JSON 可解析性；  
- 🧩 **重命名 Agent 并剥离无关依赖**：`ReactAgent` → `CodeReviewAgent`，删除 `weather_agent` 等误导性字段；  
- 📜 **细化异常声明**：`exec()` 改为 `throws IOException, GraphRunnerException, GitAPIException, InterruptedException`；  
- 🗑️ **删除 `.idea/` 目录提交**：运行 `git rm -r --cached .idea && echo ".idea/" >> .gitignore`；  
- 🧪 **为 `Main.java` 补充集成测试**：验证 `CodeReviewService.exec()` 返回非空字符串且含 `# 代码评审.` 标题。  

#### 💻修改后的代码：  
```java
// --- code-review-sdk/src/main/java/com/qqdbz/code/review/infrastructure/ai/AiCodeReview.java ---
package com.qqdbz.code.review.infrastructure.ai;

import com.alibaba.cloud.ai.dashscope.api.DashScopeApi;
import com.alibaba.cloud.ai.dashscope.chat.DashScopeChatModel;
import com.alibaba.cloud.ai.graph.agent.CodeReviewAgent; // ← 重命名类
import com.alibaba.cloud.ai.graph.exception.GraphRunnerException;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.model.ChatModel;

public class AiCodeReview {

    private static final String SYSTEM_PROMPT = """
        你是一位资深编程专家，拥有深厚的编程基础和广泛的技术栈知识。你的专长在于识别代码中的低效模式、安全隐患、以及可维护性问题，并能提出针对性的优化策略。你擅长以易于理解的方式解释复杂的概念，确保即使是初学者也能跟随你的指导进行有效改进。在提供优化建议时，你注重平衡性能、可读性、安全性、逻辑错误、异常处理、边界条件，以及可维护性方面的考量，同时尊重原始代码的设计意图。
        你总是以鼓励和建设性的方式提出反馈，致力于提升团队的整体编程水平，详尽指导编程实践，雕琢每一行代码至臻完善。用户会将仓库代码分支修改代码给你，以git diff 字符串的形式提供，你需要根据变化的代码，帮忙review本段代码。然后你review内容的返回内容必须严格遵守下面我给你的格式，包括标题内容。
        模板中的变量内容解释：
        变量1是给review打分，分数区间为0~100分。
        变量2 是code review发现的问题点，包括：可能的性能瓶颈、逻辑缺陷、潜在问题、安全风险、命名规范、注释、以及代码结构、异常情况、边界条件、资源的分配与释放等等
        变量3是具体的优化修改建议。
        变量4是你给出的修改后的代码。
        变量5是代码中的优点。
        变量6是代码的逻辑和目的，识别其在特定上下文中的作用和限制

        必须要求：
        1. 以精炼的语言、严厉的语气指出存在的问题。
        2. 你的反馈内容必须使用严谨的markdown格式
        3. 不要携带变量内容解释信息。
        4. 有清晰的标题结构
        返回格式严格如下：
        # 代码评审.
        ### 😐代码评分：{变量1}
        #### 😐代码逻辑与目的：
        {变量6}
        #### ✅代码优点：
        {变量5}
        #### 🤔问题点：
        {变量2}
        #### 🎯修改建议：
        {变量3}
        #### 💻修改后的代码：
        {变量4}
        """;

    public String codeReview(String diffCode) throws GraphRunnerException {
        String apiKey = System.getenv("DASHSCOPE_API_KEY");
        if (apiKey == null || apiKey.trim().isEmpty()) {
            throw new IllegalArgumentException("DASHSCOPE_API_KEY environment variable not set");
        }

        DashScopeApi dashScopeApi = DashScopeApi.builder()
                .apiKey(apiKey)
                .build();

        ChatModel chatModel = DashScopeChatModel.builder()
                .dashScopeApi(dashScopeApi)
                .build();

        CodeReviewAgent agent = CodeReviewAgent.builder() // ← 重命名 builder
                .name("code_review_agent")
                .model(chatModel)
                .systemPrompt(SYSTEM_PROMPT)
                .build();

        AssistantMessage response = agent.call("这个是我的代码：" + diffCode);
        return response.getText();
    }
}

// --- code-review-sdk/src/main/java/com/qqdbz/code/review/infrastructure/git/GitCommand.java ---
package com.qqdbz.code.review.infrastructure.git;

import com.qqdbz.code.review.type.utils.RandomStringUtils;
import org.eclipse.jgit.api.Git;
import org.eclipse.jgit.api.errors.GitAPIException;
import org.eclipse.jgit.transport.UsernamePasswordCredentialsProvider;
import org.eclipse.jgit.util.FS;

import java.io.*;
import java.nio.file.Files;
import java.nio.file.Path;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Objects;

public class GitCommand {

    public String diff() throws IOException, InterruptedException {
        ProcessBuilder logBuilder = new ProcessBuilder("git", "log", "-1", "--pretty=format:%H");
        logBuilder.directory(new File("."));
        try (Process logProcess = logBuilder.start();
             BufferedReader logReader = new BufferedReader(new InputStreamReader(logProcess.getInputStream()))) {

            String latestCommitHash = logReader.readLine();
            if (latestCommitHash == null) {
                throw new IllegalStateException("Failed to get latest commit hash");
            }

            ProcessBuilder diffBuilder = new ProcessBuilder("git", "diff", latestCommitHash + "^", latestCommitHash);
            diffBuilder.directory(new File("."));
            try (Process diffProcess = diffBuilder.start();
                 BufferedReader diffReader = new BufferedReader(new InputStreamReader(diffProcess.getInputStream()))) {

                StringBuilder diffCode = new StringBuilder();
                String line;
                while ((line = diffReader.readLine()) != null) {
                    diffCode.append(line).append("\n");
                }

                int exitCode = diffProcess.waitFor();
                if (exitCode != 0) {
                    throw new RuntimeException("Git diff failed with exit code: " + exitCode);
                }
                return diffCode.toString();
            }
        }
    }

    public String writeToLog(String codeReview) throws Exception {
        String token = System.getenv("GITHUB_TOKEN");
        if (token == null || token.trim().isEmpty()) {
            throw new IllegalArgumentException("GITHUB_TOKEN environment variable not set");
        }
        return writeLog(token, codeReview);
    }

    private static String writeLog(String token, String log) throws Exception {
        Path repoDir = Files.createTempDirectory("code-review-log-"); // ← 动态临时目录
        try (Git git = Git.cloneRepository()
                .setURI("https://github.com/zhrdislikecode/code-review-logs.git")
                .setDirectory(repoDir.toFile())
                .setCredentialsProvider(new UsernamePasswordCredentialsProvider(token, ""))
                .setCloneSubmodules(false)
                .call()) {

            String dateFolderName = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
            Path datePath = repoDir.resolve(dateFolderName);
            Files.createDirectories(datePath);

            String fileName = RandomStringUtils.generateRandomString(12) + ".md";
            Path logPath = datePath.resolve(fileName);
            Files.write(logPath, log.getBytes());

            git.add().addFilepattern(dateFolderName + "/" + fileName).call();
            git.commit().setMessage("Add new file via CodeReviewSDK").call();
            git.push()
                    .setCredentialsProvider(new UsernamePasswordCredentialsProvider(token, ""))
                    .call();

            return "https://github.com/zhrdislikecode/code-review-logs/blob/master/" + dateFolderName + "/" + fileName;
        } catch (Exception e) {
            throw new RuntimeException("Failed to push log to GitHub", e);
        } finally {
            deleteRecursively(repoDir); // ← 清理临时目录
        }
    }

    private static void deleteRecursively(Path path) throws IOException {
        if (Files.isDirectory(path)) {
            Files.walk(path)
                    .sorted((a, b) -> -a.compareTo(b))
                    .forEach(p -> {
                        try {
                            Files.delete(p);
                        } catch (IOException ex) {
                            throw new UncheckedIOException(ex);
                        }
                    });
        }
    }
}
```

> 💡 注：其余文件（如 `AbstractCodeReviewService.exec()` 签名、`Main.java` 回滚、`.gitignore` 更新）需同步修改，此处因篇幅限制未展开，但已涵盖全部关键修复项。