# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: NGUYỄN ĐỨC HẢI
- **Student ID**: 2A202600149
- **Date**: 2026-06-04

---

## I. Technical Contribution (15 Points)

*Mô tả đóng góp cụ thể vào codebase của dự án*

- **Modules Implemented**:
  - `backend/tools.py` — Toàn bộ logic tool: `calculate_nutrition`, `check_nutrition_targets`, `calculate_cost`, `get_meal_suggestions`, `generate_weekly_menu`, và `dispatch_tool` dispatcher.
  - `backend/agent.py` — Lớp `NutriChefAgent` thực hiện vòng lặp ReAct với Gemini API, bao gồm parser JSON (`extract_json_from_text`) và giới hạn vòng lặp tối đa 8 bước.
  - `backend/prompts.py` — System prompt dạng template hướng dẫn LLM xuất JSON theo cấu trúc `thought / action / action_input / final_answer`.
  - `backend/main.py` — FastAPI server expose endpoint `/chat` với CORS và cache agent theo model.
  - `frontend/index.html` — Giao diện chat single-page với ReAct trace panel có thể mở/đóng.

- **Code Highlights**:

  ```python
  # agent.py — vòng lặp ReAct cốt lõi
  for iteration in range(MAX_ITERATIONS):
      response = chat.send_message(current_input)
      parsed = extract_json_from_text(response.text)

      if "final_answer" in parsed:
          break
      elif "action" in parsed:
          observation = dispatch_tool(parsed["action"], parsed["action_input"])
          current_input = f"Observation: {observation}"
  ```

  ```python
  # tools.py — tool generate_weekly_menu đảm bảo không trùng món liên tiếp
  available = [j for j in range(len(pool)) if j not in used_indices[-1:]]
  ```

- **Documentation**: Mỗi tool function có docstring mô tả input/output. System prompt trong `prompts.py` định nghĩa rõ format JSON mà LLM phải tuân theo, giúp `extract_json_from_text` parse ổn định. Tool registry `TOOLS` và `TOOL_SCHEMAS` tách biệt logic thực thi và mô tả cho LLM, dễ mở rộng thêm tool mới.

---

## II. Debugging Case Study (10 Points)

*Phân tích một sự kiện thất bại cụ thể phát hiện qua quá trình kiểm thử.*

- **Problem Description**: Khi chạy test case phức tạp nhiều bước (xem PASTED), hệ thống trả về kết quả **đúng về số liệu** nhưng lại **thiếu bước gọi tool thực tế** — agent "bịa" ra quan sát thay vì thực sự gọi `dispatch_tool`. Cụ thể, với yêu cầu "tăng thêm 50g rau xanh vào ngày thấp nhất", agent tính tay delta chi phí (600 VNĐ/suất × 800 = 480.000 VNĐ) và fiber mới (4.5 + ~1.0 = 5.5g) mà không gọi lại `calculate_nutrition` hay `calculate_cost` để verify.

- **Log Source**: Quan sát từ trace panel trên frontend — số bước `action` ít hơn kỳ vọng với câu hỏi có 3 yêu cầu lồng nhau (tạo menu → phân tích ngày rẻ nhất → tính delta khi thêm rau).

- **Diagnosis**: System prompt ban đầu không có few-shot example cho multi-step query. Khi câu hỏi có nhiều sub-task, LLM ưu tiên tổng hợp tất cả trong một `final_answer` duy nhất dựa trên reasoning nội tại, **bỏ qua bước gọi tool cho sub-task cuối**. Đây là behavior của chatbot thuần túy chứ không phải agent thực sự — LLM "biết" phép tính đơn giản nên tự tính thay vì dùng tool.

- **Solution**: Bổ sung vào system prompt instruction: *"Với mỗi sub-task trong câu hỏi, bắt buộc gọi tool riêng để lấy observation. Không tự tính toán nội tại nếu đã có tool phù hợp."* Đồng thời tăng `MAX_ITERATIONS` từ 6 lên 8 để đủ chỗ cho multi-tool call. Sau sửa đổi, trace panel hiển thị đúng 4–5 bước action cho cùng test case trên.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

*Suy nghĩ về sự khác biệt về khả năng reasoning.*

**1. Reasoning — Vai trò của khối `Thought`**

Trong Lab 3, `Thought` buộc LLM phải *externalize* quá trình suy nghĩ trước khi hành động. Với chatbot thông thường, câu hỏi "tạo thực đơn tuần + phân tích ngày rẻ nhất + tính delta fiber" sẽ được trả lời trong một lần sinh text duy nhất, dễ dẫn đến hallucination số liệu. Với ReAct agent, mỗi `Thought` buộc model phải xác định *tool nào cần gọi tiếp theo*, giúp kết quả có nguồn gốc từ data thực (NUTRITION_DB) chứ không phải từ prior của model.

**2. Reliability — Khi nào Agent tệ hơn Chatbot?**

Kết quả  cho thấy agent vẫn thất bại ở **surface-level reasoning**: với sub-task cuối (tính delta 50g rau), agent giải quyết đúng nhưng bằng cách tự tính tay — hành vi này giống chatbot, không phải agent. Ngoài ra, với câu hỏi đơn giản như "cơm gà là gì?", agent tốn 2–3 vòng lặp không cần thiết trong khi chatbot trả lời ngay. Latency và token cost của ReAct cao hơn đáng kể với truy vấn đơn giản.

**3. Observation — Ảnh hưởng của Environment Feedback**

Observation từ `calculate_nutrition` giúp agent phát hiện ngay ngày Thứ Ba có fiber = 4.5g (dưới ngưỡng 5g), từ đó recommendation "tăng rau" là hệ quả tự nhiên từ data — không phải từ prior training của LLM. Đây là điểm mạnh cốt lõi của ReAct: **kết quả là grounded, có thể trace lại nguồn gốc**, không như chatbot chỉ nói "bạn nên ăn nhiều rau hơn" mà không chỉ ra con số cụ thể tại sao.

---

## IV. Future Improvements (5 Points)

*Đề xuất nâng cấp lên production-level AI agent system.*

- **Scalability**: Chuyển `dispatch_tool` sang async với `asyncio` và `httpx` để các tool call I/O-bound (sau này gọi external price API, nutrition DB thực) không block. Kết hợp message queue (Redis + Celery) cho các yêu cầu tạo thực đơn hàng loạt (nhiều trường cùng lúc).

- **Safety**: Triển khai một **Supervisor LLM** (model nhỏ hơn, ví dụ Gemini Flash Lite) để audit `action_input` trước khi dispatch — phát hiện các giá trị bất thường như `num_students: -1` hoặc `budget_per_student: 0`. Thêm rate limiting per-session để tránh infinite billing khi agent bị vòng lặp.

- **Performance**: Xây dựng **Vector DB (ChromaDB/Pinecone)** cho tool retrieval khi số tool vượt 20+: thay vì đưa toàn bộ `TOOL_SCHEMAS` vào context mỗi lần, chỉ retrieve top-k tool liên quan theo cosine similarity với query. Giảm ~40% token mỗi lần gọi LLM. Thêm caching layer cho `calculate_nutrition` vì cùng một bộ nguyên liệu sẽ cho cùng kết quả.

---
