日期：2025-01-21
## 折腾来折腾去的发现还是写这个版本更好用
### 取消关键词自动回复，别问为什么，不够稳定，需要时刻的监控wx的聊天记录 。
### 更新日期文本框 使得监控的文件路径 可以根据 日期文本框变化， 这样就不用每月手动更新路径。
### 增加开启和关闭提示消息！
### 贴出最终版的python代码备份,以备不时之需。
```python 
import os
import time
import tkinter as tk
from tkinter import scrolledtext, messagebox, Entry
import requests
import json
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import threading
import pyautogui
import pyperclip
import subprocess
import wxauto
import win32clipboard
import pythoncom
import traceback
from datetime import datetime


class DumpFileHandler(FileSystemEventHandler):
    def __init__(self, text_widget, upload_callback):
        self.text_widget = text_widget
        self.upload_callback = upload_callback
        self.event_processed = {}

    def on_created(self, event):
        if event.is_directory:
            return
        file_path = event.src_path
        if file_path.endswith(('.dump', '.bin', '.txt')):
            if file_path not in self.event_processed:
                self.text_widget.insert(tk.END, f'新文件创建: {file_path}\n')
                self.text_widget.yview(tk.END)
                self.upload_callback(filepath=file_path)
                self.event_processed[file_path] = True
            else:
                print(f"已处理过 {file_path} 的创建事件，跳过重复处理")

    def on_modified(self, event):
        if event.is_directory:
            return
        file_path = event.src_path
        if file_path.endswith(('.dump', '.bin', '.txt')):
            if file_path not in self.event_processed:
                self.text_widget.insert(tk.END, f'文件修改: {file_path}\n')
                self.text_widget.yview(tk.END)
                self.upload_callback(filepath=file_path)
                self.event_processed[file_path] = True
            else:
                print(f"已处理过 {file_path} 的修改事件，跳过重复处理")

    def on_deleted(self, event):
        if event.is_directory:
            return
        file_path = event.src_path
        if file_path.endswith(('.dump', '.bin', '.txt')):
            if file_path not in self.event_processed:
                self.text_widget.insert(tk.END, f'文件删除: {file_path}\n')
                self.text_widget.yview(tk.END)
                self.upload_callback(filepath=file_path)
                self.event_processed[file_path] = True
            else:
                print(f"已处理过 {file_path} 的删除事件，跳过重复处理")


def upload(filepath: str) -> str:
    url = "http://47.98.235.44:5005/upload"
    files = [
        ('upload_dump', (os.path.basename(filepath), open(filepath, 'rb'), 'application/octet-stream'))
    ]
    try:
        response = requests.post(url, files=files)
        if response.status_code!= 200:
            raise Exception(f"Error uploading file: Upload failed with status code {response.status_code}")
        return response.text
    except Exception as e:
        raise Exception(f"Error uploading file: {str(e)}")


def analysis(uuid: str) -> str:
    url = "http://47.98.235.44:5005/analysis"
    payload = json.dumps({"uuid": uuid})
    headers = {
        'Content-Type': 'application/json'
    }
    response = requests.post(url, headers=headers, data=payload)
    return response.text


def copy_to_clipboard(output_text: tk.Text):
    try:
        root.clipboard_clear()
        result = output_text.get(1.0, tk.END).strip()
        root.clipboard_append(result)
        output_text.insert(tk.END, "分析结果已复制到剪贴板\n")
    except Exception as e:
        output_text.insert(tk.END, f"错误: 复制到剪贴板时发生错误: {str(e)}\n")


def send_to_wechat(message: str, output_text: tk.Text, target_group_name):
    try:
        pythoncom.CoInitialize()
        win32clipboard.OpenClipboard()
        win32clipboard.EmptyClipboard()
        win32clipboard.SetClipboardText(message)
        win32clipboard.CloseClipboard()
        wechat = wxauto.WeChat()
        session_list = wechat.GetSessionList()
        target_session = None
        for session in session_list:
            if target_group_name in session:
                target_session = session
                break
        if target_session:
            wechat.ChatWith(target_session)
            wechat.SendMsg(message)
            output_text.insert(tk.END, "消息已发送到微信窗口\n")
            pythoncom.CoUninitialize()
            return True
        else:
            output_text.insert(tk.END, f"错误: 未找到群聊 {target_group_name}\n")
            pythoncom.CoUninitialize()
            return False
    except Exception as e:
        output_text.insert(tk.END, f"错误: 发送消息时发生错误: {str(e)}\n")
        traceback.print_exc()
        return False


def process_file(filepath: str, output_text: tk.Text, target_group_name):
    should_send_message = False
    message_sent = False
    try:
        try:
            result = upload(filepath)
        except Exception as e:
            output_text.delete(1.0, tk.END)
            output_text.insert(tk.END, f"错误: 发生错误: {str(e)}\n")
            error_message = "分析接口目前被攻击稍后再试！"
            send_to_wechat(message=error_message, output_text=output_text, target_group_name=target_group_name)
            return
        response_data = json.loads(result)
        uuid = response_data.get("uuid")
        if uuid:
            analysis_result = analysis(uuid)
            analysis_data = json.loads(analysis_result)
            if analysis_data.get("code") == 200:
                data = analysis_data.get("data", [])
                if data and data[0]:
                    messages = []
                    for item in data[0]:
                        if isinstance(item, dict):
                            name = item.get("name", "")
                            values = item.get("value", [])
                            for value in values:
                                if isinstance(value, dict):
                                    b = value.get("b", "")
                                    messages.append(f"{name}: {b}")
                    if messages:
                        result_message = "\n".join(messages)
                        output_text.delete(1.0, tk.END)
                        output_text.insert(tk.END, f"解析成功，返回数据:\n{result_message}\n")
                        copy_to_clipboard(output_text)
                        should_send_message = True
                else:
                    warning_message = "该数据暂未收录"
                    output_text.delete(1.0, tk.END)
                    output_text.insert(tk.END, warning_message + "\n")
                    copy_to_clipboard(output_text)
                    should_send_message = True
            else:
                warning_message = f"警告: 解析失败: {analysis_data.get('message')}\n"
                output_text.delete(1.0, tk.END)
                output_text.insert(tk.END, warning_message)
                copy_to_clipboard(output_text)
                should_send_message = True
        else:
            warning_message = "警告: 未能获取UUID\n"
            output_text.delete(1.0, tk.END)
            output_text.insert(tk.END, warning_message)
            copy_to_clipboard(output_text)
            should_send_message = True

        if should_send_message and not message_sent:
            output_text.delete("1.0", tk.END)
            if'result_message' in locals():
                message_sent = send_to_wechat(result_message, output_text, target_group_name)
            else:
                message_sent = send_to_wechat(warning_message, output_text, target_group_name)
            if message_sent:
                time.sleep(3)
                try:
                    command = f'del /f /q "{filepath}"'
                    subprocess.run(command, shell=True, check=True)
                    output_text.insert(tk.END, f"文件 {filepath} 已成功删除\n")
                except subprocess.CalledProcessError as e:
                    output_text.insert(tk.END, f"错误: 删除文件 {filepath} 时出错: {str(e)}\n")
                except Exception as e:
                    output_text.insert(tk.END, f"其他错误: {str(e)}\n")
    except json.JSONDecodeError:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, "错误: 返回数据格式错误，无法解析JSON\n")
    except Exception as e:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, f"错误: 发生错误: {str(e)}\n")


class MonitorApp:
    def __init__(self, master):
        self.master = master
        self.master.title("微信机器人")
        self.master.geometry("500x450")

        self.label = tk.Label(master, text="基础监控目录:")
        self.label.pack(pady=10)

        self.directory_entry = tk.Entry(master, width=50)
        self.directory_entry.pack(pady=10)

        self.group_label = tk.Label(master, text="目标群名:")
        self.group_label.pack(pady=10)

        self.group_entry = tk.Entry(master, width=50, justify=tk.CENTER)
        self.group_entry.pack(pady=10)

        self.date_label = tk.Label(master, text="当前日期:")
        self.date_label.pack(pady=10)

        self.date_entry = tk.Entry(master, width=20, justify=tk.CENTER)
        self.date_entry.pack(pady=10)
        self.update_date_display()

        self.start_button = tk.Button(master, text="开始监控", command=self.start_monitoring)
        self.start_button.pack(pady=10)

        self.stop_button = tk.Button(master, text="停止监控", command=self.stop_monitoring)
        self.stop_button.pack(pady=10)

        self.output_text = scrolledtext.ScrolledText(master, width=60, height=15)
        self.output_text.pack(pady=10)

        self.observer = None
        self.is_running = False

    def update_date_display(self):
        current_date = datetime.now().strftime("%Y-%m")
        self.date_entry.delete(0, tk.END)
        self.date_entry.insert(0, current_date)
        self.master.after(86400000, self.update_date_display)

    def start_monitoring(self):
        base_directory = self.directory_entry.get().strip()
        date = self.date_entry.get().strip()
        if not os.path.isdir(base_directory):
            messagebox.showerror("错误", "请输入有效的基础目录路径！")
            return
        if not date:
            messagebox.showerror("错误", "请确保日期文本框有值！")
            return
        directory = os.path.join(base_directory, date)
        if not os.path.isdir(directory):
            try:
                os.makedirs(directory)
            except Exception as e:
                messagebox.showerror("错误", f"创建目录 {directory} 失败: {str(e)}")
                return
        target_group_name = self.group_entry.get().strip()
        if not target_group_name:
            messagebox.showerror("错误", "请输入有效的群名！")
            return

        self.is_running = True
        self.output_text.insert(tk.END, "分析机器人已开启！\n")
        self.output_text.yview(tk.END)
        send_to_wechat("分析机器人已开启！", self.output_text, target_group_name)
        self.output_text.insert(tk.END, f'开始监控目录: {directory}\n')
        self.output_text.yview(tk.END)

        event_handler = DumpFileHandler(self.output_text, lambda filepath: process_file(filepath, self.output_text, target_group_name))
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
        # 先在输出文本框插入信息
        self.output_text.insert(tk.END, "分析机器人已关闭！\n")
        self.output_text.yview(tk.END)
        # 发送信息到微信窗口
        target_group_name = self.group_entry.get().strip()
        send_to_wechat("分析机器人已关闭！", self.output_text, target_group_name)
        self.output_text.insert(tk.END, '监控已停止。\n')
        self.output_text.yview(tk.END)


if __name__ == "__main__":
    root = tk.Tk()
    app = MonitorApp(root)
    root.mainloop()

```
