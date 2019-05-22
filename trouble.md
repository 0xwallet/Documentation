## Trouble Shooting

- elixir 报错无法启动 apps.

可能是由于在 dockerfile 中设置了非 root 用户, 导致权限不够, 无法写入某些文件. 在 Dockerfile 里设置 `User root` 即可.

- 连接不上数据库.

可能是数据库的 hostname, password 或者 port 配置错误.

- Daocloud 流水线在发布环节失败

删除应用, 重新部署.

- shell 中写入文件时换行符 `\n` 丢失

将环境变量 IFS 设置为 "" (空的), 即可关闭自动剪裁.

- 部署 dev 环境的 dashboard, 显示数据库已创建, 但打开 /auth 页面后, 输入邮箱点击 Enter, 不能进入下一步.

由于4000 端口未开启, 所以访问到的并非 dev 环境.

- 让 DaoCloud 上的 Stack 和主机上的其它应用运行在同一网络下

在单个 service 内使用 `network_mode: bridge` 的配置
