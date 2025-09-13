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
    `1.1.x` 需要换成实际的openssl 版本，可通过 `openssl version` 命令查看。

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
    `/api/janus` 需要根据实际的 {janus_url} 替换。
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
9. 部署到**皮肤站服务器**，删除 `node_modules` 文件夹，运行命令安装生产依赖并生成 Prisma 客户端：
    ```bash
    npm i --omit=dev
    npx prisma generate
    ```
    将 `dist`、`node_modules` 和 `.env` 打包上传到**皮肤站服务器**，部署目录自选。修改`.env`中的数据库配置为**皮肤站服务器**访问数据库的地址。SQLite 把**你的 PC** 的数据库文件上传覆盖回**皮肤站服务器**。

    将**皮肤站目录**下的 storage/oauth-private.key 复制到 **Janus 部署目录**。

    使用 `node dist/main.js` 命令试运行看有无报错。

10. 配置守护进程，这里以 systemd 为例，在 `/etc/systemd/system/` 下创建 `janus-daemon.service` 文件，内容如下：
    ```toml
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

## 一些坑

1. `binaryTargets` 不好确定可以先不设置，部署到**皮肤站服务器**后根据运行报错在**你的 PC** 设置后重复步骤 9。