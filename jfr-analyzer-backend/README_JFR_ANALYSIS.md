# JFR分析器火焰图功能实现文档

## 概述

本项目实现了基于Java Mission Control (JMC) 的JFR文件分析功能，能够生成火焰图来可视化Java应用程序的性能数据。

## 功能特性

### 1. 支持的分析维度

- **CPU时间分析** (`cpu`): 分析CPU使用时间
- **CPU采样分析** (`cpu-sample`): 分析CPU采样数据
- **挂钟时间分析** (`wall-clock`): 分析实际执行时间
- **本地执行采样** (`native-execution-samples`): 分析本地方法执行
- **内存分配分析** (`alloc`): 分析内存分配情况
- **内存使用分析** (`mem`): 分析内存使用情况
- **文件IO时间** (`file-io-time`): 分析文件IO耗时
- **文件读取大小** (`file-read-size`): 分析文件读取数据量
- **文件写入大小** (`file-write-size`): 分析文件写入数据量
- **Socket读取时间** (`socket-read-time`): 分析网络读取耗时
- **Socket读取大小** (`socket-read-size`): 分析网络读取数据量
- **Socket写入时间** (`socket-write-time`): 分析网络写入耗时
- **Socket写入大小** (`socket-write-size`): 分析网络写入数据量
- **同步分析** (`synchronization`): 分析同步操作
- **线程挂起** (`thread-park`): 分析线程挂起情况
- **类加载次数** (`class-load-count`): 分析类加载次数
- **类加载时间** (`class-load-wall-time`): 分析类加载耗时
- **线程睡眠** (`thread-sleep`): 分析线程睡眠时间

### 2. 火焰图数据结构

火焰图包含以下核心数据：

```json
{
  "data": [
    ["1,2,3", 1000, "main"],
    ["1,2,4", 500, "main"]
  ],
  "threadSplit": {
    "main": 1500,
    "worker-1": 800
  },
  "symbolTable": {
    "1": "com.example.Main.main",
    "2": "com.example.Service.process",
    "3": "com.example.Util.helper",
    "4": "com.example.Database.query"
  }
}
```

- **data**: 火焰图节点数据，每个元素包含 `[符号ID数组, 样本数量, 线程名称]`
- **threadSplit**: 线程分割信息，显示每个线程的总样本数
- **symbolTable**: 符号表，将符号ID映射到实际的方法名

## API接口

### 1. 分析JFR文件并生成火焰图

**接口**: `POST /api/jfr/analyze`

**参数**:
- `filePath` (String, 必需): JFR文件路径
- `dimension` (String, 必需): 分析维度
- `include` (boolean, 可选): 是否包含指定的任务集，默认true
- `taskSet` (List<String>, 可选): 任务集
- `options` (Map<String, String>, 可选): 分析选项

**示例请求**:
```bash
curl -X POST "http://localhost:8200/api/jfr/analyze" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "filePath=/path/to/application.jfr&dimension=cpu&include=true"
```

**响应示例**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "data": [
      ["1,2,3", 1000, "main"],
      ["1,2,4", 500, "main"]
    ],
    "threadSplit": {
      "main": 1500
    },
    "symbolTable": {
      "1": "com.example.Main.main",
      "2": "com.example.Service.process",
      "3": "com.example.Util.helper",
      "4": "com.example.Database.query"
    }
  }
}
```

### 2. 获取分析元数据

**接口**: `GET /api/jfr/metadata`

**响应示例**:
```json
{
  "code": 1,
  "msg": "success",
  "data": {
    "perfDimensions": [
      "cpu", "cpu-sample", "wall-clock", "native-execution-samples",
      "alloc", "mem", "file-io-time", "file-read-size", "file-write-size",
      "socket-read-time", "socket-read-size", "socket-write-time", "socket-write-size",
      "synchronization", "thread-park", "class-load-count", "class-load-wall-time", "thread-sleep"
    ]
  }
}
```

### 3. 验证JFR文件

**接口**: `GET /api/jfr/validate?filePath=/path/to/file.jfr`

**响应示例**:
```json
{
  "code": 1,
  "msg": "success",
  "data": true
}
```

### 4. 获取支持的维度

**接口**: `GET /api/jfr/dimensions`

**响应示例**:
```json
{
  "code": 1,
  "msg": "success",
  "data": [
    "cpu", "cpu-sample", "wall-clock", "native-execution-samples",
    "alloc", "mem", "file-io-time", "file-read-size", "file-write-size",
    "socket-read-time", "socket-read-size", "socket-write-time", "socket-write-size",
    "synchronization", "thread-park", "class-load-count", "class-load-wall-time", "thread-sleep"
  ]
}
```

## 核心实现架构

### 1. 数据模型层

- **AnalysisRequest**: 分析请求模型
- **AnalysisResult**: 分析结果模型
- **TaskResultBase**: 任务结果基类
- **TaskCPUTime**: CPU时间任务结果
- **StackTrace**: 堆栈跟踪模型
- **Frame**: 堆栈帧模型
- **RecordedEvent**: 记录事件模型

### 2. 服务层

- **JFRAnalyzer**: JFR分析器接口
- **JFRAnalyzerImpl**: JFR分析器实现
- **JFRAnalysisService**: JFR分析服务接口
- **JFRAnalysisServiceImpl**: JFR分析服务实现

### 3. 控制器层

- **JFRAnalysisController**: JFR分析REST控制器

### 4. 工具类

- **DimensionBuilder**: 维度构建器
- **Extractor**: 事件提取器接口

## 使用示例

### 1. 基本CPU分析

```java
// 创建分析服务
JFRAnalysisService service = new JFRAnalysisServiceImpl();

// 分析CPU使用情况
Path jfrFile = Paths.get("/path/to/application.jfr");
FlameGraph flameGraph = service.analyzeAndGenerateFlameGraph(
    jfrFile, "cpu", true, null, null);

// 处理火焰图数据
Object[][] data = flameGraph.getData();
Map<String, Long> threadSplit = flameGraph.getThreadSplit();
Map<Integer, String> symbolTable = flameGraph.getSymbolTable();
```

### 2. 内存分配分析

```java
// 分析内存分配情况
FlameGraph allocGraph = service.analyzeAndGenerateFlameGraph(
    jfrFile, "alloc", true, null, null);
```

### 3. 特定线程分析

```java
// 只分析特定线程
List<String> taskSet = Arrays.asList("main", "worker-1");
FlameGraph threadGraph = service.analyzeAndGenerateFlameGraph(
    jfrFile, "cpu", true, taskSet, null);
```

## 配置说明

### 1. 依赖配置

项目使用以下JMC依赖：

```xml
<dependency>
    <groupId>org.openjdk.jmc</groupId>
    <artifactId>flightrecorder</artifactId>
    <version>8.3.1</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmc</groupId>
    <artifactId>common</artifactId>
    <version>8.3.1</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmc</groupId>
    <artifactId>flightrecorder.rules</artifactId>
    <version>8.3.1</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmc</groupId>
    <artifactId>flightrecorder.rules.jdk</artifactId>
    <version>8.3.1</version>
</dependency>
```

### 2. 应用配置

在 `application.yml` 中配置JFR存储路径：

```yaml
arthas:
  jfr-storage-path: ${user.home}/arthas-jfr-storage
```

## 性能优化

### 1. 并行处理

- 支持多线程并行解析JFR事件
- 可配置并行工作线程数
- 使用CountDownLatch确保所有任务完成

### 2. 内存优化

- 使用符号表压缩重复的方法名
- 流式处理大型JFR文件
- 及时释放不需要的数据

### 3. 缓存策略

- 可缓存分析结果
- 支持增量分析
- 元数据缓存

## 错误处理

### 1. 文件验证

- 检查文件是否存在
- 验证文件可读性
- 检查文件大小
- 验证文件格式

### 2. 异常处理

- 详细的错误日志
- 友好的错误消息
- 异常分类处理

### 3. 参数验证

- 维度参数验证
- 文件路径验证
- 任务集验证

## 扩展性

### 1. 新增分析维度

1. 在 `ProfileDimension` 枚举中添加新维度
2. 实现对应的 `Extractor` 类
3. 在 `JFRAnalyzerImpl` 中注册新的提取器
4. 更新元数据信息

### 2. 自定义分析规则

1. 实现 `IRule` 接口
2. 注册到 `RuleRegistry`
3. 在问题分析中使用

### 3. 输出格式扩展

1. 实现新的输出格式接口
2. 添加格式转换器
3. 支持多种可视化方式

## 注意事项

1. **内存使用**: 大型JFR文件可能消耗大量内存，建议配置足够的堆内存
2. **处理时间**: 复杂分析可能需要较长时间，建议使用异步处理
3. **文件格式**: 确保JFR文件格式正确，支持JFR-8、JFR-11、JFR-17+等版本
4. **权限要求**: 确保应用有读取JFR文件的权限
5. **并发限制**: 避免同时处理过多JFR文件，防止资源耗尽

## 未来改进

1. **异步处理**: 支持异步分析，提供进度查询
2. **增量分析**: 支持JFR文件的增量分析
3. **可视化增强**: 提供更多可视化选项
4. **性能优化**: 进一步优化内存和CPU使用
5. **扩展性**: 支持插件化的分析器扩展 