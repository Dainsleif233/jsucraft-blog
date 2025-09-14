## 前言

本文记录了在 Union 皮肤站上升级 Yggdrasil-Connect 插件并部署 Janus 的步骤。使用的机器为 Debian 11 x86_64 0.5G，因此也是在小内存机器上部署的教程。使用的皮肤站数据库为 Sqlite。

## 备份皮肤站

操作前请务必备份皮肤站数据库和插件，以免过程中出现错误导致损失。

## 升级 Yggdrasil-Connect 插件

1. **禁用**并删除旧版插件。
2. 在 `插件管理-从远程下载-URL` 中填写下载连接 `https://mc.sjtu.cn/union` 安装新版插件。在 `插件管理` 中启用插件。
3. 连接终端并打开皮肤站根目录，执行：
    ```shell
    php artisan yggc:fix-uuid-table
    php artisan yggc:create-personal-access-client
    ```
    第一条命令清除了 UUID 表中可能存在的异常数据。第二条命令创建了一个个人访问客户端，返回 `Client ID`。
4. 在 `.env` 中新建一条配置
    ```env
    PASSPORT_PERSONAL_ACCESS_CLIENT_ID=
    ```
    值为命令返回的个人访问客户端的 `Client ID`。
5. 在 `插件管理` 中禁用和启用插件一次。

## 部署 Janus

本节中：
- `{site_url}` 表示皮肤站 Url，例如：`https://skin.jsumc.fun`。
- `{janus_url}` 表示 Janus 的 Url，必须为 https，例如：`https://skin.jsumc.fun/api/janus`。
- `{client_id}` 表示上文获取的 `Client ID`，例如：`1`。

注意替换。

1. 皮肤站禁用 `隐藏「高级功能」菜单` 插件，在 `高级功能-OAuth2 应用` 中对应上文 `Client ID` 的 `回调 URL` 中填入 `{site_url}/yggc/client/public`。
2. 安装 Node.js 22 到**你的 PC** 和**皮肤站服务器**。
3. 下载 Janus 源码到**你的 PC**，仓库地址：
    - MySQL：https://github.com/bs-community/janus.git
    - PostgreSQL：https://github.com/Tobby-000/janus.git
    - SQLite：https://github.com/Dainsleif233/janus.git
4. 设置数据表前缀，检查**皮肤站**的 `.env` 配置，若 `DB_PREFIX` 不存在或为空则跳过这步。将 **Janus** 的 `prisma` 目录下的 `schema.prisma.example` 文件重命名为 `schema.prisma` 并编辑，在**每一个** Model 的 @@map() 中填写的数据表名前添加前缀：
    ```prisma
    model AuthorizationCode {
        // ...

        @@map("yggc_authorization_codes") // <- 修改这里的数据表名，添加表前缀
        // @@map("skin_yggc_authorization_codes") <- 如果你的 DB_PREFIX 是 skin_，就在表名前面加上 skin_
    }
    ```
5. 修改源码：

    在 `schema.prisma` 中修改 `generator client` 为以下内容：
    ```prisma
    generator client {
        provider = "prisma-client-js"
        binaryTargets = ["native", "debian-openssl-1.1.x"]
    }
    ```
    `debian-openssl-1.1.x` 换成**皮肤站服务器**实际的系统：
    ```
    Windows: windows
    macOS Intel: darwin
    macOS M1/M2/M3: darwin-arm64
    Ubuntu/Debian x64: debian-openssl-1.1.x
    Ubuntu/Debian ARM64: linux-arm64-openssl-1.1.x
    Alpine Linux x64: linux-musl
    Alpine Linux ARM64: linux-musl-arm64-openssl-1.1.x
    RHEL/CentOS/Oracle: rhel-openssl-1.1.x
    ```
    `1.1.x` 需要换成实际的 openssl 版本，可通过 `openssl version` 命令查看。

    在 `src/app.controller.ts` 中修改 `getHello`(L36) 为以下内容：
    ```typescript
    @All("/{*path}")
    getHello(@Req() req: Request, @Res() res: Response): Promise<void> {
        req.url = req.originalUrl.replace("/api/janus", "");
        return this.appService.callback(req, res);
    }
    ```

    在 `src/main.ts` 中在 `app.enable('trust proxy');`(L27) 后插入以下内容：
    ```typescript
    app.setGlobalPrefix('api/janus');
    ```

    在 `src/oidc-provider.service.ts` 中修改 `url`(L127) 为以下内容：
    ```typescript
    url(ctx, interaction) {
        const prompt = interaction.prompt;
        return `/api/janus/interaction/${interaction.uid}`;
    },
    ```
    `/api/janus` 需要根据实际的 `{janus_url}` 替换。
6. 修改配置文件，将 **Janus** 根目录的 `.env.example` 重命名为 `.env` 并编辑：
    ```env
    # 服务端口
    PORT=3000

    # MySQL / PostgreSQL
    # 数据库 IP
    DB_HOST=
    # 数据库端口
    DB_PORT=
    # 数据库用户名
    DB_USERNAME=
    # 数据库密码
    DB_PASSWORD=
    # 数据库名称
    DB_NAME=

    # SQLite
    # 数据库文件路径（相对于 prisma 目录），推荐使用绝对路径
    DATABASE_URL="file:/path/to/data.db"

    ISSUER="{janus_url}"
    BS_SITE_URL="{site_url}"
    SHARED_CLIENT_ID="{client_id}"

    # 其余配置项无需修改
    ```
    先将数据库配置设置为能从**你的 PC** 访问的地址，SQLite 把数据库文件下载到**你的 PC**。
7. 安装依赖并构建应用，运行命令：
    ```bash
    npm i
    npm run build
    ```
8. 迁移数据库，确保能连接到数据库，可使用 `npx prisma migrate status` 命令检查数据库迁移状态，执行命令迁移数据库：
    ```bash
    npx prisma migrate resolve --applied 0_init
    npx prisma migrate deploy
    ```
9. 部署到**皮肤站服务器**，运行命令移除开发依赖并生成 Prisma 客户端：
    ```bash
    npm i --omit=dev
    npx prisma generate
    ```
    将 `dist`、`node_modules` 和 `.env` 打包上传到**皮肤站服务器**，部署目录自选。修改`.env`中的数据库配置为**皮肤站服务器**访问数据库的地址。SQLite 把**你的 PC** 的数据库文件上传覆盖回**皮肤站服务器**。

    将**皮肤站目录**下的 storage/oauth-private.key 复制到 **Janus 部署目录**。

    使用 `node dist/main.js` 命令试运行看有无报错。

10. 配置守护进程，这里以 systemd 为例，在 `/etc/systemd/system/` 下创建 `janus-daemon.service` 文件，内容如下：
    ```ini
    [Unit]
    Description=Janus-Daemon

    [Service]
    WorkingDirectory=/path/to/janus
    ExecStart=node dist/main.js
    ExecReload=kill -s QUIT $MAINPID
    ExecStop=kill -s QUIT $MAINPID

    [Install]
    WantedBy=multi-user.target
    ```
    `/path/to/janus` 设置为实际的 **Janus 部署目录**。

    使用以下命令启用守护进程并设置开机自启：
    ```bash
    systemctl daemon-reload
    systemctl start janus-daemon.service
    systemctl enable janus-daemon.service
    ```
11. 配置反向代理，这里以 Nginx 为例，反代 Janus 服务端口到皮肤站地址，这样可以直接使用皮肤站的 SSL，在皮肤站 nginx 文件的 server 块中添加以下内容：
    ```nginx
    location /api/janus {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_pass http://localhost:3000;
    }
    ```
    `/api/janus` 和 端口 `3000` 根据实际配置设置。**注意**：`/api/janus` 和 `http://localhost:3000` 后不要加 `/`

    运行 `systemctl restart nginx` 命令重启 Nginx。

## 修改前端

修改前端页面以统一授权页面风格，这一步是**步骤5**的拓展，修改完成后需要重新构建 `dist` 目录上传到**皮肤站服务器**。

1. 修改 `oidc-provider.service.ts` 文件，在 `OIDCProviderService` 类中添加以下代码：
    ```typescript
    // 填写皮肤站站点名和站点图标 Url
    private readonly siteName = '';
    private readonly favicon = '';

    generateHtml(title: string, content: string) {
        return `<!DOCTYPE html>
            <html lang="zh-CN">
                <head>
                    <meta charset="utf-8">
                    <meta http-equiv="X-UA-Compatible" content="IE=edge">
                    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
                    <link rel="stylesheet" href="https://cdn.bootcdn.net/ajax/libs/font-awesome/5.15.4/css/all.min.css" crossorigin="">
                    <link href="https://bs-cdn.yecdn.com/6.0.2/public/app/style.7eb5d06.css" rel="stylesheet" crossorigin="anonymous">
                    <link rel="shortcut icon" href="${this.favicon}">
                    <link rel="icon" type="image/png" href="${this.favicon}" sizes="192x192">
                    <link rel="apple-touch-icon" href="${this.favicon}" sizes="180x180">
                    <link href="https://bs-cdn.yecdn.com/6.0.2/public/app/home-css.bef20ec.css" rel="stylesheet" crossorigin="anonymous">
                    <title>${title} - ${this.siteName}</title>
                </head>
                <body class="hold-transition login-page">
                    <div class="login-box">
                        <div class="login-logo">
                            <a href="${this.siteUrl}">${this.siteName}</a>
                        </div>
                        <div class="card">
                            <div class="card-body login-card-body">
                                ${content}
                            </div>
                        </div>
                    </div>
                </body>
            </html>`
    }

    async successSource(ctx: oidc.KoaContextWithOIDC) {
        const content = `
            <div class="text-center py-5">
                <i class="far fa-check-circle text-success fa-5x mb-4" aria-hidden="true"></i>
                <h5 class="text-success mb-0">登录成功</h5>
            </div>`;
        ctx.body = this.generateHtml('登录成功', content);
    }

    async userCodeConfirmSource(ctx: oidc.KoaContextWithOIDC, form: String, _client: any, _deviceInfo: any, userCode: String) {
        const content = `
            <p class="login-box-msg">登录至 ${ctx.oidc.client?.clientName || ctx.oidc.client?.clientId}</p>
            <main>
                <div class="alert alert-info">请确认以下授权码与您的应用中显示的授权码相符。</div>
                    <div class="mb-3 text-center" style="font-size: 1.6em; font-weight: bold; font-family: Minecraft;">${userCode}</div>
                <div class="alert alert-warning">
                    <i class="icon fas fa-exclamation-triangle"></i>如果您没有发起此操作，或者该授权码与您的应用中显示的授权码不匹配，请关闭此窗口或点击取消。
                </div>
                ${form}
                <button class="btn btn-success btn-block" type="submit" form="op.deviceConfirmForm">继续</button>
                <button class="btn btn-default btn-block" type="submit" form="op.deviceConfirmForm" value="yes" name="abort">取消</button>
            </main>`;
        ctx.body = this.generateHtml('授权', content);
    }

    async userCodeInputSource(ctx: oidc.KoaContextWithOIDC, form: String, _out: any, err: any) {
        let msg: string;
        if (err && (err.userCode || err.name === 'NoCodeError')) msg = '您输入的代码不正确，请重试';
        else if (err && err.name === 'AbortedError') msg = '登录请求被中断';
        else if (err) msg = '处理请求时发生错误';
        else msg = '请输入您设备上显示的代码';

        const content = `
            <p class="login-box-msg">授予应用访问权限</p>
            <main>
                <div class="alert alert-danger">${msg}</div>
                <div class="form-group">${form}</div>
                <div class="alert alert-warning">
                    <i class="icon fas fa-exclamation-triangle"></i>请勿输入来自你不信任的来源的授权码，以免造成个人隐私泄露和账号安全问题。
                </div>
                <button class="btn btn-success btn-block" type="submit" form="op.deviceInputForm">继续</button>
            </main>
            <script>
                input = document.getElementsByName('user_code')[0];
                input.placeholder = '输入应用中显示的授权码';
                input.classList.add('form-control');
            </script>`;
        ctx.body = this.generateHtml('授权', content);
    }
    ```

2. 修改 `provider` 构建方法中的 `deviceFlow` 为：
    ```typescript
    deviceFlow: {
        enabled: true,
        successSource: this.successSource.bind(this),
        userCodeConfirmSource: this.userCodeConfirmSource.bind(this),
        userCodeInputSource: this.userCodeInputSource.bind(this)
    }
    ```

## 一些坑

1. `binaryTargets` 不好确定可以先不设置，部署到**皮肤站服务器**后根据运行报错在**你的 PC** 设置后重复步骤 9。