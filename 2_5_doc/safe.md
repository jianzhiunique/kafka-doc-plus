# Kafka 安全演进

0.9开始引入安全机制，可以在以下情况下进行认证

1. 客户端到kafka、kafka内部、工具到kafka
2. kafka到zookeeper
3. 客户端到kafka、kafka内部、工具到kafka
4. 客户端读写鉴权

支持的机制

1. SASL/GSSAPI (Kerberos) - starting at version 0.9.0.0
2. SASL/PLAIN - starting at version 0.10.0.0
3. SASL/SCRAM-SHA-256 and SASL/SCRAM-SHA-512 - starting at version 0.10.2.0
4. SASL/OAUTHBEARER - starting at version 2.0

在运行的集群上启用安全
http://kafka.apache.org/25/documentation.html#security_rolling_upgrade