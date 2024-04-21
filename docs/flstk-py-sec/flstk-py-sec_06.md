# 第五章：非对称加密

本章内容包括

+   介绍密钥分发问题

+   使用`cryptography`包演示非对称加密

+   通过数字签名确保不可否认性

在上一章中，你学会了如何使用对称加密确保机密性。不幸的是，对称加密并非万灵药。单独来说，对称加密不适用于密钥分发，这是密码学中的一个经典问题。在本章中，你将学习如何使用非对称加密解决这个问题。在此过程中，你将更多地了解名为`cryptography`的 Python 包。最后，我将向你展示如何通过数字签名确保不可否认性。

## 5.1 密钥分发问题

当加密者和解密者是同一方时，对称加密效果很好，但它的扩展性不佳。假设 Alice 想要向 Bob 发送一条保密消息。她加密消息并将密文发送给 Bob。Bob 需要 Alice 的密钥来解密消息。现在 Alice 必须找到一种方法将密钥分发给 Bob，而不被 Eve，一个窃听者，拦截密钥。Alice 可以用第二个密钥加密她的密钥，但她如何安全地将第二个密钥发送给 Bob？Alice 可以用第三个密钥加密她的第二个密钥，但她如何...你明白了。密钥分发是一个递归问题。

如果 Alice 想向像 Bob 这样的 10 个人发送消息，问题就会变得更加严重。即使 Alice 将密钥物理分发给所有方，如果 Eve 从任何一个人那里获取了密钥，她将不得不重复这项工作。更换密钥的概率和成本将增加十倍。另外，Alice 可以为每个人管理不同的密钥——工作量增加一个数量级。这个*密钥分发问题*是非对称加密的灵感之一。

## 5.2 非对称加密

如果一个加密算法，如 AES，使用相同的密钥进行加密和解密，我们称之为*对称*。如果一个加密算法使用两个不同的密钥进行加密和解密，我们称之为*非对称*。密钥被称为*密钥对*。

密钥对由一个*私钥*和一个*公钥*组成。私钥由所有者隐藏。公钥公开分发给任何人；它不是一个秘密。私钥可以解密公钥加密的内容，反之亦然。

*非对称加密*，如图 5.1 所示，是解决密钥分发问题的经典解决方案。假设 Alice 想要安全地向 Bob 发送一条保密消息，使用公钥加密。Bob 生成一个密钥对。私钥保密，公钥公开分发给 Alice。如果 Eve 看到 Bob 向 Alice 发送的公钥，没关系；那只是一个公钥。现在 Alice 使用 Bob 的公钥加密她的消息。她公开将密文发送给 Bob。Bob 接收到密文，并使用他的私钥解密它——唯一可以解密 Alice 消息的密钥。

![CH05_F01_Byrne](img/CH05_F01_Byrne.png)

图 5.1 Alice 通过公钥加密机密地向 Bob 发送消息。

此解决方案解决了两个问题。首先，密钥分发问题已经解决。如果 Eve 设法获取到 Bob 的公钥和 Alice 的密文，她无法解密消息。只有 Bob 的私钥才能解密由 Bob 的公钥产生的密文。其次，此解决方案可扩展。如果 Alice 想把她的消息发送给 10 个人，每个人只需要生成自己的唯一密钥对。如果 Eve 成功地破坏了某个人的私钥，这不会影响其他参与者。

此部分演示了公钥加密的基本思想。下一节演示了如何使用史上最广泛使用的公钥密码系统在 Python 中执行此操作。

### 5.2.1 RSA 公钥加密

*RSA* 是一种经受住时间考验的经典非对称加密的例子。这个公钥密码系统是由 Ron Rivest、Adi Shamir 和 Leonard Adleman 在 1970 年代末期开发的。这个缩写代表了创建者的姓氏。

以下的 `openssl` 命令演示了如何使用 `genpkey` 子命令生成一个 3072 位的 RSA 私钥。在撰写本文时，RSA 密钥应至少为 2048 位：

```py
$ openssl genpkey -algorithm RSA \      ❶
    -out private_key.pem \              ❷
    -pkeyopt rsa_keygen_bits:3072       ❸
```

❶ 生成 RSA 密钥

❷ 生成私钥文件到这个路径

❸ 使用 3072 位的密钥大小

注意 RSA 密钥和 AES 密钥之间的大小差异。为了达到可比较的强度，RSA 密钥需要比 AES 密钥大得多。例如，AES 密钥的最大大小是 256 位：这样大小的 RSA 密钥就是个笑话。这种对比反映了这些算法用于加密数据的基础数学模型。RSA 加密使用整数因子分解；AES 加密使用替换-置换网络。一般来说，用于非对称加密的密钥需要比用于对称加密的密钥更大。

以下 `openssl` 命令演示了如何使用 `rsa` 子命令从私钥文件中提取 RSA 公钥：

```py
$ openssl rsa -pubout -in private_key.pem -out public_key.pem
```

私钥和公钥有时存储在文件系统中。重要的是要管理这些文件的访问权限。私钥文件不应该对除所有者以外的任何人可读或可写。另一方面，公钥文件可以被任何人读取。以下命令演示了如何在类 Unix 系统上限制对这些文件的访问：

```py
$ chmod 600 private_key.pem   ❶
$ chmod 644 public_key.pem    ❷
```

❶ 拥有者具有读取和写入权限。

❷ 任何人都可以读取这个文件。

注意：与对称密钥一样，非对称密钥在生产源代码或文件系统中没有用武之地。这样的密钥应该安全地存储在诸如亚马逊的 AWS 密钥管理服务（[`aws.amazon.com/kms/`](https://aws.amazon.com/kms/)）和谷歌的 Cloud 密钥管理服务（[`cloud.google.com/security-key-management`](https://cloud.google.com/security-key-management)）之类的密钥管理服务中。

OpenSSL 将密钥串行化到磁盘上的格式称为*增强隐私邮件*（*PEM*）。PEM 是编码密钥对的事实标准方式。如果您已经使用过 PEM 格式的文件，您可能会在每个文件中看到下面粗体显示的`-----BEGIN`头部：

```py
-----BEGIN PRIVATE KEY-----
MIIG/QIBADANBgkqhkiG9w0BAQEFAASCBucwggbjAgEAAoIBgQDJ2Psz+Ub+VKg0
vnlZmm671s5qiZigu8SsqcERPlSk4KsnnjwbibMhcRlGJgSo5Vv13SMekaj+oCTl
...

-----BEGIN PUBLIC KEY-----
MIIBojANBgkqhkiG9w0BAQEFAAOCAY8AMIIBigKCAYEAydj7M/lG/lSoNL55WZpu
u9bOaomYoLvErKnBET5UpOCrJ548G4mzIXEZRiYEqOVb9d0jHpGo/qAk5VCwfNPG
...
```

或者，可以使用`cryptography`包生成密钥。列表 5.1 演示了如何使用`rsa`模块生成私钥。`generate_private_key`的第一个参数是本书不讨论的 RSA 实现细节（有关更多信息，请访问[www.imperialviolet.org/2012/03/16/rsae.html](https://cloud.google.com/security-key-management)）。第二个参数是密钥大小。生成私钥后，从中提取公钥。

用 Python 生成 RSA 密钥对的列表 5.1

```py
from cryptography.hazmat.backends import default_backend    ❶
from cryptography.hazmat.primitives import serialization    ❶
from cryptography.hazmat.primitives.asymmetric import rsa   ❶

private_key = rsa.generate_private_key(                     ❷
    public_exponent=65537,                                  ❷
    key_size=3072,                                          ❷
    backend=default_backend(), )                            ❷

public_key = private_key.public_key()                       ❸
```

❶ 复杂的低级 API

❷ 私钥生成

❸ 公钥提取

注意 生产密钥对的生成在 Python 中很少进行。通常，这是通过命令行工具如`openssl`或`ssh-keygen`完成的。

下面的列表演示了如何将内存中的两个密钥序列化为磁盘上的 PEM 格式。

用 Python 序列化 RSA 密钥对的列表 5.2

```py
private_bytes = private_key.private_bytes(                    ❶
    encoding=serialization.Encoding.PEM,                      ❶
    format=serialization.PrivateFormat.PKCS8,                 ❶
    encryption_algorithm=serialization.NoEncryption(), )      ❶

with open('private_key.pem', 'xb') as private_file:           ❶
    private_file.write(private_bytes)                         ❶

public_bytes = public_key.public_bytes(                       ❷
    encoding=serialization.Encoding.PEM,                      ❷
    format=serialization.PublicFormat.SubjectPublicKeyInfo, ) ❷

with open('public_key.pem', 'xb') as public_file:             ❷
    public_file.write(public_bytes)                           ❷
```

❶ 私钥序列化

❷ 公钥序列化

不管密钥对如何生成，都可以使用下面列表中显示的代码将其加载到内存中。

用 Python 反序列化 RSA 密钥对的列表 5.3

```py
with open('private_key.pem', 'rb') as private_file:            ❶
   loaded_private_key = serialization.load_pem_private_key(    ❶
       private_file.read(),                                    ❶
       password=None,                                          ❶
       backend=default_backend()                               ❶
   )                                                           ❶

with open('public_key.pem', 'rb') as public_file:              ❷
   loaded_public_key = serialization.load_pem_public_key(      ❷
       public_file.read(),                                     ❷
       backend=default_backend()                               ❷
   )                                                           ❷
```

❶ 私钥反序列化

❷ 公钥反序列化

下一个列表演示了如何使用公钥加密并用私钥解密。与对称块密码一样，RSA 使用填充方案加密数据。

注意 最佳非对称加密填充（OAEP）是 RSA 加密和解密的推荐填充方案。

用 Python 进行 RSA 公钥加密和解密的列表 5.4

```py
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding

padding_config = padding.OAEP(                           ❶
   mgf=padding.MGF1(algorithm=hashes.SHA256()),          ❶
   algorithm=hashes.SHA256(),                            ❶
   label=None, )                                         ❶

plaintext = b'message from Alice to Bob'

ciphertext = loaded_public_key.encrypt(                  ❷
   plaintext=plaintext,                                  ❷
   padding=padding_config, )                             ❷

decrypted_by_private_key = loaded_private_key.decrypt(   ❸
   ciphertext=ciphertext,                                ❸
   padding=padding_config)                               ❸

assert decrypted_by_private_key == plaintext
```

❶ 使用 OAEP 填充

❷ 用公钥加密

❸ 用私钥解密

非对称加密是双向的。你可以用公钥加密，用私钥解密；或者，你可以反向操作——用私钥加密，用公钥解密。这给我们提供了保密性和数据认证之间的权衡。用公钥加密的数据是*保密的*；只有私钥的所有者才能解密消息，但任何人都可能是其作者。用私钥加密的数据是*认证的*；接收者知道消息只能由私钥进行授权，但任何人都可以解密它。

本节演示了公钥加密如何确保保密性。下一节演示了私钥加密如何确保不可否认性。

## 5.3 不可否认性

在第三章，你学会了 Alice 和 Bob 如何通过密钥散列来确保消息认证。Bob 发送了一条消息以及一个哈希值给 Alice。Alice 也对消息进行了哈希。如果 Alice 的哈希值与 Bob 的哈希值匹配，她可以得出两个结论：消息具有完整性，并且 Bob 是消息的创建者。

现在从第三方 Charlie 的角度考虑这种情况。Charlie 知道谁创建了这条消息吗？不，因为 Alice 和 Bob 都共享一把密钥。Charlie 知道消息是由他们中的一个创建的，但他不知道是哪一个。没有任何东西能阻止 Alice 在声称消息是由 Bob 发送的同时创建消息。没有任何东西能阻止 Bob 在声称消息是由 Alice 创建的同时发送消息。Alice 和 Bob 都知道消息的作者是谁，但他们无法向任何其他人证明作者是谁。

当一个系统阻止参与者否认他们的行为时，我们称之为*不可否认性*。在这种情况下，Bob 将无法否认他的行为，即发送消息。在现实世界中，不可否认性通常在消息代表在线交易时使用。例如，销售点系统可能以不可否认性作为将商业伙伴法律约束以履行协议的一种方式。这些系统允许第三方，如法律机构，验证每笔交易。

如果 Alice、Bob 和 Charlie 想要不可否认性，Alice 和 Bob 将不得不停止共享密钥并开始使用数字签名。

### 5.3.1 数字签名

*数字签名*比数据验证和数据完整性更进一步，以确保不可否认性。数字签名允许任何人，而不仅仅是接收者，回答两个问题：谁发送了消息？消息在传输过程中是否被修改？数字签名与手写签名有许多相似之处：

+   两种签名类型都是签名者独特的。

+   两种签名类型都可以用来将签署者与合同法律约束起来。

+   两种签名类型都难以伪造。

数字签名通常是通过将哈希函数与公钥加密相结合而创建的。要对消息进行数字签名，发送方首先对消息进行哈希处理。哈希值和发送者的私钥然后成为一个非对称加密算法的*输入*；此算法的*输出*是消息发送者的数字签名。换句话说，明文是哈希值，密文是数字签名。然后一起传输消息和数字签名。图 5.2 描述了 Bob 如何实现此协议。

![CH05_F02_Byrne](img/CH05_F02_Byrne.png)

图 5.2 Bob 在发送给 Alice 之前使用私钥加密数字签名消息。

数字签名是与消息一起公开传输的；它不是一个秘密。有些程序员很难接受这一点。在一定程度上这是可以理解的：签名是密文，攻击者可以很容易地使用公钥解密它。请记住，尽管密文通常是隐藏的，但数字签名是一个例外。数字签名的目标是确保不可否认性，而不是保密性。如果攻击者解密了数字签名，他们不会获得私人信息。

### 5.3.2 RSA 数字签名

列表 5.5 展示了 Bob 对图 5.2 中所示想法的实现。此代码展示了如何使用 SHA-256、RSA 公钥加密以及一种名为概率签名方案（PSS）的填充方案对消息进行签名。`RSAPrivateKey.sign` 方法结合了这三个元素。

列表 5.5 Python 中的 RSA 数字签名

```py
import json
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.primitives import hashes

message = b'from Bob to Alice'

padding_config = padding.PSS(                     ❶
    mgf=padding.MGF1(hashes.SHA256()),            ❶
    salt_length=padding.PSS.MAX_LENGTH)           ❶

private_key = load_rsa_private_key()              ❷
signature = private_key.sign(                     ❸
    message,                                      ❸
    padding_config,                               ❸
    hashes.SHA256())                              ❸

signed_msg = {                                    ❹
    'message': list(message),                     ❹
    'signature': list(signature),                 ❹
}                                                 ❹
outbound_msg_to_alice = json.dumps(signed_msg)    ❹
```

❶ 使用 PSS 填充

❷ 使用列表 5.3 中所示的方法加载私钥

❸ 使用 SHA-256 进行签名

❹ 为 Alice 准备带有数字签名的消息

警告 RSA 数字签名和 RSA 公钥加密的填充方案不同。推荐使用 OAEP 填充进行 RSA 加密；推荐使用 PSS 填充进行 RSA 数字签名。这两种填充方案不能互换。

在接收到 Bob 的消息和签名后，但在信任消息之前，Alice 验证签名。

### 5.3.3 RSA 数字签名验证

在 Alice 接收到 Bob 的消息和数字签名后，她会执行三件事：

1.  她对消息进行哈希。

1.  她使用 Bob 的公钥解密签名。

1.  她比较哈希值。

如果 Alice 的哈希值与解密的哈希值匹配，她就知道可以信任该消息。图 5.3 描绘了 Alice，接收方，如何实现协议的一部分。

![CH05_F03_Byrne](img/CH05_F03_Byrne.png)

图 5.3 Alice 接收 Bob 的消息并使用公钥解密验证他的签名。

列表 5.6 展示了 Alice 对图 5.3 中所示协议的实现。数字签名验证的所有三个步骤都委托给了 `RSAPublicKey.verify`。如果计算的哈希值与 Bob 解密的哈希值不匹配，`verify` 方法将抛出 `InvalidSignature` 异常。如果哈希值匹配，Alice 就知道消息没有被篡改，消息只能由拥有 Bob 的私钥的人发送，大概是 Bob。

列表 5.6 Python 中的 RSA 数字签名验证

```py
import json
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.exceptions import InvalidSignature

def receive(inbound_msg_from_bob):
    signed_msg = json.loads(inbound_msg_from_bob)    ❶
    message = bytes(signed_msg['message'])           ❶
    signature = bytes(signed_msg['signature'])       ❶

    padding_config = padding.PSS(                    ❷
        mgf=padding.MGF1(hashes.SHA256()),           ❷
        salt_length=padding.PSS.MAX_LENGTH)          ❷

    private_key = load_rsa_private_key()             ❸
    try:
        private_key.public_key().verify(             ❹
            signature,                               ❹
            message,                                 ❹
            padding_config,                          ❹
            hashes.SHA256())                         ❹
        print('Trust message')
    except InvalidSignature:
        print('Do not trust message')
```

❶ 接收消息和签名

❷ 使用 PSS 填充

❸ 使用列表 5.3 中所示的方法加载私钥

❹ 将签名验证委托给 verify 方法

Charlie，第三方，可以像 Alice 一样验证消息的来源。因此，Bob 的签名确保了不可否认性。他不能否认自己是消息的发送者，除非他还声称自己的私钥已被泄露。

Eve，一个中间人，如果她试图干预协议，将会失败。她可以尝试在传输到 Alice 的过程中修改消息、签名或公钥。在这三种情况下，签名都将无法通过验证。修改消息会影响 Alice 计算的哈希值。修改签名或公钥会影响 Alice 解密的哈希值。

本节深入探讨了数字签名作为非对称加密的应用。使用 RSA 密钥对进行这样的操作是安全、可靠且经过实战检验的。不幸的是，非对称加密并不是数字签名的最佳方式。下一节将介绍一个更好的替代方案。

### 5.3.4 椭圆曲线数字签名

与 RSA 一样，椭圆曲线密码系统围绕密钥对的概念展开。与 RSA 密钥对一样，椭圆曲线密钥对用于签署数据和验证签名；与 RSA 密钥对不同的是，椭圆曲线密钥对不对数据进行非对称加密。换句话说，RSA 私钥解密其公钥加密的内容，反之亦然。椭圆曲线密钥对不支持这种功能。

那么，为什么有人会选择椭圆曲线而不是 RSA？椭圆曲线密钥对可能无法对数据进行非对称加密，但在签署数据方面速度更快。因此，椭圆曲线密码系统已成为数字签名的现代方法，吸引人们摆脱 RSA，降低计算成本。

RSA 并不不安全，但椭圆曲线密钥对在签署数据和验证签名方面效率更高。例如，256 位椭圆曲线密钥的强度可与 3072 位 RSA 密钥相媲美。椭圆曲线和 RSA 之间的性能对比反映了这些算法使用的基础数学模型。椭圆曲线密码系统使用椭圆曲线；RSA 数字签名使用整数因子分解。

列表 5.7 演示了 Bob 如何生成一个椭圆曲线密钥对，并使用 SHA-256 对消息进行签名。与 RSA 相比，这种方法需要更少的 CPU 周期和更少的代码行数。私钥是使用 NIST 批准的椭圆曲线 SECP384R1 或 P-384 生成的。

列表 5.7 在 Python 中椭圆曲线数字签名

```py
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec

message = b'from Bob to Alice'

private_key = ec.generate_private_key(ec.SECP384R1(), default_backend())

signature = private_key.sign(message, ec.ECDSA(hashes.SHA256()))    ❶
```

❶ 使用 SHA-256 进行签名

列表 5.8 继续上一列表 5.7，演示了 Alice 如何验证 Bob 的签名。与 RSA 一样，公钥从私钥中提取；如果签名未通过验证，`verify` 方法会抛出 `InvalidSignature`。

列表 5.8 在 Python 中椭圆曲线数字签名验证

```py
from cryptography.exceptions import InvalidSignature

public_key = private_key.public_key()   ❶

try:
    public_key.verify(signature, message, ec.ECDSA(hashes.SHA256()))
except InvalidSignature:                ❷
    pass                                ❷
```

❶ 提取公钥

❷ 处理验证失败

有时重新对消息进行哈希是不可取的。当处理大型消息或大量消息时，通常会出现这种情况。`sign` 方法，针对 RSA 密钥和椭圆曲线密钥，通过让调用者负责生成哈希值来适应这些情况。这使调用者可以选择高效地对消息进行哈希或重用先前计算的哈希值。下一个列表演示了如何使用 `Prehashed` 实用类对大型消息进行签名。

列表 5.9 在 Python 中高效签署大型消息

```py
import hashlib
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import ec, utils

large_msg = b'from Bob to Alice ...'              ❶
sha256 = hashlib.sha256()                         ❶
sha256.update(large_msg[:8])                      ❶
sha256.update(large_msg[8:])                      ❶
hash_value = sha256.digest()                      ❶

private_key = ec.generate_private_key(ec.SECP384R1(), default_backend())

signature = private_key.sign(                     ❷
    hash_value,                                   ❷
    ec.ECDSA(utils.Prehashed(hashes.SHA256())))   ❷
```

❶ 调用者高效地对消息进行哈希

❷ 使用 Prehashed 实用类进行签名

到目前为止，您已经掌握了散列、加密和数字签名的工作知识。您学到了以下内容：

+   散列确保数据的完整性和数据的认证。

+   加密确保机密性。

+   数字签名确保不可否认。

本章介绍了`cryptography`包中的许多低级示例，供教学目的使用。这些低级示例为您准备了下一章我将介绍的高级解决方案，即传输层安全性所需的一切。这种网络协议将您迄今所学的关于散列、加密和数字签名的一切内容汇集在一起。

## 总结

+   非对称加密算法使用不同的密钥进行加密和解密。

+   公钥加密是解决密钥分发问题的方案。

+   RSA 密钥对是一种经典且安全的非对称加密数据的方式。

+   数字签名保证不可否认。

+   椭圆曲线数字签名比 RSA 数字签名更有效。
