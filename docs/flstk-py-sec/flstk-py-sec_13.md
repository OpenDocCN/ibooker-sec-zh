# 第十一章：OAuth 2

本章内容

+   注册 OAuth 客户端

+   请求对受保护资源的授权

+   授权而不暴露身份验证凭据

+   访问受保护的资源

*OAuth 2*是由 IETF 定义的行业标准授权协议。这个协议，我简称为*OAuth*，使用户能够授权第三方访问受保护的资源。最重要的是，它允许用户在不向第三方暴露他们的身份验证凭据的情况下这样做。在本章中，我将解释 OAuth 协议，并与 Alice、Bob 和 Charlie 一起详细介绍它。Eve 和 Mallory 也会出现。我还会向你展示如何使用两个很棒的工具 Django OAuth Toolkit 和`requests-oauthlib`来实现这个协议。

你可能已经使用过 OAuth 了。你是否曾经访问过 medium.com 这样的网站，可以使用“使用 Google 登录”或“使用 Twitter 登录”？这个功能，称为*社交登录*，旨在简化账户创建。这些网站不会纠缠你的个人信息，而是要求你允许它们从社交媒体网站检索你的个人信息。在底层，这通常是使用 OAuth 实现的。

在我们深入研究这个主题之前，我将用一个例子来说明一些词汇术语。这些术语由 OAuth 规范定义；它们在本章中反复出现。当你去 medium.com 并使用 Google 登录时

+   你的谷歌账户信息是*受保护的资源*。

+   你是*资源所有者*；资源所有者是一个实体，通常是最终用户，有权授权访问受保护的资源。

+   Medium.com 是*OAuth 客户端*，一个第三方实体，当资源所有者允许时可以访问受保护的资源。

+   谷歌托管着*授权服务器*，允许资源所有者授权第三方访问受保护的资源。

+   谷歌还托管着*资源服务器*，守护着受保护的资源。

在现实世界中，资源服务器有时被称为*API*。在本章中，我避免使用这个术语，因为它有歧义。授权服务器和资源服务器几乎总是属于同一组织；对于小型组织来说，它们甚至是同一个服务器。图 11.1 展示了这些角色之间的关系。

![CH11_F01_Byrne](img/CH11_F01_Byrne.png)

图 11.1 Google 通过 OAuth 进行社交登录

谷歌和第三方网站通过实现一个工作流程进行合作。这个工作流程，或*授权类型*，由 OAuth 规范定义。在下一节中，你将详细了解这个授权类型。

## 11.1 授权类型

*授权类型*定义了资源所有者如何授予对受保护资源的访问权限。OAuth 规范定义了四种授权类型。在本书中，我只讨论了一种，即授权码。这种授权类型占据了绝大多数 OAuth 使用情况；现在先不要关注其他三种。以下列表概述了每种授权类型及其适用的用例：

+   *授权码授权*适用于网站、移动应用程序和基于浏览器的应用程序。

+   *隐式授权*曾经是移动和基于浏览器的应用程序的推荐授权类型。但这种授权类型已经被弃用。

+   *密码授权*通过要求资源所有者通过第三方提供其凭据，从而消除了对授权服务器的需求。

+   *客户端凭据授权*适用于资源所有者和第三方是同一实体的情况。

在您的工作和个人生活中，您可能只会看到授权码授权。隐式授权已被弃用，密码授权固有地不太安全，客户端凭据授权的用例很少见。下一节将介绍授权码流，OAuth 的主要部分。

### 11.1.1 授权码流

*授权* *码流*由一个明确定义的协议实现。在此协议开始之前，第三方必须首先注册为授权服务器的 OAuth 客户端。OAuth 客户端注册为协议建立了几个前提条件，包括 OAuth 客户端的名称和凭据。协议中的每个参与者在协议的各个阶段使用此信息。

授权码流协议分为四个阶段：

1.  请求授权

1.  授予权限

1.  执行令牌交换

1.  访问受保护的资源

四个阶段中的第一个始于资源所有者访问 OAuth 客户端站点时。

请求授权

在协议的此阶段（图 11.2 中所示）期间，OAuth 客户端通过将资源所有者发送到授权服务器来请求授权。通过普通链接、HTTP 重定向或 JavaScript，站点将资源所有者指向*授权 URL*。这是授权服务器托管的授权表单的地址。

![CH11_F02_Byrne](img/CH11_F02_Byrne.png)

图 11.2 资源所有者访问第三方站点；该站点将其指向授权服务器托管的授权表单。

下一阶段开始时，授权服务器向资源所有者呈现授权表单。

授予权限

在协议的此阶段（图 11.3 中所示）期间，资源所有者通过授权服务器向 OAuth 客户端授予对受保护资源的访问权限。授权表单负责确保资源所有者做出知情决定。然后，资源所有者通过提交授权表单来授予权限。

接下来，授权服务器将资源所有者重定向回到他们来自的地方，即 OAuth 客户端站点。这是通过将他们重定向到一个称为*重定向 URI*的 URL 来完成的。第三方在 OAuth 客户端注册过程中预先设置了重定向 URI。

![CH11_F03_Byrne](img/CH11_F03_Byrne.png)

图 11.3 资源所有者通过提交授权表单来授予授权；授权服务器会使用授权码将所有者重定向回第三方站点。

授权服务器将一个重要的查询参数附加到重定向 URI 中；此查询参数被命名为`code`，如*authorization code*中所示。换句话说，授权服务器通过将其反射到资源所有者身上将授权码传递给 OAuth 客户端。

第三阶段始于 OAuth 客户端从入站重定向 URI 中解析授权码。

执行令牌交换

在此阶段，如图 11.4 所示，OAuth 客户端会将授权码交换为访问令牌。然后，该代码将与 OAuth 客户端注册凭据一起直接发送回到它来自的地方，即授权服务器。

授权服务器验证代码和 OAuth 客户端凭据。代码必须熟悉、未使用、最近的，并且与 OAuth 客户端标识符关联。客户端凭据必须有效。如果满足每个标准，授权服务器将响应一个访问令牌。

![CH11_F04_Byrne](img/CH11_F04_Byrne.png)

图 11.4 从重定向 URI 解析授权码后，OAuth 客户端将其发送回到它来自的地方；授权服务器将响应一个访问令牌。

最后一个阶段始于 OAuth 客户端向资源服务器发送请求。

访问受保护资源

在此阶段，如图 11.5 所示，OAuth 客户端使用访问令牌来访问受保护的资源。此请求在标头中携带访问令牌。资源服务器负责验证访问令牌。如果令牌有效，则授予 OAuth 客户端对受保护资源的访问权限。

![CH11_F05_Byrne](img/CH11_F05_Byrne.png)

图 11.5 使用访问令牌，第三方站点向资源服务器请求受保护的资源。

图 11.6 描述了从开始到结束的授权码流程。

![CH11_F06_Byrne](img/CH11_F06_Byrne.png)

图 11.6 我们的 OAuth 授权码流程

在下一节中，我将再次与 Alice、Bob 和 Charlie 一起详细介绍此协议。

## 11.2 Bob 授权 Charlie

在之前的章节中，你为 Alice 制作了一个网站；Bob 注册为它的用户。在这个过程中，Bob 信任 Alice 的个人信息，即他的电子邮件。在本节中，Alice、Bob 和 Charlie 合作开展一个新的工作流程。Alice 将她的网站转变为授权服务器和资源服务器。Charlie 的新网站请求 Bob 的许可，以从 Alice 的网站检索 Bob 的电子邮件。Bob 授权 Charlie 的网站，而不会暴露他的身份验证凭据。在下一节中，我将向你展示如何实现这个工作流程。

这个工作流程是之前介绍的授权授予类型的实现。它从 Charlie 开始，他在 Python 中构建一个新网站。Charlie 决定通过 OAuth 与 Alice 的网站集成。这提供了以下好处：

+   Charlie 可以要求 Bob 提供他的电子邮件地址。

+   Bob 更有可能分享他的电子邮件地址，因为他不需要输入它。

+   Charlie 避免构建用户注册和电子邮件确认的工作流程。

+   Bob 少记住一个密码。

+   Charlie 不需要承担管理 Bob 密码的责任。

+   Bob 节省了时间。

作为 authorize.alice.com 的超级用户，Alice 通过她的网站的管理控制台为 Charlie 注册了一个 OAuth 客户端。图 11.7 展示了 OAuth 客户端注册表单。请花一分钟观察这个表单有多少熟悉的字段。这个表单包含了 OAuth 客户端凭据、名称和重定向 URI 的字段。请注意，授权码选项被选中为授权授予类型字段。

![CH11_F07_Byrne](img/CH11_F07_Byrne.png)

图 11.7 Django 管理控制台中的 OAuth 客户端注册表单

### 11.2.1 请求授权

Bob 访问 Charlie 的网站，client.charlie.com。Bob 对这个网站不熟悉，所以它呈现了接下来的链接。这个链接的地址是一个授权 URL；它是由授权服务器 authorize.alice.com 托管的授权表单的地址。授权 URL 的前两个查询参数是必需的，以粗体字显示。`response_type`参数设置为`code`，就像*授权码*一样。第二个参数是 Charlie 的 OAuth 客户端 ID：

```py
<a href='https:/./authorize.alice.com/o/authorize/?
➥ response_type=code&                                    # ❶
➥ client_id=Q7kuJVjbGbZ6dGlwY49eFP7fNFEUFrhHGGG84aI3&    # ❶
➥ state=ju2rUmafnEIxvSqphp3IMsHvJNezWb'>                 # ❷
    What is your email?
</a>
```

❶ 必需的查询参数

❷ 一个可选的安全功能

`state`参数是一个可选的安全功能。稍后，当 Bob 授权 Charlie 的网站后，Alice 的授权服务器将通过将其附加到重定向 URI 来将此参数回显到 Charlie 的网站。我稍后会解释为什么，在本节的结尾。

### 11.2.2 授予权限

Bob 通过点击链接导航到 authorize.alice.com。Bob 碰巧已经登录，所以 authorize.alice.com 不会麻烦他进行身份验证；授权表单立即呈现。这个表单的目的是确保 Bob 做出知情决定。表单询问 Bob 是否愿意将他的电子邮件地址提供给 Charlie 的网站，使用 Charlie 的 OAuth 客户端的名称。

Bob 通过提交授权表单来授予权限。然后，Alice 的授权服务器将他重定向回 Charlie 的站点。重定向 URI 包含两个参数。授权码由 code 参数携带，如粗体所示；Charlie 的站点稍后将用此来交换访问令牌。state 参数的值与通过授权 URL 到达的值匹配：

```py
https:/./client.charlie.com/oauth/callback/?    # ❶
➥ code=CRN7DwyquEn99mrWJg5iAVVlJZDTzM&        # ❷
➥ state=ju2rUmafnEIxvSqphp3IMsHvJNezWb        # ❸
```

❶ 重定向 URI

❷ 授权码

❸ 将状态回映到 Charlie 的站点

### 11.2.3 令牌交换

Charlie 的站点通过解析重定向 URI 中的代码并将其直接发送回 Alice 的授权服务器来开始此阶段。Charlie 通过调用一个称为*令牌端点*的服务来执行此操作。其目的是验证传入的授权码并将其交换为访问令牌。此令牌包含在令牌端点响应的主体中。

访问令牌很重要；任何拥有此令牌的人或机器都可以请求 Bob 的电子邮件，而无需他的用户名或密码。Charlie 的站点甚至不让 Bob 看到令牌。由于这个令牌非常重要，它受到*可以用于什么*和*可以使用多长时间*的限制。这些限制由令牌端点响应中的两个附加字段指定：`scope`和`expires_in`。

接下来显示了令牌端点响应主体。访问令牌，范围和到期时间如粗体所示。此响应指示 Alice 的授权服务器允许 Charlie 的站点使用一个 36,000 秒（10 小时）有效的访问令牌访问 Bob 的电子邮件：

```py
{
 'access_token': 'A2IkdaPkmAjetNgpCRNk0zR78DUqoo',   # ❶
 'token_type': 'Bearer'                              # ❶
 'scope': 'email',                                   # ❷
 'expires_in': 36000,                                # ❷
 ...
}
```

❶ 指定权限

❷ 通过范围和时间限制权限

### 11.2.4 访问受保护的资源

最后，Charlie 的站点使用访问令牌从 Alice 的资源服务器检索 Bob 的电子邮件。此请求通过`Authorization`请求头将访问令牌传递到资源服务器。访问令牌如粗体所示：

```py
GET /protected/name/ HTTP/1.1
Host: resource.alice.com
Authorization: Bearer A2IkdaPkmAjetNgpCRNk0zR78DUqoo
```

Alice 的资源服务器有责任验证访问令牌。这意味着受保护的资源，即 Bob 的电子邮件，处于范围内，并且访问令牌尚未过期。最后，Charlie 的站点收到一个包含 Bob 的电子邮件的响应。最重要的是，Charlie 的站点在没有 Bob 的用户名或密码的情况下完成了这个操作。

阻止 Mallory

你还记得 Charlie 的站点在授权 URL 中附加了一个 state 参数吗？然后 Alice 的授权服务器通过在重定向 URI 中附加完全相同的参数来回映它？Charlie 的站点通过将 state 参数设置为随机字符串使每个授权 URL 变得唯一。当字符串返回时，站点将其与发送的本地副本进行比较。如果值匹配，Charlie 的站点得出结论，Bob 只是按预期从 Alice 的授权服务器返回。

如果重定向 URI 中的状态值与授权 URL 的状态值不匹配，查理的站点将中止流程；甚至不会尝试将授权代码交换为访问令牌。为什么？因为如果鲍勃是从爱丽丝那里获取重定向 URI 的话，这种情况是不可能发生的。而只有当鲍勃从其他人那里获取重定向 URI（比如玛洛丽）时，才会发生这种情况。

假设爱丽丝和查理不支持这个可选的安全检查。玛洛丽注册为爱丽丝网站的用户。然后，她从爱丽丝的服务器请求授权表单。玛洛丽提交了授权表单，允许查理的站点访问她帐户的电子邮件地址。但是，她没有按照重定向 URI 返回到查理的站点，而是把重定向 URI 发送给鲍勃，作为恶意电子邮件或聊天消息。鲍勃上钩了，按照玛洛丽的重定向 URI 进行了跟踪。这将他带到了查理的站点，并带有玛洛丽账户的有效授权代码。

查理的站点将玛洛丽的代码交换为有效的访问令牌。它使用访问令牌检索玛洛丽的电子邮件地址。玛洛丽现在有机会欺骗查理和鲍勃。首先，查理的站点可能会错误地将玛洛丽的电子邮件地址分配给鲍勃。其次，鲍勃可能会从查理的站点获取有关自己个人信息的错误印象。现在想象一下，如果查理的站点请求其他形式的个人信息——例如健康记录——情况会有多严重。图 11.8 描绘了玛洛丽的攻击。

![CH11_F08_Byrne](img/CH11_F08_Byrne.png)

图 11.8 玛洛丽诱使鲍勃将她的授权代码提交给查理。

在这一节中，你看到了爱丽丝、鲍勃和查理在对抗玛洛丽的同时合作进行工作流程。这个工作流程涵盖了客户注册、授权、令牌交换和资源访问。在接下来的两节中，你将学习如何使用两个新工具构建这个工作流程，即 Django OAuth Toolkit 和 `requests-oauthlib`。

## 11.3 Django OAuth Toolkit

在本节中，我将向你展示如何将任何 Django 应用服务器转换为授权服务器、资源服务器或两者兼具。在此过程中，我将向你介绍一个重要的 OAuth 构造，称为 *scopes*。Django OAuth Toolkit（DOT）是一个在 Python 中实现授权和资源服务器的优秀库。DOT 借助一系列可定制的视图、装饰器和实用程序将 OAuth 带到 Django 中。它还与 `requests-oauthlib` 很好地配合；这两个框架都将繁重的工作委托给一个称为 `oauthlib` 的第三方组件。

注意 `oauthlib` 是一个通用的 OAuth 库，没有 Web 框架依赖；这使得它可以在各种 Python Web 框架中使用，而不仅仅是 Django。

在你的虚拟环境中，使用以下命令安装 DOT：

```py
$ pipenv install django-oauth-toolkit
```

接下来，在你的 Django 项目的 `settings` 模块中安装 `oauth2_provider` Django 应用程序。这行代码，如下所示，属于授权服务器和资源服务器，而不是 OAuth 客户端应用程序：

```py
INSTALLED_APPS = [
    ...
    'oauth2_provider',     # ❶
]
```

❶ 将你的 Django 项目转换为授权服务器、资源服务器，或两者兼有

使用以下命令运行已安装的 `oauth2_provider` 应用的迁移。这些迁移创建的表存储授权代码、访问令牌以及注册的 OAuth 客户端的账户详情：

```py
$ python manage.py migrate oauth2_provider
```

在 urls.py 中添加以下路径条目。这包括一打负责 OAuth 客户端注册、授权、令牌交换等的端点：

```py
urlpatterns = [
    ...
    path('o/', include(
     'oauth2_provider.urls', namespace='oauth2_provider')),
]
```

重新启动服务器并登录到管理员控制台，在 /admin/ 路径下。管理员控制台欢迎页面除了认证和授权之外还有一个新的菜单用于 Django OAuth Toolkit。管理员可以从这个菜单中管理令牌、授权和 OAuth 客户端。

注意在现实世界中，授权服务器和资源服务器几乎总是属于同一组织。对于中小型实施（例如，不是 Twitter 或 Google），授权服务器和资源服务器是同一服务器。在本节中，我分别介绍了它们的角色，但出于简单起见，将它们的实现合并在一起。

在接下来的两个部分中，我将分解你的授权服务器和资源服务器的职责。这些职责包括支持一个重要的 OAuth 功能，称为*范围*。

### 11.3.1 授权服务器职责

DOT 提供用于处理授权服务器职责的 Web 用户界面、配置设置和工具。这些职责包括以下内容：

+   定义范围

+   验证资源所有者

+   生成重定向 URI

+   管理授权代码

定义范围

资源所有者通常希望对第三方访问进行细粒度的控制。例如，Bob 可能愿意与 Charlie 分享他的电子邮件，但不分享他的聊天记录或健康记录。OAuth 通过范围满足了这种需求。*范围*需要协议的每个参与者进行协调；它们由授权服务器定义，由 OAuth 客户端请求，并由资源服务器执行。

范围在授权服务器的 `settings` 模块中使用 `SCOPES` 设置进行定义。此设置是一组键值对。每个键表示范围对机器的意义；每个值表示范围对人的意义。键最终出现在授权 URL 和重定向 URI 的查询参数中；值在授权表单中显示给资源所有者。

确保你的授权服务器配置了一个邮件范围，如下面代码中的粗体所示。与其他 DOT 配置设置一样，`SCOPES` 位于方便的 `OAUTH2_PROVIDER` 命名空间下：

```py
OAUTH2_PROVIDER = {     # ❶
    ...
 'SCOPES': {
 'email': 'Your email',
 'name': 'Your name',
 ...
 },
    ...
}
```

❶ Django OAuth Toolkit 配置命名空间

范围是由 OAuth 客户端可选请求的。这是通过将一个可选的查询参数附加到授权 URL 上实现的。该参数名为 `scope`，伴随着 `client_id` 和 `state` 参数。

如果授权 URL 没有 `scope` 参数，授权服务器将回退到一组默认范围。默认范围由授权服务器中的 `DEFAULT_SCOPES` 设置定义。该设置表示在授权 URL 没有范围参数时要使用的范围列表。如果未指定，该设置默认为 `SCOPES` 中的所有内容：

```py
OAUTH2_PROVIDER = {
    ...
 'DEFAULT_SCOPES': ['email', ],
    ...
}
```

资源所有者身份验证

身份验证是授权的先决条件；因此，如果资源所有者尚未登录，则服务器必须向其挑战以获取身份验证凭据。DOT 通过利用 Django 身份验证来避免重复发明轮子。资源所有者使用与直接进入网站时相同的常规登录页面进行身份验证。

您的登录页面只需添加一个额外的隐藏输入字段。这个字段在这里用粗体显示，让服务器在用户登录后将用户重定向到授权表单：

```html
<html>
    <body>

       <form method='POST'>
         {% csrf_token %}                                  <!-- ❶ -->
         {{ form.as_p }}                                   <!-- ❷ -->
 <input type="hidden" name="next" value="{{ next }}" />    <!-- ❸ -->
         <button type='submit'>Login</button>
       </form>

    </body>
</html>
```

❶ 必要，但在第十六章中已涵盖

❷ 动态呈现为用户名和密码表单字段

❸ 隐藏的 HTML 字段

生成重定向 URI

DOT 为您生成重定向 URI，但默认情况下将支持 HTTP 和 HTTPS。以这种方式推送您的系统到生产环境是一个非常糟糕的主意。

警告 每个生产重定向 URI 应该使用 HTTPS，而不是 HTTP。在授权服务器中强制执行这一点，而不是在每个 OAuth 客户端中。

假设 Alice 的授权服务器通过 HTTP 将 Bob 重定向回 Charlie 的站点，并使用重定向 URI。这将向网络窃听者 Eve 显示代码和状态参数。Eve 现在有可能在 Charlie 之前将 Bob 的授权码交换为访问令牌。图 11.9 展示了 Eve 的攻击。当然，她需要 Charlie 的 OAuth 客户端凭据才能成功。

![CH11_F09_Byrne](img/CH11_F09_Byrne.png)

图 11.9 Bob 收到 Alice 的授权码；Eve 拦截该代码并在 Charlie 之前将其发送回 Alice。

将 `ALLOWED_REDIRECT_URI_SCHEMES` 设置添加到 `settings` 模块中，如下所示，以强制所有重定向 URI 使用 HTTPS。该设置是一个字符串列表，表示允许重定向 URI 具有哪些协议：

```py
OAUTH2_PROVIDER = {
    ...
 'ALLOWED_REDIRECT_URI_SCHEMES': ['https'],
    ...
}
```

管理授权码

每个授权码都有一个过期时间。资源所有者和 OAuth 客户端负责在此时间限制内操作。授权服务器不会将过期的授权码交换为访问令牌。这对于攻击者来说是一种威慑，对于资源所有者和 OAuth 客户端来说是一个合理的障碍。如果攻击者设法拦截授权码，他们必须能够快速将其交换为访问令牌。

使用`AUTHORIZATION_CODE_EXPIRE_SECONDS`设置来配置授权码的过期时间。此设置表示授权码的生存时间，以秒为单位。此设置在授权服务器中配置，并由其执行。此设置的默认值为 1 分钟；OAuth 规范建议最长为 10 分钟。以下示例配置 DOT 拒绝任何早于 10 秒的授权码：

```py
OAUTH2_PROVIDER = {
    ...
 'AUTHORIZATION_CODE_EXPIRE_SECONDS': 10,
    ...
}
```

DOT 提供了一个授权码管理的管理控制台 UI。通过点击管理员控制台欢迎页面上的授权码链接或导航到/admin/oauth2_provider/grant/来访问授权页面。管理员使用此页面搜索和手动删除授权码。

管理员通过点击任何授权码来导航到授权码详情页面。该页面允许管理员查看或修改授权码属性，如过期时间、重定向 URI 或范围。

### 11.3.2 资源服务器责任

与授权服务器开发一样，DOT 提供了用于处理资源服务器责任的 Web 界面、配置设置和实用程序。这些责任包括以下内容：

+   管理访问令牌

+   为受保护的资源提供服务

+   强制作用域

管理访问令牌

与授权码一样，访问令牌也有一个过期时间。资源服务器通过拒绝任何带有过期访问令牌的请求来执行此过期。这不会阻止访问令牌落入错误手中，但如果发生这种情况，可以限制损害。

使用`ACCESS_TOKEN_EXPIRE_SECONDS`设置来配置每个访问令牌的生存时间。默认值在这里以粗体显示，为 36,000 秒（10 小时）。在您的项目中，此值应尽可能短，但足以让 OAuth 客户端完成其工作：

```py
OAUTH2_PROVIDER = {
    ...
 'ACCESS_TOKEN_EXPIRE_SECONDS': 36000,
    ...
}
```

DOT 提供了一个类似于授权码管理页面的访问令牌管理界面。管理员可以通过点击管理员控制台欢迎页面上的访问令牌链接或导航到/admin/oauth2_provider/accesstoken/来访问访问令牌页面。管理员使用此页面搜索和手动删除访问令牌。

从访问令牌页面，管理员导航到访问令牌详情页面。管理员使用访问令牌详情页面来查看和修改访问令牌属性，如过期时间。

为受保护的资源提供服务

与未受保护的资源一样，受保护的资源由视图提供服务。在您的资源服务器中添加清单 11.1 中的视图定义。注意`EmailView`扩展了粗体显示的`ProtectedResourceView`。这确保了只有持有有效访问令牌的授权 OAuth 客户端才能访问用户的电子邮件。

清单 11.1 使用 ProtectedResourceView 为受保护资源提供服务

```py
from django.http import JsonResponse
from oauth2_provider.views import ProtectedResourceView

class EmailView(ProtectedResourceView):     # ❶
    def get(self, request):                 # ❷
        return JsonResponse({               # ❸
            'email': request.user.email,    # ❸
        })                                  # ❸
```

❶ 需要有效的访问令牌

❷ 被像 client.charlie.com 这样的 OAuth 客户端调用

❸ 为 Bob 的电子邮件等受保护的资源提供服务

当 OAuth 客户端请求受保护的资源时，它肯定不会发送用户的 HTTP 会话 ID。（在第七章，您了解到会话 ID 是一个用户与一个服务器之间的重要秘密。）那么，资源服务器如何确定请求适用于哪个用户？它必须从访问令牌开始工作。DOT 通过`OAuth2TokenMiddleware`透明地执行这一步。这个类从访问令牌推断用户，并将`request.user`设置为如果受保护的资源请求直接来自用户。

打开您的设置文件，并将`OAuth2TokenMiddleware`，如下面的加粗所示，添加到`MIDDLEWARE`中。确保您将此组件放在`SecurityMiddleware`之后：

```py
MIDDLEWARE = [
    ...
 'oauth2_provider.middleware.OAuth2TokenMiddleware',
]
```

`OAuth2TokenMiddleware`通过`OAuth2Backend`的帮助解析用户，如下面的加粗所示。将此组件添加到`settings`模块中的`AUTHENTICATION_BACKENDS`中。确保内置的`ModelBackend`仍然完好无损；这个组件对终端用户身份验证是必要的：

```py
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',    # ❶
 'oauth2_provider.backends.OAuth2Backend',       # ❷
]
```

❶ 验证用户

❷ 验证 OAuth 客户端

强制执行范围

DOT 资源服务器使用`ScopedProtectedResourceView`强制执行范围。从这个类继承的视图不仅需要一个有效的访问令牌；它们还确保受保护的资源在访问令牌的范围内。

列表 11.2 定义了`ScopedEmailView`，它是`ScopedProtectedResourceView`的子类。与列表 11.1 中的`EmailView`相比，`ScopedEmailView`只有两个小差别，如下面的加粗所示。首先，它继承自`ScopedProtectedResourceView`而不是`ProtectedResourceView`。其次，`required_scopes`属性定义了要强制执行的范围。

列表 11.2 使用`ScopedProtectedResourceView`提供受保护的资源

```py
from django.http import JsonResponse
from oauth2_provider.views import ScopedProtectedResourceView

class ScopedEmailView(ScopedProtectedResourceView):    # ❶
 required_scopes = ['email', ]                      # ❷

    def get(self, request):
        return JsonResponse({
            'email': request.user.email,
        })
```

❶ 需要有效的访问令牌并强制执行范围

❷ 指定要强制执行的范围

将范围分为两类通常很有用：读取或写入。这使资源所有者能够更精细地控制。例如，Bob 可能授予 Charlie 对他的电子邮件的读取访问权限和对他的姓名的写入访问权限。这种方法有一个不幸的副作用：它使范围的数量翻倍。DOT 通过本地支持读取和写入范围的概念来避免这个问题。

DOT 资源服务器使用`ReadWriteScopedResourceView`自动强制执行读取和写入范围。这个类比`ScopedProtectedResourceView`进一步验证入站访问令牌的范围与请求的方法是否匹配。例如，如果请求方法是`GET`，则访问令牌必须具有读取范围；如果请求方法是`POST`或`PATCH`，则必须具有写入范围。

列表 11.3 定义了`ReadWriteEmailView`，它是`ReadWriteScopedResourceView`的子类。`ReadWriteEmailView`允许 OAuth 客户端通过使用`get`方法和`patch`方法分别读取和写入资源所有者的电子邮件。传入的访问令牌必须具有读取和邮件范围以使用`get`方法；它必须具有写入和邮件范围以使用`patch`方法。读取和写入范围不会出现在`required_scopes`中；它们是隐式的。

列表 11.3 使用 ReadWriteScopedResourceView 提供保护服务

```py
import json
from django.core.validators import validate_email
from oauth2_provider.views import ReadWriteScopedResourceView

class ReadWriteEmailView(ReadWriteScopedResourceView):
    required_scopes = ['email', ]

    def get(self, request):                   # ❶
        return JsonResponse({                 # ❶
            'email': request.user.email,      # ❶
        })                                    # ❶

    def patch(self, request):                 # ❷
        body = json.loads(request.body)       # ❷
        email = body['email']                 # ❷
        validate_email(email)                 # ❷
        user = request.user                   # ❷
        user.email = email                    # ❷
        user.save(update_fields=['email'])    # ❷
        return HttpResponse()                 # ❷
```

❶ 需要读取和邮件范围

❷ 需要写入和邮件范围

基于函数的视图

DOT 为基于函数的视图提供函数装饰器。这里粗体显示的`@protected_resource`装饰器在功能上类似于`ProtectedResourceView`和`ScopedProtectedResourceView`。单独使用此装饰器确保调用者拥有访问令牌。`scopes`参数确保访问令牌具有足够的范围：

```py
from oauth2_provider.decorators import protected_resource

@protected_resource()                        # ❶
def protected_resource_view_function(request):
    ...
    return HttpResponse()

@protected_resource(scopes=['email'])        # ❷
def scoped_protected_resource_view_function(request):
    ...
    return HttpResponse()
```

❶ 需要有效的访问令牌

❷ 需要有效的带有邮件范围的访问令牌

这里粗体显示的`rw_protected_resource` `decorator`在功能上类似于`ReadWriteScopedResourceView`。对于带有`rw_protected_resource`修饰的视图的 GET 请求必须携带具有读取范围的访问令牌。对于同一视图的 POST 请求必须携带具有写入范围的访问令牌。`scopes`参数指定了其他范围：

```py
from oauth2_provider.decorators import rw_protected_resource

@rw_protected_resource()                     # ❶
def read_write_view_function(request):
    ...
    return HttpResponse()

@rw_protected_resource(scopes=['email'])     # ❷
def scoped_read_write_view_function(request):
    ...
    return HttpResponse()
```

❶ GET 需要读取范围，POST 需要写入范围

❷ GET 需要读取和邮件范围，POST 需要写入和邮件范围

大多数主要使用 OAuth 的程序员是从客户端进行操作的。像查理这样的人比像爱丽丝这样的人更常见；OAuth 客户端比 OAuth 服务器自然要多。在下一节中，您将学习如何使用`requests-oauthlib`实现 OAuth 客户端。

## 11.4 requests-oauthlib

`requests-oauthlib`是在 Python 中实现 OAuth 客户端的出色库。此库将另外两个组件粘合在一起：`requests`包和`oauthlib`。在您的虚拟环境中，运行以下命令来安装`requests_oauthlib`：

```py
$ pipenv install requests_oauthlib
```

在第三方项目中声明一些常量，从客户端注册凭据开始。在本例中，我将客户端密钥存储在 Python 中。在生产系统中，您的客户端密钥应该安全地存储在密钥管理服务中，而不是您的代码库中：

```py
CLIENT_ID = 'Q7kuJVjbGbZ6dGlwY49eFP7fNFEUFrhHGGG84aI3'
CLIENT_SECRET = 'YyP1y8BCCqfsafJr0Lv9RcOVeMjdw3HqpvIPJeRjXB...'
```

接下来，定义授权表单、令牌交换端点和受保护资源的 URL：

```py
AUTH_SERVER = 'https:/./authorize.alice.com'
AUTH_FORM_URL = '%s/o/authorize/' % AUTH_SERVER
TOKEN_EXCHANGE_URL = '%s/o/token/' % AUTH_SERVER
RESOURCE_URL = 'https:/./resource.alice.com/protected/email/'
```

域名

在本章中，我使用诸如`authorize.alice.com`和`client.charlie.com`等域名，以避免将您与对 localhost 的含糊引用混淆。为了跟上内容，您不必在本地开发环境中这样做；使用 localhost 就可以了。

只需确保你的第三方服务器绑定到与授权服务器不同的端口即可。服务器的端口由`bind`参数指定，如下所示加粗显示：

```py
$ gunicorn third.wsgi --bind localhost:8001 \              # ❶
                      --keyfile path/to/private_key.pem \
                      --certfile path/to/certificate.pem
```

❶ 将服务器绑定到 8001 端口

在下一节中，你将使用这些配置设置来请求授权、获取访问令牌和访问受保护资源。

### 11.4.1 OAuth 客户端职责

`requests-oauthlib` 使用 `OAuth2Session` 处理 OAuth 客户端的职责，它是 Python OAuth 客户端的瑞士军刀。该类旨在自动完成以下操作：

+   生成授权 URL

+   将授权码交换为访问令牌

+   请求受保护资源

+   撤销访问令牌

将列表 11.4 中的视图添加到你的第三方项目中。`WelcomeView` 在用户的 HTTP 会话中查找访问令牌。然后，它请求两者之一：用户的授权或来自资源服务器的电子邮件。如果没有访问令牌可用，则渲染一个带有授权 URL 的欢迎页面；如果有访问令牌可用，则渲染一个带有用户电子邮件的欢迎页面。

列表 11.4 OAuth 客户端 WelcomeView

```py
from django.views import View
from django.shortcuts import render
from requests_oauthlib import OAuth2Session

class WelcomeView(View):
    def get(self, request):
        access_token = request.session.get('access_token')
        client = OAuth2Session(CLIENT_ID, token=access_token)
        ctx = {}

        if not access_token:
            url, state = client.authorization_url(AUTH_FORM_URL)    # ❶
            ctx['authorization_url'] = url                          # ❶
            request.session['state'] = state                        # ❶
        else:
            response = client.get(RESOURCE_URL)                     # ❷
            ctx['email'] = response.json()['email']                 # ❷

        return render(request, 'welcome.html', context=ctx)
```

❶ 请求授权

❷ 访问受保护资源

`OAuth2Session` 用于生成授权 URL 或检索受保护资源。请注意，状态值的副本存储在用户的 HTTP 会话中；期望授权服务器在协议的后续阶段回显此值。

接下来，将以下欢迎页面模板添加到你的第三方项目中。如果用户的电子邮件已知，则渲染用户的电子邮件。否则，渲染授权链接（加粗显示）：

```html
<html>
    <body>
        {% if email %}
            Email: {{ email }}
        {% else %}
            <a href='{{ authorization_url }}'>    <!-- ❶ -->
                What is your email?               <!-- ❶ -->
            </a>                                  <!-- ❶ -->
        {% endif %}
    </body>
</html>
```

❶ 请求授权

请求授权

有许多请求授权的方法。在本章中，我为了简单起见使用链接来完成。或者，你可以通过重定向来完成。此重定向可以在 JavaScript、视图或自定义中间件组件中进行。

接下来，将列表 11.5 中的视图添加到你的第三方项目中。与 `WelcomeView` 一样，`OAuthCallbackView` 首先通过会话状态初始化 `OAuth2Session`。此视图将令牌交换委托给 `OAuth2Session`，并提供重定向 URI 和客户端密钥。然后将访问令牌存储在用户的 HTTP 会话中，`WelcomeView` 可以访问它。最后，用户被重定向回欢迎页面。

列表 11.5 OAuth 客户端 OAuthCallbackView

```py
from django.shortcuts import redirect
from django.urls import reverse
from django.views import View

class OAuthCallbackView(View):
    def get(self, request):
        state = request.session.pop('state')
        client = OAuth2Session(CLIENT_ID, state=state)

        redirect_URI = request.build_absolute_uri()
        access_token = client.fetch_token(          # ❶
            TOKEN_EXCHANGE_URL,                     # ❶
            client_secret=CLIENT_SECRET,            # ❶
            authorization_response=redirect_URI)    # ❶
        request.session['access_token'] = access_token

        return redirect(reverse('welcome'))         # ❷
```

❶ 请求授权

❷ 将用户重定向回欢迎页面

`fetch_token` 方法为 `OAuthCallbackView` 执行了大量工作。首先，此方法从重定向 URI 中解析代码和状态参数。然后，它将入站状态参数与从用户的 HTTP 会话中提取的状态进行比较。如果两个值不匹配，则引发 `MismatchingStateError`，并且授权码永远不会被使用。如果两个状态值匹配，则 `fetch_token` 方法将授权码和客户端密钥发送到令牌交换端点。

撤销令牌

当你完成一个访问令牌后，通常没有理由继续持有它。你不再需要它，而且只有当它落入错误的手中时才会对你造成危害。因此，通常最好在访问令牌完成其目的后撤销每个访问令牌。一旦被撤销，访问令牌就无法用于访问受保护的资源。

DOT 通过一个专门的端点来处理令牌撤销。这个端点需要一个访问令牌和 OAuth 客户端凭据。以下代码演示了如何访问令牌撤销。请注意，资源服务器会用 403 状态码回应后续请求：

```py
>>> data = {
...     'client_id': CLIENT_ID,
...     'client_secret': CLIENT_SECRET,
...     'token': client.token['access_token']
... }
>>> client.post('%s/o/revoke_token/' % AUTH_SERVER, data=data)    # ❶
<Response [200]>                                                  # ❶
>>> client.get(RESOURCE_URL)                                      # ❷
<Response [403]>                                                  # ❷
```

❶ 撤销访问令牌

❷ 后续访问被拒绝

大型 OAuth 提供商通常允许你手动撤销为你的个人数据发布的访问令牌。例如，访问[`myaccount.google.com/permissions`](https://myaccount.google.com/permissions)查看为你的 Google 账户发布的所有有效访问令牌的列表。这个用户界面让你查看每个访问令牌的详细信息，并撤销它们。为了保护你的隐私，你应该撤销对任何你不打算很快使用的客户端应用程序的访问权限。

在这一章中，你学到了很多关于 OAuth 的知识。你从资源所有者、OAuth 客户端、授权服务器和资源服务器的角度了解了这个协议是如何工作的。你还接触到了 Django OAuth Toolkit 和`requests-oauthlib`。这些工具在它们的工作中表现出色，文档完善，并且彼此之间相互配合良好。

## 总结

+   你可以在不分享密码的情况下分享你的数据。

+   授权码流是目前最常用的 OAuth 授权类型。

+   授权码被交换为访问令牌。

+   通过限制访问令牌的时间和范围来降低风险。

+   范围由 OAuth 客户端请求，由授权服务器定义，并由资源服务器强制执行。
