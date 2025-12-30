日期：2025-02-23
## 概览
NoneBot2 是一个现代、跨平台、可扩展的 Python 聊天机器人框架（下称 NoneBot），它基于 Python 的类型注解和异步优先特性（兼容同步），能够为你的需求实现提供便捷灵活的支持。同时，NoneBot 拥有大量的开发者为其开发插件，用户无需编写任何代码，仅需完成环境配置及插件安装，就可以正常使用 NoneBot。

需要注意的是，NoneBot 仅支持 Python 3.9 以上版本

## 步骤
先部署好 NoneBot2 环境 再接入 LLOneBot 添加 WS 反向地址
NoneBot2 安装代码
1.安装pipx
```
python -m pip install --user pipx
```
```
python -m pipx ensurepath
```
2.安装脚手架
```
pipx install nb-cli
```
3.创建启动 NoneBot2 项目
```
nb create
```

## LLOneBot是什么？
LLOneBot基于 NTQQ 实现的 QQ BOT
## WS 反向地址：
```
ws://127.0.0.1:8080/onebot/v11/ws
```
## qq分析机器人代码 需要特别注意 代码中的路径修改 和插件命名规范 
 
插件命名  ：
```
plugin_example.py
```

```python
import os
import asyncio
import aiohttp
import shutil
import requests
import json
import time
from nonebot import on_notice, on_message
from nonebot.adapters.onebot.v11 import Bot, GroupUploadNoticeEvent, GroupMessageEvent, Message
from nonebot.permission import SUPERUSER
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler, FileModifiedEvent, FileMovedEvent
import logging

# 配置日志
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# 保存文件的目录
SAVE_DIR = r"C:\Users\Administrator\Downloads"

# 检查保存文件的目录是否存在，不存在则创建
if not os.path.exists(SAVE_DIR):
    os.makedirs(SAVE_DIR)

# 创建一个通知事件监听器，用于监听群文件上传事件
group_file_upload = on_notice()


@group_file_upload.handle()
async def handle_group_file_upload(bot: Bot, event: GroupUploadNoticeEvent):
    file_info = event.file
    file_name = file_info.name
    file_id = file_info.id
    group_id = event.group_id
    user_id = event.user_id  # 获取发送文件的群成员的 QQ 号

    logging.info(f"收到群 {group_id} 的文件上传通知，文件名: {file_name}，文件 ID: {file_id}，发送者 QQ: {user_id}")

    try:
        # 获取文件下载链接
        logging.info(f"尝试获取文件 {file_name} 的下载链接，群 ID: {group_id}，文件 ID: {file_id}")
        result = await bot.call_api("get_group_file_url", group_id=group_id, file_id=file_id)
        download_url = result.get("url")
        if not download_url:
            logging.error(f"获取文件 {file_name} 的下载链接失败，返回结果: {result}")
            return
        logging.info(f"成功获取文件 {file_name} 的下载链接: {download_url}")

        file_path = None  # 初始化 file_path 变量
        if download_url.startswith('file://'):
            # 处理本地文件链接
            local_file_path = download_url[7:]  # 去掉 'file://' 前缀
            file_path = os.path.join(SAVE_DIR, file_name)
            try:
                shutil.copy2(local_file_path, file_path)
                logging.info(f"文件 {file_name} 复制成功，保存路径: {file_path}")
            except Exception as e:
                # 仅在复制文件真正出错时打印错误日志
                if not os.path.exists(file_path):
                    logging.error(f"复制文件 {file_name} 时发生错误: {str(e)}")
        else:
            # 处理网络链接
            async with aiohttp.ClientSession() as session:
                async with session.get(download_url) as response:
                    if response.status == 200:
                        file_path = os.path.join(SAVE_DIR, file_name)
                        with open(file_path, 'wb') as f:
                            while True:
                                chunk = await response.content.read(1024)
                                if not chunk:
                                    break
                                f.write(chunk)
                        logging.info(f"文件 {file_name} 下载成功，保存路径: {file_path}")
                    else:
                        # 仅在网络下载真正出错时打印错误日志
                        if not os.path.exists(os.path.join(SAVE_DIR, file_name)):
                            logging.error(f"下载文件 {file_name} 失败，状态码: {response.status}")

        if file_path:
            try:
                # 调用文件分析函数
                analysis_result = process_file(file_path, group_id, bot)
                at_msg = f"[CQ:at,qq={user_id}]\n"  # 构造 @ 消息
                if analysis_result == "未能提取到任何信息":
                    full_message = at_msg + "该文件目前无法分析"
                else:
                    filtered_result = filter_analysis_result(analysis_result)  # 过滤分析结果
                    full_message = at_msg + filtered_result  # 拼接 @ 消息和过滤后的分析结果
                logging.info(f"尝试发送消息，群 ID: {group_id}，消息内容: {full_message}")
                if hasattr(bot, 'send_group_msg'):
                    await bot.send_group_msg(group_id=group_id, message=Message(full_message))
                else:
                    logging.error("bot 对象没有 send_group_msg 方法")

                # 发送完消息后删除文件
                if os.path.exists(file_path):
                    os.remove(file_path)
                    logging.info(f"文件 {file_name} 已删除")
                else:
                    logging.warning(f"文件 {file_name} 不存在，无法删除")

            except Exception as e:
                logging.error(f"调用 process_file 或发送消息时出现错误: {str(e)}")
        else:
            logging.error("文件路径为空，无法进行文件分析和消息发送")

    except aiohttp.ClientError as e:
        # 仅在网络请求真正出错且文件未下载成功时打印错误日志
        if not os.path.exists(os.path.join(SAVE_DIR, file_name)):
            logging.error(f"下载文件 {file_name} 时发生网络错误: {str(e)}")
    except Exception as e:
        # 仅在出现未知错误且文件未下载成功时打印错误日志
        if not os.path.exists(os.path.join(SAVE_DIR, file_name)):
            logging.error(f"下载文件 {file_name} 时发生未知错误: {str(e)}")


def upload(filepath: str) -> str:
    url = "http://47.98.235.44:5005/upload"
    # 获取文件的实际名称
    filename = os.path.basename(filepath)
    files = [
        ('upload_dump', (filename, open(filepath, 'rb'), 'application/octet-stream'))
    ]
    response = requests.post(url, files=files)
    return response.text


def analysis(uuid: str) -> str:
    url = "http://47.98.235.44:5005/analysis"
    payload = json.dumps({"uuid": uuid})
    headers = {
        'Content-Type': 'application/json'
    }
    response = requests.post(url, headers=headers, data=payload)
    return response.text


class FileChangeHandler(FileSystemEventHandler):
    ALLOWED_EXTENSIONS = ['.dump', '.bin', '.txt']
    CHECK_INTERVAL = 1  # 检查文件大小变化的时间间隔（秒）
    CHECK_DURATION = 3  # 持续检查的总时长（秒）

    def process_file(self, filepath, group_id=None, bot=None):
        file_extension = filepath.lower().rsplit('.', 1)[-1] if '.' in filepath else ''
        file_extension = '.' + file_extension

        if file_extension in self.ALLOWED_EXTENSIONS:
            start_time = time.time()
            last_size = os.path.getsize(filepath)
            while time.time() - start_time < self.CHECK_DURATION:
                time.sleep(self.CHECK_INTERVAL)
                new_size = os.path.getsize(filepath)
                if new_size != last_size:
                    last_size = new_size
                    start_time = time.time()

            try:
                result = upload(filepath)
                print(f"文件 {os.path.basename(filepath)} 上传成功，服务器返回: {result}")
                response_data = json.loads(result)
                uuid = response_data.get("uuid")
                if uuid:
                    analysis_result = analysis(uuid)
                    analysis_data = json.loads(analysis_result)
                    print("分析结果:", analysis_data)
                    if analysis_data.get("code") == 200:
                        messages = []
                        for item in analysis_data.get("data", [])[0]:
                            if isinstance(item, dict):
                                name = item.get("name", "")
                                values = item.get("value", [])
                                for value in values:
                                    if isinstance(value, dict):
                                        b = value.get("b", "")
                                        messages.append(f"{name}: {b}")
                        if messages:
                            result_message = "\n".join(messages)
                            print(f"解析成功，返回数据:\n{result_message}")
                            return result_message
                        else:
                            print("未能提取到任何信息")
                            return "未能提取到任何信息"
                    else:
                        print(f"解析失败: {analysis_data.get('message')}")
                        return f"解析失败: {analysis_data.get('message')}"
                else:
                    print("未能获取 UUID")
                    return "未能获取 UUID"
            except json.JSONDecodeError:
                print("返回数据格式错误，无法解析 JSON")
                return "返回数据格式错误，无法解析 JSON"
            except Exception as e:
                print(f"发生错误: {str(e)}")
                return f"发生错误: {str(e)}"
        else:
            print(f"文件 {filepath} 扩展名不允许，跳过上传和分析。")
            return f"文件 {filepath} 扩展名不允许，跳过上传和分析。"

    def on_modified(self, event: FileModifiedEvent):
        if not event.is_directory:
            self.process_file(event.src_path)

    def on_moved(self, event: FileMovedEvent):
        if not event.is_directory:
            self.process_file(event.dest_path)


process_file = FileChangeHandler().process_file


def filter_analysis_result(result):
    lines = result.split('\n')
    # 如果结果只有一行，直接返回原结果
    if len(lines) == 1:
        return result
    filtered_lines = []
    for line in lines:
        if "系统" in line:  # 过滤包含 "系统" 的行
            filtered_lines.append(line)
    return '\n'.join(filtered_lines)


# 启动文件监控
event_handler = FileChangeHandler()
observer = Observer()
path = r"C:\Users\Administrator\Downloads"
observer.schedule(event_handler, path=path, recursive=False)
observer.start()
```
## 必装库
需要进入NoneBot2虚拟环境安装
```
pip install aiohttp requests watchdog -i https://mirrors.aliyun.com/pypi/simple/
```
