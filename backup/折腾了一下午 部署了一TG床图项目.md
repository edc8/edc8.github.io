### 本着能白嫖的就白嫖的原则多亏了cf 大善人 让我白嫖了一床图系统 

~~这下也能在博客上添加图片了~~
算了 貌似不能使用这个床图系统的外链  显示裂图 

### 不过 也算学习了一下部署方法 
我太粗心了 一直用的Workers部署一直出错 这点要注意，需要用pages部署 步骤如下：

点击左侧菜单的 Workers & Pages。

点击 Create application (创建应用程序)。

关键： 页面上方有两个标签页，请务必点击 Pages（通常在右侧），而不是默认的 Workers。

点击 Connect to Git (连接到 Git)。

选择你的仓库并点击 Begin setup (开始设置)。

在 Pages 模式下的填写配置

现在你会发现界面变了，这时候它不会强制要求你填部署命令。请这样填写：

Framework preset (框架预设): 选择 None。

Build command (构建命令): 保持完全空白（不要填任何东西）。

Build output directory (构建输出目录): 只填一个斜杠 /。

Root directory (根目录): 保持为 /。

点击 Save and Deploy。

绑定 KV（给它一个数据库）
去 Settings -> Functions -> KV namespace bindings。

点击 Add binding。

变量名称必须填：img_kv。

KV 命名空间选择你创建的那个。

设置后台（防止被盗刷）
去 Settings -> Environment variables。

添加 BASIC_USER (用户名) 和 BASIC_PASS (密码)。
