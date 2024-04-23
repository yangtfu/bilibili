<div align="center">

# 哔哩哔哩-API-Go版本

![](https://img.shields.io/github/go-mod/go-version/CuteReimu/bilibili "语言")
[![](https://img.shields.io/github/actions/workflow/status/CuteReimu/bilibili/golangci-lint.yml?branch=master)](https://github.com/CuteReimu/bilibili/actions/workflows/golangci-lint.yml "代码分析")
[![](https://img.shields.io/github/contributors/CuteReimu/bilibili)](https://github.com/CuteReimu/bilibili/graphs/contributors "贡献者")
[![](https://img.shields.io/github/license/CuteReimu/bilibili)](https://github.com/CuteReimu/bilibili/blob/master/LICENSE "许可协议")
</div>

本项目是基于Go语言编写的哔哩哔哩API调用。目前常用的接口已经基本完成。

**本项目不会编写单元测试代码**。一则因为各项数据会频繁变动，难以写成固定的结果；二则因为每次单元测试都要大量请求B站API，会对其产生不必要的压力。
如果你发现有**接口bug**或者**有你需要但是本库尚未实现的接口**，可以[提交issue](https://github.com/CuteReimu/bilibili/issues/new/choose)或者[提交pull request](#如何为仓库做贡献)。
如果因为B站修改了接口导致接口突然不可用，不一定能够及时更新，很大程度上需要依赖各位的告知。

现在是v2版本，v2版本使用了泛型，因此确认不支持Go1.17及之前的版本。[如果还想使用v1版本可以点击这里跳转](https://github.com/CuteReimu/bilibili/tree/v1)。

**声明**：

1. 本项目遵守 AGPL 开源协议。
2. 本项目基于 [SocialSisterYi/bilibili-API-collect](https://github.com/SocialSisterYi/bilibili-API-collect)
   中描述的接口编写。请尊重该项目作者的努力，遵循该项目的开源要求，禁止一切商业使用。
3. **请勿滥用，本项目仅用于学习和测试！利用本项目提供的接口、文档等造成不良影响及后果与本人无关。**
4. 由于本项目的特殊性，可能随时停止开发或删档
5. 本项目为开源项目，不接受任何形式的催单和索取行为，更不容许存在付费内容

PS：目前，B站调用接口时强制使用 `https` 协议

## 快速开始
### 安装

```bash
go get -u github.com/CuteReimu/bilibili/v2
```

在项目中引用即可使用

```go
import "github.com/CuteReimu/bilibili/v2"

var client = bilibili.New()
```

### 首次登录

**下文为了篇幅更短，示例中把很多显而易见的`err`校验忽略成了`_`，实际使用请自行校验`err`。**

#### 方法一：扫码登录

首先获取二维码：

```go
qrCode, _ := client.GetQRCode()
buf, _ := qrCode.Encode()
img, _ := png.Decode(buf) // 或者写入文件 os.WriteFile("qrcode.png", buf, 0644)
// 也可以调用 qrCode.Print() 将二维码打印在控制台
```

扫码并确认成功后，发送登录请求：

```go
result, err := client.LoginWithQRCode(bilibili.LoginWithQRCodeParam{
    QrcodeKey: qrCode.QrcodeKey,
})
if err == nil && result.Code == 0 {
    log.Println("登录成功")
}
```

#### 方法二：账号密码登录

首先获取人机验证参数：

```go
captchaResult, _ := client.Captcha()
```

将`captchaResult`中的`gt`和`challenge`值保存下来，自行使用 [手动验证器](https://kuresaru.github.io/geetest-validator/) 进行人机验证，并获得`validate`和`seccode`。然后使用账号密码进行登录即可：

```go
result, err := client.LoginWithPassword(bilibili.LoginWithPasswordParam{
    Username:  userName,
    Password:  password,
    Token:     captchaResult.Token,
    Challenge: captchaResult.Geetest.Challenge,
    Validate:  validate,
    Seccode:   seccode,
})
if err == nil && result.Status == 0 {
    log.Println("登录成功")
}
```

#### 方法三：使用短信验证码登录（不推荐）

首先用上述方法二相同的方式获取人机验证参数并进行人机验证。然后获取国际地区代码：

```go
countryCrownResult, _ := client.GetCountryCrown()
```

当然，如果你已经确定`cid`的值，这一步可以跳过。中国大陆的`cid`就是`86`。

然后发送短信验证码：*（[这个接口大概率返回86103错误](https://github.com/SocialSisterYi/bilibili-API-collect/issues/756)）*

```go
sendSMSResult, _ := client.SendSMS(bilibili.SendSMSParam{
    Cid:       cid,
    Tel:       tel,
    Source:    "main_web",
    Token:     captchaResult.Token,
    Challenge: captchaResult.Geetest.Challenge,
    Validate:  validate,
    Seccode:   seccode,
})
```

然后就可以使用手机验证码登录了：

```go
result, err := client.LoginWithSMS(bilibili.LoginWithSMSParam{
    Cid:        cid,
    Tel:        tel,
    Code:       123456, // 短信验证码
    Source:     "main_web",
    CaptchaKey: sendSMSResult.CaptchaKey,
})
if err == nil && result.Status == 0 {
    log.Println("登录成功")
}
```

### 储存Cookies

使用上述任意方式登录成功后，Cookies值就已经设置好了。你可以保存Cookies值方便下次启动程序时不需要重新登录。

```go
// 获取cookiesString，自行存储，方便下次启动程序时不需要重新登录
cookiesString := client.GetCookiesString()

// 设置cookiesString，就不需要登录操作了
client.SetCookiesString(cookiesString)
// 你也可以直接把浏览器的Cookie复制过来调用SetCookiesString，这样也可以不需要登录操作了
```

### 其它接口

你可以很方便的调用其它接口，以下举个例子：

```go
videoInfo, err := client.GetVideoInfo(bilibili.VideoParam{
    Aid: 12345678,
})
```

参数中非必填字段你可以不填（可以通过是否有`omitempty`来判断这个字段是否为非必填字段）。

方法都是按照对应功能的英文翻译命名的，因此你可以方便地使用IDE找到想要的方法，配合注释便能够知道如何使用。

### 可能用到的工具接口

```go
// 解析短连接
typ, id, err := client.UnwrapShortUrl("https://b23.tv/xxxxxx")

// 获取服务器当前时间
now, err := client.Now()

// av号转bv号
bvid := bilibili.AvToBv(111298867365120)

// bv号转av号
aid := bilibili.BvToAv("BV1L9Uoa9EUx")

// 通过ip确定地理位置
zoneLocation, err := client.GetZoneLocation()

// 获取分区当日投稿稿件数
regionDailyCount, err := client.GetRegionDailyCount()
```

### 设置*resty.Client的一些参数

调用`client.Resty()`就可以获取到`*resty.Client`，然后自行操作即可。**但是不要做一些离谱的操作**~~（比如把Cookies删了）~~

```go
client.Resty().SetTimeout(20 * time.Second) // 设置超时时间
client.Resty().SetLogger(logger) // 自定义logger
```

## Star History

<a href="https://star-history.com/#CuteReimu/bilibili&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=CuteReimu/bilibili&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=CuteReimu/bilibili&type=Date" />
   <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=CuteReimu/bilibili&type=Date" />
 </picture>
</a>

## 如何为仓库做贡献？

不知道在哪些方面可以做贡献？[点击这里看看吧！](https://github.com/CuteReimu/bilibili/contribute)

命名规范请参考[CONTRIBUTING.md](.github/CONTRIBUTING.md)
