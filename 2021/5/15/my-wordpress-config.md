# 又一个 WordPress 博客的初始配置

站点结构：`nginx` => `WordPress`

## 配置 HTTPS

```conf
# 生成公私钥对
ssh-keygen ...

# 生成证书申请请求
openssl req -new -sha256 -key cert.key -subj "/C=CN/ST=Anhui/L=Wuhu/O=whit/CN=hamflx.cn" \
    -reqexts SAN -config <(cat /etc/pki/tls/openssl.cnf \
    <(printf "[SAN]\nsubjectAltName=DNS:hamflx.cn,DNS:*.hamflx.cn")) \
    >cert.csr

# 申请通配符证书
docker run -it --rm \
    -e DP_Id=DP_Id \
    -e DP_Key=DP_Key \
    -v $PWD:/acme.sh \
    neilpang/acme.sh \
    --signcsr \
    --csr /acme.sh/cert.csr \
    --dns dns_dp
```

强制流量从 `http` 重定向到 `https` 的 `nginx` 的配置：

```conf
server {
  listen 80 default_server;
  listen [::]:80 default_server;
  server_name _;
  return 301 https://$host$request_uri;
}
```

## 配置 HTTP 2.0

```conf
server {
    listen       443 default_server ssl http2;
    listen  [::]:443 default_server ssl http2;
    server_name  www.hamflx.cn hamflx.cn;

    ssl_certificate     /hamflx.cn/fullchain.cer;
    ssl_certificate_key /hamflx.cn/cert.key;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;

        # 注意这里一定要加，不然在登录的时候会一直重定向。
        proxy_set_header   X-Forwarded-Proto $scheme;

        proxy_pass https://www.hamflx.cn;
    }
}
```

## 配置邮箱

这里并不使用 `WP-SMTP` 类似的插件，而是通过“`My Custom Functions`”插件来实现这个功能。这个插件与邮箱功能无关，但是可以插入代码到 `WordPress`，这里只需要写一个邮箱功能初始化的函数插入到 `WordPress` 即可。

但是启用邮箱之后，激活邮件和重置密码邮件中的链接点击都无效，这是 `WordPress` 的一个 `BUG`。代码中的 `h_mail_filter` 函数即是为解决这个问题的。

在开始之前你需要一个具有 `SMTP` 功能的邮件服务器用来发送邮件，如果没有可以使用 `QQ` 邮箱。

## 配置清单

在 `My Custom Functions` 插件中的 `Settings` 页面中加入如下代码：

```php
function h_override_mail_from_name($email) {
    return '幻梦';
}

function h_override_mail_from($email) {
    return 'service@hamflx.cn';
}

function h_mail_filter($args) {
    $args['message'] = preg_replace("/<(.*?)>/", "$1", $args['message']);
    return $args;
}

function h_mail_smtp($phpmailer) {
    $phpmailer->IsSMTP();
    $phpmailer->SMTPAuth = true;
    $phpmailer->Port = 465;
    $phpmailer->SMTPSecure = "ssl";
    $phpmailer->Host = "smtp.qq.com";
    $phpmailer->Username = "service@hamflx.cn";
    $phpmailer->Password = "Your Token";
}

add_filter('wp_mail_from_name', 'h_override_mail_from_name');
add_filter('wp_mail_from', 'h_override_mail_from');
add_filter('wp_mail', 'h_mail_filter');

add_action("phpmailer_init", "h_mail_smtp");
```

## 启用 QQ 邮箱 SMTP 功能

若使用 `QQ` 邮箱，则需要启用 `QQ` 邮箱的 `SMTP` 功能并生成授权码（作为 `SMTP` 登录时的密码）。

在 `QQ` 邮箱的设置页面中进入到“账户”选项卡，找到下图的配置，启用“`POP3/SMTP`”服务、生成授权码。

## 参考资料

- [自己动手实现 WordPress 的邮件通知功能](https://leonax.net/p/6391/implement-wordpress-email-notification-myself/)
- [WordPress中各种邮件内容及标题的自定义](https://www.jerryzone.cn/diy-wp-email-content/)
- [完美解决wordpress邮件链接无效的问题](https://www.cnblogs.com/kenshinobiy/p/7441781.html)
