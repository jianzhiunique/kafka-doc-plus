# 不易懂的参数意思
### control.plane.listener.name
Name of listener used for communication between controller and brokers. 
Broker will use the control.plane.listener.name to locate the endpoint in listeners list, 
to listen for connections from the controller. 
If not explicitly configured, the default value will be null 
and there will be no dedicated endpoints for controller connections.

### inter.broker.listener.name 默认null
Name of listener used for communication between brokers. 
If this is unset, the listener name is defined by security.inter.broker.protocol. 
It is an error to set this and security.inter.broker.protocol properties at the same time.

### security.inter.broker.protocol 默认PLAINTEXT
Security protocol used to communicate between brokers. 
Valid values are: PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL. 
It is an error to set this and inter.broker.listener.name properties at the same time.


### listener.security.protocol.map

If the listener name is not a security protocol, listener.security.protocol.map must also be set.

Map between listener names and security protocols. 
This must be defined for the same security protocol to be usable in more than one port or IP. 
For example, internal and external traffic can be separated even if SSL is required for both. 
Concretely, the user could define listeners with names INTERNAL and EXTERNAL and this property as: `INTERNAL:SSL,EXTERNAL:SSL`. 
As shown, key and value are separated by a colon and map entries are separated by commas. 
Each listener name should only appear once in the map. 
Different security (SSL and SASL) settings can be configured for each listener by adding a normalised prefix (the listener name is lowercased) to the config name. 
For example, to set a different keystore for the INTERNAL listener, a config with name listener.name.internal.ssl.keystore.location would be set. 
If the config for the listener name is not set, the config will fallback to the generic config (i.e. ssl.keystore.location).
侦听器名称和安全协议之间的映射。
必须对同一安全协议进行定义，才能在多个端口或IP中使用。
例如，内部和外部流量可以分开，即使两者都需要SSL。
具体地说，用户可以用名称INTERNAL和EXTERNAL定义侦听器，该属性为：`内部：SSL,外部：SSL`.
如图所示，键和值用冒号分隔，映射条目用逗号分隔。
每个侦听器名称在映射中只应出现一次。
通过向配置名称添加标准化前缀（侦听器名称为小写），可以为每个侦听器配置不同的安全性（SSL和SASL）设置。
例如，要为内部侦听器设置不同的密钥库，请使用listener.name.internal.ssl.keystore.location.位置会被设定的。
如果未设置侦听器名称的配置，则配置将回退到通用配置（即。ssl.keystore.location.位置).