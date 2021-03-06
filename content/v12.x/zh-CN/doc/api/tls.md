# TLS (SSL)

<!--introduced_in=v0.10.0-->

> 稳定性：2 - 稳定

`tls` 模块提供了对传输层安全性 (TLS) 及构建于 OpenSSL 之上的安全套接字层 （SSL）的实现。 此模块可通过如下方式访问：

```js
const tls = require('tls');
```

## TLS/SSL 概念

TLS/SSL 是一个基于公钥/私钥的架构 （PKI）。 For most common cases, each client and server must have a *private key*.

私钥可通过多种方式生成。 如下示例子演示了如何通过 OpenSSL 命令行界面生成 2048 位的 RSA 私钥：

```sh
openssl genrsa -out ryans-key.pem 2048
```

With TLS/SSL, all servers (and some clients) must have a *certificate*. Certificates are *public keys* that correspond to a private key, and that are digitally signed either by a Certificate Authority or by the owner of the private key (such certificates are referred to as "self-signed"). The first step to obtaining a certificate is to create a *Certificate Signing Request* (CSR) file.

OpenSSL 命令行界面可被用于生成一个针对私钥的 CSR。

```sh
openssl req -new -sha256 -key ryans-key.pem -out ryans-csr.pem
```

一旦 CSR 文件生成，它可被发送给证书颁发机构以获取签名，或者用于生成自签名证书。

下面的示例演示了如何使用 OpenSSL 命令行界面创建一个自签名证书。

```sh
openssl x509 -req -in ryans-csr.pem -signkey ryans-key.pem -out ryans-cert.pem
```

一旦证书被生成，它就可以被用于生成 `.pfx` 或 `.p12` 文件：

```sh
openssl pkcs12 -export -in ryans-cert.pem -inkey ryans-key.pem \
      -certfile ca-cert.pem -out ryans.pfx
```

其中：

* `in`：是已签名的证书
* `inkey`：是关联的私钥
* `certfile`：是所有证书颁发机构（CA）的证书连接而成的单一文件，即：`cat ca1-cert.pem ca2-cert.pem > ca-cert.pem`

### 完美前向安全

<!-- type=misc -->

The term "[Forward Secrecy](https://en.wikipedia.org/wiki/Perfect_forward_secrecy)" or "Perfect Forward Secrecy" describes a feature of key-agreement (i.e., key-exchange) methods. That is, the server and client keys are used to negotiate new temporary keys that are used specifically and only for the current communication session. Practically, this means that even if the server's private key is compromised, communication can only be decrypted by eavesdroppers if the attacker manages to obtain the key-pair specifically generated for the session.

完美前向安全是通过针对每一个 TLS/SSL 握手的密钥协商随机生成密钥对（与针对所有会话使用相同的密钥相比）来实现的。 实现此技术的方法被称为 "ephemeral"。

目前有两种常见方法来实现完美前向安全（注意：字符 "E" 被追加到传统缩写之后）：

* [DHE](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange): An ephemeral version of the Diffie Hellman key-agreement protocol.
* [ECDHE](https://en.wikipedia.org/wiki/Elliptic_curve_Diffie%E2%80%93Hellman): An ephemeral version of the Elliptic Curve Diffie Hellman key-agreement protocol.

由于密钥生成的系统开销很大，ephemeral 方法可能在性能上会有缺陷。

要想通过 `DHE` 方法和 `tls` 模块来使用完美前向安全，要求生成 Diffie-Hellman 参数并在 [`tls.createSecureContext()`][] 的 `dhparam` 参数中进行指定。 下面演示了如何通过 OpenSSL 命令行界面来生成这些参数：

```sh
openssl dhparam -outform PEM -out dhparam.pem 2048
```

如果通过 `ECDHE` 方法使用完美前向安全，Diffie-Hellman 参数不是必须的，默认的 ECDHE 曲线将被使用。 The `ecdhCurve` property can be used when creating a TLS Server to specify the list of names of supported curves to use, see [`tls.createServer()`][] for more info.

Perfect Forward Secrecy was optional up to TLSv1.2, but it is not optional for TLSv1.3, because all TLSv1.3 cipher suites use ECDHE.

### ALPN and SNI

<!-- type=misc -->

ALPN (Application-Layer Protocol Negotiation Extension) and SNI (Server Name Indication) are TLS handshake extensions:

* ALPN: Allows the use of one TLS server for multiple protocols (HTTP, HTTP/2)
* SNI: Allows the use of one TLS server for multiple hostnames with different SSL certificates.

### Pre-shared keys

<!-- type=misc -->

TLS-PSK support is available as an alternative to normal certificate-based authentication. It uses a pre-shared key instead of certificates to authenticate a TLS connection, providing mutual authentication. TLS-PSK and public key infrastructure are not mutually exclusive. Clients and servers can accommodate both, choosing either of them during the normal cipher negotiation step.

TLS-PSK is only a good choice where means exist to securely share a key with every connecting machine, so it does not replace PKI (Public Key Infrastructure) for the majority of TLS uses. The TLS-PSK implementation in OpenSSL has seen many security flaws in recent years, mostly because it is used only by a minority of applications. Please consider all alternative solutions before switching to PSK ciphers. Upon generating PSK it is of critical importance to use sufficient entropy as discussed in [RFC 4086](https://tools.ietf.org/html/rfc4086). Deriving a shared secret from a password or other low-entropy sources is not secure.

PSK ciphers are disabled by default, and using TLS-PSK thus requires explicitly specifying a cipher suite with the `ciphers` option. The list of available ciphers can be retrieved via `openssl ciphers -v 'PSK'`. All TLS 1.3 ciphers are eligible for PSK but currently only those that use SHA256 digest are supported they can be retrieved via `openssl ciphers -v -s -tls1_3 -psk`.

According to the [RFC 4279](https://tools.ietf.org/html/rfc4279), PSK identities up to 128 bytes in length and PSKs up to 64 bytes in length must be supported. As of OpenSSL 1.1.0 maximum identity size is 128 bytes, and maximum PSK length is 256 bytes.

The current implementation doesn't support asynchronous PSK callbacks due to the limitations of the underlying OpenSSL API.

### 客户端发起的重新协商攻击缓解

<!-- type=misc -->

TLS 协议允许客户端重新协商 TLS 会话的特定方面。 遗憾的是，会话重新协商需要不成比例的服务器端资源，这使得它成为拒绝服务攻击的潜在载体。

为了减轻风险，重新协商以每十分钟三次为上限。 当超出此上限时，在 [`tls.TLSSocket`][] 实例上会发出 `'error'` 事件。 此限制是可以配置的：

* `tls.CLIENT_RENEG_LIMIT` {number} 指定重新协商请求的数量。 **Default:** `3`.
* `tls.CLIENT_RENEG_WINDOW` {number} 指定以秒计的重新协商请求时间段。 **Default:** `600` (10 minutes).

The default renegotiation limits should not be modified without a full understanding of the implications and risks.

TLSv1.3 does not support renegotiation.

### Session Resumption

Establishing a TLS session can be relatively slow. The process can be sped up by saving and later reusing the session state. There are several mechanisms to do so, discussed here from oldest to newest (and preferred).

***Session Identifiers*** Servers generate a unique ID for new connections and send it to the client. Clients and servers save the session state. When reconnecting, clients send the ID of their saved session state and if the server also has the state for that ID, it can agree to use it. Otherwise, the server will create a new session. See [RFC 2246](https://www.ietf.org/rfc/rfc2246.txt) for more information, page 23 and
30.

Resumption using session identifiers is supported by most web browsers when making HTTPS requests.

For Node.js, clients wait for the [`'session'`][] event to get the session data, and provide the data to the `session` option of a subsequent [`tls.connect()`][] to reuse the session. Servers must implement handlers for the [`'newSession'`][] and [`'resumeSession'`][] events to save and restore the session data using the session ID as the lookup key to reuse sessions. To reuse sessions across load balancers or cluster workers, servers must use a shared session cache (such as Redis) in their session handlers.

***Session Tickets*** The servers encrypt the entire session state and send it to the client as a "ticket". When reconnecting, the state is sent to the server in the initial connection. This mechanism avoids the need for server-side session cache. If the server doesn't use the ticket, for any reason (failure to decrypt it, it's too old, etc.), it will create a new session and send a new ticket. See [RFC 5077](https://tools.ietf.org/html/rfc5077) for more information.

Resumption using session tickets is becoming commonly supported by many web browsers when making HTTPS requests.

For Node.js, clients use the same APIs for resumption with session identifiers as for resumption with session tickets. For debugging, if [`tls.TLSSocket.getTLSTicket()`][] returns a value, the session data contains a ticket, otherwise it contains client-side session state.

With TLSv1.3, be aware that multiple tickets may be sent by the server, resulting in multiple `'session'` events, see [`'session'`][] for more information.

Single process servers need no specific implementation to use session tickets. To use session tickets across server restarts or load balancers, servers must all have the same ticket keys. There are three 16-byte keys internally, but the tls API exposes them as a single 48-byte buffer for convenience.

Its possible to get the ticket keys by calling [`server.getTicketKeys()`][] on one server instance and then distribute them, but it is more reasonable to securely generate 48 bytes of secure random data and set them with the `ticketKeys` option of [`tls.createServer()`][]. The keys should be regularly regenerated and server's keys can be reset with [`server.setTicketKeys()`][].

Session ticket keys are cryptographic keys, and they ***must be stored securely***. With TLS 1.2 and below, if they are compromised all sessions that used tickets encrypted with them can be decrypted. They should not be stored on disk, and they should be regenerated regularly.

If clients advertise support for tickets, the server will send them. The server can disable tickets by supplying `require('constants').SSL_OP_NO_TICKET` in `secureOptions`.

Both session identifiers and session tickets timeout, causing the server to create new sessions. The timeout can be configured with the `sessionTimeout` option of [`tls.createServer()`][].

For all the mechanisms, when resumption fails, servers will create new sessions. Since failing to resume the session does not cause TLS/HTTPS connection failures, it is easy to not notice unnecessarily poor TLS performance. The OpenSSL CLI can be used to verify that servers are resuming sessions. Use the `-reconnect` option to `openssl s_client`, for example:

```sh
$ openssl s_client -connect localhost:443 -reconnect
```

Read through the debug output. The first connection should say "New", for example:

```text
New, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
```

Subsequent connections should say "Reused", for example:

```text
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
```

## 修改默认的 TLS 密码套件

Node.js 是使用默认的已启用和已禁用的 TLS 密码套件来构建的。 当前，默认的密码套件是：

```txt
TLS_AES_256_GCM_SHA384:
TLS_CHACHA20_POLY1305_SHA256:
TLS_AES_128_GCM_SHA256:
ECDHE-RSA-AES128-GCM-SHA256:
ECDHE-ECDSA-AES128-GCM-SHA256:
ECDHE-RSA-AES256-GCM-SHA384:
ECDHE-ECDSA-AES256-GCM-SHA384:
DHE-RSA-AES128-GCM-SHA256:
ECDHE-RSA-AES128-SHA256:
DHE-RSA-AES128-SHA256:
ECDHE-RSA-AES256-SHA384:
DHE-RSA-AES256-SHA384:
ECDHE-RSA-AES256-SHA256:
DHE-RSA-AES256-SHA256:
HIGH:
!aNULL:
!eNULL:
!EXPORT:
!DES:
!RC4:
!MD5:
!PSK:
!SRP:
!CAMELLIA
```

This default can be replaced entirely using the [`--tls-cipher-list`][] command line switch (directly, or via the [`NODE_OPTIONS`][] environment variable). For instance, the following makes `ECDHE-RSA-AES128-GCM-SHA256:!RC4` the default TLS cipher suite:

```sh
node --tls-cipher-list="ECDHE-RSA-AES128-GCM-SHA256:!RC4" server.js

export NODE_OPTIONS=--tls-cipher-list="ECDHE-RSA-AES128-GCM-SHA256:!RC4"
node server.js
```

The default can also be replaced on a per client or server basis using the `ciphers` option from [`tls.createSecureContext()`][], which is also available in [`tls.createServer()`][], [`tls.connect()`][], and when creating new [`tls.TLSSocket`][]s.

The ciphers list can contain a mixture of TLSv1.3 cipher suite names, the ones that start with `'TLS_'`, and specifications for TLSv1.2 and below cipher suites.  The TLSv1.2 ciphers support a legacy specification format, consult the OpenSSL [cipher list format](https://www.openssl.org/docs/man1.1.1/man1/ciphers.html#CIPHER-LIST-FORMAT) documentation for details, but those specifications do *not* apply to TLSv1.3 ciphers.  The TLSv1.3 suites can only be enabled by including their full name in the cipher list. They cannot, for example, be enabled or disabled by using the legacy TLSv1.2 `'EECDH'` or `'!EECDH'` specification.

Despite the relative order of TLSv1.3 and TLSv1.2 cipher suites, the TLSv1.3 protocol is significantly more secure than TLSv1.2, and will always be chosen over TLSv1.2 if the handshake indicates it is supported, and if any TLSv1.3 cipher suites are enabled.

The default cipher suite included within Node.js has been carefully selected to reflect current security best practices and risk mitigation. 更改默认的密码套件会对应用程序的安全性产生重大影响。 `--tls-cipher-list` 开关以及 `ciphers` 选项只有在绝对必要时被使用。

The default cipher suite prefers GCM ciphers for \[Chrome's 'modern cryptography' setting\]\[\] and also prefers ECDHE and DHE ciphers for Perfect Forward Secrecy, while offering *some* backward compatibility.

128 bit AES is preferred over 192 and 256 bit AES in light of \[specific attacks affecting larger AES key sizes\]\[\].

依赖于不安全和已弃用的 RC4 或基于 DES 密码的旧客户端（如：Internet Explorer 6），在默认设置下无法完成握手过程。 If these clients _must_ be supported, the [TLS recommendations](https://wiki.mozilla.org/Security/Server_Side_TLS) may offer a compatible cipher suite. For more details on the format, see the OpenSSL [cipher list format](https://www.openssl.org/docs/man1.1.1/man1/ciphers.html#CIPHER-LIST-FORMAT) documentation.

There are only 5 TLSv1.3 cipher suites:

* `'TLS_AES_256_GCM_SHA384'`
* `'TLS_CHACHA20_POLY1305_SHA256'`
* `'TLS_AES_128_GCM_SHA256'`
* `'TLS_AES_128_CCM_SHA256'`
* `'TLS_AES_128_CCM_8_SHA256'`

The first 3 are enabled by default. The last 2 `CCM`-based suites are supported by TLSv1.3 because they may be more performant on constrained systems, but they are not enabled by default since they offer less security.

## Class: `tls.Server`
<!-- YAML
added: v0.3.2
-->

* Extends: {net.Server}

Accepts encrypted connections using TLS or SSL.

### Event: `'keylog'`
<!-- YAML
added: v12.3.0
-->

* `line` {Buffer} Line of ASCII text, in NSS `SSLKEYLOGFILE` format.
* `tlsSocket` {tls.TLSSocket} The `tls.TLSSocket` instance on which it was generated.

The `keylog` event is emitted when key material is generated or received by a connection to this server (typically before handshake has completed, but not necessarily). This keying material can be stored for debugging, as it allows captured TLS traffic to be decrypted. It may be emitted multiple times for each socket.

A typical use case is to append received lines to a common text file, which is later used by software (such as Wireshark) to decrypt the traffic:

```js
const logFile = fs.createWriteStream('/tmp/ssl-keys.log', { flags: 'a' });
// ...
server.on('keylog', (line, tlsSocket) => {
  if (tlsSocket.remoteAddress !== '...')
    return; // Only log keys for a particular IP
  logFile.write(line);
});
```

### Event: `'newSession'`
<!-- YAML
added: v0.9.2
-->

在创建新的 TLS 会话时，`'newSession'` 事件会被发出。 这可被用于将会话保存在外部存储中。 The data should be provided to the [`'resumeSession'`][] callback.

当被调用时，监听器回调函数将被赋予三个参数：

* `sessionId` {Buffer} The TLS session identifier
* `sessionData` {Buffer} The TLS session data
* `callback` {Function} 为了在安全连接中发送和接收数据的，不接受任何参数的回调函数。

Listening for this event will have an effect only on connections established after the addition of the event listener.

### Event: `'OCSPRequest'`
<!-- YAML
added: v0.11.13
-->

当客户端发出证书状态查询时，`'OCSPRequest'` 事件被发送。 当被调用时，监听器回调函数将被赋予三个参数：

* `certificate` {Buffer} 服务器端证书
* `issuer` {Buffer} 签发者证书
* `callback` {Function} 一个必须被调用以提供 OCSP 请求结果的回调函数。

服务器的当前证书可被解析以获取 OCSP URL及证书 ID；在获取 OCSP 响应之后，`callback(null, resp)` 将被调用，其中 `resp` 是一个包含 OCSP 响应的 `Buffer` 实例。 `certificate` 和 `issuer` 都是 `Buffer` 的DER 形式表示的主要及颁发机构的证书。 这些可被用于获取 OCSP 证书 ID以及 OCSP 终端 URL。

或者，可以调用 `callback(null, null)`，来表明没有 OCSP 响应。

调用 `callback(err)` 将会导致 `socket.destroy(err)` 被调用。

典型的 OCSP 请求流程如下所示：

1. Client connects to the server and sends an `'OCSPRequest'` (via the status info extension in ClientHello).
2. Server receives the request and emits the `'OCSPRequest'` event, calling the listener if registered.
3. Server extracts the OCSP URL from either the `certificate` or `issuer` and performs an [OCSP request](https://en.wikipedia.org/wiki/OCSP_stapling) to the CA.
4. Server receives `'OCSPResponse'` from the CA and sends it back to the client via the `callback` argument
5. Client validates the response and either destroys the socket or performs a handshake.

The `issuer` can be `null` if the certificate is either self-signed or the issuer is not in the root certificates list. （在建立 TLS 连接时，issuer 可以通过 `ca` 选项来提供。）

Listening for this event will have an effect only on connections established after the addition of the event listener.

An npm module like [asn1.js](https://www.npmjs.com/package/asn1.js) may be used to parse the certificates.

### Event: `'resumeSession'`
<!-- YAML
added: v0.9.2
-->

当客户端请求继续以前的 TLS 会话时，将发出 `'resumeSession'` 事件。 当被调用时，监听器回调函数将被赋予两个参数：

* `sessionId` {Buffer} The TLS session identifier
* `callback` {Function} A callback function to be called when the prior session has been recovered: `callback([err[, sessionData]])`
  * `err` {Error}
  * `sessionData` {Buffer}

The event listener should perform a lookup in external storage for the `sessionData` saved by the [`'newSession'`][] event handler using the given `sessionId`. If found, call `callback(null, sessionData)` to resume the session. If not found, the session cannot be resumed. `callback()` must be called without `sessionData` so that the handshake can continue and a new session can be created. It is possible to call `callback(err)` to terminate the incoming connection and destroy the socket.

Listening for this event will have an effect only on connections established after the addition of the event listener.

下面演示了如何恢复 TLS 会话：

```js
const tlsSessionStore = {};
server.on('newSession', (id, data, cb) => {
  tlsSessionStore[id.toString('hex')] = data;
  cb();
});
server.on('resumeSession', (id, cb) => {
  cb(null, tlsSessionStore[id.toString('hex')] || null);
});
```

### Event: `'secureConnection'`
<!-- YAML
added: v0.3.2
-->

在新建连接的握手过程成功完成后，将发出 `'secureConnection'` 事件。 当被调用时，监听器回调函数将被赋予一个参数：

* `tlsSocket` {tls.TLSSocket} 已建立的 TLS 套接字。

`tlsSocket.authorized` 属性为 `boolean` 类型，它指示客户端是否已由提供的服务器证书颁发机构之一进行了验证。 如果 `tlsSocket.authorized` 的值为 `false`，则 `socket.authorizationError` 被赋值以描述授权如何失败。 Depending on the settings of the TLS server, unauthorized connections may still be accepted.

The `tlsSocket.alpnProtocol` property is a string that contains the selected ALPN protocol. When ALPN has no selected protocol, `tlsSocket.alpnProtocol` equals `false`.

`tlsSocket.servername` 属性是包含通过 SNI 请求的服务器名称的字符串。

### Event: `'tlsClientError'`
<!-- YAML
added: v6.0.0
-->

在建立安全连接之前发生错误时，将发出 `'tlsClientError'` 事件。 当被调用时，监听器回调函数将被赋予两个参数：

* `exception` {Error} 用于描述错误的 `Error` 对象
* `tlsSocket` {tls.TLSSocket} 产生错误根源的 `tls.TLSSocket` 实例。

### `server.addContext(hostname, context)`
<!-- YAML
added: v0.5.3
-->

* `hostname` {string} 一个 SNI 主机名或通配符 （例如：`'*'`）
* `context` {Object} 包含来自于 [`tls.createSecureContext()`][] `options` 参数 （例如：`key`, `cert`, `ca`等）中任何可能属性的对象。

The `server.addContext()` method adds a secure context that will be used if the client request's SNI name matches the supplied `hostname` (or wildcard).

### `server.address()`
<!-- YAML
added: v0.6.0
-->

* 返回：{Object}

返回操作系统报告的绑定地址，地址系列名，以及服务器端口号。 请查阅 [`net.Server.address()`][] 以获取更多信息。

### `server.close([callback])`
<!-- YAML
added: v0.3.2
-->

* `callback` {Function} A listener callback that will be registered to listen for the server instance's `'close'` event.
* 返回：{tls.Server}

`server.close()` 方法会阻止服务器接受新连接。

此函数以异步方式运行。 当服务器没有更多的打开连接时，将发出 `'close'` 事件。

### `server.connections`
<!-- YAML
added: v0.3.2
deprecated: v0.9.7
-->

> 稳定性：0 - 已弃用：改为使用 [`server.getConnections()`][]。

* {number}

返回服务器上当前的并发连接数。

### `server.getTicketKeys()`
<!-- YAML
added: v3.0.0
-->

* Returns: {Buffer} A 48-byte buffer containing the session ticket keys.

Returns the session ticket keys.

See [Session Resumption](#tls_session_resumption) for more information.

### `server.listen()`

启动监听加密连接的服务器。 此方法与 [`net.Server`][] 中的 [`server.listen()`][] 相同。

### `server.setSecureContext(options)`
<!-- YAML
added: v11.0.0
-->

* `options` {Object} An object containing any of the possible properties from the [`tls.createSecureContext()`][] `options` arguments (e.g. `key`, `cert`, `ca`, etc).

The `server.setSecureContext()` method replaces the secure context of an existing server. Existing connections to the server are not interrupted.

### `server.setTicketKeys(keys)`
<!-- YAML
added: v3.0.0
-->

* `keys` {Buffer} A 48-byte buffer containing the session ticket keys.

Sets the session ticket keys.

Changes to the ticket keys are effective only for future server connections. Existing or currently pending server connections will use the previous keys.

See [Session Resumption](#tls_session_resumption) for more information.

## Class: `tls.TLSSocket`
<!-- YAML
added: v0.11.4
-->

* Extends: {net.Socket}

Performs transparent encryption of written data and all required TLS negotiation.

`tls.TLSSocket` 的实例实现了双工的 [Stream](stream.html#stream_stream) 接口。

Methods that return TLS connection metadata (e.g. [`tls.TLSSocket.getPeerCertificate()`][] will only return data while the connection is open.

### `new tls.TLSSocket(socket[, options])`
<!-- YAML
added: v0.11.4
changes:
  - version: v12.2.0
    pr-url: https://github.com/nodejs/node/pull/27497
    description: The `enableTrace` option is now supported.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2564
    description: ALPN options are supported now.
-->

* `socket` {net.Socket|stream.Duplex} 在服务器端，任何的 `Duplex` 流。 在客户端，[`net.Socket`][] 的任何实例 （对于客户端的通用 `Duplex` 流支持，必须使用 [`tls.connect()`][]）。
* `options` {Object}
  * `enableTrace`: See [`tls.createServer()`][]
  * `isServer`：SSL/TLS 协议是不对称的，TLSSockets 必须知道它们是否要作为服务器还是客户端。 如果值为 `true`，TLS 套接字将被实例化为服务器。 **Default:** `false`.
  * `server` {net.Server} A [`net.Server`][] instance.
  * `requestCert`：是否通过请求证书来对远程对等方进行身份验证。 客户端始终请求服务器的证书。 Servers (`isServer` is true) may set `requestCert` to true to request a client certificate.
  * `rejectUnauthorized`: See [`tls.createServer()`][]
  * `ALPNProtocols`: See [`tls.createServer()`][]
  * `SNICallback`: See [`tls.createServer()`][]
  * `session` {Buffer} A `Buffer` instance containing a TLS session.
  * `requestOCSP` {boolean} 如果值为 `true`，指定 OCSP 状态请求扩展将被添加到客户端 hello 消息中，且在建立安全通信之前发出 `'OCSPResponse'` 事件
  * `secureContext`: TLS context object created with [`tls.createSecureContext()`][]. If a `secureContext` is _not_ provided, one will be created by passing the entire `options` object to `tls.createSecureContext()`.
  * ...: [`tls.createSecureContext()`][] options that are used if the `secureContext` option is missing. Otherwise, they are ignored.

从现有的 TCP 套接字中构造一个新的 `tls.TLSSocket` 对象。

### Event: `'keylog'`
<!-- YAML
added: v12.3.0
-->

* `line` {Buffer} Line of ASCII text, in NSS `SSLKEYLOGFILE` format.

The `keylog` event is emitted on a client `tls.TLSSocket` when key material is generated or received by the socket. This keying material can be stored for debugging, as it allows captured TLS traffic to be decrypted. It may be emitted multiple times, before or after the handshake completes.

A typical use case is to append received lines to a common text file, which is later used by software (such as Wireshark) to decrypt the traffic:

```js
const logFile = fs.createWriteStream('/tmp/ssl-keys.log', { flags: 'a' });
// ...
tlsSocket.on('keylog', (line) => logFile.write(line));
```

### Event: `'OCSPResponse'`
<!-- YAML
added: v0.11.13
-->

当 `tls.TLSSocket` 被创建且已收到 OCSP 响应时，如果 `requestOCSP` 选项被设置，将发出 `'OCSPResponse'` 事件。 当被调用时，监听器回调函数将被赋予一个参数：

* `response` {Buffer} 服务器的 OCSP 响应

通常，`response` 是来自服务器 CA 的数字签名对象，它包含服务器证书废止的状态信息。

### Event: `'secureConnect'`
<!-- YAML
added: v0.11.4
-->

在新建连接的握手过程成功完成后，将发出 `'secureConnect'` 事件。 无论服务器的证书是否已被授权，监听器回调函数都将被调用。 客户端有责任检查 `tlsSocket.authorized` 属性来确定服务器证书是否由指定的 CA 之一签名。 如果 `tlsSocket.authorized === false`，可通过检查 `tlsSocket.authorizationError` 属性来发现错误。 If ALPN was used, the `tlsSocket.alpnProtocol` property can be checked to determine the negotiated protocol.

### Event: `'session'`
<!-- YAML
added: v11.10.0
-->

* `session` {Buffer}

The `'session'` event is emitted on a client `tls.TLSSocket` when a new session or TLS ticket is available. This may or may not be before the handshake is complete, depending on the TLS protocol version that was negotiated. The event is not emitted on the server, or if a new session was not created, for example, when the connection was resumed. For some TLS protocol versions the event may be emitted multiple times, in which case all the sessions can be used for resumption.

On the client, the `session` can be provided to the `session` option of [`tls.connect()`][] to resume the connection.

See [Session Resumption](#tls_session_resumption) for more information.

For TLSv1.2 and below, [`tls.TLSSocket.getSession()`][] can be called once the handshake is complete.  For TLSv1.3, only ticket-based resumption is allowed by the protocol, multiple tickets are sent, and the tickets aren't sent until after the handshake completes. So it is necessary to wait for the `'session'` event to get a resumable session.  Applications should use the `'session'` event instead of `getSession()` to ensure they will work for all TLS versions.  Applications that only expect to get or use one session should listen for this event only once:

```js
tlsSocket.once('session', (session) => {
  // The session can be used immediately or later.
  tls.connect({
    session: session,
    // Other connect options...
  });
});
```

### `tlsSocket.address()`
<!-- YAML
added: v0.11.4
-->

* 返回：{Object}

Returns the bound `address`, the address `family` name, and `port` of the underlying socket as reported by the operating system: `{ port: 12346, family: 'IPv4', address: '127.0.0.1' }`.

### `tlsSocket.authorizationError`
<!-- YAML
added: v0.11.4
-->

返回对等方证书未被验证的原因。 只有在 `tlsSocket.authorized === false` 时，此属性被设置。

### `tlsSocket.authorized`
<!-- YAML
added: v0.11.4
-->

* 返回：{boolean}

在创建 `tls.TLSSocket` 实例时，如果对等方证书由指定的 CA 之一签名，返回 `true`，否则返回 `false`。

### `tlsSocket.disableRenegotiation()`
<!-- YAML
added: v8.4.0
-->

禁用此 `TLSSocket` 实例的 TLS 重新协商。 Once called, attempts to renegotiate will trigger an `'error'` event on the `TLSSocket`.

### `tlsSocket.enableTrace()`
<!-- YAML
added: v12.2.0
-->

When enabled, TLS packet trace information is written to `stderr`. This can be used to debug TLS connection problems.

Note: The format of the output is identical to the output of `openssl s_client
-trace` or `openssl s_server -trace`. While it is produced by OpenSSL's `SSL_trace()` function, the format is undocumented, can change without notice, and should not be relied on.

### `tlsSocket.encrypted`
<!-- YAML
added: v0.11.4
-->

总是返回 `true`。 它也可以被用于区分 TLS 套接字和常规的 `net.Socket` 实例。

### `tlsSocket.getCertificate()`
<!-- YAML
added: v11.2.0
-->

* 返回：{Object}

Returns an object representing the local certificate. 返回的对象具有和证书中字段相对应的属性。

See [`tls.TLSSocket.getPeerCertificate()`][] for an example of the certificate structure.

If there is no local certificate, an empty object will be returned. If the socket has been destroyed, `null` will be returned.

### `tlsSocket.getCipher()`
<!-- YAML
added: v0.11.4
changes:
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26625
    description: Return the minimum cipher version, instead of a fixed string
      (`'TLSv1/SSLv3'`).
  - version: v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30637
    description: Return the IETF cipher name as `standardName`.
-->

* 返回：{Object}
  * `name` {string} OpenSSL name for the cipher suite.
  * `standardName` {string} IETF name for the cipher suite.
  * `version` {string} The minimum TLS protocol version supported by this cipher suite.

Returns an object containing information on the negotiated cipher suite.

例如：
```json
{
    "name": "AES128-SHA256",
    "standardName": "TLS_RSA_WITH_AES_128_CBC_SHA256",
    "version": "TLSv1.2"
}
```

See [SSL_CIPHER_get_name](https://www.openssl.org/docs/man1.1.1/man3/SSL_CIPHER_get_name.html) for more information.

### `tlsSocket.getEphemeralKeyInfo()`
<!-- YAML
added: v5.0.0
-->

* 返回：{Object}

返回一个代表类型，名称，以及在客户端连接的 [完美前向安全](#tls_perfect_forward_secrecy) 中用于交换的 ephemeral 密钥大小的对象。 当密钥交换不是 ephemeral 时，返回一个空对象。 因为这只在客户端套接字中支持，如果在服务器套接字上调用，则返回 `null`。 支持的类型为：`'DH'` 和 `'ECDH'`。 The `name` property is available only when type is `'ECDH'`.

For example: `{ type: 'ECDH', name: 'prime256v1', size: 256 }`.

### `tlsSocket.getFinished()`
<!-- YAML
added: v9.9.0
-->

* Returns: {Buffer|undefined} The latest `Finished` message that has been sent to the socket as part of a SSL/TLS handshake, or `undefined` if no `Finished` message has been sent yet.

As the `Finished` messages are message digests of the complete handshake (with a total of 192 bits for TLS 1.0 and more for SSL 3.0), they can be used for external authentication procedures when the authentication provided by SSL/TLS is not desired or is not enough.

Corresponds to the `SSL_get_finished` routine in OpenSSL and may be used to implement the `tls-unique` channel binding from [RFC 5929](https://tools.ietf.org/html/rfc5929).

### `tlsSocket.getPeerCertificate([detailed])`
<!-- YAML
added: v0.11.4
-->

* `detailed` {boolean} 如果值为 `true`，则包含完整的证书链，否则只包含对等方的证书。
* Returns: {Object} A certificate object.

返回代表对等方证书的对象。 If the peer does not provide a certificate, an empty object will be returned. If the socket has been destroyed, `null` will be returned.

If the full certificate chain was requested, each certificate will include an `issuerCertificate` property containing an object representing its issuer's certificate.

#### Certificate Object
<!-- YAML
changes:
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/24358
    description: Support Elliptic Curve public key info.
-->

A certificate object has properties corresponding to the fields of the certificate.

* `raw` {Buffer} The DER encoded X.509 certificate data.
* `subject` {Object} The certificate subject, described in terms of Country (`C:`), StateOrProvince (`ST`), Locality (`L`), Organization (`O`), OrganizationalUnit (`OU`), and CommonName (`CN`). The CommonName is typically a DNS name with TLS certificates. Example: `{C: 'UK', ST: 'BC', L: 'Metro', O: 'Node Fans', OU: 'Docs', CN: 'example.com'}`.
* `issuer` {Object} The certificate issuer, described in the same terms as the `subject`.
* `valid_from` {string} The date-time the certificate is valid from.
* `valid_to` {string} The date-time the certificate is valid to.
* `serialNumber` {string} The certificate serial number, as a hex string. Example: `'B9B0D332A1AA5635'`.
* `fingerprint` {string} The SHA-1 digest of the DER encoded certificate. It is returned as a `:` separated hexadecimal string. Example: `'2A:7A:C2:DD:...'`.
* `fingerprint256` {string} The SHA-256 digest of the DER encoded certificate. It is returned as a `:` separated hexadecimal string. Example: `'2A:7A:C2:DD:...'`.
* `ext_key_usage` {Array} (Optional) The extended key usage, a set of OIDs.
* `subjectaltname` {string} (Optional) A string containing concatenated names for the subject, an alternative to the `subject` names.
* `infoAccess` {Array} (Optional) An array describing the AuthorityInfoAccess, used with OCSP.
* `issuerCertificate` {Object} (Optional) The issuer certificate object. For self-signed certificates, this may be a circular reference.

The certificate may contain information about the public key, depending on the key type.

For RSA keys, the following properties may be defined:

* `bits` {number} The RSA bit size. Example: `1024`.
* `exponent` {string} The RSA exponent, as a string in hexadecimal number notation. Example: `'0x010001'`.
* `modulus` {string} The RSA modulus, as a hexadecimal string. Example: `'B56CE45CB7...'`.
* `pubkey` {Buffer} The public key.

For EC keys, the following properties may be defined:

* `pubkey` {Buffer} The public key.
* `bits` {number} The key size in bits. Example: `256`.
* `asn1Curve` {string} (Optional) The ASN.1 name of the OID of the elliptic curve. Well-known curves are identified by an OID. While it is unusual, it is possible that the curve is identified by its mathematical properties, in which case it will not have an OID. Example: `'prime256v1'`.
* `nistCurve` {string} (Optional) The NIST name for the elliptic curve, if it has one (not all well-known curves have been assigned names by NIST). Example: `'P-256'`.

Example certificate:

```text
{ subject:
   { OU: [ 'Domain Control Validated', 'PositiveSSL Wildcard' ],
     CN: '*.nodejs.org' },
  issuer:
   { C: 'GB',
     ST: 'Greater Manchester',
     L: 'Salford',
     O: 'COMODO CA Limited',
     CN: 'COMODO RSA Domain Validation Secure Server CA' },
  subjectaltname: 'DNS:*.nodejs.org, DNS:nodejs.org',
  infoAccess:
   { 'CA Issuers - URI':
      [ 'http://crt.comodoca.com/COMODORSADomainValidationSecureServerCA.crt' ],
     'OCSP - URI': [ 'http://ocsp.comodoca.com' ] },
  modulus: 'B56CE45CB740B09A13F64AC543B712FF9EE8E4C284B542A1708A27E82A8D151CA178153E12E6DDA15BF70FFD96CB8A88618641BDFCCA03527E665B70D779C8A349A6F88FD4EF6557180BD4C98192872BCFE3AF56E863C09DDD8BC1EC58DF9D94F914F0369102B2870BECFA1348A0838C9C49BD1C20124B442477572347047506B1FCD658A80D0C44BCC16BC5C5496CFE6E4A8428EF654CD3D8972BF6E5BFAD59C93006830B5EB1056BBB38B53D1464FA6E02BFDF2FF66CD949486F0775EC43034EC2602AEFBF1703AD221DAA2A88353C3B6A688EFE8387811F645CEED7B3FE46E1F8B9F59FAD028F349B9BC14211D5830994D055EEA3D547911E07A0ADDEB8A82B9188E58720D95CD478EEC9AF1F17BE8141BE80906F1A339445A7EB5B285F68039B0F294598A7D1C0005FC22B5271B0752F58CCDEF8C8FD856FB7AE21C80B8A2CE983AE94046E53EDE4CB89F42502D31B5360771C01C80155918637490550E3F555E2EE75CC8C636DDE3633CFEDD62E91BF0F7688273694EEEBA20C2FC9F14A2A435517BC1D7373922463409AB603295CEB0BB53787A334C9CA3CA8B30005C5A62FC0715083462E00719A8FA3ED0A9828C3871360A73F8B04A4FC1E71302844E9BB9940B77E745C9D91F226D71AFCAD4B113AAF68D92B24DDB4A2136B55A1CD1ADF39605B63CB639038ED0F4C987689866743A68769CC55847E4A06D6E2E3F1',
  exponent: '0x10001',
  pubkey: <Buffer ... >,
  valid_from: 'Aug 14 00:00:00 2017 GMT',
  valid_to: 'Nov 20 23:59:59 2019 GMT',
  fingerprint: '01:02:59:D9:C3:D2:0D:08:F7:82:4E:44:A4:B4:53:C5:E2:3A:87:4D',
  fingerprint256: '69:AE:1A:6A:D4:3D:C6:C1:1B:EA:C6:23:DE:BA:2A:14:62:62:93:5C:7A:EA:06:41:9B:0B:BC:87:CE:48:4E:02',
  ext_key_usage: [ '1.3.6.1.5.5.7.3.1', '1.3.6.1.5.5.7.3.2' ],
  serialNumber: '66593D57F20CBC573E433381B5FEC280',
  raw: <Buffer ... > }
```

### `tlsSocket.getPeerFinished()`
<!-- YAML
added: v9.9.0
-->

* Returns: {Buffer|undefined} The latest `Finished` message that is expected or has actually been received from the socket as part of a SSL/TLS handshake, or `undefined` if there is no `Finished` message so far.

As the `Finished` messages are message digests of the complete handshake (with a total of 192 bits for TLS 1.0 and more for SSL 3.0), they can be used for external authentication procedures when the authentication provided by SSL/TLS is not desired or is not enough.

Corresponds to the `SSL_get_peer_finished` routine in OpenSSL and may be used to implement the `tls-unique` channel binding from [RFC 5929](https://tools.ietf.org/html/rfc5929).

### `tlsSocket.getProtocol()`
<!-- YAML
added: v5.7.0
-->

* 返回：{string|null}

返回一个包含当前连接的已协商过的 SSL/TLS 协议版本号的字符串。 对于尚未完成握手过程的已连接套接字，将返回 `'unknown'` 值。 对于服务器套接字或断开连接的客户端套接字，将返回 `null` 值。

Protocol versions are:

* `'SSLv3'`
* `'TLSv1'`
* `'TLSv1.1'`
* `'TLSv1.2'`
* `'TLSv1.3'`

See the OpenSSL [`SSL_get_version`][] documentation for more information.

### `tlsSocket.getSession()`
<!-- YAML
added: v0.11.4
-->

* {Buffer}

Returns the TLS session data or `undefined` if no session was negotiated. On the client, the data can be provided to the `session` option of [`tls.connect()`][] to resume the connection. On the server, it may be useful for debugging.

See [Session Resumption](#tls_session_resumption) for more information.

Note: `getSession()` works only for TLSv1.2 and below. For TLSv1.3, applications must use the [`'session'`][] event (it also works for TLSv1.2 and below).

### `tlsSocket.getSharedSigalgs()`
<!-- YAML
added: v12.11.0
-->

* Returns: {Array} List of signature algorithms shared between the server and the client in the order of decreasing preference.

See [SSL_get_shared_sigalgs](https://www.openssl.org/docs/man1.1.1/man3/SSL_get_shared_sigalgs.html) for more information.

### `tlsSocket.getTLSTicket()`
<!-- YAML
added: v0.11.4
-->

* {Buffer}

For a client, returns the TLS session ticket if one is available, or `undefined`. For a server, always returns `undefined`.

It may be useful for debugging.

See [Session Resumption](#tls_session_resumption) for more information.

### `tlsSocket.isSessionReused()`
<!-- YAML
added: v0.5.6
-->

* Returns: {boolean} `true` if the session was reused, `false` otherwise.

See [Session Resumption](#tls_session_resumption) for more information.

### `tlsSocket.localAddress`
<!-- YAML
added: v0.11.4
-->

* {string}

返回代表本地 IP 地址的字符串。

### `tlsSocket.localPort`
<!-- YAML
added: v0.11.4
-->

* {number}

返回代表本地端口号的数字。

### `tlsSocket.remoteAddress`
<!-- YAML
added: v0.11.4
-->

* {string}

返回代表远程 IP 地址的字符串。 例如： `'74.125.127.100'` 或 `'2001:4860:a005::68'`。

### `tlsSocket.remoteFamily`
<!-- YAML
added: v0.11.4
-->

* {string}

返回代表远程 IP 地址系列名的字符串。 `'IPv4'` 或 `'IPv6'`。

### `tlsSocket.remotePort`
<!-- YAML
added: v0.11.4
-->

* {number}

返回代表远程端口号的数字。 例如：`443`。

### `tlsSocket.renegotiate(options, callback)`
<!-- YAML
added: v0.11.8
-->

* `options` {Object}
  * `rejectUnauthorized` {boolean} If not `false`, the server certificate is verified against the list of supplied CAs. 如果验证失败，则会发出 `'error'` 事件；`err.code` 包含 OpenSSL 错误代码。 **Default:** `true`.
  * `requestCert`
* `callback` {Function} If `renegotiate()` returned `true`, callback is attached once to the `'secure'` event. If `renegotiate()` returned `false`, `callback` will be called in the next tick with an error, unless the `tlsSocket` has been destroyed, in which case `callback` will not be called at all.

* Returns: {boolean} `true` if renegotiation was initiated, `false` otherwise.

`tlsSocket.renegotiate()` 方法启动 TLS 重新协商过程。 结束后，`callback` 函数会被赋予一个参数，该参数为 `Error` （如果请求失败）或 `null`。

This method can be used to request a peer's certificate after the secure connection has been established.

When running as the server, the socket will be destroyed with an error after `handshakeTimeout` timeout.

For TLSv1.3, renegotiation cannot be initiated, it is not supported by the protocol.

### `tlsSocket.setMaxSendFragment(size)`
<!-- YAML
added: v0.11.11
-->

* `size` {number} TLS 片段大小的最大值。 The maximum value is `16384`. **Default:** `16384`.
* 返回：{boolean}

`tlsSocket.setMaxSendFragment()` 方法会设置 TLS 片段大小的最大值。 如果成功设置限制，则返回 `true`，否则返回 `false`。

较小的片段大小会减少客户端的缓冲延迟；在收到完整的片段且其完整性已被验证之前，较大的片段会在 TLS 层被缓冲；较大的片段可能需要多次的交互才能完成数据的传输，同时由于数据包丢失或重新排序，其处理可能会延迟。 然而，较小的片段会增加额外的 TLS 帧字节和 CPU 开销，这可能会降低服务器的整体吞吐量。

## `tls.checkServerIdentity(hostname, cert)`
<!-- YAML
added: v0.8.4
-->

* `hostname` {string} The host name or IP address to verify the certificate against.
* `cert` {Object} A [certificate object](#tls_certificate_object) representing the peer's certificate.
* 返回：{Error|undefined}

Verifies the certificate `cert` is issued to `hostname`.

Returns {Error} object, populating it with `reason`, `host`, and `cert` on failure. 成功时，返回 {undefined}。

This function can be overwritten by providing alternative function as part of the `options.checkServerIdentity` option passed to `tls.connect()`. The overwriting function can call `tls.checkServerIdentity()` of course, to augment the checks done with additional verification.

This function is only called if the certificate passed all other checks, such as being issued by trusted CA (`options.ca`).

## `tls.connect(options[, callback])`
<!-- YAML
added: v0.11.3
changes:
  - version: v12.16.0
    pr-url: https://github.com/nodejs/node/pull/23188
    description: The `pskCallback` option is now supported.
  - version: v12.9.0
    pr-url: https://github.com/nodejs/node/pull/27836
    description: Support the `allowHalfOpen` option.
  - version: v12.4.0
    pr-url: https://github.com/nodejs/node/pull/27816
    description: The `hints` option is now supported.
  - version: v12.2.0
    pr-url: https://github.com/nodejs/node/pull/27497
    description: The `enableTrace` option is now supported.
  - version: v11.8.0
    pr-url: https://github.com/nodejs/node/pull/25517
    description: The `timeout` option is supported now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12839
    description: The `lookup` option is supported now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/11984
    description: The `ALPNProtocols` option can be a `TypedArray` or
     `DataView` now.
  - version: v5.3.0, v4.7.0
    pr-url: https://github.com/nodejs/node/pull/4246
    description: The `secureContext` option is supported now.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2564
    description: ALPN options are supported now.
-->

* `options` {Object}
  * `enableTrace`: See [`tls.createServer()`][]
  * `host` {string} 客户端应连接到的主机。 **Default:** `'localhost'`.
  * `port` {number} 客户端应连接到的端口。
  * `path` {string} Creates Unix socket connection to path. 如果此选项被指定，则 `host` 和 `port` 会被忽略。
  * `socket` {stream.Duplex} 建立到给定套接字的安全连接，而不是创建一个新的套接字。 通常情况下，这是 [`net.Socket`][] 的实例，但允许任何的 `Duplex` 流。 如果指定此选项，除了在证书验证时，`path`, `host` 和 `port` 将被忽略。 通常，当传递给 `tls.connect()` 时，套接字已连接，但它也可在稍后被连接。 Connection/disconnection/destruction of `socket` is the user's responsibility; calling `tls.connect()` will not cause `net.connect()` to be called.
  * `allowHalfOpen` {boolean} If the `socket` option is missing, indicates whether or not to allow the internally created socket to be half-open, otherwise the option is ignored. See the `allowHalfOpen` option of [`net.Socket`][] for details. **Default:** `false`.
  * `rejectUnauthorized` {boolean} If not `false`, the server certificate is verified against the list of supplied CAs. 如果验证失败，则会发出 `'error'` 事件；`err.code` 包含 OpenSSL 错误代码。 **Default:** `true`.
  * `pskCallback` {Function}
    * hint: {string} optional message sent from the server to help client decide which identity to use during negotiation. Always `null` if TLS 1.3 is used.
    * Returns: {Object} in the form `{ psk: <Buffer|TypedArray|DataView>, identity: <string> }` or `null` to stop the negotiation process. `psk` must be compatible with the selected cipher's digest. `identity` must use UTF-8 encoding. When negotiating TLS-PSK (pre-shared keys), this function is called with optional identity `hint` provided by the server or `null` in case of TLS 1.3 where `hint` was removed. It will be necessary to provide a custom `tls.checkServerIdentity()` for the connection as the default one will try to check hostname/IP of the server against the certificate but that's not applicable for PSK because there won't be a certificate present. More information can be found in the [RFC 4279](https://tools.ietf.org/html/rfc4279).
  * `ALPNProtocols`: {string[]|Buffer[]|TypedArray[]|DataView[]|Buffer| TypedArray|DataView} An array of strings, `Buffer`s or `TypedArray`s or `DataView`s, or a single `Buffer` or `TypedArray` or `DataView` containing the supported ALPN protocols. `Buffer`s should have the format `[len][name][len][name]...` e.g. `'\x08http/1.1\x08http/1.0'`, where the `len` byte is the length of the next protocol name. Passing an array is usually much simpler, e.g. `['http/1.1', 'http/1.0']`. Protocols earlier in the list have higher preference than those later.
  * `servername`: {string} SNI (服务器名称指示) TLS 扩展的服务器名。 It is the name of the host being connected to, and must be a host name, and not an IP address. It can be used by a multi-homed server to choose the correct certificate to present to the client, see the `SNICallback` option to [`tls.createServer()`][].
  * `checkServerIdentity(servername, cert)` {Function} A callback function to be used (instead of the builtin `tls.checkServerIdentity()` function) when checking the server's hostname (or the provided `servername` when explicitly set) against the certificate. This should return an {Error} if verification fails. The method should return `undefined` if the `servername` and `cert` are verified.
  * `session` {Buffer} 包含 TLS 会话的 `Buffer` 实例。
  * `minDHSize` {number} 用来接受 TLS 连接的，以位为单位的 DH 参数大小的最小值。 当服务器提供的 DH 参数的大小小于 `minDHSize` 时，TLS 连接将被销毁，且将抛出错误。 **Default:** `1024`.
  * `secureContext`: TLS context object created with [`tls.createSecureContext()`][]. If a `secureContext` is _not_ provided, one will be created by passing the entire `options` object to `tls.createSecureContext()`.
  * ...: [`tls.createSecureContext()`][] options that are used if the `secureContext` option is missing, otherwise they are ignored.
  * ...: Any [`socket.connect()`][] option not already listed.
* `callback` {Function}
* 返回：{tls.TLSSocket}

如果指定了 `callback` 函数，则该函数将被添加为 [`'secureConnect'`][] 事件的监听器。

`tls.connect()` 返回一个 [`tls.TLSSocket`][] 对象。

The following illustrates a client for the echo server example from [`tls.createServer()`][]:

```js
// Assumes an echo server that is listening on port 8000.
const tls = require('tls');
const fs = require('fs');

const options = {
  // Necessary only if the server requires client certificate authentication.
  key: fs.readFileSync('client-key.pem'),
  cert: fs.readFileSync('client-cert.pem'),

  // Necessary only if the server uses a self-signed certificate.
  ca: [ fs.readFileSync('server-cert.pem') ],

  // Necessary only if the server's cert isn't for "localhost".
  checkServerIdentity: () => { return null; },
};

const socket = tls.connect(8000, options, () => {
  console.log('client connected',
              socket.authorized ? 'authorized' : 'unauthorized');
  process.stdin.pipe(socket);
  process.stdin.resume();
});
socket.setEncoding('utf8');
socket.on('data', (data) => {
  console.log(data);
});
socket.on('end', () => {
  console.log('server ends connection');
});
```

## `tls.connect(path[, options][, callback])`
<!-- YAML
added: v0.11.3
-->

* `path` {string} `options.path` 的默认值。
* `options` {Object} 请参阅 [`tls.connect()`][]。
* `callback` {Function} 请参阅 [`tls.connect()`][]。
* 返回：{tls.TLSSocket}

与 [`tls.connect()`][] 相同，但不同之处在于 `path` 可以通过参数，而不是选项的方式提供。

A path option, if specified, will take precedence over the path argument.

## `tls.connect(port[, host][, options][, callback])`
<!-- YAML
added: v0.11.3
-->

* `port` {number} `options.port` 的默认值。
* `host` {string} Default value for `options.host`.
* `options` {Object} 请参阅 [`tls.connect()`][]。
* `callback` {Function} 请参阅 [`tls.connect()`][]。
* 返回：{tls.TLSSocket}

与 [`tls.connect()`][] 相同，但不同之处在于 `port` 和 `host` 可以通过参数，而不是选项的方式提供。

A port or host option, if specified, will take precedence over any port or host argument.

## `tls.createSecureContext([options])`
<!-- YAML
added: v0.11.13
changes:
  - version: v12.12.0
    pr-url: https://github.com/nodejs/node/pull/28973
    description: Added `privateKeyIdentifier` and `privateKeyEngine` options
                 to get private key from an OpenSSL engine.
  - version: v12.11.0
    pr-url: https://github.com/nodejs/node/pull/29598
    description: Added `sigalgs` option to override supported signature
                 algorithms.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26209
    description: TLSv1.3 support added.
  - version: v11.5.0
    pr-url: https://github.com/nodejs/node/pull/24733
    description: The `ca:` option now supports `BEGIN TRUSTED CERTIFICATE`.
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/24405
    description: The `minVersion` and `maxVersion` can be used to restrict
                 the allowed TLS protocol versions.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19794
    description: The `ecdhCurve` cannot be set to `false` anymore due to a
                 change in OpenSSL.
  - version: v9.3.0
    pr-url: https://github.com/nodejs/node/pull/14903
    description: The `options` parameter can now include `clientCertEngine`.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15206
    description: The `ecdhCurve` option can now be multiple `':'` separated
                 curve names or `'auto'`.
  - version: v7.3.0
    pr-url: https://github.com/nodejs/node/pull/10294
    description: If the `key` option is an array, individual entries do not
                 need a `passphrase` property anymore. `Array` entries can also
                 just be `string`s or `Buffer`s now.
  - version: v5.2.0
    pr-url: https://github.com/nodejs/node/pull/4099
    description: The `ca` option can now be a single string containing multiple
                 CA certificates.
-->

* `options` {Object}
  * `ca` {string|string[]|Buffer|Buffer[]} 可选的覆盖受信任 CA 的证书。 默认值为信任由 Mozilla 策展的知名 CA。 当使用此选项显式指定 CA 时，会彻底替换 Mozilla 的 CA。 The value can be a string or `Buffer`, or an `Array` of strings and/or `Buffer`s. Any string or `Buffer` can contain multiple PEM CAs concatenated together. 对等方证书必须可以链接到受服务器信任的CA，才能对连接进行身份验证。 当使用不能链接到知名 CA 的证书时，证书的 CA 必须被显式指定为受信任的，否则连接无法进行身份验证。 如果对等方使用一个不匹配，或和默认 CA 无法链接的证书，请使用 `ca` 选项来提供一个 CA 证书以便对等方证书可以匹配或链入。 对于自签名证书，证书就是自己的 CA，因此必须被提供。 For PEM encoded certificates, supported types are "TRUSTED CERTIFICATE", "X509 CERTIFICATE", and "CERTIFICATE". See also [`tls.rootCertificates`][].
  * `cert` {string|string[]|Buffer|Buffer[]} Cert chains in PEM format. One cert chain should be provided per private key. Each cert chain should consist of the PEM formatted certificate for a provided private `key`, followed by the PEM formatted intermediate certificates (if any), in order, and not including the root CA (the root CA must be pre-known to the peer, see `ca`). When providing multiple cert chains, they do not have to be in the same order as their private keys in `key`. If the intermediate certificates are not provided, the peer will not be able to validate the certificate, and the handshake will fail.
  * `sigalgs` {string} Colon-separated list of supported signature algorithms. The list can contain digest algorithms (`SHA256`, `MD5` etc.), public key algorithms (`RSA-PSS`, `ECDSA` etc.), combination of both (e.g 'RSA+SHA384') or TLS v1.3 scheme names (e.g. `rsa_pss_pss_sha512`). See [OpenSSL man pages](https://www.openssl.org/docs/man1.1.1/man3/SSL_CTX_set1_sigalgs_list.html) for more info.
  * `ciphers` {string} Cipher suite specification, replacing the default. For more information, see [modifying the default cipher suite](#tls_modifying_the_default_tls_cipher_suite). Permitted ciphers can be obtained via [`tls.getCiphers()`][]. Cipher names must be uppercased in order for OpenSSL to accept them.
  * `clientCertEngine` {string} Name of an OpenSSL engine which can provide the client certificate.
  * `crl` {string|string[]|Buffer|Buffer[]} PEM formatted CRLs (Certificate Revocation Lists).
  * `dhparam` {string|Buffer} Diffie Hellman 参数，是 [完美前向安全](#tls_perfect_forward_secrecy) 的必选参数。 运行 `openssl dhparam` 来创建参数。 The key length must be greater than or equal to 1024 bits or else an error will be thrown. Although 1024 bits is permissible, use 2048 bits or larger for stronger security. 如果忽略或无效，参数将以静默方式被丢弃，DHE 密码将不可用。
  * `ecdhCurve` {string} A string describing a named curve or a colon separated list of curve NIDs or names, for example `P-521:P-384:P-256`, to use for ECDH key agreement. 设置为 `auto` 来自动选择曲线。 Use [`crypto.getCurves()`][] to obtain a list of available curve names. On recent releases, `openssl ecparam -list_curves` will also display the name and description of each available elliptic curve. **Default:** [`tls.DEFAULT_ECDH_CURVE`][].
  * `honorCipherOrder` {boolean} 尝试使用服务器的，而不是客户端的密码套件偏好。 当值为 `true` 时，将 `SSL_OP_CIPHER_SERVER_PREFERENCE` 赋值给 `secureOptions`，请参阅 [OpenSSL Options](crypto.html#crypto_openssl_options) 以获取更多信息。
  * `key` {string|string[]|Buffer|Buffer[]|Object[]} Private keys in PEM format. PEM 允许对私钥选项进行加密。 Encrypted keys will be decrypted with `options.passphrase`. Multiple keys using different algorithms can be provided either as an array of unencrypted key strings or buffers, or an array of objects in the form `{pem: <string|buffer>[, passphrase: <string>]}`. 对象形式只能在数组中使用。 `object.passphrase` 是可选的。 Encrypted keys will be decrypted with `object.passphrase` if provided, or `options.passphrase` if it is not.
  * `privateKeyEngine` {string} Name of an OpenSSL engine to get private key from. Should be used together with `privateKeyIdentifier`.
  * `privateKeyIdentifier` {string} Identifier of a private key managed by an OpenSSL engine. Should be used together with `privateKeyEngine`. Should not be set together with `key`, because both options define a private key in different ways.
  * `maxVersion` {string} Optionally set the maximum TLS version to allow. One of `'TLSv1.3'`, `'TLSv1.2'`, `'TLSv1.1'`, or `'TLSv1'`. Cannot be specified along with the `secureProtocol` option, use one or the other. **Default:** [`tls.DEFAULT_MAX_VERSION`][].
  * `minVersion` {string} Optionally set the minimum TLS version to allow. One of `'TLSv1.3'`, `'TLSv1.2'`, `'TLSv1.1'`, or `'TLSv1'`. Cannot be specified along with the `secureProtocol` option, use one or the other. It is not recommended to use less than TLSv1.2, but it may be required for interoperability. **Default:** [`tls.DEFAULT_MIN_VERSION`][].
  * `passphrase` {string} Shared passphrase used for a single private key and/or a PFX.
  * `pfx` {string|string[]|Buffer|Buffer[]|Object[]} PFX or PKCS12 encoded private key and certificate chain. `pfx` is an alternative to providing `key` and `cert` individually. PFX 通常是加密的，如果是的话，`passphrase` 将被用于对其进行解密。 可以提供多个 PFX，其提供方式可以是一个未加密的 PFX 缓冲区数组，或是以 `{buf: <string|buffer>[, passphrase: <string>]}` 格式提供的对象数组。 对象形式只能在数组中使用。 `object.passphrase` 是可选的。 Encrypted PFX will be decrypted with `object.passphrase` if provided, or `options.passphrase` if it is not.
  * `secureOptions` {number} 可选的能够影响 OpenSSL 协议行为的选项，通常是不需要的。 如果有的话，应小心使用！ 其值是 [OpenSSL Options](crypto.html#crypto_openssl_options) 中 `SSL_OP_*` 选项的数字位掩码。
  * `secureProtocol` {string} Legacy mechanism to select the TLS protocol version to use, it does not support independent control of the minimum and maximum version, and does not support limiting the protocol to TLSv1.3.  Use `minVersion` and `maxVersion` instead.  The possible values are listed as [SSL_METHODS](https://www.openssl.org/docs/man1.1.1/man7/ssl.html#Dealing-with-Protocol-Methods), use the function names as strings.  For example, use `'TLSv1_1_method'` to force TLS version 1.1, or `'TLS_method'` to allow any TLS protocol version up to TLSv1.3.  It is not recommended to use TLS versions less than 1.2, but it may be required for interoperability. **Default:** none, see `minVersion`.
  * `sessionIdContext` {string} Opaque identifier used by servers to ensure session state is not shared between applications. 未被客户端使用。

[`tls.createServer()`][] sets the default value of the `honorCipherOrder` option to `true`, other APIs that create secure contexts leave it unset.

[`tls.createServer()`][] uses a 128 bit truncated SHA1 hash value generated from `process.argv` as the default value of the `sessionIdContext` option, other APIs that create secure contexts have no default value.

The `tls.createSecureContext()` method creates a `SecureContext` object. It is usable as an argument to several `tls` APIs, such as [`tls.createServer()`][] and [`server.addContext()`][], but has no public methods.

A key is *required* for ciphers that make use of certificates. `key` 或 `pfx` 都可被用于该目的。

If the `ca` option is not given, then Node.js will default to using [Mozilla's publicly trusted list of CAs](https://hg.mozilla.org/mozilla-central/raw-file/tip/security/nss/lib/ckfw/builtins/certdata.txt).

## `tls.createServer([options][, secureConnectionListener])`
<!-- YAML
added: v0.3.2
changes:
  - version: v12.3.0
    pr-url: https://github.com/nodejs/node/pull/27665
    description: The `options` parameter now supports `net.createServer()`
                 options.
  - version: v9.3.0
    pr-url: https://github.com/nodejs/node/pull/14903
    description: The `options` parameter can now include `clientCertEngine`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/11984
    description: The `ALPNProtocols` option can be a `TypedArray` or
     `DataView` now.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2564
    description: ALPN options are supported now.
-->

* `options` {Object}
  * `ALPNProtocols`: {string[]|Buffer[]|TypedArray[]|DataView[]|Buffer| TypedArray|DataView} An array of strings, `Buffer`s or `TypedArray`s or `DataView`s, or a single `Buffer` or `TypedArray` or `DataView` containing the supported ALPN protocols. `Buffer`s should have the format `[len][name][len][name]...` e.g. `0x05hello0x05world`, where the first byte is the length of the next protocol name. Passing an array is usually much simpler, e.g. `['hello', 'world']`. (协议应按其优先级进行排序。)
  * `clientCertEngine` {string} Name of an OpenSSL engine which can provide the client certificate.
  * `enableTrace` {boolean} If `true`, [`tls.TLSSocket.enableTrace()`][] will be called on new connections. Tracing can be enabled after the secure connection is established, but this option must be used to trace the secure connection setup. **Default:** `false`.
  * `handshakeTimeout` {number} 如果 SSL/TLS 握手过程没有在指定的毫秒数内完成，则终止连接。 如果握手过程超时，则会在 `tls.Server` 对象上发出 `'tlsClientError'`。 **Default:** `120000` (120 seconds).
  * `rejectUnauthorized` {boolean} If not `false` the server will reject any connection which is not authorized with the list of supplied CAs. 只有当 `requestCert` 的值为 `true` 时，此选项才有效。 **Default:** `true`.
  * `requestCert` {boolean} 如果值为 `true`，服务器会从发出连接的客户端请求一个证书，并尝试验证该证书。 **Default:** `false`.
  * `sessionTimeout` {number} The number of seconds after which a TLS session created by the server will no longer be resumable. See [Session Resumption](#tls_session_resumption) for more information. **Default:** `300`.
  * `SNICallback(servername, cb)` {Function} 当客户端支持 SNI TLS 扩展时将被调用的函数。 当被调用时，将传递两个参数：`servername` 和 `cb`。 `SNICallback` should invoke `cb(null, ctx)`, where `ctx` is a `SecureContext` instance. (`tls.createSecureContext(...)` can be used to get a proper `SecureContext`.) If `SNICallback` wasn't provided the default callback with high-level API will be used (see below).
  * `ticketKeys`: {Buffer} 48-bytes of cryptographically strong pseudo-random data. See [Session Resumption](#tls_session_resumption) for more information.
  * `pskCallback` {Function}
    * socket: {tls.TLSSocket} the server [`tls.TLSSocket`][] instance for this connection.
    * identity: {string} identity parameter sent from the client.
    * Returns: {Buffer|TypedArray|DataView} pre-shared key that must either be a buffer or `null` to stop the negotiation process. Returned PSK must be compatible with the selected cipher's digest. When negotiating TLS-PSK (pre-shared keys), this function is called with the identity provided by the client. If the return value is `null` the negotiation process will stop and an "unknown_psk_identity" alert message will be sent to the other party. If the server wishes to hide the fact that the PSK identity was not known, the callback must provide some random data as `psk` to make the connection fail with "decrypt_error" before negotiation is finished. PSK ciphers are disabled by default, and using TLS-PSK thus requires explicitly specifying a cipher suite with the `ciphers` option. More information can be found in the [RFC 4279](https://tools.ietf.org/html/rfc4279).
  * `pskIdentityHint` {string} optional hint to send to a client to help with selecting the identity during TLS-PSK negotiation. Will be ignored in TLS 1.3. Upon failing to set pskIdentityHint `'tlsClientError'` will be emitted with `'ERR_TLS_PSK_SET_IDENTIY_HINT_FAILED'` code.
  * ...: Any [`tls.createSecureContext()`][] option can be provided. For servers, the identity options (`pfx`, `key`/`cert` or `pskCallback`) are usually required.
  * ...: Any [`net.createServer()`][] option can be provided.
* `secureConnectionListener` {Function}
* 返回：{tls.Server}

Creates a new [`tls.Server`][]. 如果提供了 `secureConnectionListener`，它会被自动设置为 [`'secureConnection'`][] 事件的监听器。

The `ticketKeys` options is automatically shared between `cluster` module workers.

如下演示了一个简单的 echo 服务器：

```js
const tls = require('tls');
const fs = require('fs');

const options = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),

  // This is necessary only if using client certificate authentication.
  requestCert: true,

  // This is necessary only if the client uses a self-signed certificate.
  ca: [ fs.readFileSync('client-cert.pem') ]
};

const server = tls.createServer(options, (socket) => {
  console.log('server connected',
              socket.authorized ? 'authorized' : 'unauthorized');
  socket.write('welcome!\n');
  socket.setEncoding('utf8');
  socket.pipe(socket);
});
server.listen(8000, () => {
  console.log('server bound');
});
```

The server can be tested by connecting to it using the example client from [`tls.connect()`][].

## `tls.getCiphers()`
<!-- YAML
added: v0.10.2
-->

* Returns: {string[]}

Returns an array with the names of the supported TLS ciphers. The names are lower-case for historical reasons, but must be uppercased to be used in the `ciphers` option of [`tls.createSecureContext()`][].

Cipher names that start with `'tls_'` are for TLSv1.3, all the others are for TLSv1.2 and below.

```js
console.log(tls.getCiphers()); // ['aes128-gcm-sha256', 'aes128-sha', ...]
```

## `tls.rootCertificates`
<!-- YAML
added: v12.3.0
-->

* {string[]}

An immutable array of strings representing the root certificates (in PEM format) used for verifying peer certificates. This is the default value of the `ca` option to [`tls.createSecureContext()`][].

## `tls.DEFAULT_ECDH_CURVE`
<!-- YAML
added: v0.11.13
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/16853
    description: Default value changed to `'auto'`.
-->

在 tls 服务器中用于 ECDH 密钥协议的默认曲线名称。 The default value is `'auto'`. See [`tls.createSecureContext()`][] for further information.

## `tls.DEFAULT_MAX_VERSION`
<!-- YAML
added: v11.4.0
-->

* {string} The default value of the `maxVersion` option of [`tls.createSecureContext()`][]. It can be assigned any of the supported TLS protocol versions, `'TLSv1.3'`, `'TLSv1.2'`, `'TLSv1.1'`, or `'TLSv1'`. **Default:** `'TLSv1.3'`, unless changed using CLI options. Using `--tls-max-v1.2` sets the default to `'TLSv1.2'`.  Using `--tls-max-v1.3` sets the default to `'TLSv1.3'`. If multiple of the options are provided, the highest maximum is used.

## `tls.DEFAULT_MIN_VERSION`
<!-- YAML
added: v11.4.0
-->

* {string} The default value of the `minVersion` option of [`tls.createSecureContext()`][]. It can be assigned any of the supported TLS protocol versions, `'TLSv1.3'`, `'TLSv1.2'`, `'TLSv1.1'`, or `'TLSv1'`. **Default:** `'TLSv1.2'`, unless changed using CLI options. Using `--tls-min-v1.0` sets the default to `'TLSv1'`. Using `--tls-min-v1.1` sets the default to `'TLSv1.1'`. Using `--tls-min-v1.3` sets the default to `'TLSv1.3'`. If multiple of the options are provided, the lowest minimum is used.

## 已弃用的 API

### Class: `CryptoStream`
<!-- YAML
added: v0.3.4
deprecated: v0.11.3
-->

> 稳定性：0 - 已弃用：改为使用 [`tls.TLSSocket`][]。

`tls.CryptoStream` 类表示加密数据流。 This class is deprecated and should no longer be used.

#### `cryptoStream.bytesWritten`
<!-- YAML
added: v0.3.4
deprecated: v0.11.3
-->

The `cryptoStream.bytesWritten` property returns the total number of bytes written to the underlying socket *including* the bytes required for the implementation of the TLS protocol.

### Class: `SecurePair`
<!-- YAML
added: v0.3.2
deprecated: v0.11.3
-->

> 稳定性：0 - 已弃用：改为使用 [`tls.TLSSocket`][]。

由 [`tls.createSecurePair()`][] 返回。

#### Event: `'secure'`
<!-- YAML
added: v0.3.2
deprecated: v0.11.3
-->

一旦建立了安全连接，`SecurePair` 对象就会发出 `'secure'` 事件。

As with checking for the server [`'secureConnection'`](#tls_event_secureconnection) event, `pair.cleartext.authorized` should be inspected to confirm whether the certificate used is properly authorized.

### `tls.createSecurePair([context][, isServer][, requestCert][, rejectUnauthorized][, options])`
<!-- YAML
added: v0.3.2
deprecated: v0.11.3
changes:
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2564
    description: ALPN options are supported now.
-->

> 稳定性：0 - 已弃用：改为使用 [`tls.TLSSocket`][]。

* `context` {Object} 由 `tls.createSecureContext()` 返回的安全上下文对象。
* `isServer` {boolean} `true` 指定 TLS 连接是否以服务器方式打开。
* `requestCert` {boolean} `true` 指定服务器是否应该请求连接客户端的证书。 只适用于当 `isServer` 的值为 `true`时。
* `rejectUnauthorized` {boolean} If not `false` a server automatically reject clients with invalid certificates. 只适用于当 `isServer` 的值为 `true`时。
* `options`
  * `enableTrace`: See [`tls.createServer()`][]
  * `secureContext`: A TLS context object from [`tls.createSecureContext()`][]
  * `isServer`：如果值为 `true`，则 TLS 套接字将被以服务器模式初始化。 **Default:** `false`.
  * `server` {net.Server} A [`net.Server`][] instance
  * `requestCert`: See [`tls.createServer()`][]
  * `rejectUnauthorized`: See [`tls.createServer()`][]
  * `ALPNProtocols`: See [`tls.createServer()`][]
  * `SNICallback`: See [`tls.createServer()`][]
  * `session` {Buffer} A `Buffer` instance containing a TLS session.
  * `requestOCSP` {boolean} If `true`, specifies that the OCSP status request extension will be added to the client hello and an `'OCSPResponse'` event will be emitted on the socket before establishing a secure communication.

创建具有两个流的新安全对对象，其中一个读取和写入加密数据，另一个读取和写入明文数据。 通常，加密流和传入的加密数据流通过管道传输，而明文流则用于替换初始加密流。

`tls.createSecurePair()` 返回含有 `cleartext` 和 `encrypted` 流属性的 `tls.SecurePair` 对象。

Using `cleartext` has the same API as [`tls.TLSSocket`][].

The `tls.createSecurePair()` method is now deprecated in favor of `tls.TLSSocket()`. 例如，代码：

```js
pair = tls.createSecurePair(/* ... */);
pair.encrypted.pipe(socket);
socket.pipe(pair.encrypted);
```

可被替换为：

```js
secureSocket = tls.TLSSocket(socket, options);
```

where `secureSocket` has the same API as `pair.cleartext`.
