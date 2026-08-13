## 请求链路

#### traceId
在两个用户同时查询信息时，日志信息有可能交错，不知道哪个日志对应哪个用户链路

这个时候就可以给每个请求都分配一个traceId，这样在日志记录的时候，每个请求链路就会有特有的traceId

这个时候查询日志的时候根据traceId就能找到整个请求的日志

#### MDC
MDC是暂存traceId的口袋，如果没有MDC，那么每一层的traceID都需要手动传递

一般业务流程
```
请求进入
-> MDC 放入 traceId=A100
-> Controller 打日志，自动带 A100
-> Service 打日志，自动带 A100
-> 请求结束
-> 从 MDC 删除 A100
```
#### Filter 
位于Controller之前，可以生成traceId
```
请求到达
-> Filter 生成 traceId
-> 放入 MDC
-> Controller
-> Service
-> Redis / MySQL
-> Filter 记录状态码和耗时
-> 清理 MDC
```

生成的traceId必须要在finally中清理，因为如果在请求执行过程中如果失败了，traceId不在finally中就不会被清理，可能下一次来新请求时MDC还是记录就traceID

finally字段无论是请求执行成功还是失败都会执行，就不用考虑这种情况

对于日志记录Logger的引用要引对对应的包
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
```
并且在配置配置文件
```
logging:
  pattern:
    console: '%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [traceId=%X{traceId:-}] %logger{36} - %msg%n'
```

## RAG

#### R、A、G 的准确含义

- R — Retrieval，检索：根据用户问题，从知识库取回相关片段。
- A — Augmentation，增强：把检索结果作为上下文，加入 Prompt。
- G — Generation，生成：模型根据“问题 + 上下文”生成回答。

对用查找用户问题的流程
```
用户问题
→ 生成问题向量
→ 与文档向量比较相似度
→ 找到向量对应的原始文本块
→ 把原始文本交给生成模型
```
#### TopK
是在知识库中检索出TopK个相似度最高的文本块

主要风险是：
- 混入无关或冲突内容
- Prompt 噪声增加
- 上下文 token、模型耗时和费用增加
- 最终回答可能被错误材料误导

#### 结构化chunk
提高检索准确率和答案完整性
结构化 chunk 仍不能保证绝对正确，因为还可能存在：
- 知识库内容本身错误或过期
- 用户问题存在歧义
- 相似度排序错误
- 模型没有遵守上下文
- 知识库根本没有对应答案


RAG 在生成前检索外部知识，为回答提供依据。

离线阶段完成清洗、分块、文档向量化以及原文、向量和元数据存储。

在线阶段完成问题向量化、相似度检索、TopK、阈值过滤、上下文构造和模型生成。

RAG 提高事实准确性的机会并降低幻觉风险，但不保证回答绝对正确。

