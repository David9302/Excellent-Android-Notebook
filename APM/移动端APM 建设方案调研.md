# 移动端APM 建设方案调研

# 名词解释：

- TTI：用户实际可交互时长. （客户端：首屏完成加载，并可实际点击、滑动）.
- LCP：最大内容绘制完成时长.（客户端：往往是首屏全貌完成渲染时间节点.）
- FMP：核心组件绘制完成时长. （客户端：对应页面核心组件完成绘制时长）.
- 跳出率： 在对应页面、模块**未完成完全加载**之前跳出的次数 / 总进入次数. （为：**负向指标**）
- 秒开率：可以基于 FMP → LCP → TTI 三项指标（ps: 难度不一）来任意制定为分子， 其分母为点击对应跳转按钮为起始时间.
- Heart：为Google 2010年推出的用户体验模型框架，并将用户体验拆解出**五大方向**并加以**定义及测量方法.**
- GSM：是Google Heart 同期推出的目标制定方法论. 以**Goal** → **Signal** → **Metrics** 为导向. 用于校验目标达成效果及负反馈收集方案.
- 任务完成率，如：
- 在成交目标下，可以理解 U2O 转化率 .就是任务完成率
- 在Steam绑定目标下，可以理解 Steam 绑定成功率. 就是任务完成率

# 一、Why - 移动端用户体验模型

基于Google Heart 模型及 Google GSM 方法论。 可以清晰的理解到使用何种框架 及 方法来制定/测量一款拥有好的体验的应用软件.

目前国内除几家大型交互团队以外，更多的是去直接沿用 **Google Heart** 模型 结合 **Google GSM** 方法论 来制定用户体验提升目标，以下为为大致呈现：

| **维度** | **目标** | **信号/动作** | **指标** | **评估方法** |
| --- | --- | --- | --- | --- |
| 愉悦度 | 用户觉得对其有帮助的，使用产品感觉的愉悦的. | 1. 满意度是否有所下降？2. 产品评分5分居多？3. 愿意推荐给别人使用的. | 满意度、NPS、易用性、愿意转发分享 | NPS调研问卷，用户行为埋点 + 数据分析， APM系统. |
| 参与度（核心功能） | 用户对产品内容感兴趣并且愿意经常使用 | 1. 一段时间内多次访问产品或者使用某个功能 | 平均用户的每周的访问频率，7天内的活跃用户数量、热门期间重复访问量 | 用户行为埋点 + 数据分析 |
| 接受度（新功能） | 用户看到新产品或功能使用意愿度 | 1. 新产品功能访问量高2. 愿意注册3. 愿意使用新功能 | 页面浏览量，产品访问量，点击率、注册率、下载量、操作时长等等 | 用户行为埋点 + 数据分析 |
| 留存率 | 在一个时间内用户愿意回来继续使用产品 | 1. 活跃度高2. 流失率低3. 复购率高 | 次日留存率，7日留存率，30日留存率。复购率 | 用户行为埋点 + 数据分析 |
| 任务完成率 | 用户能够高效、准确的完成指定任务 | 1. 指定页面中、指定按钮点击（下一步/退出）2. 实际页面跳转3. 任务完成时间4. 操作过程出错事件 | 页面停留时长、转化率、可用率、错误率、页面跳出率、任务完成百分比 | 用户行为埋点 + 数据分析 + APM系统. |

以上表格中我们可以敲重点来看通用的部分：**评估方法.**

可以看到该模型下除了在**愉悦度**指标项需要用户主观的评估（NPS调研），其余五项完整的均需要用户 **"行为埋点 + 数据分析."**

而APM系统出现在了 **愉悦度 及 任务完成率**。它的职责更多的是去评估用户是否在操作过程中出现过异常不佳的场景，乃至出现严重的Block.

- 举例Case 见下图：

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741933871906-2c67d0a6-70c3-44a0-a02c-34d4d86b9dee.png)

# 二、What - 当前现状及可改进的部分.

- 当前我们已接入了 **友盟 + Arms**，原则上应该是可以满足应用**Google Heart** 模型建设。
- 但通过调研Arms后发现当前功能无法支持后续的**指标制定及数据测量，并对故障排查较为困难**.

### 原因如下：

| **维度** | **优势** | **劣势** | **其劣势带来的意义** |
| --- | --- | --- | --- |
| 会话跟踪 | 可观测单一用户、会话进行行为追溯（含行为、页面追溯、接口追溯） | 1. 用户行为漏报较多，并只有简单的click事件2. 没有聚合信息后的时间滑动窗口看板，可视化能力较差. | 直观的感知用户全生命周期的的行为路径及系统发生事件. 有待提升. |
| 页面访问 | 无 | 1. 该页面Native 只有加载耗时统计 | 从实际耗时来看 - 该口径为onResume 生命周期（Window 挂载时间节点）。与实际渲染无关. |
| 资源加载 | H5较为友好，对js、html、img、css等各项资源均进行网络监控. | Native 目前只有API 请求资源监听，对File相关资源无监听 | Native的图片资源、文件读写等监控缺失。其对内存、卡顿两部分缺乏对应模块的归因能力. |
| API请求 | 有完善的trace系统，可基于API 定位后端耗时分布. | trace ID 未与端进行绑定，客户端接口与接口之间交互无法通过trace进行关联. | trace 没有做到真正意义上的全链路追踪，当前仅限于服务侧. |
| 异常统计 | 优于Bugly的异常类型归类 | 1. 缺乏实际调用堆栈信息.2. ANR 信息严重缺失.3. 缺乏现场还原信息：内存、CPU、用户行为等数据 | 现场还原能力不足从而导致RD问题定位效率较低. |
| 卡顿统计 | 无 | 附带信息较少，只有基础的堆栈信息 | 排查难度较大，全是系统层面通用堆栈。参考价值较小 |
- **页面加载速度**
    
    ![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741934052891-46445983-3e30-4273-9249-731a6774c16f.png)
    

### 基于上Arms各维度分析发现：

**优势：** 用户体验的每个大模块均涉及，较为全面. 并在后端Traces追溯较为优秀.

**劣势：**

1. 指标模糊：每个模块下的均有大致指标，但这些指标无法建设正向的用户体验模型（对应 Heart）
2. 排查困难：当前异常情况下，现场还原信息较少.

漏报数据项，当前没有手段去科学的证实， 不作为当前分析项.

# 三、How - 怎么做？

基于上分析点，我可以从Arms缺失的部分并集合我们当前的问题进行联动改造。

APM建设为辅。从而真正做到可定义、可测量的路径。

### 1. 以下为 Android 启动时段的火焰图，其截止到首页渲染的第一帧： 7s

### FMP. 实际耗时： 7s（ps：本地分析 约有5 ~ 8%的性能损耗 ，并且包含600ms的 欢迎页）

**实际耗时预测公式：7000 * 0.92 - 600  = 5840ms （5.84s）**

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741935995334-69b5cd66-1eab-44da-b8b9-97a27125d52b.png)

### 2. 其中 Application.onCreate() 执行完成前：耗时：2.5s

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936126789-4bff43d2-4404-4306-82f0-dda0628005ab.png)

- **具体Code.**
    
    ```jsx
    fun initThirdSdk() {
        UmengInitializer.init(Utils.getApp())
        if (!ProcessUtil.isMainProcess()) {
            return
        }
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(
            DYJuLiangInitializer::class.java
        )
        UULogan.init()
        LoganBlackListHelper.init()
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(
                TencentX5Initializer::class.java
        )
        initPreloader()
        RouteUtil.get(IJSPluginService::class.java)
                ?.registerPluginFactory(GLOBAL_JS_PLUGIN_FACTORY, AppModuleJSPluginFactory())
        // 七鱼SDK，必须在App中初始化不能放在CustomerServiceInitializer中
        // 七鱼SDK，必须在App中初始化不能放在CustomerServiceInitializer中
        UUNeTalkHelperV2.getInstance().initUnicorn(Utils.getApp())
    //    initUnicorn(Utils.getApp())
    
        AppInitializer.getInstance(Utils.getApp())
                .initializeComponent(ABConfigExperimentInitializer::class.java)
    //        if (SharePHelper.getInstance().getAliyunReport() == 1)
    //            AppInitializer.getInstance(Utils.getApp()).initializeComponent(AliHaInitializer.class);
        //        if (SharePHelper.getInstance().getAliyunReport() == 1)
    //            AppInitializer.getInstance(Utils.getApp()).initializeComponent(AliHaInitializer.class);
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(ConfigInitializer::class.java)
        if (SharePHelper.getInstance().buglyOpenSwitch == 1) AppInitializer.getInstance(Utils.getApp())
                .initializeComponent(
                        BuglyInitializer::class.java
                )
    
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(
                TrackAndApmInitializer::class.java
        )
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(
                OneKeyLoginInitializer::class.java
        )
        UUPushManager.init(Utils.getApp())
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(
                VerifyFaceInitializer::class.java
        )
        AppInitializer.getInstance(Utils.getApp()).initializeComponent(
                VolcanoInitializer::class.java
        )
        if (DomainSwitch.enable()) {
            DomainHelper.init(Utils.getApp(), DomainSwitch.getHttpErrorThreshold(), false,if (!isPrivacyPolicyAgreed()) {
                UTDevice.getUtdid(Utils.getApp())
            } else {
                ""
            },UUABSwitchManager.getConfig("android_H5ErrorCodeSwitch_v1"))
            Log.d("DomainHelper", BuildConfig.FLAVOR + "")
        }
    
        UUPerformance.enableReport = UUABSwitchManager.hitABTestLogic("enable_apm_android")
    
        initArms(application = CONTEXT.application)
        NormalJsPreloaderHelper.run {
            init(
            blackList = mutableListOf(LeaseTransferShelfH5OrderFragment::class.java,LeaseTransferShelfH5Fragment::class.java, SalePutShelfH5Fragment::class.java, ShelfH5Fragment::class.java),
            pluginList = listOf(NetUtilsPlugin())
        )
        }
    }
    ```
    

**主要耗时分布为：**

| 类执行： | 耗时 | 备注 |
| --- | --- | --- |
| DetailWebViewPreloadHelper | 976 ms | 初始化 WebView Pool |
| SharePreInitializer | 244 ms | SharedPreferences 初始化 |
| ArmsInit | 239 ms | Arms 初始化 |
| NormalJsPreloaderHelper | 191 ms | 解析url WebView 预加载页面 |
| **总结：** | 1650 ms |  |

### 3. 首页启动 + 渲染第一帧耗时 3.68s

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936173311-69375850-fa86-42be-8724-be6251d3770e.png)

- **具体Code**
    
    ```java
    fun gainConfig(block: ((Boolean) -> Unit)? = null) {
    launchs(block = {
            val model = api.gainUserConfig()
    model.data?.let {
        if (it.contains("730")) {
            it.getJSONObject("730")?.let { csgo ->
                                           RentSettingViewHelper.syncConfig(csgo)
                                           //求购自动接收报价服务开关
                                           if (csgo.contains("purchaseAutoReceiveSwitch")) Ukv.putBoolean(
                KEY_ASK_BUY_AUTO_RECEIVE,
                csgo.getInteger("purchaseAutoReceiveSwitch") == 1
            )
    
                if (csgo.contains("inventoryViewSwitch")) {
                SharePHelper.getInstance()
                .saveViewMapStatus(csgo.getInteger("inventoryViewSwitch") == 1)
            }
                                           if (csgo.contains("inventoryThinModeSwitch")) {
                                               SharePHelper.getInstance()
                                               .saveSimpleType(csgo.getInteger("inventoryThinModeSwitch") == 1)
                                           }
                                           if (csgo.contains("inventoryTemplateMergeSwitch")) {
                                               SharePHelper.getInstance()
                                               .saveStockMergeStatus(csgo.getInteger("inventoryTemplateMergeSwitch") == 1)
                                           }
                                           if (csgo.contains("autoRecordPurchasePriceSwitch")) {
                                               SharePHelper.getInstance()
                                               .setPreference_remarkPrice(csgo.getInteger("autoRecordPurchasePriceSwitch") == 1)
                                           }
                                           if (csgo.contains("transactionMode")) {
                                               SharePHelper.getInstance()
                                               .setDefaultTransactionMode(csgo.getInteger("transactionMode"))
                                           }
                                           if (csgo.contains("preSaleCoolEndSellSwitch")) {
                                               SharePHelper.getInstance()
                                               .setDefaultPresaleCoolingMode(csgo.getInteger("preSaleCoolEndSellSwitch"))
                                           }
                                           if (csgo.contains("defaultRentalActivitySwitch")) {
                                               SharePHelper.getInstance()
                                               .setDefaultJoinRentActivity(csgo.getInteger("defaultRentalActivitySwitch") == 1)
                                           }
                                           if (csgo.contains("leaseDays")) {
                                               SharePHelper.getInstance()
                                               .synLeaseDays(csgo.getJSONArray("leaseDays") as MutableList<Int>?)
                                           }
                                           if (csgo.contains("defaultLeaseDays")) {
                                               SharePHelper.getInstance().synDefautLeaseDays(
                                                   csgo.getJSONArray("defaultLeaseDays").toList() as MutableList<Int>?
                                               )
                                           }
                                           if (csgo.contains("leaseListSort")) {
                                SharePHelper.getInstance().putString(
                                    SharePHelper.KEY.K_SORT_PREFERENCE_SELECT,
                                    csgo.getString("leaseListSort")
                                        ?: ""
                                )
                            }
                            if (csgo.contains("easyCompensationOrdinarySwitch")) {
                                // 默认选中赔付标准的开关
                                csgo["easyCompensationOrdinarySwitch"]?.let { value ->
                                    LocalDataHelper.setEasyCompensationOrdinarySwitchStatus(value == "1".toDouble())
                                }
                            }
                            if (csgo.contains("compensateSwitch")) {
                                LocalDataHelper.synCompensateSwitch(csgo.getInteger("compensateSwitch"))
                            }
                            if (csgo.contains("depositCompensateType")) {
                                LocalDataHelper.synCompensateType(csgo.getInteger("depositCompensateType"))
                            }
                            if (csgo.contains("autoFillDepositSwitch")) {
                                val enabled = csgo.getInteger("autoFillDepositSwitch")
                                SharePHelper.getInstance().isAutoFillDeposit = enabled == true.toInt()
                            }
                            if (csgo.contains("depositCoefficient")) {
                                csgo["depositCoefficient"]?.let { value ->
                                    SharePHelper.getInstance()
                                        .setAutoFillDepositCoefficient(value.toString().toFloat())
                                }
                            }
                            if (csgo.contains("fillInRentSwitch")) {
                                val enabled = csgo.getInteger("fillInRentSwitch")
                                Ukv.putBoolean(KEY_AUTO_FILL_LONG_RENT, enabled == true.toInt())
                            }
                            if (csgo.contains("longRentCoefficient")) {
                                csgo["longRentCoefficient"]?.let { value ->
                                    Ukv.putFloat(
                                        KEY_AUTO_FILL_LONG_RENT_COEFFICIENT,
                                        value.toString().toFloat()
                                    )
                                }
                            }
                            SaleCommodityMergeHelper.saveSaleListMergeMode(csgo)
                            NoCDHelper.onGetCurrent(csgo)
                        }
                    }
                    ACache.updateConfig(BaseConfig.USER_CUSTOM_CONFIG, it.toJson())
                }
                block?.invoke(false)
            }, onError = {
                block?.invoke(false)
                UUToastUtils.showShort(it.message)
            })
        }
    ```
    
    ```java
    private fun initFragments(savedInstanceState: Bundle?, mainActivity: MainActivity) {
            mFragments = arrayOfNulls(MAX)
            if (savedInstanceState == null) {
                mFragments!![SECOND] = HomePageV2Fragment.newInstance(HOMETAG)
                mFragments!![THIRD] = StockV2Fragment.newInstance(STOCKTAG)
                mFragments!![FOURTH] = SellV2Fragment.newInstance(SELLTAG)
                mFragments!![FIFTH] = newInstance(ASKTAG)
                mFragments!![SIX] = getUserFragment(mainActivity, false)
                loadMultipleRootFragment(
                    R.id.c_content, GameHelper.currentIndex,
                    mFragments!![SECOND],
                    mFragments!![THIRD],
                    mFragments!![FOURTH],
                    mFragments!![FIFTH],
                    mFragments!![SIX]
                )
            } else {
                mFragments!![SECOND] = findChildFragment(HomePageV2Fragment::class.java)
                mFragments!![THIRD] = findChildFragment(StockV2Fragment::class.java)
                mFragments!![FOURTH] = findChildFragment(SellV2Fragment::class.java)
                mFragments!![FIFTH] = findChildFragment(AskFragment::class.java)
                mFragments!![SIX] = findChildFragment(UserFragmentV3::class.java)
            }
        }
    ```
    

| 类执行 | 耗时 | 备注 |
| --- | --- | --- |
| MainViewModel | 132 ms | 初始化MainViewModel，主要为加载配置项信息. |
| MainActivity.setContent() + Fragment View创建 | 1300 ms | 创建 首页静态布局、View |
| MainActivity.onResume() | 326 ms | 挂载 Window |
| Choreographer.doFrame() | 1100 ms | 将创建好的View 绘制 至 Windows |
| **总结：** | 2858 ms |  |

### 4. 结论：

- 以上为 启动项的静态分析，可以看到拆出来部分耗时为 **4.3s** 左右. 该部分后续可能持续进行优化.
- 目前为静态分析并且机型为：一加 Ace 2V 其 12G运存. 并不为低端机.
- APM接入后，更能反应出线上的实际情况.

💡关于APM建设，也可以考虑到行为埋点的联动.

# 四、Roadmap - 该如何制定

关于APM 建设，其中分为四个部分：**Client采样系统 + Logs日志系统 + Traces 系统（链路追踪） + Metrics（报表统计）**

服务端：[https://github.com/apache/skywalking](https://github.com/apache/skywalking) **24.2K Star**

# 4.1 Logs 日志系统：

基于历史经验及行业较为通用的解决方案：ELF (Elastic + Logstash + Kibana）

其中技术公开性较好，其主要成本在于 ES 服务、存储成本 + Logstash 实例成本.

- ES 服务
    
    ![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936275100-042f5303-9dcd-4654-9506-eb8caba24bc8.png)
    
- ES 服务存储：
    
    ![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936291991-2c1c949a-e1c6-4b7f-9684-fb506ea1c59e.png)
    
- Logstash 实例 + 存储：
    
    ![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936320761-be5f5791-f7f9-454c-8c4d-472f2640ad90.png)
    

# 4.2 Trace 系统:

历史调研，使用Twitter 开源服务：Zipkin (

[https://zipkin.io/](https://zipkin.io/)

)

**17.1k Star**

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936362539-fd2aef12-f8ac-4efb-9ffa-354f01e1f112.png)

# 4.3 Metrics 系统：

点评：Cat 系统. （ [https://github.com/dianping/cat）](https://github.com/dianping/cat%EF%BC%89) 18.8K Star

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936543492-bdb04510-fc53-4ee9-9843-ef5f39d8dd90.png)

# 4.4 客户端：Andrroid / iOS 采样

### 性能监控：Matrix : [https://github.com/Tencent/matrix](https://github.com/Tencent/matrix) **11.8k Star**

- 其中涵盖 Android/iOS 性能监控模块。 但通过排查Android 的基本源码，会发现大多数更新为3年前。 会存在一定的兼容性问题（但可以解决）.

### 行为埋点：

- Android 已有相对成熟方案. 基于Gradle Plugin + ASM 达成 AOP切面. 并支持手动埋点 填充业务数据.
- 并且 神策团队 对外开源 ：[https://github.com/sensorsdata](https://github.com/sensorsdata) 目前看最新更新时间为2周前，可支持Web、小程序、iOS、Android等各端行为无埋日志采样 SDK. （ps: Android Gradle Plugin 较老）

# 4.5 Web / H5 数据采样 **5.1K Star**

webfunny_monitor : [https://github.com/a597873885/webfunny_monitor](https://github.com/a597873885/webfunny_monitor)

最后更新于2年前. 其中涵盖埋点 + 性能监控模块.

# 4.6 统一解决方案 （开源，但由于涉及面较多。未完成相对全面的调研.）

目前通过调研发现国外一家名为：Sentry的企业 - 专注于APM 并对外宣传开源所有代码：包含 前端 + 客户端 + 后端数据采样 及 Traces + Metrics 、可视化页面等多项领域一并开源。 提供了相对全面的解决方案.

[https://github.com/getsentry/sentry](https://github.com/getsentry/sentry) （**40.3K Star**）为目前APM类别开源项目中 Star 最高.

其中开源的可视化界面：

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936569849-9648c67d-7e4d-4069-8b78-ed65e72068b6.png)

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936593304-c3bb7200-d9f7-444e-a96a-8bf0ea8d9bd2.png)

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936654614-118e21cc-5bd6-46c5-877b-a14112358cc6.png)

![](https://cdn.nlark.com/yuque/0/2025/png/1401610/1741936678153-f14cdc50-05c1-4f85-aeda-f6b5dd75b9c3.png)

💡建议可参考该开源项目，进行定制化迭代开发. 打造企业专注APM.

# 参考文献：

- Google Heart：[https://www.interaction-design.org/literature/article/google-s-heart-framework-for-measuring-ux](https://www.interaction-design.org/literature/article/google-s-heart-framework-for-measuring-ux)
- Application Performance Monitor：
- [https://github.com/a597873885/webfunny_monitor](https://github.com/a597873885/webfunny_monitor)
- [https://github.com/BiBoyang/GoldHouse-for-iOS](https://github.com/BiBoyang/GoldHouse-for-iOS)
- [https://github.com/apache/skywalking](https://github.com/apache/skywalking)
- [https://github.com/openzipkin/zipkin](https://github.com/openzipkin/zipkin)
- [https://github.com/dianping/cat](https://github.com/dianping/cat)
- Sentry Github Repositories：[https://github.com/getsentry/sentry](https://github.com/getsentry/sentry)
- 行为埋点：
- 神策 Gihub Repositories ： [https://github.com/sensorsdata](https://github.com/sensorsdata)