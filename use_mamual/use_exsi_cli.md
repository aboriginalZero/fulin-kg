```shell
# 查看虚拟机进程
esxcli vm process list
# 类比 htop
esxtop
```

杀死对应进程

1. 按 C 切换到 CPU 资源利用率屏幕。
2. 按 Shift+V 将视图限定为虚拟机。这样会更容易在步骤 7 中找到 Leader World ID。
3. 按 F 显示字段列表。
4. 按 C 添加 Leader World ID 列。
5. 通过目标虚拟机的名称和领导者域 ID (`LWID`) 来标识目标虚拟机。
6. 按 K。
7. 在 `World to kill` 提示符处，键入步骤 7 中获取的 Leader World ID，然后按 Enter。
8. 等待 30 秒后验证该进程是否已不再列出。





kb.vmware.com 中较为权威