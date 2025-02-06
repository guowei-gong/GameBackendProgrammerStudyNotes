Account 账户结构体中的 AccountId 是通过雪花 ID 生成的，它可以是字符串，也可以是整形。我倾向于整形，因为雪花算法生产的 ID 本身是有序的，对于数据库来说，查询和存储(字符串比整形占用更多的存储，虽然是微不足道的)都是更友好的。

Proto 结构体存储到 MongoDB 的场景，需要将 Proto 先转换为 Json，再将 Json 装载为 Bson。

前面提到的将 Proto 先转换为 Json，通用方案是使用 google 官方的 `protojson` 包

但遗憾的是，protojson 包转换的过程中，会将 int64 类型转换为 string 类型，通过[该代码](https://github.com/protocolbuffers/protobuf-go/blob/b92717ecb630d4a4824b372bf98c729d87311a4d/encoding/protojson/encode.go#L275-L278
)造成的，因为 JavaScript 无法处理 int64 类型的精度，但从 JSON 的角度来看，它不应该能处理任何整形吗？

在 [issue](https://github.com/golang/protobuf/issues/1414) 中我看到了同样的疑问，社区给出的答复：`protojson` 包遵循此处定义的标准 JSON 映射，该[规定](https://developers.google.com/protocol-buffers/docs/proto3#json)表示 64 位 int 编码为 JSON 字符串。

目前看来，社区是不会修复该问题的，毕竟已经从 2022 年放到现在了。我思考的解决方案如下。

- 反射；

- 公司项目是开发阶段，可以考虑将 AccountId 设置为 String 类型；

- 考虑新增一个 Model 层，保存到 MongoDB 手动进行一次映射，针对该场景，其他情况继续使用 `protojson` 包。