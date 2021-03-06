---
description: 结合管理和技术手段杜绝GitHub敏感信息泄露问题，做到近实时监控预警。分享开发GitHub敏感信息监控过程中的特征采集、最佳告警响应方式、误报规则类型、漏报处理思路以及整体GitHub敏感信息泄露的现状等等一些经验。
---

# GitHub敏感信息泄露监控
Feei <feei#feei> 01/2018

## 0x01 漏洞背景
GitHub作为开源代码平台，给程序员提供了交流学习的地方，提供了很多便利，但如果使用不当，比如将包含了账号密码、密钥等配置文件的代码上传了，导致攻击者能发现并进一步利用这些泄露的信息，就是一个典型的GitHub敏感信息泄露漏洞。

## 0x02 解决思路
### 制度为辅
新入职员工往往这方面的意识较为薄弱，通过对新入职员工的专项安全培训，着重普及下在工作和生活中因为意识不足所导致的一些安全问题及应对方法。让新人对此有个基础的意识，知道上传GitHub代码会有风险，并将其加入到内部的信息安全守则或红线中，有据可依也有据可罚，对恶意上传者起到一定的威慑，对于老员工也有一定的意识提升。进一步可以细化GitHub开源规范，让开源流程化，也让安全团队介入其中进行评估。

### 技术为主
#### 内外网隔离
从代码泄露最终的结果上看，直接影响的是外网可访问的接口被利用，比如邮箱账号密码泄露的话会被攻击者所登陆并拿到邮件信息、组织架构及花名册等；数据库使用外网IP会导致数据被盗取等。
所以可以先从内外网隔离来做，规范代码中统一使用内部接口、内部IP，迫不得已的外部接口可以通过增加二次验证、限制访问源IP等策略进行加固。
这样这样泄露出去的代码无法对企业造成直接影响，至于代码本身存在的一些漏洞就需要依靠白盒代码审计日积月累了。

#### 实时监控
经过内外网隔离后，代码虽然无法造成直接的危害，但毕竟是公司资产，涉及公司声誉，还是需要进一步解决。所以**实时**的监控很有必要，我们如果能在第一时间内发现泄露的信息并找到对应人及时删除，造成的影响将会被降到最低。需要再次强调的是，**实时**是整个技术最核心指标。

### 技术方案
监控核心思路可以通过GitHub搜索代码中的特征来发现泄露的信息。

同类型项目Hawkeye使用的是代理爬取的方式进行定期搜索，但代理爬取的方式有诸多弊端，最大缺点是爬取的时候GitHub是一个黑盒，你不知道他是什么频率限制规则，也不知道他什么时候改变规则，一切都存在不确定性。

虽然GitHub API也有频率限制，但他的文档中明确了频率限制的策略，我们可以通过做好频率控制避免触发GitHub API限制规则。

对于绝大多数的企业，做好以下两条就不会被触发GitHub频率限制规则。目前已经试验过的2000条扫描规则15个token在多种时间频率下未发现问题。

- **将扫描规则按优先级拆分成不同类型并以不同的速度去运行**
- **注册多个账号申请多个token**

## 0x03 最佳实践
### 收集特征
那如何找到我们的特征呢，GitHub作为一个开源**代码**托管平台，代码首当其中，但也会有人将GitHub仅当做版本控制系统来管理自己的文档、资料。这类资料都有一个共同特征，里面包含企业的专属特征：代码中接口的调用包含内部API地址、代码头部包含企业的Copyright信息、代码包名包含企业的主域名等。

#### 常见几类特征
- 内部域名，内网接口使用的域名、测试域名等，比如`mogujie.org`
- 外部域名，对外使用的域名，可以选一些风险较高的，比如`mail.mogujie.com`
- 代码版权，代码头部注释中的版权声明，比如`Copyright 2018 Meili-Inc. All Rights Reserved`
- 主机域名，服务器HOST域名，比如`goods.db.mogujie.host`
- 代码包名，Java代码包名，比如`com.mogujie.goods`

#### 如何找到其它企业内部特征？
可以从域名特征入手，最简单的通过一些特定关键词:

- `corp`
- `dev`
- `inc`
- `pre`
- `test`
- `copyright`

搜索GitHub代码，根据代码人工确认是否是误报。找到某个确定是该企业的工程后，分析改工程其它类型的特征即可，对于下属企业可以通过域名注册人反查并循环前面的逻辑。
比如要找蘑菇街的内部特征，可以通过在GitHub中搜索`mogujie dev`即可找到属于蘑菇街的工程，继而我们再从这类工程中找到真正的内部域名和其它类型特征。

**实践例子**
> 声明：通过本篇内容任何人都可以轻易找到任何公司的内部特征，例子只为提升理解，若觉得侵权请联系我删除。
> 仅列取部分典型厂商的典型内部特征作为讲解。


- 百度（BAIDU.COM）：公网域名和内网域名混用
  - `yx01.baidu.com`
  - `vm.baidu.com`
  - `epc.baidu.com`
  - `iwm.name`
  - `xdb.all.serv`
- 阿里巴巴（ALIBABA.COM）：INC和.NET是最喜欢用作内网域名
  - `alibaba-inc.com`
  - `aliyun-inc.com`
  - `alibaba-inc.vipserver`
  - `alibaba.net`
  - `alipay.net`
  - `fliggy.net`
  - `taobao.net`
- 腾讯(TENCENT.COM)：用了一个不是自己的域名
  - `tapd.oa.com`
  - `proxy.tencent.com`
  - `proxy.oa.com`
  - `npm.oa.com`
  - `tencentdb.com`
  - `tencent.com`
- 京东（JD.COM）：特征独一无二，比较准
  - `jd.local`
- 360（360.CN）：特征独一无二，比较准
  - `qihoo.net`
- Bilibili（BILIBILI.COM）：CO域名和COM域名在搜索分词容易出问题，需要注意
  - `bilibili.co`
- 饿了么(ELE.ME)：特征独一无二，比较准
  - `elenet.me`
- 陌陌（IMMOMO.COM）：特征独一无二，比较准
  - `wemomo.com`
- 搜狐（sohu.com）：喜欢INC
  - `sohu-inc.com`
  - `sohuno.com`
  - `cyou-inc.com`
- 途牛（TUNIU.COM）：喜欢ORG
  - `tuniu.org`
  - `tuniu-cie.org`
  - `tunie-sit.org`
- 去哪儿（QUNAR.COM）：和百度类似，但用二级域来区分，比百度清楚
  - `dev.qunar.com`
  - `package.qunar.com`
  - `test.qunar.com`
  - `beta.qunar.com`
  - `local.qunar.com`
  - `corp.qunar.com`
- 携程（CTRIP.COM）：：特征独一无二，比较准
  - `ctripqa.com`
  - `ctripcorp.com`
- 爱奇艺（IQIYI.COM）：爱奇艺是区分的最好的，数据库、应用间调用、人访问的都区分开了
  - `qiyi.domain`
  - `qiyi.virtual`
  - `qiyi.db`

#### 了解搜索特性有利于特征收集
- 像其它搜索引擎一样，GitHub搜索会对内容进行分词，所以搜索的时候应该以单词为单位，如果有词组可以拆开用空格拼接起来搜索。
- 如果遇到一些比较容易出现误报的词，可以使用双引号括起来就行强制搜索，比如`"mi.com"`。
- 特殊字符不会被索引到。
- 使用时间倒序的方式，先处理最近上传的。

### 告警与响应
在告警这块早期为了告警速度与响应速度，选择了邮件的方式，在后续的实践过程中我们也尝试想改成管理后台、Slack等形式，但综合评估后觉得邮件是最佳实践。

#### 漏洞告警实践
- 调快邮件客户端收信的轮询时间，第一时间获取漏洞信息
- 将所有客户端调整为imap模式，这样能保证我们在电脑上操作的漏洞结果在手机上状态能同步过来
- 邮件发送频率限制可以通过配置多个邮箱甚至是多个不同运营商的邮箱解决

#### 漏洞响应实践
使用邮件来管理这类漏洞，证实存在或有研究价值的漏洞可以打上不同颜色星标，继而提交给对应厂商或发送给应急响应小组。而证实误报的可以删除到垃圾桶，定期通过优化代码和规则解决；

### 误报与漏报

#### 误报
> 我们目前积累的常见的误报规则

- GitHub博客，比如`github.io github.com`
- Android项目（甲方自家监测建议去掉此条规则），比如`app/src/main`
- 爬虫，比如`crawler spider scrapy 爬虫`
- 插件，比如`surge adblock .pac`
- Link，比如`"<a href <iframe src ]("`
- 其它，比如`linux_command_set domains jquery sdk linux contact readme hosts authors .html .apk`

#### 漏报
我们可以不断补充规则来覆盖尽可能多的特征场景，但还是避免不了漏报，毕竟不是所有的项目都遵守或符合我们的特征。如果企业安全团队推动力比较强，可以考虑给所有项目都加上一个文件，内容可以是一个索引标记，之后就根据这些索引标记输出为监控规则来实现无漏报。

## 0x04 漏洞处理
通过[GSIL](https://github.com/FeeiCN/GSIL)发现泄露的信息后，可以通过以下方式处理。

### 找得到作者
先通过以下方法找到作者

- 查看Commits记录，看其中作者名字和邮箱（往往大家在Git Config的时候会全局配置自己的提交用户名和邮箱）
- 查看作者GitHub个人主页，根据Nickname/Username/Email等关联出上传人的真实身份
- 查看项目类型，比如是前端项目可以找到前端负责人，让其在团队内部询问下
- 查看项目代码中的蛛丝马迹，比如注释中会有编写人姓名和邮箱、通过配置信息关联出申请人

找到作者后，联系其先不要删除，保留[项目Traffic页面](https://github.com/FeeiCN/GSIL/graphs/traffic)的截图以作为风险判断依据，再删除项目。

最后根据项目泄露相关信息进行整改，比如泄露了项目配置信息（账号密码、密钥等）则进行更改。

### 找不到作者

找不到作者先整改泄露的信息，优先级由外而内，把危害降到最低。
再让公司法务或信息安全团队通过[GitHub DCMA（数字千年版权法案处理规则）](https://help.github.com/articles/guide-to-submitting-a-dmca-takedown-notice/)向GitHub发送邮件进行处理，一般三天左右会有答复。

几个注意的点

- 邮件格式和注意事项在[GitHub DCMA处理规则](https://help.github.com/articles/guide-to-submitting-a-dmca-takedown-notice/)中有详细说明，请严格按照说明编写
- 根据泄露的信息提供相对应的证据，比如泄露域名相关信息可提供域名证书等能证明所有权相关证据
- 希望GitHub官网采取的操作，一般是删除对应GitHub仓库
- 提供公司的联系方式
- 签名等

## 0x05 漏洞现状
为了优化速度及误报问题，我们将国内的漏洞盒子、补天及乌云历史上的厂商都加入到监控列表中进行测试，我们仅拿着这一类漏洞刷某漏洞平台全部厂商进了排名榜前十，漏洞报告太多导致触发邮箱频率限制规则被ban了，由此可见这类问题在国内的多么普遍。

有SRC（漏洞应急响应中心）的厂商我们都会尽量将漏洞反馈过去，**每月单漏洞收入超过¥10k+**，但大多数厂商是没有对外接收漏洞的渠道，为避免这类漏洞被人恶意利用，我们开源了整个项目，所有厂商可以免费使用，移步[https://github.com/FeeiCN/GSIL](https://github.com/FeeiCN/GSIL)，据了解国内已有数十家大型互联网公司、证券公司、银行等在使用[GSIL](https://github.com/FeeiCN/GSIL)，包括蘑菇街、美丽说、小店、大搜车、农信银行、泰康保险、传化集团、投哪儿网、广发证券等。

除了针对各个企业的特征外，我们还发现了一些通用特征，这些特征往往造成的影响更大，比如针对**企业邮箱**、**私密文档**、**公私密钥**、**通用连接组件**的规则所捕获的信息也经常会涉及到各家企业，比如之前某家云厂商员工将某个系统上传到GitHub，其中包含了可通过公开API调用的公私密钥，可控制其数万台服务器全部权限。

而且越来越多的线索表明这类漏洞越来越多的被恶意利用，比如之前曾发现黑灰产通过大批量监控收集所有泄露的**AppId**和**AppSecret**来控制微信公众号进一步达到盈利目的、通过泄露的**SSH/Redis账号密码**获取主机权限并挖矿等。

只有将问题充分暴露出来，大家才会正视并解决问题。

#### 引用参考
- [GSIL](https://github.com/FeeiCN/GSIL)
- [安全+沙龙第四期-杭州站](https://mp.weixin.qq.com/s/pSZFLHPc4MczHsDiPjS7WQ)
