###  python部分
`import requests
from datetime import datetime
import os

# 币印 API 配置
PUID = " "
COIN_TYPE = "btc"
TOKEN = " "

def fetch():
    url = "https://api-prod.poolin.com/api/public/v2/payment/stats"
    headers = {
        "authorization": f"Bearer {TOKEN}",
        "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X) AppleWebKit/605.1.15"
    }
    params = {"puid": PUID, "coin_type": COIN_TYPE}
    
    try:
        # 增加超时设置防止卡死
        res = requests.get(url, headers=headers, params=params, timeout=15)
        data = res.json()
        if data.get('err_no') == 0:
            d = data.get('data', {})
            y_earn = float(d.get('yesterday_amount', 0)) / 100000000
            balance = d.get('balance', '0.00')
            msg = "数据同步成功"
        else:
            y_earn, balance = 0.0, "0.00"
            msg = f"API报错: {data.get('err_msg')}"
    except Exception as e:
        y_earn, balance = 0.0, "0.00"
        msg = f"连接失败: {str(e)}"

    # 生成 HTML (适配手机，内联 CSS 加速访问)
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    html_content = f"""<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0">
    <title>BTC 收益监控</title>
    <style>
        body {{ background: #0f172a; color: #f8fafc; font-family: -apple-system, sans-serif; margin: 0; display: flex; justify-content: center; align-items: center; min-height: 100vh; }}
        .card {{ background: #1e293b; padding: 2rem; border-radius: 1.5rem; border: 1px solid #334155; width: 90%; max-width: 360px; text-align: center; box-sizing: border-box; box-shadow: 0 20px 25px -5px rgba(0,0,0,0.5); }}
        h2 {{ color: #f59e0b; margin-top: 0; font-size: 1.4rem; }}
        .label {{ color: #94a3b8; font-size: 0.85rem; margin-bottom: 5px; }}
        .val {{ color: #fff; font-size: 1.8rem; font-weight: bold; margin-bottom: 20px; font-family: monospace; }}
        hr {{ border: 0; border-top: 1px solid #334155; margin: 20px 0; }}
        .footer {{ font-size: 0.75rem; color: #64748b; line-height: 1.6; }}
    </style>
</head>
<body>
    <div class="card">
        <h2>BTC 收益监控</h2>
        <div class="label">昨日收益 (BTC)</div>
        <div class="val">{y_earn:.8f}</div>
        <div class="label">当前余额</div>
        <div class="val">{balance}</div>
        <hr>
        <div class="footer">
            状态: {msg}<br>
            更新: {now}<br>
            <span style="color:#475569;">永久在线模式已启动 (20min更新)</span>
        </div>
    </div>
</body>
</html>"""
    
    with open("index.html", "w", encoding="utf-8") as f:
        f.write(html_content)
    print("✅ index.html 写入成功")

if __name__ == "__main__":
    fetch()`
### yml文件部分
`name: BTC-Monitor-Stable-Permanent

on:
  schedule:
    - cron: '*/20 * * * *'  # 每20分钟运行一次，避开风控
  workflow_dispatch:        # 允许手动触发，用于激活和测试

permissions:
  contents: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 1. 检出代码
        uses: actions/checkout@v4

      - name: 2. 设置 Python 环境
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: 3. 安装必要依赖
        run: pip install requests

      - name: 4. 执行更新脚本
        run: python update_data.py

      - name: 5. 部署到 GitHub Pages
        uses: JamesIves/github-pages-deploy-action@v4
        with:
          folder: .
          branch: gh-pages
          single-commit: true

      - name: 6. 自动保活 (防止60天停转)
        uses: gautamkrishnar/keepalive-workflow@v2
        with:
          use_api: true`
