# Triển khai Telegram Bot + Codex CLI + Home Assistant MCP trên Armbian

> **Trạng thái:** Hướng dẫn triển khai nền tảng và bot **chỉ lập kế hoạch**. Chưa bật ghi MCP tự động, backup/rollback qua MCP hoặc trả URL dashboard tự động cho đến khi xác minh danh sách công cụ MCP thực tế.
>
> **Kiến trúc:** Telegram Android → Python Bot (Docker) → Codex CLI (đăng nhập ChatGPT) → Home Assistant MCP → Home Assistant. Không dùng n8n hoặc PC Windows.

## 1. Thông số môi trường

| Thành phần | Giá trị |
|---|---|
| Máy chủ | Armbian ARM64, IP `192.168.1.234` |
| Home Assistant | Docker, `network_mode: host` |
| MCP URL | `http://192.168.1.234/api/mcp` (đúng theo URL đã cung cấp) |
| HA Web URL | Xác minh thực tế: thường là `http://192.168.1.234:8123` |
| Agent | Codex CLI đăng nhập ChatGPT |
| Điều khiển | Telegram Bot (long polling) |
| Bí mật | `.env`, không commit Git |

**Quan trọng:** MCP chính thức của Home Assistant có thể không có công cụ sửa dashboard hoặc automation. Khả năng ghi chỉ được xác nhận sau khi liệt kê công cụ của endpoint. Nếu thiếu, cần thêm MCP quản trị phù hợp hoặc cơ chế chỉnh sửa file có kiểm soát.

## 2. Kiểm tra Docker và mạng

```bash
ssh root@192.168.1.234
uname -m
docker compose version
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
curl -i http://192.168.1.234/api/mcp
```

GET có thể trả `405 Method Not Allowed` với MCP Streamable HTTP; đó chưa phải bằng chứng endpoint lỗi. Không công khai cổng MCP ra Internet.

## 3. Tạo Telegram Bot

1. Mở Telegram, tìm **@BotFather**.
2. Gửi `/newbot`, đặt tên và username kết thúc bằng `bot`.
3. Lưu Bot Token; xác định Telegram **numeric user ID** của tài khoản được phép dùng bot.
4. Không gửi token vào hội thoại công khai.

## 4. Tạo thư mục và quyền

```bash
mkdir -p /opt/ha-ai-agent/{bot,workspace,codex-data,backups,state}
cd /opt/ha-ai-agent
chown -R 10001:10001 workspace codex-data backups state
chmod 700 codex-data backups state
```

Không mount thư mục `/root/hass/config` hoặc Docker socket vào bot trong giai đoạn này.

## 5. Tạo `.env`

```dotenv
TELEGRAM_BOT_TOKEN=REPLACE_ME
TELEGRAM_USER_ID=123456789
HA_MCP_URL=http://192.168.1.234/api/mcp
HA_MCP_TOKEN=REPLACE_ME
HA_URL=http://192.168.1.234:8123
CODEX_HOME=/home/agent/.codex
```

```bash
chmod 600 .env
```

`HA_URL` là địa chỉ giao diện, **không nhất thiết trùng** với `HA_MCP_URL`. Token MCP phải phù hợp với phương thức xác thực endpoint (ví dụ Bearer); không giả định token HA luôn dùng được với mọi MCP proxy.

## 6. Tạo `Dockerfile`

```dockerfile
FROM node:22-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-pip git curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g @openai/codex
RUN pip3 install --break-system-packages python-telegram-bot==22.5
RUN useradd -m -u 10001 agent
RUN mkdir -p /workspace /app/bot /backups /state \
    && chown -R agent:agent /workspace /app /backups /state

ENV HOME=/home/agent
ENV CODEX_HOME=/home/agent/.codex
WORKDIR /workspace
USER agent
CMD ["python3", "/app/bot/main.py"]
```

> Đối với production, khóa phiên bản image và dependency sau khi kiểm thử. Nếu bản `@openai/codex` mới yêu cầu Node cao hơn, điều chỉnh base image tương ứng.

## 7. Tạo `docker-compose.yml`

```yaml
services:
  ha-ai-bot:
    build: .
    container_name: ha-ai-bot
    restart: unless-stopped
    init: true
    env_file:
      - .env
    volumes:
      - ./bot:/app/bot:ro
      - ./workspace:/workspace
      - ./codex-data:/home/agent/.codex
      - ./backups:/backups
      - ./state:/state
    working_dir: /workspace
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    mem_limit: 1g
    pids_limit: 128
```

Build:

```bash
docker compose build
```

## 8. Đăng nhập Codex bằng ChatGPT

```bash
docker compose run --rm --entrypoint codex ha-ai-bot login --device-auth
```

Mở URL được hiển thị trên điện thoại và hoàn tất đăng nhập. Kiểm tra:

```bash
docker compose run --rm --entrypoint codex ha-ai-bot login status
```

Không đưa thông tin đăng nhập, access token hoặc file xác thực Codex vào Git.

## 9. Cấu hình MCP cho Codex

Tạo `codex-data/config.toml`:

```toml
[mcp_servers.homeassistant]
url = "http://192.168.1.234/api/mcp"
bearer_token_env_var = "HA_MCP_TOKEN"
```

Kiểm tra cấu hình và thử truy vấn chỉ đọc:

```bash
docker compose run --rm --entrypoint codex ha-ai-bot mcp list

docker compose run --rm --entrypoint codex ha-ai-bot exec \
  --sandbox read-only --skip-git-repo-check \
  'Liệt kê các công cụ MCP homeassistant có sẵn và mô tả ngắn. Không thực hiện thay đổi.'
```

**Cảnh báo:** `--sandbox read-only` chỉ hạn chế filesystem; **không chặn công cụ MCP có khả năng ghi**. Trước khi dùng với MCP chứa tool ghi, phải giới hạn quyền ở server/proxy hoặc chỉ kết nối đến endpoint chỉ đọc. Lời nhắc “không sửa” không phải biện pháp bảo mật đáng tin cậy.

## 10. Mã bot Telegram bản lập kế hoạch (`bot/main.py`)

Bot dưới đây **không thực thi khi bấm Duyệt**. Đây là bản an toàn để thử Telegram và Codex; chỉ dùng với MCP đã được giới hạn quyền đọc.

```python
import os
import asyncio
import uuid
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler,
    CallbackQueryHandler, filters,
)

TOKEN = os.environ["TELEGRAM_BOT_TOKEN"]
OWNER = int(os.environ["TELEGRAM_USER_ID"])
TASKS = {}

async def run_codex(prompt: str) -> str:
    proc = await asyncio.create_subprocess_exec(
        "codex", "exec", "--sandbox", "read-only",
        "--skip-git-repo-check", prompt,
        cwd="/workspace",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(proc.communicate(), timeout=180)
    except asyncio.TimeoutError:
        proc.kill()
        await proc.communicate()
        raise RuntimeError("Codex timeout")
    if proc.returncode:
        raise RuntimeError(stderr.decode(errors="replace")[-1000:])
    return stdout.decode(errors="replace")[-3000:]

async def start(update: Update, context):
    if update.effective_user and update.effective_user.id == OWNER:
        await update.message.reply_text("HA AI Bot sẵn sàng lập kế hoạch (chưa triển khai).")

async def message(update: Update, context):
    if not update.effective_user or update.effective_user.id != OWNER:
        return
    request = update.message.text
    task_id = uuid.uuid4().hex[:10]
    waiting = await update.message.reply_text("Đang phân tích...")
    prompt = (
        "Bạn là chuyên gia Home Assistant. Chỉ phân tích, không thay đổi hệ thống. "
        "Không gọi công cụ MCP có tác dụng ghi. Nêu đối tượng, bước làm, "
        "rủi ro, kiểm tra và rollback. Yêu cầu: " + request
    )
    try:
        plan = await run_codex(prompt)
        TASKS[task_id] = {"request": request, "plan": plan, "status": "pending"}
        buttons = InlineKeyboardMarkup([[
            InlineKeyboardButton("Duyệt", callback_data=f"approve:{task_id}"),
            InlineKeyboardButton("Từ chối", callback_data=f"reject:{task_id}"),
        ]])
        await waiting.edit_text(
            f"Kế hoạch #{task_id}\n\n{plan}", reply_markup=buttons
        )
    except Exception as exc:
        await waiting.edit_text(f"Không thể phân tích: {str(exc)[:400]}")

async def callback(update: Update, context):
    query = update.callback_query
    await query.answer()
    if not query.from_user or query.from_user.id != OWNER:
        return
    action, task_id = query.data.split(":", 1)
    task = TASKS.get(task_id)
    if not task or task["status"] != "pending":
        await query.edit_message_text("Tác vụ không tồn tại hoặc đã xử lý.")
        return
    if action == "reject":
        task["status"] = "rejected"
        await query.edit_message_text(f"Đã từ chối #{task_id}.")
    elif action == "approve":
        task["status"] = "approved"
        await query.edit_message_text(
            f"Đã ghi nhận phê duyệt #{task_id}. "
            "CHƯA triển khai: thiếu bộ thực thi và rollback đã kiểm thử."
        )

def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, message))
    app.add_handler(CallbackQueryHandler(callback))
    app.run_polling(drop_pending_updates=True)

if __name__ == "__main__":
    main()
```

Chạy:

```bash
docker compose up -d
docker logs -f ha-ai-bot
```

Gửi `/start` và yêu cầu: “Đề xuất dashboard Zigbee tối màu, không thay đổi gì”. Trạng thái `TASKS` chỉ lưu trong RAM; khi restart bot, đề xuất sẽ mất. Phiên bản production cần lưu tác vụ vào SQLite, đặt thời hạn và chống duyệt lại.

## 11. Thiết kế bản production: Duyệt → Ghi MCP → Rollback

Chỉ triển khai sau khi xác minh chính xác tool và schema của MCP.

1. **Đọc:** Agent lấy trạng thái hiện tại và tạo bản đề xuất dạng dữ liệu có cấu trúc. Giai đoạn này chỉ cấp tool đọc.
2. **Khóa bản đề xuất:** Lưu target, nội dung thay đổi, phiên bản/hash của trạng thái gốc và mã tác vụ vào SQLite.
3. **Duyệt:** Telegram kiểm tra numeric user ID, task ID, thời hạn và hash. Chỉ chấp nhận một lần.
4. **Backup:** Trước khi ghi, lấy bản cấu hình hiện tại qua MCP, lưu snapshot vào `/backups` với quyền hạn chế. Nếu không thể backup, dừng.
5. **Kiểm tra xung đột:** Đọc lại target và so với phiên bản/hash đã duyệt. Nếu khác, yêu cầu duyệt lại.
6. **Thực thi:** Một service triển khai gọi đúng tool ghi với tham số đã duyệt. Không cho AI tùy ý thực thi shell hoặc mở rộng phạm vi thay đổi.
7. **Xác minh:** Đọc lại trạng thái, kiểm tra kết quả và ghi audit log.
8. **Rollback:** Chỉ phục hồi snapshot nếu trạng thái hiện tại vẫn khớp phiên bản do tác vụ tạo; nếu có thay đổi mới, yêu cầu xử lý xung đột.
9. **URL:** Với dashboard, lấy đường dẫn thực tế từ dữ liệu dashboard và ghép với `HA_URL`; không suy đoán slug.

**Phạm vi ban đầu nên giới hạn dashboard**, sau đó mới thêm automation và script. Không cấp quyền chỉnh sửa khóa cửa, báo động, token hoặc cấu hình mạng cho agent.

## 12. Kiểm tra và nghiệm thu

- [ ] Docker bot chạy ổn định.
- [ ] Chỉ Telegram user ID được cấp phép nhận phản hồi.
- [ ] Codex đăng nhập ChatGPT thành công.
- [ ] Kết nối MCP thành công.
- [ ] Đã liệt kê và xác minh công cụ MCP.
- [ ] Có cơ chế chặn tool ghi **thực sự** trước phê duyệt.
- [ ] Có backup, kiểm tra xung đột, rollback và audit log.
- [ ] Có đường dẫn dashboard đúng, mở được từ điện thoại trên mạng LAN hoặc VPN.

## 13. Lỗi thường gặp

| Triệu chứng | Cách xử lý |
|---|---|
| MCP `401` | Kiểm tra token, kiểu Bearer/OAuth và quyền |
| MCP `404` | Kiểm tra đường dẫn, reverse proxy và cổng |
| MCP GET `405` | Thử bằng client MCP hỗ trợ Streamable HTTP, không kết luận từ GET |
| Codex không thấy tool | Kiểm tra `config.toml`, MCP transport, xác thực và log |
| Codex không đăng nhập | Kiểm tra volume `codex-data` và quyền UID 10001 |
| Telegram im lặng | Kiểm tra bot token, numeric user ID và `docker logs` |
| URL dashboard không mở | Kiểm tra HA Web URL, mạng LAN/VPN và slug thật |

## 14. Tài liệu tham khảo

- [OpenAI Codex CLI](https://github.com/openai/codex)
- [Home Assistant MCP Server](https://www.home-assistant.io/integrations/mcp_server/)
- [HA-MCP cộng đồng](https://github.com/homeassistant-ai/ha-mcp)
- [python-telegram-bot](https://docs.python-telegram-bot.org/)

---

**Kết luận:** Tài liệu này dựng được phần Telegram → Codex → MCP để khảo sát và lập kế hoạch. Không coi nó là bản production có ghi và rollback tự động cho đến khi kiểm chứng các MCP tools thực tế và hoàn thiện module triển khai có kiểm soát.
