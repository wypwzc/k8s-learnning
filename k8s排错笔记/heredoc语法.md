### 基本格式

```bash
command <<EOF
内容...
EOF
```

| 部分         | 作用                                                         |
| :----------- | :----------------------------------------------------------- |
| `<<`         | 重定向操作符，表示 here-doc 开始                             |
| `EOF`        | **定界符**（delimiter），可以换成任意字符串，如 `END`、`MARKER` |
| 中间的内容   | 原样传递给 `command` 的标准输入                              |
| 最后的 `EOF` | 单独一行，表示 here-doc 结束                                 |

------

### 你例子中的实际效果

```bash
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

等价于执行：

```bash
cat > /etc/modules-load.d/k8s.conf
```

然后手动输入：

```plain
overlay
br_netfilter
```

再按 Ctrl+D 结束。

所以最终就是把 `overlay\nbr_netfilter\n` 写进 `/etc/modules-load.d/k8s.conf` 文件。

------

### 常见变体

| 写法                  | 含义                                          |
| :-------------------- | :-------------------------------------------- |
| `<<EOF`               | 变量会被展开，`$VAR` 会替换                   |
| `<<'EOF'` 或 `<<\EOF` | **禁止变量展开**，原样保留 `$`                |
| `<<-EOF`              | 忽略每行开头的 **Tab** 缩进（方便脚本里排版） |