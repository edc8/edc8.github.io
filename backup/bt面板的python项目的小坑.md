## 这几天瞎逛youtube,发个爬取SSR链接的脚本 
### 于是在AI加持下，配合飞书的webhook将自定义服务的消息推送至飞书便有有如下的代码；
```python
import requests
import base64
import json
import pyaes
import binascii
from datetime import datetime


def send_feishu_message(webhook_url, message):
    headers = {
        "Content-Type": "application/json"
    }
    payload = {
        "msg_type": "text",
        "content": {
            "text": message
        }
    }
    response = requests.post(webhook_url, json=payload, headers=headers)
    if response.status_code == 200:
        print("消息发送成功")
    else:
        print(f"消息发送失败，状态码: {response.status_code}，错误信息: {response.text}")


print("      H͜͡E͜͡L͜͡L͜͡O͜͡ ͜͡W͜͡O͜͡R͜͡L͜͡D͜͡ ͜͡E͜͡X͜͡T͜͡R͜͡A͜͡C͜͡ ͜͡S͜͡S͜͡ ͜͡N͜͡O͜͡D͜͡E͜͡")
print("𓆝 𓆟 𓆞 𓆟 𓆝 𓆟 𓆞 𓆟 𓆝 𓆟 𓆞 𓆟")
print("Author : 𝐼𝑢")
print(f"Date   : {datetime.today().strftime('%Y-%m-%d')}")
print("Version: 1.0")
print("𓆝 𓆟 𓆞 𓆟 𓆝 𓆟 𓆞 𓆟 𓆝 𓆟 𓆞 𓆟")
print("𝐼𝑢:")
print(r"""
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⡠⠤⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡴⠁⠀⡰⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⢀⡎⢀⠀⠀⠁⠀⠀⠀⠀⠀⠀⠀
⣀⠀⠀⠀⠀⠀⠀⡠⠬⡡⠬⡋⠀⡄⠀⠀⠀⠀⠀⠀⠀
⡀⠁⠢⡀⠀⠀⢰⠠⢷⠰⠆⡅⠀⡇⠀⠀⠀⣀⠔⠂⡂
⠱⡀⠀⠈⠒⢄⡸⡑⠊⢒⣂⣦⠄⢃⢀⠔⠈⠀⠀⡰⠁
⠀⠱⡀⠀⠀⡰⣁⣼⡿⡿⢿⠃⠠⠚⠁⠀⠀⢀⠜⠀⠀
⠀⠀⠐⢄⠜⠀⠈⠓⠒⠈⠁⠀⠀⠀⠀⠀⡰⠃⠀⠀⠀
⠀⠀⢀⠊⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠾⡀⠀⠀⠀⠀
⠀⠀⢸⣄⠀⠀⡀⠀⠀⠀⠀⠀⠀⠀⠀⣀⡇⠀⠀⠀⠀
⠀⠀⠸⢸⣳⠦⣍⣁⣀⣀⣀⣀⣠⠴⠚⠁⠇⠀⠀⠀⠀
⠀⠀⠀⢳⣿⠄⠸⠢⠍⠉⠉⠀⠀⡠⢒⠎⠀⠀⠀⠀⠀
⠀⠀⠀⠣⣀⠁⠒⡤⠤⢤⠀⠀⠐⠙⡇⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠣⡀⡼⠀⠀⠈⠱⡒⠂⡸⠁⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠑⢒⠁⠀⠀⠀⠀⠀⠀⠀
""")


a = 'http://api.skrapp.net/api/serverlist'
b = {
    'accept': '/',
    'accept-language': 'zh-Hans-CN;q=1, en-CN;q=0.9',
    'appversion': '1.3.1',
    'user-agent': 'SkrKK/1.3.1 (iPhone; iOS 13.5; Scale/2.00)',
    'content-type': 'application/x-www-form-urlencoded',
    'Cookie': 'PHPSESSID=fnffo1ivhvt0ouo6ebqn86a0d4'
}
c = {'data': '4265a9c353cd8624fd2bc7b5d75d2f18b1b5e66ccd37e2dfa628bcb8f73db2f14ba98bc6a1d8d0d1c7ff1ef0823b11264d0addaba2bd6a30bdefe06f4ba994ed'}
d = b'65151f8d966bf596'
e = b'88ca0f0ea1ecf975'


def f(g, d, e):
    h = pyaes.AESModeOfOperationCBC(d, iv=e)
    i = b''.join(h.decrypt(g[j:j+16]) for j in range(0, len(g), 16))
    return i[:-i[-1]]


j = requests.post(a, headers=b, data=c)


if j.status_code == 200:
    k = j.text.strip()
    l = binascii.unhexlify(k)
    m = f(l, d, e)
    n = json.loads(m)
    message_list = []
    for o in n['data']:
        p = f"aes-256-cfb:{o['password']}@{o['ip']}:{o['port']}"
        q = base64.b64encode(p.encode('utf-8')).decode('utf-8')
        r = f"ss://{q}#{o['title']}"
        message_list.append(r)
    # 飞书机器人的 Webhook 地址，需替换为实际地址
    webhook_url = "you Webhook 地址"
    # 将列表中的消息组合成一个字符串
    message = "\n".join(message_list)
    send_feishu_message(webhook_url, message)
```
## 宝塔遇到的坑；
### 打算利用宝塔面板的定时执行，跑这个脚本推送到小飞书群组，结果 和之前了解的不一样了，折腾了几天时间 才无意间解决；下面贴出图片配置。

![](https://preview.cloud.189.cn/image/imageAction?param=612F10B9B4E2F0A77F4DAF50C92DB1AFD117AEA127A5DE1F0A6658D304A3DD62AE209FF53D2D7F77B1EAB6AD2D8D98A2516C5C6DBECEEA5F372C9740BEAD8903FF89DFE87F09FC0148EEEBCEED1EE99092EB133207290B5205D4BAAF263A7548D0A13CB23501FDB438C7BE4298AC54DE)
## 这个没什么好说的；基本设置下面这个才是重点,需要将这设置打开
![](https://preview.cloud.189.cn/image/imageAction?param=F0A37F21588A27DE6CDBFB0280DF9D0C2B661814FE041DAB7042DC6E336DC00FE6BD3036426CFFF24E2C585A13E02788CCAF0442F3BFB1745549ECFC7C8E31395C7D1483C878A9C386CA4764487F7421FAEBA316C7E79906E4537D6615AD1A4813B987DD4B792A9831D9B33529DA9BCB)
## 然后才能实现定时运行该python脚本
![](https://preview.cloud.189.cn/image/imageAction?param=82F27986CBC0E899613D6EE5356D59641396678D0E271F13430743713E1AAAC42253F975565DBC26F4AA52CF07E8FAA03265D88E48D5BD9A323E8905C52E151931D0CAF765858EFCFC9FC11CF60225AAE65B6A433606EA2E052628F942E9997CA39EC8C7869D0CA0439547EA3F432477)
### 题外话 现在让人搬个站 换个系统居然要80快，不过现在没这个精力去折腾网站了；再说吧，等闲下来 把网站后台全部找回备份好说不定就搞！