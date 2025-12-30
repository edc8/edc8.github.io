日期：2025-01-20

## 得益于AI技术的普及，让我这个非专业选手也能实现使用python开发出自己想要的功能
### 在此记录贴出代码用作备份的 
### 无自动回复功能版 
```python
import os
import time
import tkinter as tk
from tkinter import scrolledtext, messagebox
import requests
import json
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import threading
import wxauto
import win32clipboard
import pythoncom


class DumpFileHandler(FileSystemEventHandler):
    def __init__(self, text_widget, upload_callback):
        self.text_widget = text_widget
        self.upload_callback = upload_callback
        self.event_processed = {}  

    def on_created(self, event):
        self.handle_event(event, '新文件创建')

    def on_modified(self, event):
        self.handle_event(event, '文件修改')

    def on_deleted(self, event):
        self.handle_event(event, '文件删除')

    def handle_event(self, event, event_type):
        if event.is_directory:
            return
        file_path = event.src_path
        if file_path.endswith(('.dump', '.bin', '.txt')):
            if file_path not in self.event_processed:
                self.text_widget.insert(tk.END, f'{event_type}: {file_path}\n')
                self.text_widget.yview(tk.END)
                self.upload_callback(filepath=file_path)  
                self.event_processed[file_path] = True
            else:
                print(f"已处理过 {file_path} 的{event_type}事件，跳过重复处理")


def upload(filepath: str) -> str:
    url = "http://47.98.235.44:5005/upload"
    try:
        with open(filepath, 'rb') as file:
            files = [
                ('upload_dump', (os.path.basename(filepath), file, 'application/octet-stream'))
            ]
            response = requests.post(url, files=files)
            if response.status_code == 200:
                return response.text
            else:
                raise Exception(f"Upload failed with status code {response.status_code}")
    except Exception as e:
        raise Exception(f"Error uploading file: {str(e)}")


def analysis(uuid: str) -> str:
    url = "http://47.98.235.44:5005/analysis"
    payload = json.dumps({"uuid": uuid})
    headers = {
        'Content-Type': 'application/json'
    }
    try:
        response = requests.post(url, headers=headers, data=payload)
        if response.status_code == 200:
            return response.text
        else:
            raise Exception(f"Analysis request failed with status code {response.status_code}")
    except Exception as e:
        raise Exception(f"Error sending analysis request: {str(e)}")


def copy_to_clipboard(output_text: tk.Text):
    try:
        root.clipboard_clear()
        result = output_text.get(1.0, tk.END).strip()
        root.clipboard_append(result)
        output_text.insert(tk.END, "分析结果已复制到剪贴板\n")
    except Exception as e:
        output_text.insert(tk.END, f"错误: 复制到剪贴board时发生错误: {str(e)}\n")


def process_file(filepath: str, output_text: tk.Text, target_group_name):
    def build_message(data):
        messages = []
        if data and data[0]:
            for item in data[0]:
                if isinstance(item, dict):
                    name = item.get("name", "")
                    values = item.get("value", [])
                    for value in values:
                        if isinstance(value, dict):
                            b = value.get("b", "")
                            messages.append(f"{name}: {b}")
        return "\n".join(messages) if messages else None

    def handle_analysis_result(analysis_data):
        if analysis_data.get("code") == 200:
            data = analysis_data.get("data", [])
            result_message = build_message(data)
            if result_message:
                output_text.delete(1.0, tk.END)
                output_text.insert(tk.END, f"解析成功，返回数据:\n{result_message}\n")
                copy_to_clipboard(output_text)
                send_to_wechat(result_message, target_group_name)  # 调用发送微信消息的函数并传入群名称
                return result_message
            else:
                warning_message = "该数据暂未收录"
                output_text.delete(1.0, tk.END)
                output_text.insert(tk.END, warning_message + "\n")
                copy_to_clipboard(output_text)
                send_to_wechat(warning_message, target_group_name)  # 调用发送微信消息的函数并传入群名称
                return warning_message
        else:
            warning_message = f"警告: 解析失败: {analysis_data.get('message')}\n"
            output_text.delete(1.0, tk.END)
            output_text.insert(tk.END, warning_message)
            copy_to_clipboard(output_text)
            send_to_wechat(warning_message, target_group_name)  # 调用发送微信消息的函数并传入群名称
            return warning_message

    def handle_file_deletion(filepath):
        import subprocess
        import sys
        if sys.platform.startswith('win32'):
            command = f'del /f /q "{filepath}"'
        else:
            command = f'rm -f "{filepath}"'
        try:
            subprocess.run(command, shell=True, check=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            output_text.insert(tk.END, f"文件 {filepath} 已成功删除\n")
        except subprocess.CalledProcessError as e:
            output_text.insert(tk.END, f"错误: 删除文件 {filepath} 时出错: {str(e)}\n")
        except Exception as e:
            output_text.insert(tk.END, f"其他错误: {str(e)}\n")

    try:
        result = upload(filepath)
        response_data = json.loads(result)
        uuid = response_data.get("uuid")
        if uuid:
            analysis_result = analysis(uuid)
            analysis_data = json.loads(analysis_result)
            message = handle_analysis_result(analysis_data)
            handle_file_deletion(filepath)
        else:
            warning_message = "警告: 未能获取UUID\n"
            output_text.delete(1.0, tk.END)
            output_text.insert(tk.END, warning_message)
            copy_to_clipboard(output_text)
            send_to_wechat(warning_message, target_group_name)  # 调用发送微信消息的函数并传入群名称
    except json.JSONDecodeError:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, "错误: 返回数据格式错误，无法解析JSON\n")
        send_to_wechat("错误: 返回数据格式错误，无法解析JSON\n", target_group_name)  # 调用发送微信消息的函数并传入群名称
    except Exception as e:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, f"错误: 发生错误: {str(e)}\n")
        send_to_wechat(f"错误: 发生错误: {str(e)}\n", target_group_name)  # 调用发送微信消息的函数并传入群名称


class MonitorApp:
    def __init__(self, master):
        self.master = master
        self.master.title("文件监控工具")
        self.master.geometry("500x450")  # 增加高度以容纳新组件

        self.label_group = tk.Label(master, text="目标群名称:")  # 新增标签
        self.label_group.pack(pady=10)

        self.group_entry = tk.Entry(master, width=50)  # 新增输入框
        self.group_entry.pack(pady=10)

        self.label = tk.Label(master, text="监控目录:")
        self.label.pack(pady=10)

        self.directory_entry = tk.Entry(master, width=50)
        self.directory_entry.pack(pady=10)

        self.start_button = tk.Button(master, text="开始监控", command=self.start_monitoring)
        self.start_button.pack(pady=10)

        self.stop_button = tk.Button(master, text="停止监控", command=self.stop_monitoring)
        self.stop_button.pack(pady=10)

        self.output_text = scrolledtext.ScrolledText(master, width=60, height=15)
        self.output_text.pack(pady=10)

        self.observer = None
        self.is_running = False

    def start_monitoring(self):
        group_name = self.group_entry.get().strip()  # 获取群名称
        directory = self.directory_entry.get().strip()
        if not os.path.isdir(directory):
            messagebox.showerror("错误", "请输入有效的目录路径！")
            return

        self.is_running = True
        self.output_text.insert(tk.END, f'开始监控目录: {directory}\n')
        self.output_text.yview(tk.END)

        event_handler = DumpFileHandler(self.output_text, lambda filepath: process_file(filepath, self.output_text, group_name))  # 传入群名称
        self.observer = Observer()
        self.observer.schedule(event_handler, directory, recursive=False)
        self.observer.start()
        threading.Thread(target=self.run_observer, daemon=True).start()

    def run_observer(self):
        try:
            while self.is_running:
                time.sleep(1)
        except Exception as e:
            self.output_text.insert(tk.END, f'监控出错: {str(e)}\n')
            self.output_text.yview(tk.END)

    def stop_monitoring(self):
        self.is_running = False
        if self.observer:
            self.observer.stop()
            self.observer.join()
        self.output_text.insert(tk.END, '监控已停止。\n')
        self.output_text.yview(tk.END)


def get_clipboard_content():
    win32clipboard.OpenClipboard()
    data = win32clipboard.GetClipboardData(win32clipboard.CF_TEXT)
    win32clipboard.CloseClipboard()
    return data.decode('utf-8')


def send_to_wechat(message, target_group_name):  # 接收群名称作为参数
    pythoncom.CoInitialize()
    wechat = wxauto.WeChat()
    session_list = wechat.GetSessionList()
    target_group_session = None
    for session in session_list:
        if target_group_name in session:
            target_group_session = session
            break
    if target_group_session:
        # 先选中目标群会话
        wechat.ChatWith(target_group_session)
        # 向目标群会话发送消息
        wechat.SendMsg(message)


if __name__ == "__main__":
    root = tk.Tk()
    app = MonitorApp(root)
    root.mainloop()
 ```
## 有关键词自动回复功能版；
## 两个版本的回复思路完全不一样，同时这也是编程逻辑的魅力所在。
```python
import os
import time
import tkinter as tk
from tkinter import scrolledtext, messagebox
import requests
import json
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import threading
import wxauto
import win32clipboard
import pythoncom
import logging
from datetime import datetime


# 设置日志记录为输出到命令行
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)
logging.getLogger().addHandler(console_handler)


# 存储分析结果的全局变量
analysis_result_content = ""
# 用于通知有新分析结果的事件
new_result_event = threading.Event()


class DumpFileHandler(FileSystemEventHandler):
    def __init__(self, text_widget, upload_callback):
        self.text_widget = text_widget
        self.upload_callback = upload_callback
        self.event_processed = {}  

    def on_created(self, event):
        self.handle_event(event, '新文件创建')

    def on_modified(self, event):
        self.handle_event(event, '文件修改')

    def on_deleted(self, event):
        self.handle_event(event, '文件删除')

    def handle_event(self, event, event_type):
        if event.is_directory:
            return
        file_path = event.src_path
        if file_path.endswith(('.dump', '.bin', '.txt')):
            if file_path not in self.event_processed:
                self.text_widget.insert(tk.END, f'{event_type}: {file_path}\n')
                self.text_widget.yview(tk.END)
                self.upload_callback(filepath=file_path)  
                self.event_processed[file_path] = True
            else:
                print(f"已处理过 {file_path} 的{event_type}事件，跳过重复处理")


def upload(filepath: str) -> str:
    url = "http://47.98.235.44:5005/upload"
    try:
        with open(filepath, 'rb') as file:
            files = [
                ('upload_dump', (os.path.basename(filepath), file, 'application/octet-stream'))
            ]
            response = requests.post(url, files=files)
            if response.status_code == 200:
                return response.text
            else:
                raise Exception(f"Upload failed with status code {response.status_code}")
    except Exception as e:
        raise Exception(f"Error uploading file: {str(e)}")


def analysis(uuid: str) -> str:
    url = "http://47.98.235.44:5005/analysis"
    payload = json.dumps({"uuid": uuid})
    headers = {
        'Content-Type': 'application/json'
    }
    try:
        response = requests.post(url, headers=headers, data=payload)
        if response.status_code == 200:
            return response.text
        else:
            raise Exception(f"Analysis request failed with status code {response.status_code}")
    except Exception as e:
        raise Exception(f"Error sending analysis request: {str(e)}")


def copy_to_clipboard(output_text: tk.Text):
    try:
        root.clipboard_clear()
        result = output_text.get(1.0, tk.END).strip()
        root.clipboard_append(result)
        output_text.insert(tk.END, "分析结果已复制到剪贴板\n")
    except Exception as e:
        output_text.insert(tk.END, f"错误: 复制到剪贴板时发生错误: {str(e)}\n")


def process_file(filepath: str, output_text: tk.Text, target_group_name):
    def build_message(data):
        messages = []
        if data and data[0]:
            for item in data[0]:
                if isinstance(item, dict):
                    name = item.get("name", "")
                    values = item.get("value", [])
                    for value in values:
                        if isinstance(value, dict):
                            b = value.get("b", "")
                            messages.append(f"{name}: {b}")
        return "\n".join(messages) if messages else None

    def handle_analysis_result(analysis_data):
        global analysis_result_content  # 使用全局变量
        if analysis_data.get("code") == 200:
            data = analysis_data.get("data", [])
            result_message = build_message(data)
            if result_message:
                output_text.delete(1.0, tk.END)
                output_text.insert(tk.END, f"解析成功，返回数据:\n{result_message}\n")
                copy_to_clipboard(output_text)
                analysis_result_content = result_message  # 更新全局变量
                send_to_wechat(result_message, target_group_name)
                new_result_event.set()  # 通知有新结果
                return result_message
            else:
                warning_message = "该数据暂未收录"
                output_text.delete(1.0, tk.END)
                output_text.insert(tk.END, warning_message + "\n")
                copy_to_clipboard(output_text)
                analysis_result_content = warning_message  # 更新全局变量
                send_to_wechat(warning_message, target_group_name)
                new_result_event.set()  # 通知有新结果
                return warning_message
        else:
            warning_message = f"警告: 解析失败: {analysis_data.get('message')}\n"
            output_text.delete(1.0, tk.END)
            output_text.insert(tk.END, warning_message)
            copy_to_clipboard(output_text)
            analysis_result_content = warning_message  # 更新全局变量
            send_to_wechat(warning_message, target_group_name)
            new_result_event.set()  # 通知有新结果
            return warning_message

    def handle_file_deletion(filepath):
        import subprocess
        import sys
        if sys.platform.startswith('win32'):
            command = f'del /f /q "{filepath}"'
        else:
            command = f'rm -f "{filepath}"'
        try:
            subprocess.run(command, shell=True, check=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            output_text.insert(tk.END, f"文件 {filepath} 已成功删除\n")
        except subprocess.CalledProcessError as e:
            output_text.insert(tk.END, f"错误: 删除文件 {filepath} 时出错: {str(e)}\n")
        except Exception as e:
            output_text.insert(tk.END, f"其他错误: {str(e)}\n")

    try:
        result = upload(filepath)
        response_data = json.loads(result)
        uuid = response_data.get("uuid")
        if uuid:
            analysis_result = analysis(uuid)
            analysis_data = json.loads(analysis_result)
            message = handle_analysis_result(analysis_data)
            handle_file_deletion(filepath)
        else:
            warning_message = "警告: 未能获取UUID\n"
            output_text.delete(1.0, tk.END)
            output_text.insert(tk.END, warning_message)
            copy_to_clipboard(output_text)
            send_to_wechat(warning_message, target_group_name)
    except json.JSONDecodeError:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, "错误: 返回数据格式错误，无法解析JSON\n")
        send_to_wechat("错误: 返回数据格式错误，无法解析JSON\n", target_group_name)
    except Exception as e:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, f"错误: 发生错误: {str(e)}\n")
        send_to_wechat(f"错误: 发生错误: {str(e)}\n", target_group_name)


def start_wechat_listener(target_group_name, keyword_reply_dict):
    max_retries = 5  # 最大重试次数
    retry_delay = 5  # 重试延迟（秒）
    retries = 0
    while retries < max_retries:
        try:
            pythoncom.CoInitialize()  # 确保在使用 COM 之前初始化
            wx = wxauto.WeChat()
            logging.info("> 初始化成功，获取到已登录窗口")

            wx.AddListenChat(who=target_group_name, savepic=False)

            replied_messages = {}
            wait = 1
            reply_time_window = 60

            while True:
                msgs = wx.GetListenMessage()
                new_messages = []
                current_time = time.time()

                for msg_identifier in list(replied_messages.keys()):
                    if current_time - replied_messages[msg_identifier] > reply_time_window:
                        del replied_messages[msg_identifier]

                for chat in msgs:
                    group_name = chat.who
                    group_msgs = msgs.get(chat)

                    for msg in group_msgs:
                        msgtype = msg.type
                        content = msg.content
                        msg_identifier = (chat.who, msg.id)

                        logging.info(f"收到消息 - 群聊: {group_name}, 类型: {msgtype}, 内容: {content}")

                        if msgtype == "friend":
                            if msg_identifier not in replied_messages:
                                new_messages.append((msg_identifier, msg))

                for msg_identifier, msg in new_messages:
                    content = msg.content.lower()
                    for keyword, reply in keyword_reply_dict.items():
                        if keyword in content:
                            chat.SendMsg(reply)
                            logging.info(f"回复 {keyword} 消息 - 群聊: {chat.who}, 消息ID: {msg.id}")
                            break
                    if content == "[文件]":  # 检查消息内容是否为 [文件]
                        try:
                            # 发送分析结果作为回复
                            new_result_event.wait()  # 等待新结果
                            chat.SendMsg(analysis_result_content)
                            logging.info(f"回复 [文件] 消息 - 群聊: {chat.who}, 消息ID: {msg.id}")
                            new_result_event.clear()  # 清除事件
                        except Exception as e:
                            logging.error(f"发送回复消息时出错: {e}")

                    replied_messages[msg_identifier] = current_time

                time.sleep(wait)
        except Exception as e:
            logging.error(f"发生异常: {e}")
            retries += 1
            time.sleep(retry_delay)  # 延迟一段时间后重试
        finally:
            pythoncom.CoUninitialize()  # 确保在结束时反初始化


class MonitorApp:
    def __init__(self, master):
        self.master = master
        self.master.title("wxdump机器人")
        self.master.geometry("500x550")

        self.label_group = tk.Label(master, text="目标群名称:")
        self.label_group.pack(pady=10)

        self.group_entry = tk.Entry(master, width=50)
        self.group_entry.pack(pady=10)

        self.label = tk.Label(master, text="监控目录:")
        self.label.pack(pady=10)

        self.directory_entry = tk.Entry(master, width=50)
        self.directory_entry.pack(pady=10)

        self.keyword_frame = tk.Frame(master)
        self.keyword_frame.pack(pady=10)

        self.add_keyword_button = tk.Button(master, text="添加关键词", command=self.add_keyword_field)
        self.add_keyword_button.pack(pady=10)

        self.keyword_fields = []

        self.start_button = tk.Button(master, text="开始监控", command=self.start_monitoring)
        self.start_button.pack(pady=10)

        self.stop_button = tk.Button(master, text="停止监控", command=self.stop_monitoring)
        self.stop_button.pack(pady=10)

        self.output_text = scrolledtext.ScrolledText(master, width=60, height=15)
        self.output_text.pack(pady=10)

        self.observer = None
        self.is_running = False

    def add_keyword_field(self):
        frame = tk.Frame(self.keyword_frame)
        frame.pack(pady=5)

        keyword_label = tk.Label(frame, text="关键词:")
        keyword_label.pack(side=tk.LEFT)
        keyword_entry = tk.Entry(frame, width=20)
        keyword_entry.pack(side=tk.LEFT)

        reply_label = tk.Label(frame, text="回复:")
        reply_label.pack(side=tk.LEFT)
        reply_entry = tk.Entry(frame, width=20)
        reply_entry.pack(side=tk.LEFT)

        self.keyword_fields.append((keyword_entry, reply_entry))

    def start_monitoring(self):
        group_name = self.group_entry.get().strip()
        base_directory = self.directory_entry.get().strip()
        if not base_directory:
            messagebox.showerror("错误", "请输入有效的基础目录路径！")
            return

        # 获取当前年月
        current_year_month = datetime.now().strftime("%Y-%m")
        directory = os.path.join(base_directory, current_year_month)
        if not os.path.isdir(directory):
            messagebox.showerror("错误", "请确保目录存在或输入有效的基础目录路径！")
            return

        keyword_reply_dict = {}
        for keyword_entry, reply_entry in self.keyword_fields:
            keyword = keyword_entry.get().strip()
            reply = reply_entry.get().strip()
            if keyword and reply:
                keyword_reply_dict[keyword] = reply

        self.is_running = True
        self.output_text.insert(tk.END, f'开始监控目录: {directory}\n')
        self.output_text.yview(tk.END)

        event_handler = DumpFileHandler(self.output_text, lambda filepath: process_file(filepath, self.output_text, group_name))
        self.observer = Observer()
        self.observer.schedule(event_handler, directory, recursive=False)
        self.observer.start()
        threading.Thread(target=self.run_observer, daemon=True).start()

        # 启动微信监听线程
        threading.Thread(target=start_wechat_listener, args=(group_name, keyword_reply_dict), daemon=True).start()

    def run_observer(self):
        try:
            while self.is_running:
                time.sleep(1)
        except Exception as e:
            self.output_text.insert(tk.END, f'监控出错: {str(e)}\n')
            self.output_text.yview(tk.END)

    def stop_monitoring(self):
        self.is_running = False
        if self.observer:
            self.observer.stop()
            self.observer.join()
        self.output_text.insert(tk.END, '监控已停止。\n')
        self.output_text.yview(tk.END)


def send_to_wechat(message, target_group_name):
    pythoncom.CoInitialize()  # 确保在使用 COM 之前初始化
    wechat = wxauto.WeChat()
    session_list = wechat.GetSessionList()
    target_group_session = None
    for session in session_list:
        if target_group_name in session:
            target_group_session = session
            break
    if target_group_session:
        wechat.ChatWith(target_group_session)
        wechat.SendMsg(message)
    pythoncom.CoUninitialize()  # 确保在结束时反初始化


if __name__ == "__main__":
    root = tk.Tk()
    app = MonitorApp(root)
    root.mainloop()
```
