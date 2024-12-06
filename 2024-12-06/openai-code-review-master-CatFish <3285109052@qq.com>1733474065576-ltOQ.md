### 代码评审

#### 文件：`FeiShu.java`

**变更点：**
1. 新增了`import cn.cat.middleware.sdk.types.enums.CommitInfo;`。
2. 修改了`sendTestMessage`方法，增加了`Map<String, Map<String, String>> data`参数。
3. 在`sendTestMessage`方法中，增加了从`data`参数中提取提交信息的逻辑。
4. 新增了`get`辅助方法，用于从`data`中提取特定提交信息。

**评审意见：**
- **优点：**
  - 增加了提交信息的详细展示，使得飞书消息更加丰富和有用。
  - 使用了枚举`CommitInfo`来管理提交信息的键，提高了代码的可读性和可维护性。
  - `get`方法的引入简化了数据提取逻辑，使代码更简洁。

- **改进建议：**
  - `get`方法中的异常处理：当前`get`方法假设`data`和嵌套的`Map`总是存在的，可能会引发`NullPointerException`。建议增加空值检查或使用`Optional`来避免潜在的错误。
  - 日志记录：在`get`方法中增加日志记录，以便在数据提取失败时能够追踪问题。
  - 参数校验：在`sendTestMessage`方法中，对`redirectUri`和`data`参数进行非空校验，以防止传入无效参数。

**示例代码改进：**
```java
private static String get(Map<String, Map<String, String>> data, CommitInfo commitInfo) {
    if (data == null || data.get(commitInfo.getCode()) == null) {
        logger.warn("Data for {} is missing", commitInfo.getDesc());
        return "N/A";
    }
    return data.get(commitInfo.getCode()).getOrDefault("value", "N/A");
}
```

#### 文件：`FeiShuMessageStrategy.java`

**变更点：**
1. 修改了`sendMessage`方法，增加了`data`参数传递给`FeiShu.sendTestMessage`方法。

**评审意见：**
- **优点：**
  - 与`FeiShu.java`的变更保持一致，确保了消息发送逻辑的完整性。

- **改进建议：**
  - 参数校验：在`sendMessage`方法中，对`logUrl`和`data`参数进行非空校验，以防止传入无效参数。
  - 异常处理：当前仅记录了异常日志，可以考虑增加更详细的错误处理逻辑，例如重试机制或错误通知。

**示例代码改进：**
```java
public void sendMessage(String logUrl, Map<String, Map<String, String>> data) {
    if (logUrl == null || data == null) {
        logger.error("Invalid parameters: logUrl or data is null");
        return;
    }
    try {
        String botWebhook = EnvUtils.getEnv("FEISHU_BOT_WEBHOOK");
        FeiShu feiShu = new FeiShu(botWebhook);
        feiShu.sendTestMessage(logUrl, data);
    } catch (Exception e) {
        logger.error("发送飞书消息失败", e);
        // 可以考虑增加重试逻辑或错误通知
    }
}
```

#### 文件：`CommitInfo.java`

**变更点：**
1. 新增了`CommitInfo`枚举类，用于管理提交信息的键和描述。

**评审意见：**
- **优点：**
  - 使用枚举类管理常量，提高了代码的可读性和可维护性。
  - 提供了`getCode`和`getDesc`方法，方便在代码中使用。

- **改进建议：**
  - 枚举值的命名：可以考虑将枚举值的命名改为全大写，以符合常见的枚举命名规范。
  - 文档注释：建议为枚举类和每个枚举值添加详细的文档注释，说明其用途和含义。

**示例代码改进：**
```java
/**
 * 提交信息枚举类，用于管理代码提交的相关信息。
 */
public enum CommitInfo {
    REPO_NAME("repo_name", "项目名称"),
    BRANCH_NAME("branch_name", "分支名称"),
    COMMIT_AUTHOR("commit_author", "提交者"),
    COMMIT_MESSAGE("commit_message", "提交信息");

    private final String code;
    private final String desc;

    CommitInfo(String code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    public String getCode() {
        return code;
    }

    public String getDesc() {
        return desc;
    }
}
```

### 总结
整体来看，这些变更提高了代码的可读性和功能完整性，但也存在一些潜在的异常处理和参数校验问题。建议根据上述改进建议进行优化，以确保代码的健壮性和可维护性。