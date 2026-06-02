# Day 04 Lab v2 — Research Agent Tool Eval

## Team Information

- **Team**: Nhóm 4
- **Members**: Phạm Ngọc Vinh - 2A202600563, Nguyễn Huy Bảo - 2A202600997, Phan Quốc Anh - 2A202600890
- **Provider/Model**: Gemini - `gemini-3.1-flash-lite`
- **Public UI**: https://labai.xn--ngcvinh-dx4c.vn/

## Brief

Trong lab này, nhóm build một research agent nhỏ nhưng chạy thật. Agent nhận request của user, chọn tool, truyền arguments, chạy tool thật, lưu full JSON log, rồi dùng log đó để tối ưu prompt/tool declaration qua nhiều version.

Điều cần học không phải là "chatbot trả lời hay". Điều cần học là vòng lặp evidence-driven:

1. Chạy baseline bằng API thật.
2. Đọc run JSON để biết sai tool, sai args, thiếu hỏi lại, hoặc gọi tool thừa.
3. Sửa `artifacts/system_prompt.md` hoặc `artifacts/tools.yaml`.
4. Chạy lại và ghi versioning.
5. Tự viết thêm eval case để đo những lỗi nhóm quan tâm.
6. Viết report dựa trên log thật, không dựa vào cảm giác.

## Research Agent - Tính năng chính

### 🎯 Agent làm được gì

Research Agent của nhóm được thiết kế để hỗ trợ nghiên cứu đa nguồn với khả năng:

- **Tìm kiếm web**: Tin tức, thông tin hiện tại với bộ lọc chủ đề và thời gian
- **Social Media**: Theo dõi timeline người dùng và tìm kiếm tweet/post
- **Nghiên cứu học thuật**: Tìm và đọc papers từ arXiv, tải PDF về máy
- **Tra cứu tri thức**: Wikipedia, GitHub repositories, giá cryptocurrency, video YouTube
- **Company Policy**: Tra cứu tài liệu nội bộ
- **Output**: Format digest markdown, gửi thông báo qua Telegram

### 🛠️ Các Tool có sẵn (15 tools)

#### Core Tools (6)
- **clarify**: Hỏi lại người dùng khi thiếu thông tin (handle, URL, ID) hoặc cần xác nhận
- **timeline**: Lấy tweets/posts gần đây của tài khoản cụ thể (ví dụ: `sama`, `elonmusk`)
- **social_search**: Tìm tweets/posts theo chủ đề (Latest/Top mode)
- **lookup**: Tìm kiếm web với bộ lọc topic (general/news) và timeframe
- **fetch**: Đọc/scrape nội dung từ URL cụ thể
- **format**: Format các items thành markdown digest

#### Bonus Tools (4)
- **send**: Gửi tin nhắn lên Telegram (chỉ sau khi có xác nhận)
- **policy**: Tra cứu company policy nội bộ từ markdown KB
- **papers**: Tìm papers trên arXiv theo topic
- **paper_text**: Tải PDF arXiv và trích xuất text

#### Research Extension Tools (5) ✨ **MỚI**
- **pdf_download**: Tải file PDF từ URL hoặc arXiv ID về máy (với fallback)
- **wikipedia**: Tìm kiếm thông tin bách khoa, định nghĩa, khái niệm
- **github_search**: Tìm repositories, thư viện open-source trên GitHub
- **crypto_price**: Lấy giá realtime, biến động 24h, vốn hóa của cryptocurrency
- **youtube_search**: Tìm video giải thích, bài giảng, clips

### 💡 Câu hỏi mẫu để thử

```
1. Tweet mới nhất của Sam Altman là gì?
2. Tin tức AI hôm nay có gì nổi bật?
3. Tóm tắt bài này hộ mình: https://openai.com/blog/gpt-5
4. Cho mình các tweet phổ biến nhất về OpenAI
5. Gửi tin nhắn hello qua Telegram cho tôi
6. Artificial intelligence là gì? (Wikipedia)
7. Tìm repo về transformer trên GitHub
8. Giá Bitcoin hiện tại?
9. Tìm video về deep learning
10. Tải paper này về: 2301.00001 (arXiv)
```

### 📊 Kết quả đánh giá

**Latest v5 Performance:**
- Total cases: 20
- Case accuracy: 100% (20/20)
- Tool routing accuracy: 100%
- Argument accuracy: 100%
- Multi-turn accuracy: 100%

**Version History:**
- v0: Baseline - 60% accuracy
- v1: Routing rules - 85% accuracy
- v2: Clarify response_type - 95% accuracy
- v3: Vietnamese Telegram boundary - 100% accuracy
- v4: Added pdf_download tool - 100% accuracy
- v5: Added 4 research tools + strict scope - 100% accuracy

## Architecture

### System Components

```
starter_v0/
├── Core Agent
│   ├── agent.py              # One-shot model → tool calls → execution
│   ├── chat.py               # Multi-round chat with transcript logging
│   ├── run_eval.py           # Evaluation system with metrics
│   └── ui_server.py          # Web UI server (port 8765)
│
├── Providers (LLM Adapters)
│   ├── providers/
│   │   ├── gemini_provider.py       # Google Gemini (with quota retry)
│   │   ├── openrouter_provider.py   # OpenRouter
│   │   ├── openai_provider.py       # OpenAI
│   │   └── anthropic_provider.py    # Claude
│
├── Tools (15 total)
│   ├── tools/
│   │   ├── clarify/          # Ask user for missing info
│   │   ├── timeline/         # Get user tweets
│   │   ├── social_search/    # Search tweets by topic
│   │   ├── lookup/           # Web search (news/general)
│   │   ├── fetch/            # Scrape URL content
│   │   ├── format/           # Format digest
│   │   ├── send/             # Telegram notification
│   │   ├── policy/           # Company policy KB
│   │   ├── papers/           # arXiv search
│   │   ├── paper_text/       # arXiv PDF → text
│   │   ├── pdf_download/     # Download PDFs ✨
│   │   ├── wikipedia/        # Wikipedia search ✨
│   │   ├── github_search/    # GitHub repo search ✨
│   │   ├── crypto_price/     # Crypto prices ✨
│   │   └── youtube_search/   # YouTube video search ✨
│
├── Configuration & Data
│   ├── artifacts/
│   │   ├── system_prompt.md  # Agent instructions
│   │   ├── tools.yaml        # Tool declarations
│   │   ├── version_log.csv   # Version tracking
│   │   └── REPORT.md         # Evaluation report
│   │
│   ├── data/
│   │   ├── eval_base.json    # Base evaluation cases (20)
│   │   ├── eval_group.json   # Team cases (10)
│   │   └── eval_research_extension.json
│   │
│   └── company_policy/       # Local policy markdown KB
│
├── Outputs
│   ├── runs/                 # Evaluation results (JSON)
│   ├── transcripts/          # Chat session logs
│   └── analysis/             # Parsed CSV reports
│
└── Utilities
    ├── versioning.py         # Artifact hashing
    ├── env_loader.py         # Environment config
    └── scripts/
        ├── preflight_provider.py  # Connection test
        └── parse_runs.py          # Run log parser
```

### Data Flow

```
User Input
    ↓
UI Server / Chat CLI
    ↓
Agent (system_prompt.md)
    ↓
Provider (Gemini/OpenRouter/etc)
    ↓
Tool Selection & Arguments
    ↓
Tool Execution (15 tools)
    ↓
Result Aggregation
    ↓
Response + Transcript Log
```

### Tool Architecture

Each tool follows a standard structure:

```
tools/<tool_name>/
├── TOOL.md           # Documentation with frontmatter
│   ├── name          # Tool identifier
│   ├── role          # core/bonus/extension
│   ├── provider      # API/service used
│   └── description   # What it does
│
└── tool.py           # Implementation
    └── main()        # Entry function with typed args
```

### Version Control System

The version log tracks all changes:

```csv
version,author,changed_artifact,artifact_version,
prompt_hash,tools_hash,reason,hypothesis,
metric_before,metric_after,run_file
```

Each run generates:
- Unique artifact_version (v0+hash+hash)
- Full evaluation metrics
- Tool call traces
- Failure analysis

## UI Features

### Web Interface

The UI server (`ui_server.py`) provides:

- **Provider Selection**: Gemini, OpenRouter, OpenAI, Anthropic
- **Model Configuration**: Custom model names
- **Version Control**: Track artifact versions
- **History Management**: Configurable conversation window (0-20 turns)
- **Tool Rounds**: Max tool execution rounds (1-8)
- **Sample Queries**: 10 pre-built examples across 5 categories
- **Real-time Inspection**: View tool calls, arguments, and results
- **Transcript Logging**: Auto-save all sessions
- **Download Support**: Direct PDF download links for pdf_download tool

### Sample Categories

1. **Social & Timeline**: Tweet tracking, GPT-5 discussions
2. **Web & News**: AI news, OpenAI updates
3. **Research & Knowledge**: Wikipedia, YouTube tutorials
4. **Code & Crypto**: GitHub repos, Bitcoin prices
5. **Papers & PDF**: arXiv search, paper downloads

### Keyboard Shortcuts

- **Enter**: Send message
- **Shift+Enter**: New line
- **Reset Button**: Clear history and start fresh session

## Lab Requirements

## Lab Requirements

Nhiệm vụ bắt buộc:

- ✅ Setup chạy được bằng provider thật (Gemini)
- ✅ Agent có ít nhất 5 tool trong `artifacts/tools.yaml` (có 15 tools)
- ✅ Chạy base eval với kết quả 100%
- ✅ Tối ưu ít nhất 3 vòng sau baseline: `v1`, `v2`, `v3` (đã có v5)
- ✅ Ghi `artifacts/version_log.csv` đầy đủ
- ✅ Viết thêm 5 tools mới với TOOL.md và đăng ký đầy đủ
- ✅ Tự viết thêm 10 eval case vào `data/eval_group.json` (5 single + 5 multi)
- ✅ Nộp run JSON, transcript JSON, report
- ✅ UI chạy được và có public link
- ✅ Hoàn thành `artifacts/REPORT.md` đầy đủ cả Phần A và B

Bonus (đã hoàn thành):

- ✅ Action tool `send` với confirmation
- ✅ Extra tools: `policy`, `papers`, `paper_text`
- ✅ Thêm hơn 3 tools mới: `pdf_download`, `wikipedia`, `github_search`, `crypto_price`, `youtube_search`
- ✅ UI web professional với Inspector

**Điểm thưởng (bonus point):** ✅ Team đã làm CẢ HAI — dựng được UI **và** tự viết thêm 5 tool mới (ngoài các tool có sẵn).

## Folder Map

```text
starter_v0/
  agent.py                    # one-shot model -> tool calls -> tool execution
  chat.py                     # interactive chat, multi-round tools, transcript JSON
  run_eval.py                 # eval routing + args, writes runs/*.json
  versioning.py               # prompt/tool hash
  artifacts/
    system_prompt.md          # student edits
    tools.yaml                # student edits
    version_log.csv           # student fills
    REPORT.md                 # report: Phần A debate poster + Phần B chi tiết
  data/
    eval_base.json            # fixed base eval, do not edit the cases
    eval_group.json           # team adds at least 5 cases
    eval_research_extension.json
  tools/
    README.md                 # tool folder contract
    <tool_name>/
      TOOL.md                 # frontmatter + notes
      tool.py                 # self-contained implementation
  company_policy/             # local markdown KB for bonus policy tool
  providers/                  # OpenRouter/OpenAI/Anthropic/Gemini adapters
  scripts/preflight_provider.py
  samples/                    # mock format examples (transcript, run-analysis, version_log)
```

## Tool Tracks

Phần mô tả dưới đây tóm tắt mỗi tool *làm gì*. Việc xác định *khi nào dùng* tool nào là phần nhóm tự định nghĩa trong prompt và tool declaration.

Core tools:

- `clarify`: gửi một câu hỏi cho người dùng và chờ lượt trả lời tiếp theo.
- `timeline`: lấy bài đăng gần đây của một tài khoản (`screenname`).
- `social_search`: tìm bài đăng theo từ khóa (`search_type`: Latest/Top).
- `lookup`: tìm trên web (có `topic` general/news và `timeframe`).
- `fetch`: đọc nội dung một URL.
- `format`: trình bày các item đã có thành markdown digest.

Bonus tools:

- `send`: gửi text lên Telegram channel (chỉ gửi khi `confirmed=true`).
- `policy`: tìm trong company policy markdown nội bộ.
- `papers`: tìm paper trên arXiv.
- `paper_text`: tải PDF arXiv và trích text cục bộ.

Mỗi tool nằm trong thư mục riêng dưới `starter_v0/tools/<tool_name>/`.

## ⚠️ Nếu nhóm đổi tên tool: phải đồng bộ

Eval chấm theo **đúng tên tool**. Nếu nhóm đổi tên một tool cho rõ nghĩa hơn (ví dụ `send` → `send_telegram`), **phải đổi đồng bộ ở CẢ những nơi sau**, nếu không eval báo lỗi `not declared in tools.yaml` hoặc chấm sai mọi case:

1. `artifacts/tools.yaml` — field `name`
2. `tools/__init__.py` — key trong `TOOL_FUNCTIONS`
3. `data/eval_base.json` **và** `data/eval_research_extension.json` — `expect.tool_calls[].name`

(Không cần đổi tên hàm trong `tools/<folder>/tool.py` — chỉ cần key trỏ đúng hàm. Không sửa *nội dung case* trong `eval_base.json`, chỉ đổi tên tool nếu nhóm rename.)

## Quick Start

### 1. Cài đặt môi trường

Run from `starter_v0/`:

```bash
cd starter_v0
python3 -m venv .venv

# Linux/Mac
source .venv/bin/activate

# Windows
.venv\Scripts\activate

pip install -r requirements.txt
cp .env.example .env
```

### 2. Cấu hình API Keys

Chỉnh sửa file `.env` với các API keys cần thiết:

```bash
# LLM Providers (chọn ít nhất 1)
OPENROUTER_API_KEY=...
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
GOOGLE_API_KEY=...         # Cho Gemini

# Tool APIs (bắt buộc cho các tool tương ứng)
TAVILY_API_KEY=...         # lookup (web search)
FIRECRAWL_API_KEY=...      # fetch (web scraping)
RAPIDAPI_KEY=...           # timeline, social_search
RAPIDAPI_TWITTER_HOST=twitter-api45.p.rapidapi.com

# Telegram (cho tool send)
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...

# YouTube (cho tool youtube_search)
YOUTUBE_API_KEY=...
```

Tool setup details are in [TOOL-SETUP.md](TOOL-SETUP.md).

### 3. Kiểm tra kết nối

Preflight test:

```bash
# Test provider connection
python scripts/preflight_provider.py --provider gemini

# Hoặc test provider khác
python scripts/preflight_provider.py --provider openrouter
```

If preflight fails, fix provider key, dependency, or network before running eval.

### 4. Chạy UI Test

Khởi động UI server để test tương tác:

```bash
# Chạy UI server (mặc định: http://localhost:8765)
python ui_server.py

# Chạy với custom host/port
python ui_server.py --host 0.0.0.0 --port 8080
```

Mở browser tại `http://localhost:8765` để test agent.

## Step 1 — Run Baseline

Run the fixed base eval as `v0`:

```bash
python run_eval.py \
  --provider openrouter \
  --version v0 \
  --suite base \
  --eval-cases data/eval_base.json
```

Output is saved to `runs/*.json`. Read:

- `summary.case_accuracy`
- `summary.tool_routing_accuracy`
- `summary.argument_accuracy`
- `summary.multiturn_accuracy`
- `results[*].result.failures`
- `results[*].result.observed_mismatch`

The run JSON also stores `artifact_version`, `prompt_hash`, `tools_hash`, actual tool calls, and actual tool results. That is the evidence for your report.

Optional: parse run JSON into a flat CSV table for analysis:

```bash
python scripts/parse_runs.py runs/ --output analysis/base_runs.csv
```

## Step 2 — Fix One Thing

Edit only:

- `artifacts/system_prompt.md`
- `artifacts/tools.yaml`

Do not edit the cases in `data/eval_base.json`.

Method, not memorized answers:

1. Mở run JSON. Với mỗi case FAIL, đọc `observed_mismatch` + `failures` + `actual_tool_calls`.
2. Đặt một giả thuyết: *vì sao* agent chọn sai (thiếu rule routing? thiếu convention args? prompt đang khuyến khích đoán/gửi/làm-một-bước?).
3. Sửa **một** thứ (một dòng prompt hoặc một description) để kiểm chứng giả thuyết đó.
4. Chạy lại, so metric trước/sau. Nếu không cải thiện, đổi giả thuyết.

Đổi một giả thuyết mỗi lần để version log có ý nghĩa.

## Step 3 — Run 3 Optimization Versions

Run at least three improved versions:

```bash
python run_eval.py --provider openrouter --version v1 --suite base --eval-cases data/eval_base.json
python run_eval.py --provider openrouter --version v2 --suite base --eval-cases data/eval_base.json
python run_eval.py --provider openrouter --version v3 --suite base --eval-cases data/eval_base.json
```

After each run, fill `artifacts/version_log.csv`:

```text
version,author,changed_artifact,artifact_version,prompt_hash,tools_hash,reason,hypothesis,metric_before,metric_after,run_file
```

Use hashes and run file paths from the eval output.

## Step 4 — Add Team Eval

Add at least 5 cases to `data/eval_group.json`.

Each case needs:

- `id`
- `phase`: always `"B"`
- `query` or `turns`
- `failure_type`: one of `wrong_tool`, `wrong_arg_value`, `wrong_boundary`, `unnecessary_tool`, `out_of_scope`, `missing_info`
- `expect`: `tool_calls` or `no_tool`
- `metadata.what_it_tests`

Run:

```bash
python run_eval.py \
  --provider openrouter \
  --version v3 \
  --suite group \
  --eval-cases data/eval_group.json
```

Optional extension eval:

```bash
python run_eval.py \
  --provider openrouter \
  --version v3 \
  --suite extension \
  --eval-cases data/eval_research_extension.json
```

## Step 5 — Chat Live

`chat.py` is for live multi-round interaction. It logs every turn to `transcripts/*.transcript.json`.

```bash
python chat.py --provider openrouter --version v3
```

Try at least 3 live turns, for example:

- A normal research request.
- A request thiếu thông tin (không nói rõ account/URL), rồi lượt sau bổ sung.
- Một request "đăng/gửi bản tin lên Telegram" — quan sát agent có hành động ngay hay hỏi lại trước, rồi tự quyết định hành vi nào mới đúng và sửa prompt cho khớp.

## Deploy nhanh để team khác dùng thử

### Option 1: Cloudflare Tunnel (Khuyến nghị - Nhanh nhất)

Để team cùng zone tự thử agent trong Team showdown, expose UI đang chạy local ra một link public. Cách nhanh nhất, không cần đăng ký domain hay deploy lên cloud, là **Cloudflare Tunnel** (`cloudflared`).

1. Chạy UI local trước (mặc định cổng `8765`):

   ```bash
   python ui_server.py            # → http://localhost:8765
   ```

2. Cài `cloudflared`:

   ```bash
   brew install cloudflared                          # macOS
   winget install --id Cloudflare.cloudflared        # Windows (hoặc: scoop install cloudflared)
   # Linux: xem https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
   ```

   Windows: nếu báo `cloudflared not recognized`, mở terminal mới rồi chạy lại.

3. Mở tunnel trỏ vào cổng UI — lệnh trả về ngay một URL `https://<random>.trycloudflare.com`:

   ```bash
   cloudflared tunnel --url http://localhost:8765
   ```

4. Copy URL đó, dán vào `REPORT.md` Phần A (mục "Link dùng thử") để team khác mở thử.

Lưu ý: link `trycloudflare.com` là tạm thời, sống theo phiên `cloudflared` (tắt lệnh là mất). Giữ lệnh chạy trong suốt buổi showdown.

### Option 2: Vercel / Railway / Render

Để có permanent URL, deploy lên cloud platform:

```bash
# Tạo requirements.txt cho production
pip freeze > requirements.txt

# Deploy lên Vercel (cần vercel CLI)
vercel --prod

# Hoặc deploy lên Railway
railway up

# Hoặc deploy lên Render
# Push code lên GitHub và connect repository với Render
```

### Option 3: Streamlit Cloud

Nếu convert UI sang Streamlit format:

```bash
# Push code lên GitHub repository
git push origin main

# Deploy từ https://share.streamlit.io
# Connect repository và chọn branch main
```

## Step 6 — Report + Debate Poster

Hoàn thành `artifacts/REPORT.md`. File này có 2 phần với deadline khác nhau:

- **Phần A — Giới thiệu agent** (ngắn gọn 1 trang) — **phải xong trước 16:30**, làm tài liệu phụ trợ để team khác hiểu nhanh khi demo. Chỉ cần: (1) agent làm được gì, (2) agent có những tool gì và mỗi tool làm được gì, (3) vài câu hỏi mẫu để team khác tự thử.
- **Phần B — Chi tiết / Bằng chứng** — **có thể hoàn thiện sau Team showdown để nộp bài**. Bảng đầy đủ v0–v3, failure analysis, eval cases, live chat, reflection — dựa trên log thật (run JSON, version_log).

**Format Phần A:** nộp tối thiểu bản markdown trong `REPORT.md`. Khuyến khích biến Phần A thành **poster HTML/SVG 1 trang** để show trực tiếp cho team cùng zone (ví dụ `artifacts/poster.html` hoặc `artifacts/poster.svg`) — cùng nội dung, dễ nhìn hơn. Poster HTML/SVG là tùy chọn, không thay thế bản markdown.

## Submit

Submit `starter_v0/` with:

- ✅ `artifacts/system_prompt.md` - Optimized agent instructions
- ✅ `artifacts/tools.yaml` - 15 tool declarations
- ✅ `artifacts/version_log.csv` - Versions v0 through v6
- ✅ `artifacts/REPORT.md` - Complete report (Part A + Part B)
- ✅ `data/eval_group.json` - 10 team evaluation cases
- ✅ `runs/*.json` - Multiple evaluation runs
- ✅ `analysis/*.csv` - Parsed run analytics
- ✅ `transcripts/*.transcript.json` - Live chat sessions
- ✅ `tools/*/` - 5 new tool implementations with TOOL.md
- ✅ `ui_server.py` - Web UI with inspector
- ✅ Public UI link - https://labai.xn--ngcvinh-dx4c.vn/

Do not submit `.env` or API keys.

## Development Notes

### Optimization Journey

**v0 → v1 (60% → 85%)**
- Added explicit routing rules
- Clarified missing-info behavior
- Defined publish-boundary rules

**v1 → v2 (85% → 95%)**
- Made `clarify.response_type` required
- Fixed argument mismatch errors

**v2 → v3 (95% → 100%)**
- Added Vietnamese action phrase detection
- Forced yes/no confirmation for Telegram publish

**v3 → v4 (100% → 100%)**
- Added `pdf_download` tool
- Maintained routing stability

**v4 → v5 (100% → 100%)**
- Added 4 research extension tools
- Implemented strict scope boundary
- Added offline fallbacks for all external APIs

### Key Learnings

1. **Prompt Engineering**: Clear routing rules > vague descriptions
2. **Tool Design**: Required arguments prevent guessing behavior
3. **Boundaries**: Explicit confirmation gates for actions
4. **Multi-turn**: Context carryover with correction override
5. **Evaluation**: JSON logs reveal issues better than manual testing
6. **Offline Fallbacks**: Critical for tool reliability in production

### Testing Strategy

```bash
# 1. Unit test individual tools
python -m tools.wikipedia.tool "AI"

# 2. Run baseline evaluation
python run_eval.py --provider gemini --version v5 --suite base

# 3. Test multi-turn conversations
python chat.py --provider gemini --version v5

# 4. Test UI integration
python ui_server.py

# 5. Run team evaluation
python run_eval.py --provider gemini --version v5 --suite group --eval-cases data/eval_group.json
```

### Performance Metrics

- **Response Time**: Average 2-3 seconds per tool call
- **Success Rate**: 100% on base eval (20/20 cases)
- **Multi-turn Accuracy**: 100% (context preservation)
- **Tool Routing**: 100% correct tool selection
- **Argument Accuracy**: 100% valid parameters

### Future Enhancements

Potential improvements for future versions:

1. **Caching**: Redis cache for frequent queries (Wikipedia, crypto prices)
2. **Rate Limiting**: Implement token bucket for API calls
3. **Async Tools**: Parallel tool execution for independent calls
4. **Cost Tracking**: Log API usage and costs per session
5. **A/B Testing**: Compare multiple prompts simultaneously
6. **Custom Tools**: Plugin system for easy tool additions
7. **Voice Input**: Speech-to-text integration
8. **Export Options**: PDF/Word report generation

## Troubleshooting

### Common Issues

**1. Provider API errors**
```bash
# Test provider connection
python scripts/preflight_provider.py --provider gemini

# Check API key in .env
cat .env | grep API_KEY
```

**2. Tool execution failures**
```bash
# Check tool dependencies
pip install -r requirements.txt

# Test individual tool
python -m tools.lookup.tool "AI news"
```

**3. UI not loading**
```bash
# Check port availability
netstat -an | findstr 8765  # Windows
lsof -i :8765               # Mac/Linux

# Try different port
python ui_server.py --port 9000
```

**4. Transcript not saving**
```bash
# Check write permissions
ls -la transcripts/

# Create directory if missing
mkdir -p transcripts
```

## Team Contributions

- **Phạm Ngọc Vinh**: System architecture, Gemini provider, UI server, deployment
- **Nguyễn Huy Bảo**: Tool development (wikipedia, github_search, youtube_search), evaluation framework
- **Phan Quốc Anh**: Tool development (pdf_download, crypto_price), prompt engineering, report

## References

- [Lab Instructions](README.md)
- [Tool Setup Guide](TOOL-SETUP.md)
- [Evaluation Report](starter_v0/artifacts/REPORT.md)
- [Public UI](https://labai.xn--ngcvinh-dx4c.vn/)
- [OpenRouter Documentation](https://openrouter.ai/docs)
- [Gemini API](https://ai.google.dev/tutorials/python_quickstart)
- [arXiv API](https://arxiv.org/help/api/)
- [Tavily Search API](https://tavily.com/)

## License

Educational project for VinUniversity AI Course - Day 04 Lab.

---

**Last Updated**: June 2, 2026  
**Version**: v6  
**Status**: Production Ready ✅

## Checkpoint 4h

- 15:00 - Run baseline + build UI
- 15:30 - Improve prompt/tools for v1 + build at least 1 tool
- 16:00 - Write team eval cases + improve v2
- 16:30 - Team showdown (demo sản phẩm; REPORT.md Phần A / poster là tài liệu phụ trợ để team khác hiểu nhanh agent có tool gì, làm được gì)
- 17:30 - Improve v3 + hoàn thiện report Phần B
