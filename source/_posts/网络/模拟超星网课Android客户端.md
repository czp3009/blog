---
layout: post
title: 模拟超星网课Android客户端
date: 2018-09-21 16:44:10
category: 网络
tag: [超星, android]
---

本文撰写时的 `学习通` 版本 4.0.1

这个 `超星` 有好几个名字, `慕课`, `尔雅通识课`, `泛雅`, Android APP 叫 `学习通`.

虽然这玩意有那么多名字, 但是有一件事情是亘古不变的, 那就是国内大量学校使用这个网课平台来填充学生的课程表. 而这家网课平台的课程质量普遍不佳, 分辨率低, 教师口音重, 教授内容废话连篇, 所以我们就想到了, 能不能不看这些网课呢.

这家网课平台的 Web 端的视频播放器是一个 Flash, 监听用户鼠标事件, 一旦用户鼠标移出 Flash 区域或者将浏览器最小化, 都将暂停视频直到鼠标重新放上. 而 Android 客户端就更是精彩绝伦, 视频播放的那个 Activity 直接是一个 WebView, 同样是通过 js 来监听切换到后台的事件, 令人啧啧称奇.

我们今天就来探究一下这个 `超星` 的 Android APP 是怎么运作的.

## 正常登陆
首先我们得能登陆. 我们下载一个 `学习通` 看一下正常使用是怎么登陆的.

感兴趣的同学可以去这里下载一个 https://app.chaoxing.com

其实这个 APP 在我还在使用 MI3 时是用过的, 唯一的体会就是相当卡, 卡到鸡巴旋转脱落. 甚至是 ListView 的滚动都能掉帧的那种卡顿, 恐怕放眼全球也只有 12306 的 APP 能与之相抗衡.

APP 一打开是这样的

{% asset_img Screenshot_2018-09-21-17-11-08-105_com.chaoxing.mobile.png index %}

看到下面这条非常类似 IONIC 的抽屉, 起初以为是 H5 APP, 但是看了一下各种动画效果发觉并不是.

点击下方抽屉里的 `我的` 按钮

{% asset_img Screenshot_2018-09-21-17-11-15-200_com.chaoxing.mobile.png index %}

然后点击 `请先登陆`

{% asset_img Screenshot_2018-09-21-17-11-28-696_com.chaoxing.mobile.png index %}

如果我们不输入用户名直接点 `登陆` 甚至还会有一个 Toast 来提醒用户必须输入用户名. 这个 APP 竟然有本地表单验证实在是难以置信.

## 截包
我们马上来截包一下, 当我们登陆的时候, APP 会发出这样的一个数据包

    POST /v11/loginregister?token=4faa8662c59590c6f43ae9fe5b002b42&_time=1537463981249&inf_enc=7e0a0c15d58556a991eb94011ac16cc4 HTTP/1.1
    Accept-Language: zh_CN
    Cookie: 
    Content-Length: 720
    Content-Type: multipart/form-data; boundary=P3LxQlvmN5TjoopArsDJl0scmxQ3pjpqvQ
    Host: passport2-api.chaoxing.com
    Connection: Keep-Alive
    User-Agent: Dalvik/2.1.0 (Linux; U; Android 8.0.0; MI 6 MIUI/8.9.13) com.chaoxing.mobile/ChaoXingStudy_3_4.0.1_android_phone_287_2 (@Kalimdor)_-30877643
    
    --P3LxQlvmN5TjoopArsDJl0scmxQ3pjpqvQ
    Content-Disposition: form-data; name="uname"
    Content-Type: text/plain; charset=UTF-8
    Content-Transfer-Encoding: 8bit
    
    13902095201
    --P3LxQlvmN5TjoopArsDJl0scmxQ3pjpqvQ
    Content-Disposition: form-data; name="code"
    Content-Type: text/plain; charset=UTF-8
    Content-Transfer-Encoding: 8bit
    
    19951118
    --P3LxQlvmN5TjoopArsDJl0scmxQ3pjpqvQ
    Content-Disposition: form-data; name="loginType"
    Content-Type: text/plain; charset=UTF-8
    Content-Transfer-Encoding: 8bit
    
    1
    --P3LxQlvmN5TjoopArsDJl0scmxQ3pjpqvQ
    Content-Disposition: form-data; name="roleSelect"
    Content-Type: text/plain; charset=UTF-8
    Content-Transfer-Encoding: 8bit
    
    true
    --P3LxQlvmN5TjoopArsDJl0scmxQ3pjpqvQ--

Params 有三个, `token`, `_time`, `inf_enc`.

`_time` 我们一眼就可以确定是时间戳.

通过多次登录, 我们发现 `token` 每次都是一模一样的, 说明这是类似 `appKey` 一样的固定值.

那么剩下就是 `inf_enc` 的问题. 这个参数每次都是不一样的, 所以应该是某种签名算法.

根据他的长度和只有数字和小写字母推测可能是 `MD5` 加密得到的密文, 通过现有的一些反查网站查询无果. 试验了几个可能的明文排列方案也与最终的结果不符.

于是我们决定通过反编译来获得 `inf_enc` 的生成算法.

## 反编译
为了更好地阅读源码, 我们将 `jadx` 反编译出的源码用 `IDEA` 打开.

根据多年开发经验, 通常国内的 Android 程序员很喜欢著名代码默写器 `Eclipse` 与早已停止维护多年的的 `ADT` 插件. 所以并不会被提示代码缺陷, 其中就包括硬编码问题.

所以我们只需要搜索程序中出现的一些文本, 就可以找到对应的代码所在的位置.

我们在未输入用户名的情况下点击 `登录` 按钮会被提示 `请输入你的登录账号`, 这段文本一定在这个按钮的监听器中, 我们搜索它.

{% asset_img '2018-09-21 20-58-42屏幕截图.png' search %}

搜索结果有两个, 而第二个结果里面有什么 验证码 之类的东西, 我们在 Android 客户端登录的时候是没有验证码的, 所以一定不是这个.

在类 `com.chaoxing.mobile.account.d`(已混淆) 中我们找到了一个 ClickListener

    private OnClickListener q = new OnClickListener() {
        public void onClick(View view) {
            if (!CommonUtils.isFastClick()) {
                int id = view.getId();
                if (id == R.id.tv_left) {
                    d.this.getActivity().onBackPressed();
                } else if (id == R.id.tv_right) {
                    d.this.d();
                } else if (id == R.id.iv_clear_account) {
                    d.this.e.setText("");
                    d.this.c();
                    d.this.e.requestFocus();
                    d.this.showSoftInput(d.this.e);
                } else if (id == R.id.tv_forget_password) {
                    d.this.f();
                } else if (id == R.id.btn_login) {
                    d.this.l();
                } else if (id == R.id.tv_sign_up) {
                    d.this.e();
                } else if (id == R.id.tv_login_by_phone) {
                    d.this.g();
                } else if (id == R.id.tv_login_by) {
                    d.this.i();
                }
            }
        }
    };

从布局 XML 中确认了 `btn_login` 就是那个登录按钮.

我们来看一下这个 `l()` 方法

    private boolean l() {
        e.a();
        String trim = this.e.getText().toString().trim();
        String obj = this.g.getText().toString();
        if (y.c(trim)) {
            aa.a(getActivity(), "请输入你的登录账号");
            this.e.requestFocus();
            return false;
        } else if (y.c(obj)) {
            aa.a(getActivity(), "请输入你的密码");
            this.g.requestFocus();
            return false;
        } else {
            hideSoftInput();
            a(trim, obj, 1);
            return true;
        }
    }

就是那段文本所在的方法, 如果用户名和密码都被输入过的话, 将执行 `a(trim, obj, 1)`, 而这个方法是这样的

    //login
    private void a(String username, String password, int loginType) {   //loginType is always 1
        getLoaderManager().destroyLoader(4097);
        try {
            MultipartEntity multipartEntity = new MultipartEntity();
            multipartEntity.addPart("uname", new StringBody(username, Charset.forName("UTF-8")));
            multipartEntity.addPart("code", new StringBody(password, Charset.forName("UTF-8")));
            username = com.chaoxing.mobile.unit.a.b.l;  //loginType
            StringBuilder stringBuilder = new StringBuilder();  //can be replaced with String
            stringBuilder.append(loginType);
            stringBuilder.append("");
            multipartEntity.addPart(username, new StringBody(stringBuilder.toString(), Charset.forName("UTF-8")));
            multipartEntity.addPart("roleSelect", new StringBody("true", Charset.forName("UTF-8")));    //hard coded
            Bundle bundle = new Bundle();
            bundle.putString("apiUrl", g.bq()); //g is passwordEditText(R.id.et_password)
            getLoaderManager().initLoader(4097, bundle, new a(multipartEntity));
            this.m.setVisibility(0);
        } catch (Throwable e) {
            com.google.a.a.a.a.a.a.b(e);    //Toast
        }
    }

(为了方便阅读, 一些变量已经人工反混淆并添加了一些注释, 下同)

这里构造了一个 `MultipartEntity` (来自 Apache HttpClient), 而其内容就是我们登陆的时候, 发送的那个 MultipartForm.

`uname` 就是 用户名

`code` 就是 密码(明文, 对, 你没听错, 明文)

`loginType` 实际上是硬编码的, `a(String username, String password, int loginType)` 只在一个地方被调用, IDEA 提示其值永远为 1.

`roleSelect` 也是硬编码的, 值永远为 "true"

从这里我们已经得知了整个 MultipartForm 的生成算法.

然后我们继续看, 这个 `MultipartEntity` 被构造后, 被用于构造一个 `LoaderCallbacks`.

我们看一下这个 `LoaderCallbacks` 在哪里, 然后惊人的发现, 还是他妈在这个类里面(内部类). 将 '把所有东西写在一个文件' 的作风表现到了极致.

    /* compiled from: TbsSdkJava */
    private class a implements LoaderCallbacks<Result> {
        private MultipartEntity multipartEntity;
        private String c;

        public void onLoaderReset(Loader<Result> loader) {
        }

        a() {
        }

        a(MultipartEntity multipartEntity) {
            this.multipartEntity = multipartEntity;
        }

        a(String str) {
            this.c = str;
        }

        public Loader<Result> onCreateLoader(int i, Bundle bundle) {
            if (i != 4097) {
                return null;
            }
            Loader dataLoaderWithLogin = new DataLoaderWithLogin(d.this.getContext(), bundle, this.multipartEntity);
            dataLoaderWithLogin.setOnCompleteListener(d.this.u);
            return dataLoaderWithLogin;
        }

        /* renamed from: a */
        public void onLoadFinished(Loader<Result> loader, Result result) {
            d.this.getLoaderManager().destroyLoader(loader.getId());
            if (loader.getId() == 4097) {
                d.this.a(result);
            }
        }
    }

我们之前的 `MultipartEntity` 被传入了一个叫做 `DataLoaderWithLogin` 的类里面. 这个类在包 `com.fanzhou.loader.support` 中, 似乎是什么工具类.

我们在这个类中没有找到有关 `inf_enc` 或者 `_time` 的字眼, 这个类似乎是一个通用工具类.

我们尝试在 Github 搜索 `fanzhou` 一词, 没有找到有效结果, https://github.com/fanzhou 这名用户甚至一个仓库都没有.

我们在 mvnrepository 搜索 `com.fanzhou` 一词也没有搜索到任何结果.

最后我们在 Google 搜索 fanzhou.com, 我们惊异的发现, 真的有这个网站. 这是一个连 HTTPS 都没有的网站 http://www.fanzhou.com/index.html

网站的首页是一个 Chrome 早已默认不加载的 Flash, 以及一些资源不存在的图片. 甚至还能看到著名垃圾站生成工具 '织梦CMS' 的名字.

我们推测, 这个包可能是超星的某个雇员写的, 并用自己的域名作为其包名.

既然这个类是通用工具类, 而里面并没有 `inf_enc` 相关内容, 说明这个参数应该是用某种拦截器加上去的.

我们直接搜索 `inf_enc`, 找到了类 `com.fanzhou.util.ac`

    private static String b() {
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append("token=");
        stringBuilder.append("4faa8662c59590c6f43ae9fe5b002b42");
        stringBuilder.append("&_time=");
        stringBuilder.append(System.currentTimeMillis());
        return stringBuilder.toString();
    }

    private static String f(String str) {
        return d("Z(AfY@XS", str);
    }

    private static String d(String str, String str2) {
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append(str2);
        stringBuilder.append("&DESKey=");
        stringBuilder.append(str);
        str = m.b(stringBuilder.toString());
        stringBuilder = new StringBuilder();
        stringBuilder.append(str2);
        stringBuilder.append("&inf_enc=");
        stringBuilder.append(str);
        return stringBuilder.toString();
    }

举头一看还能看到 `token` 和 `_time` 字眼.

这个 `token` 确实是硬编码的, 永远为固定值 `4faa8662c59590c6f43ae9fe5b002b42`

`_time` 为当前时间戳(不是 UnixTimeStamp, 是带毫秒的)

我们看到其中还有什么 `DESKey` 之类的东西, 难道这居然是 `DES 加密`, 后来我们知道, 我们不出所料的高估了超星程序员.

我们来看一下其中的 `m.b(String)` 是个什么方法. 它在另一个工具类 `com.fanzhou.util.m` 里面.

    public static String b(String str) {
        if (!(str == null || str.length() == 0)) {
            try {
                MessageDigest instance = MessageDigest.getInstance("MD5");
                instance.update(str.getBytes());
                byte[] digest = instance.digest();
                StringBuffer stringBuffer = new StringBuffer("");
                for (int i : digest) {  //i is nerver used
                    int i2;
                    if (i2 < 0) {
                        i2 += 256;
                    }
                    if (i2 < 16) {
                        stringBuffer.append("0");
                    }
                    stringBuffer.append(Integer.toHexString(i2));
                }
                return stringBuffer.toString();
            } catch (Throwable e) {
                a.b(e); //Toast
            }
        }
        return null;
    }

这里我们看到, 它对传入的字符串进行了 `MD5` 加密, 然后对结果 byte[] 进行了一次遍历处理, 但是不知道为何, 反编译结果中的 i2 变量并没有被初始化.

现在我们暂时还不知道他是怎么样一个处理过程, 我们先将这些方法拷贝出来, 运行一下.

{% asset_img '2018-09-21 21-54-40屏幕截图.png' 'd(String str, String str2)' %}

运行到此处时, `m.b(String)` 的传入值为

    token=4faa8662c59590c6f43ae9fe5b002b42&_time=1537538011800&DESKey=Z(AfY@XS

{% asset_img '2018-09-21 21-59-38屏幕截图.png' 'b(String str)' %}

而运行到此处, 也只是简单的把传入的字符串给 `MD5` 加密了一下.

最终程序的输出为

    token=4faa8662c59590c6f43ae9fe5b002b42&_time=1537538011800&inf_enc=00000000000000000000000000000000

我们回过头来看一下那个循环的内容

    for (int i : digest) {
        int i2;
        if (i2 < 0) {
            i2 += 256;
        }
        if (i2 < 16) {
            stringBuffer.append("0");
        }
        stringBuffer.append(Integer.toHexString(i2));
    }

循环一开始判断了 `i2` 的值是不是小于 0, 如果是, 则加 256 后再将其十六进制值(两位)拼接到字符串最后.

如果 `i2` 小于 16 则先在字符串最后拼接 0 再拼接其十六进制值(一位).

我们去 jadx 再次确认一下这个变量到底是怎么回事.

{% asset_img '2018-09-21 22-06-57屏幕截图.png' jadx %}

我们发现这一行没有行号, 说明这里有编译器优化.

那么, 这个 `i2` 只能是 `i` 赋值过去的临时变量.

我们修改一下拷贝出来的代码, 再把时间戳的值改为我们截获的数据包中的时间戳的值, 再次运行.

{% asset_img '2018-09-21 22-10-32屏幕截图.png' println %}

还记得我们截获的数据包中的 `inf_enc` 的值么, 没错, 就是这个. 我们终于破解了 `inf_enc` 的生成算法.

我们整理一下这个算法, 差不多是这样的

    "token=4faa8662c59590c6f43ae9fe5b002b42&_time=${System.currentTimeMillis()}&DESKey=Z(AfY@XS"
            .toByteArray()
            .let {
                MessageDigest.getInstance("MD5").apply {
                    update(it)
                }.digest()
            }.joinToString(separator = "") {
                var value = it.toInt()
                if (value in 0 until 16) return@joinToString "0${value.toString(16)}"
                if (value in Byte.MIN_VALUE until 0) value += 256
                value.toString(16)
            }
            .run(::println)

最后打印出来的就是 `inf_enc`.

## 模拟登陆
现在, 我们来试一试.

{% asset_img '2018-09-22 00-02-02屏幕截图.png' loginregister %}

接着我们得到了一大堆的 `Cookie`, 这些就是我们的凭证.

我们观察到, APP 在登陆后会立即访问一个地址来获取个人信息, 即 UserInfo

{% asset_img '2018-09-22 00-13-24屏幕截图.png' userLogin4Uname %}

很好, 我们成功的获得了用户信息.

## 调用接口
既然我们已经成功登陆了, 现在我们来调用一些需要登陆才能调用的接口.

{% asset_img '2018-09-22 00-27-30屏幕截图.png' mycourse %}

比如说这个接口可以获取自己需要看的网课的列表.

同样的, 我们可以获得课程里的章节列表, 单元测验等数据.

## 模拟看网课
模拟登陆只是第一步, 我们的征程是模拟看网课.

关于这一点明天再讲.
