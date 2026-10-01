---
description: "注解处理器 —— 使用基于注解的处理，在 Conductor 构建期间进行代码生成，支持 protobuf 等。"
---
# 注解处理器

此模块严格用于构建期间基于注解的代码生成任务。
目前支持 `protogen`

### 用法

下面是该模块的一个实际示例，实现在 common/build.gradle 中

```groovy
task protogen(dependsOn: jar, type: JavaExec) {
    classpath configurations.annotationsProcessorCodegen
    main = 'com.netflix.conductor.annotationsprocessor.protogen.ProtoGenTask'
    args(
            "conductor.proto",
            "com.netflix.conductor.proto",
            "github.com/netflix/conductor/client/gogrpc/conductor/model",
            "${rootDir}/grpc/src/main/proto",
            "${rootDir}/grpc/src/main/java/com/netflix/conductor/grpc",
            "com.netflix.conductor.grpc",
            jar.archivePath,
            "com.netflix.conductor.common",
    )
}
```
