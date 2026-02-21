## 又来折腾了
这次是利用cf赛博活佛和TG搭建m3u8源 ，具体弄什么资源这个都可以  
### Cloudflare Worker 脚本
```
const TOKEN = ''; // 你的机器人Token

async function handleRequest(request) {
  const url = new URL(request.url);
  const fileId = url.searchParams.get('id');

  // 如果没有 fileId，返回提示
  if (!fileId) {
    return new Response('Missing File ID', { status: 400 });
  }

  // 1. 获取 Telegram 文件信息（获取 file_path）
  const getFileUrl = `https://api.telegram.org/bot${TOKEN}/getFile?file_id=${fileId}`;
  const fileRes = await fetch(getFileUrl);
  const fileData = await fileRes.json();

  if (!fileData.ok) {
    return new Response('File Not Found in Telegram', { status: 404 });
  }

  const filePath = fileData.result.file_path;
  const downloadUrl = `https://api.telegram.org/file/bot${TOKEN}/${filePath}`;

  // 2. 向 Telegram 请求真实文件数据
  const response = await fetch(downloadUrl);

  // 3. 构建新的响应头（允许跨域，方便播放器调用）
  const newHeaders = new Headers(response.headers);
  newHeaders.set('Access-Control-Allow-Origin', '*');
  newHeaders.set('Access-Control-Allow-Methods', 'GET, HEAD, OPTIONS');
  
  // 如果是 m3u8 请求，确保 Content-Type 正确
  if (url.pathname.includes('m3u8')) {
    newHeaders.set('Content-Type', 'application/vnd.apple.mpegurl');
  } else {
    newHeaders.set('Content-Type', 'video/mp2t'); // TS 切片的标准类型
  }

  // 4. 返回流式数据给播放器
  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders,
  });
}

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});
```
### py代码
与之前提供的 Cloudflare Worker 脚本是一套完整的方案：Python 负责“加工并上传”视频到 Telegram，Worker 负责“中转并播放”视频。

它的核心逻辑是将大的 MP4 视频切成很多个小的 TS 片段，这样就可以绕过 Telegram Bot API 单个文件 20MB 的下载限制。
```
import os
import subprocess
import asyncio
import shutil
from telegram import Bot
from telegram.error import RetryAfter

# --- 基础配置 (务必确认以下信息) ---
TOKEN = ' '
CHAT_ID = '-'  # 你的频道ID

# FFmpeg 路径 (请确保 ffmpeg.exe 确实在这个位置)
FFMPEG_EXE = r'D:\ffmpeg\bin\ffmpeg.exe'

# 文件夹配置
WATCH_DIR = r'D:\vdo\new'      # 待处理文件夹 (丢入MP4)
PROCESSED_DIR = r'D:\vdo\done' # 处理完后的归档文件夹
# 播放列表文件路径 (后缀改为 .m3u 方便播放器识别)
PLAYLIST_FILE = r'D:\vdo\playlist.m3u' 

# 你的 Cloudflare Worker 域名 (务必换成真实的)
WORKER_URL = "https://edc8.de5.net" 

bot = Bot(token=TOKEN)

async def process_video(file_path):
    file_name = os.path.basename(file_path)
    base_name = os.path.splitext(file_name)[0]
    # 创建临时切片目录
    temp_hls_dir = os.path.join(os.path.dirname(file_path), f"temp_{base_name}")
    
    if not os.path.exists(temp_hls_dir): 
        os.makedirs(temp_hls_dir)

    print(f"\n🎬 正在切片: {base_name}")
    
    # --- 极致播放优化参数 ---
    # -hls_time 30: 30秒一档，平衡速度与文件数
    # independent_segments: 保证每个切片独立，拖动进度条不转圈
    cmd = [
        FFMPEG_EXE, '-y', '-i', file_path,
        '-c', 'copy', 
        '-map', '0', '-f', 'hls',
        '-hls_time', '30', 
        '-hls_flags', 'independent_segments',
        '-hls_list_size', '0',
        '-hls_segment_filename', os.path.join(temp_hls_dir, 'seg%d.ts'),
        os.path.join(temp_hls_dir, 'index.m3u8')
    ]
    
    process = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    if process.returncode != 0:
        print(f"❌ FFmpeg 报错: {process.stderr.decode('utf-8')}")
        return

    # 获取分片并按顺序排序
    ts_files = sorted([f for f in os.listdir(temp_hls_dir) if f.endswith('.ts')],  
                      key=lambda x: int(x.replace('seg','').replace('.ts','')))
    
    ts_map = {}
    print(f"📦 正在上传 {len(ts_files)} 个分片 (反限速机制已开启)...")
    
    for ts in ts_files:
        ts_path = os.path.join(temp_hls_dir, ts)
        success = False
        while not success:
            try:
                with open(ts_path, 'rb') as f:
                    msg = await bot.send_document(CHAT_ID, document=f)
                    ts_map[ts] = msg.document.file_id
                    print(f"  ✅ {ts} 已上传")
                    success = True
                    # 稳定上传间隔，避开TG高频检测
                    await asyncio.sleep(1.8) 
            except RetryAfter as e:
                wait_time = e.retry_after + 5
                print(f"⚠️ 触发TG限速，强制休息 {wait_time} 秒...")
                await asyncio.sleep(wait_time)
            except Exception as e:
                print(f"❌ 上传失败: {e}，5秒后重试...")
                await asyncio.sleep(5)
    
    # 替换 m3u8 中的本地文件名为 Worker 链接格式
    m3u8_path = os.path.join(temp_hls_dir, 'index.m3u8')
    with open(m3u8_path, 'r') as f:
        m3u8_content = f.read()
    
    for ts_name, fid in ts_map.items():
        m3u8_content = m3u8_content.replace(ts_name, f"ts?id={fid}")

    # 生成最终的 m3u8 文件并备份到频道
    final_m3u8_path = os.path.join(os.getcwd(), f"{base_name}.m3u8")
    with open(final_m3u8_path, 'w', encoding='utf-8') as f:
        f.write(m3u8_content)
    
    with open(final_m3u8_path, 'rb') as f:
        final_msg = await bot.send_document(CHAT_ID, document=f, caption=f"🎬 {base_name}")
    
    # --- 写入标准 M3U 播放列表文件 ---
    video_url = f"{WORKER_URL}/m3u8?id={final_msg.document.file_id}"
    file_exists = os.path.exists(PLAYLIST_FILE)
    
    with open(PLAYLIST_FILE, 'a', encoding='utf-8') as f:
        if not file_exists or os.path.getsize(PLAYLIST_FILE) == 0:
            f.write("#EXTM3U\n")
        f.write(f"#EXTINF:-1, {base_name}\n")
        f.write(f"{video_url}\n")
    
    print(f"⭐ 处理完成！已加入列表: {PLAYLIST_FILE}")
    
    # --- 清理现场 ---
    try:
        shutil.rmtree(temp_hls_dir)
        if os.path.exists(final_m3u8_path): os.remove(final_m3u8_path)
        
        target_path = os.path.join(PROCESSED_DIR, file_name)
        if os.path.exists(target_path): os.remove(target_path)
        shutil.move(file_path, target_path)
        print(f"🚚 文件已移至归档目录: {PROCESSED_DIR}")
    except Exception as e:
        print(f"🧹 清理文件时出现小提示: {e}")

async def main():
    # 环境检查
    if not os.path.exists(FFMPEG_EXE):
        print(f"❌ 找不到 FFmpeg 程序！请检查路径: {FFMPEG_EXE}")
        return

    # 自动创建目录
    for p in [WATCH_DIR, PROCESSED_DIR]:
        if not os.path.exists(p): os.makedirs(p)

    print("========================================")
    print("      🌟 你的私人影院后台已启动 🌟")
    print(f" 监控路径: {WATCH_DIR}")
    print(f" 播放列表: {PLAYLIST_FILE}")
    print("========================================\n")
    
    while True:
        try:
            # 扫描待处理的 MP4
            files = [f for f in os.listdir(WATCH_DIR) if f.lower().endswith('.mp4')]
            for file in files:
                full_path = os.path.join(WATCH_DIR, file)
                # 等待文件写入完成 (检测大小变化)
                size_1 = os.path.getsize(full_path)
                await asyncio.sleep(3)
                if size_1 == os.path.getsize(full_path):
                    await process_video(full_path)
        except Exception as e:
            print(f"❌ 循环任务报错: {e}")
        await asyncio.sleep(5)

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n👋 影院工厂已关闭。")

```
