from flask import Flask, request, render_template_string, jsonify
import requests
import time
import threading
from datetime import datetime

# ===================== 【配置区】你只需要改这里 =====================
SEND_KEY = "tCW5yr5"  # 改成你自己的
CHECK_INTERVAL = 600  # 检查间隔：600秒 = 10分钟
TARGET_PLAY = 700     # 报警阈值：播放量>1000触发
# ==================================================================

app = Flask(__name__)

# 监控列表格式：{"BV号": {"title": "标题", "pub_time": "发布时间", "play": 播放量, "alerted": 是否报警}}
monitor_list = {}
run_logs = []

def add_log(msg):
    global run_logs
    time_str = datetime.now().strftime("%H:%M:%S")
    run_logs.append(f"[{time_str}] {msg}")
    if len(run_logs) > 30:
        run_logs.pop(0)
# /home/xiari/mysite
# 1. 获取B站视频信息（标题、发布时间、播放量）
def get_video_info(bv):
    try:
        url = f"https://api.bilibili.com/x/web-interface/view?bvid={bv}"
        headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"}
        res = requests.get(url, headers=headers, timeout=10).json()
        # , proxies={"https": None, "http": None}
        if res["code"] == 0:
            data = res["data"]
            title = data["title"][:5]
            play = data["stat"]["view"]
            # 转换发布时间
            pub_timestamp = data["pubdate"]
            pub_time = datetime.fromtimestamp(pub_timestamp).strftime("%Y-%m-%d %H:%M:%S")
            return {"title": title, "play": play, "pub_time": pub_time}
        return None
    except Exception as e:
        print(f"获取{bv}信息失败：{e}")
        add_log(f"获取{bv}信息失败：{e}")
        return None


# 2. 发送微信报警（带标题+发布时间+播放量）
def send_wechat_alert(bv, info):
    api = f"https://miaotixing.com/trigger"

    content = f"""
    触发报警视频：
    标题：{info['title']}
    发布时间：{info['pub_time']}
    当前播放量：{info['play']}
    """

    try:
        requests.post(
            api,
            data={
                "id": SEND_KEY,   # 这里绝对不能加大括号！
                "text": content
            },
            timeout=10
            ,verify=False
        )
        print(f"[{bv}] 喵提醒发送成功")
        add_log(f"[{bv}] 喵提醒发送成功")
    except Exception as e:
        # 打印真实错误，方便排查
        print(f"[{bv}] 喵提醒发送失败，原因：{str(e)}")
        add_log(f"[{bv}] 喵提醒发送失败，原因：{str(e)}")


# 3. 定时监控任务（多视频循环检查）
# def monitor_task():
#     global monitor_list
#     print("✅ 监控线程启动成功，每10分钟检查一次...")

#     while True:
#         try:
#             # 1. 复制一份独立列表，完全脱离原字典
#             current_bvs = list(monitor_list.keys())

#             if current_bvs:
#                 print("\n===== 开始检查视频 =====")

#             for bv in current_bvs:
#                 try:
#                     # 2. 第二道锁：再次确认存在（必须加）
#                     if bv not in monitor_list:
#                         continue

#                     # 3. 第三道锁：一次性取出数据
#                     item = monitor_list.get(bv, None)
#                     if not item:
#                         continue

#                     if item.get("alerted", False):
#                         continue

#                     # 获取信息
#                     info = get_video_info(bv)
#                     if not info:
#                         continue

#                     # 4. 第四道锁：更新前再检查一次
#                     if bv not in monitor_list:
#                         continue

#                     # 更新数据
#                     monitor_list[bv]["title"] = info["title"]
#                     monitor_list[bv]["pub_time"] = info["pub_time"]
#                     monitor_list[bv]["play"] = info["play"]

#                     # 报警判断
#                     if info["play"] > TARGET_PLAY:
#                         send_wechat_alert(bv, info)
#                         monitor_list[bv]["alerted"] = True

#                     print(f"BV{bv} | 播放量：{info['play']} | 报警：{monitor_list[bv]['alerted']}")

#                 except KeyError:
#                     # 单条视频 KeyError 直接跳过
#                     continue

#         except Exception as e:
#             print(f"监控循环异常：{str(e)}")

#         time.sleep(CHECK_INTERVAL)

# 自动检查接口（免费版核心）
@app.route("/api/check")
def api_check():
    for bv in list(monitor_list.keys()):
        try:
            if monitor_list[bv]["alerted"]:
                continue

            info = get_video_info(bv)
            if not info:
                continue

            monitor_list[bv]["title"] = info["title"]
            monitor_list[bv]["pub_time"] = info["pub_time"]
            monitor_list[bv]["play"] = info["play"]
            add_log(f"检查 {bv} 播放量={info['play']}")
            if info["play"] > TARGET_PLAY:
                add_log(f"达到阈值，开始推送 {bv}")
                send_wechat_alert(bv, info)
                monitor_list[bv]["alerted"] = True
        except:
            continue
    return jsonify({"status": "ok"})

# ===================== 接口：获取最新状态 =====================
@app.route("/api/data")
def api_data():
    return jsonify(monitor_list)

# 屏蔽 Chrome 浏览器无用请求，消除 404 报错日志
@app.route('/.well-known/<path:path>')
def well_known(path):
    return '', 204

# ===================== 网页路由 =====================
@app.route("/", methods=["GET", "POST"])
def index():
    global monitor_list
    msg = ""

    # 1. 添加 BV
    if request.form.get("action") == "add":
        bv_input = request.form.get("bv_id").strip()
        bv_list = [bv.strip() for bv in bv_input.splitlines() if bv.strip().startswith("BV")]
        for bv in bv_list:
            if bv not in monitor_list:
                info = get_video_info(bv)
                if info:
                    monitor_list[bv] = {
                        "title": info["title"],
                        "pub_time": info["pub_time"],
                        "play": info["play"],
                        "alerted": False
                    }
                    msg += f"✅ 添加成功：{bv}<br>"
                    add_log(f"添加成功：{bv}<br>")
                else:
                    msg += f"❌ 添加失败：{bv}<br>"
                    add_log(f"添加失败：{bv}<br>")
            else:
                msg += f"⚠️ 已存在：{bv}<br>"
                add_log(f"已存在：{bv}<br>")

    # 2. 删除 BV
    elif request.form.get("action") == "delete":
        bv = request.form.get("bv")
        if bv in monitor_list:
            del monitor_list[bv]
            msg = f"✅ 删除成功：{bv}"
            add_log(f"删除成功：{bv}")


    # # 3. 清空全部
    # elif request.form.get("action") == "clear":
    #     monitor_list.clear()
    #     msg = "✅ 已清空所有视频"

    # 4. 刷新信息
    elif request.form.get("action") == "refresh":
        bv = request.form.get("bv")
        if bv in monitor_list:
            info = get_video_info(bv)
            if info:
                monitor_list[bv]["title"] = info["title"]
                monitor_list[bv]["pub_time"] = info["pub_time"]
                monitor_list[bv]["play"] = info["play"]
                msg = f"✅ 已刷新：{bv}"
                add_log(f"已刷新：{bv}")


    # ===================== ✅ 核心新增：手动切换报警状态 =====================
    elif request.form.get("action") == "toggle_alert":
        bv = request.form.get("bv")
        if bv in monitor_list:
            # 取反：True ↔ False
            monitor_list[bv]["alerted"] = not monitor_list[bv]["alerted"]
            status = "已报警" if monitor_list[bv]["alerted"] else "未报警"
            msg = f"✅ 已切换 {bv} 报警状态 → {status}"
            add_log(f"已切换 {bv} 报警状态 → {status}")

    # 搜索功能
    keyword = request.args.get("search", "").lower()
    filtered = {}
    for bv, item in monitor_list.items():
        if (keyword in bv.lower() or
            keyword in item["title"].lower() or
            keyword in item["pub_time"].lower()):
            filtered[bv] = item

    # 网页模板
    html = """<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>B站监控</title>
<style>
    *{margin:0;padding:0;box-sizing:border-box}
    body{font-family:-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica Neue,Arial,sans-serif;background:#f5f7fa;padding:12px;max-width:500px;margin:0 auto}
    .title{text-align:center;color:#00a1d6;margin:10px 0 12px;font-size:20px}
    .tip{text-align:center;font-size:13px;color:#666;margin-bottom:12px}
    .search{display:flex;gap:6px;margin-bottom:12px}
    .search input{flex:7;padding:10px;border-radius:10px;border:1px solid #e4e7ed}
    .search button{flex:2;padding:10px 0;border-radius:10px;background:#00a1d6;color:white;border:none;font-size:13px}
    .search .clear{background:#888}
    .card{background:white;border-radius:14px;padding:14px;margin-bottom:10px;box-shadow:0 2px 8px rgba(0,0,0,0.04)}
    textarea{width:100%;height:100px;border-radius:12px;border:1px solid #eee;padding:12px;margin-bottom:10px}
    .btn{width:100%;background:#00a1d6;color:white;padding:12px;border-radius:12px;border:none;font-weight:bold;margin-bottom:8px}
    .btn-gray{background:#888 !important; color:white; border:none; border-radius:10px; padding:10px 0; font-size:13px; flex:1.5;}
    .msg{color:#07c160;text-align:center;margin:10px 0;font-size:14px}
    .item{font-size:14px;margin-bottom:5px}
    .label{font-weight:bold}
    .status{padding:2px 6px;border-radius:10px;font-size:12px;color:white}
    .on{background:#ff4d4f}
    .off{background:#07c160}
    .btns{display:flex;gap:6px;margin-top:10px}
    .btns button{flex:1;padding:8px;border:none;border-radius:8px;font-size:12px}
    .orange{background:#ff9500;color:white}
    .gray{background:#888;color:white}
    .red{background:#ff4d4f;color:white}
    .empty{text-align:center;color:#999;padding:20px}
    .log{background:#111;color:#0f0;padding:10px;border-radius:10px;margin-top:20px;font-size:12px}
</style>
</head>
<body>
<h2 class="title">📺 B站多视频监控</h2>
<p class="tip">阈值 ≥ {{TARGET_PLAY}} | 每10分钟检查</p>

<form method="get" class="search">
    <input name="search" placeholder="搜索BV号/标题/时间" value="{{keyword}}">
    <button type="submit">搜索</button>
    <button type="button" class="clear" onclick="document.querySelector('.search input').value=''">清空</button>
</form>

<div class="card">
<form method="post">
<input type="hidden" name="action" value="add">
<textarea name="bv_id" placeholder="每行一个BV号"></textarea>
<button class="btn">✅ 批量添加</button>
</form>
<button type="button" class="btn-gray" style="width:100%" onclick="document.getElementById('bvInput').value=''">
    🗑️ 清空输入框内容
</button>

{% if msg %}<div class="msg">{{msg|safe}}</div>{% endif %}

<h3 style="margin:15px 0 8px;font-size:16px">📋 监控列表（{{filtered|length}}）</h3>

<div id="video-list">
{% for bv,item in filtered.items() %}
<div class="card" data-bv="{{ bv }}">
<div class="item"><span class="label">BV：</span>{{bv}}</div>
<div class="item"><span class="label">标题：</span><span class="title-text">{{item.title}}</span></div>
<div class="item"><span class="label">发布：</span><span class="pub-text">{{item.pub_time}}</span></div>
<div class="item"><span class="label">播放：</span><span class="play-text">{{item.play}}</span></div>
<div class="item">
<span class="label">报警：</span>
<span class="status {% if item.alerted %}on{% else %}off{% endif %} status-text">
{{'已报警' if item.alerted else '未报警'}}
</span>
</div>
<div class="btns">
<form method="post" style="flex:1"><input type="hidden" name="action" value="refresh"><input type="hidden" name="bv" value="{{bv}}"><button class="gray">刷新</button></form>
<form method="post" style="flex:1"><input type="hidden" name="action" value="toggle_alert"><input type="hidden" name="bv" value="{{bv}}"><button class="orange toggle-btn">{{'取消报警' if item.alerted else '设为报警'}}</button></form>
<form method="post" style="flex:1"><input type="hidden" name="action" value="delete"><input type="hidden" name="bv" value="{{bv}}"><button class="red">删除</button></form>
</div>
</div>
{% else %}
<div class="empty">暂无监控视频</div>
{% endfor %}
</div>

<!-- ===================== ✅ 自动同步最新状态 ===================== -->
<script>
// 自动检查监控
async function autoCheck() {
    await fetch("/api/check");
    await syncStatus();
}

// 同步状态
async function syncStatus() {
    const res = await fetch("/api/data");
    const data = await res.json();
    document.querySelectorAll(".card[data-bv]").forEach(el => {
        const bv = el.dataset.bv;
        const item = data[bv];
        if (!item) return;
        el.querySelector(".title-text").innerText = item.title;
        el.querySelector(".pub-text").innerText = item.pub_time;
        el.querySelector(".play-text").innerText = item.play;
        const statusEl = el.querySelector(".status-text");
        const btnEl = el.querySelector(".toggle-btn");
        if (item.alerted) {
            statusEl.className = "status on status-text";
            statusEl.innerText = "已报警";
            btnEl.innerText = "取消报警";
        } else {
            statusEl.className = "status off status-text";
            statusEl.innerText = "未报警";
            btnEl.innerText = "设为报警";
        }
    });
}

// 每30秒自动检查一次
setInterval(autoCheck, 10000);
</script>
</body>
</html>
"""
    return render_template_string(
        html,
        TARGET_PLAY=TARGET_PLAY,
        msg=msg,
        filtered=filtered,
        keyword=keyword
    )


# 启动程序
if __name__ == "__main__":
    # threading.Thread(target=monitor_task, daemon=True).start()
    app.run(host="0.0.0.0", port=5000)  #, debug=False, use_reloader=False
