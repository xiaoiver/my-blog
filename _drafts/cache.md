[](https://jakearchibald.com/2016/caching-best-practices/)

## 第一种情况：同一个 URL 内容不变

`Cache-Control: max-age=31536000`

由于 URL 包含 hash 也就是文件内容，当需要变更文件内容时，URL 也随之改变。
<script src="/script-f93bca2c.js"></script>

## 内容变化

`Cache-Control: no-cache`

同一个 URL，内容可能发生变化，因此每次都需要询问服务器。

不等于不能使用缓存，在使用缓存前需要向服务器确认。
如果完全禁止使用缓存，应该使用 `no-store`

这种情况下使用 ETag/Last-Modified，总归需要向服务端发送请求。

### 避免每次请求服务器

`Cache-Control: must-revalidate, max-age=600`

`must-revalidate` 并不是每次都需要向服务器确认，在 `max-age` 前都不需要确认，超过了才需要确认。
