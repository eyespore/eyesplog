# Logback

- 官方文档： http://logback.qos.ch/manual/index.html
- 参考：https://www.cnblogs.com/simpleito/p/15133654.html



> Java开源日志框架，以继承改善log4j为目的，是log4j创始人Ceki Gülcü的开源产品。声称有极佳的性能，占用空间更小，且提供其他日志系统缺失但很有用的特性。其一大特色是，在 logback-classic 中本地(native)实现了 SLF4J API（也表示依赖 slf4j-api）

- logback-core：其他俩模块基础模块，作为通用模块 其中没有 logger 的概念
- logback-classic：日志模块，完整实现了 SLF4J API
- logback-access：配合Servlet容器，提供 http 访问日志功能