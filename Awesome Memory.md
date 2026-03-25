
### Paper

|論文|重點|連結|
|---|---|---|
|**A-MEM**（NeurIPS 2025）|Zettelkasten 式動態連結記憶，每條記憶含 context / keywords / tags / links|arXiv 2502.12110|
|**MemOS**（Jul 2025）|OS-level memory 管理，統一 plaintext / KV cache / parameter 三層|arXiv Jul 2025|
|**Intrinsic Memory Agents**（Aug 2025）|Role-aligned memory template，適合 multi-agent|arXiv 2508.08997|
|**LLM-MAS Memory Survey**（TechRxiv 2025）|多 agent 系統的 memory 機制全景|TechRxiv|
|**Memory in the Age of AI Agents: A Survey**（2025/12）|統一分類框架（Forms / Functions / Dynamics）|GitHub: Shichun-Liu|
|**LLM Agent Memory: A Survey from a System-Level View**（Mar 2026）|Operation-centric 視角，覆蓋 KV cache 到 RAG|Preprints.org|
|**General Agentic Memory via Deep Research**（2025/11）|專為 deep research 情境設計的 general memory|VectorSpaceLab GitHub|

### OSS

| 框架                     | 定位                                                 | 最適情境                            |
| ---------------------- | -------------------------------------------------- | ------------------------------- |
| **Letta**（原 MemGPT）    | OS-tiered，agent 自主管理 core / archival memory        | Coding Agent（需完整 agent runtime） |
| **Mem0**（mem0ai）       | 輕量 production-ready，自動萃取 & 檢索                      | Chat Agent（快速部署）                |
| **Zep / Graphiti**     | Temporal knowledge graph，追蹤事實隨時間變化                 | Deep Research（需時序感知）            |
| **A-MEM**（開源）          | 動態連結，每條記憶互相索引                                      | 通用，尤其適合知識密集場景                   |
| **ReMe**（AgentScope）   | Compact-first，token budget 管理                      | Coding Agent 大 context 壓縮       |
| **LangMem**（LangGraph） | 原生整合 LangGraph，支援 semantic / episodic / procedural | 已有 LangGraph stack 的團隊          |



1. memory 類型 × 情境
2. **通用模組
3. **技術選型決策樹**（根據資源限制、latency 要求決定哪些模組用哪個框架）
4. **不通用的部分**（例如 symbol index、 subquery cache）


### A：Ingestion Pipeline（文字 → 記憶）

所有情境都需要「把某種文字輸入轉成可存取的記憶」

```
原始文字（對話輪次 / code chunk / tool result）
  → [Preprocessing] 清洗、切割（sentence / chunk / event 粒度）
  → [Extraction] LLM 萃取 structured facts（who, what, relation, timestamp）
  → [Embedding] 向量化（用於 semantic retrieval）
  → [Dedup check] 與已有記憶做相似度比對，決定 add / update / skip
  → [Write to store] 寫入 vector DB + optional KG
```

記憶管理涉及對已建構記憶的進一步處理與精煉，包括去重、合併與衝突解決，類似人類記憶的reconsolidation 與反思過程


### B：Storage 層

Storage paradigm 包括：累積式記憶（完整歷史追加）、反思式/摘要式記憶（定期壓縮摘要）、純文字、參數化（embedding 進模型權重），以及結構化記憶（表格、三元組或圖結構）。

| 記憶型態               | 用途               | 技術選型                        |
| ------------------ | ---------------- | --------------------------- |
| Short-term buffer  | 最近 N 輪 / loop 狀態 | in-memory dict / Redis      |
| Episodic memory    | 過去事件、已做動作        | Vector DB（Qdrant / Chroma）  |
| Procedural memory  | 決策記錄、工作流         | KV store + structured JSON  |

### C：Retrieval 策略

三個情境共用的 retrieve 介面，依 query 類型路由：

- **Semantic search**：embedding 相似度，適合"找相關內容"
- **Keyword / exact match**：適合 subquery dedup 的精確比對
- **Temporal filter**：按時間篩選，適合 deep research 的 loop 順序感知
- **Graph traversal**：適合 coding agent 的符號依賴追蹤

簡單讓 agent 自主搜尋文件（filesystem 方式）的效果，可以顯著超過專門為 single-hop retrieval 設計的特殊化 memory tools [Letta](https://www.letta.com/blog/benchmarking-ai-agent-memory)，因此 retrieval 介面的靈活性比精心設計的固定查詢更重要。

### D：Memory Lifecycle（Dedup / Decay / Eviction）

研究發現如果存入有缺陷或不相關的記憶，agent 會出現「自我退化（self-degradation）」現象，導致後續 performance 持續下降 [arXiv](https://arxiv.org/pdf/2509.25250)，因此 lifecycle 管理是通用必須模組，不只是優化項：

- **Dedup**：寫入前 cosine similarity > threshold → merge or skip
- **Conflict resolution**：同一 key 有新舊衝突時，保留 timestamp 較新者或讓 LLM 判斷
- **Decay / forgetting**：低存取頻率 + 時間久遠的記憶降低優先級（Ebbinghaus curve 啟發）
- **Compaction（壓縮）**：當 memory store 超過容量閾值，用 LLM 把多條舊記憶摘要合併

### E：Memory 注入介面（Context Assembly）

三個情境都需要決定「什麼記憶要注入這次的 LLM prompt」：

```
Retrieve(query, top_k, filters)
  → Rerank（依相關度、時效、重要性）
  → Format（轉成 prompt-friendly 文字 block）
  → Inject（system prompt 或 user message prefix）
```


---
---


1. **Ingestion (寫入與預處理層):**
    
    - Chat: 寫入純文字 (User input, AI response)。
        
    - Coding: 寫入程式碼 (需要搭配 AST parser 切割 function/class)。
        
    - Research: 寫入 Tool 的執行結果 (JSON 格式或網頁抓取的長文本)。
        
    - 通用處理: 所有的資料進來都要加上 **Metadata** (Timestamp, Agent_ID, Session_ID, Source_Type, Tags)。
    - 
    - 因為 Vector DB (向量庫) 只能比對「語意相似度」。如果不加 Metadata，當 Agent 搜尋「蘋果」時，可能會同時撈出「昨天聊天的蘋果手機」和「上個月查資料的蘋果農場」。Metadata 是用來做 **Pre-filtering (前置過濾)** 的。

		**設計範例：**
		DRA 剛剛使用 Tool 抓取了一段關於 "GPT-4 架構" 的網頁文字，準備寫入 Memory：
```
{
  "id": "mem_9a8b7c...", 
  "content": "GPT-4 是一個大型多模態模型，支援文本與圖像輸入...", // 真實要被向量化與讓 LLM 閱讀的內容
  "metadata": {
    "timestamp": "2024-05-20T10:30:00Z", // 用途：檢索時可以加上條件「只找最近3天的記憶」
    "agent_id": "deep_researcher_v1",    // 用途：區分是 Chat Agent 還是 Research Agent 存的
    "session_id": "task_req_5566",       // 用途：Research 任務專屬 ID，避免污染到下一次全新的 Research 任務
    "source_type": "tool_web_scrape",    // 用途：可以區分是 "user_input" (用戶說的), "ai_thought" (AI自己想的), 還是 "tool_output" (工具給的)
    "source_url": "https://...",         // (擴展) 紀錄來源網址，方便 Agent 寫 reference
    "tags": ["AI", "GPT-4", "Architecture"] // 用途：精準的關鍵字過濾
  }
}
```

在開發上的好處：  
當 Research Agent 進行到 Loop 3 時，Memory Manager 可以下達這樣的查詢指令給資料庫：  
「請幫我找語意跟 '模型架構' 相似的內容，但條件是 (agent_id == 'deep_researcher_v1' AND session_id == 'task_req_5566')。」  

-
1. **Storage (儲存層 - 混合儲存策略):**
    
    - 單一資料庫無法滿足。你需要：
        
        - **Vector DB (向量庫):** 存 Chat semantic search、Research 摘要、Coding 註解。
            
        - **Graph DB (圖資料庫):** 存程式碼的依賴關係 (A call B)、Research 的實體關聯。
        - 
        - Chat Agent不需要Graph DB。
			Chat Agent -> Vector DB (處理語意) + Key-Value (處理最近幾句話的順序) 

			Graph DB 情境：「關聯性 (Relationship) 的重要性大於 語意相似度 (Semantic Similarity)」時。

			- **情境 A：Coding Agent**
			    
			    - 假設檔案 main.py 呼叫了 utils.py 裡的 calculate_tax() 函數。
			        
			    - 如果你請 Agent 「修改稅率計算邏輯」。Vector DB 可以幫你精準找到 calculate_tax() 的程式碼。
			        
			    - **但 Vector DB 不知道誰呼叫了它。** 如果你改了 calculate_tax() 的回傳格式，main.py 就會報錯。
			        
			    - Graph DB 裡面存的不是文本，而是節點關聯：(main.py) -\[CALLS]-> (calculate_tax)。Agent 查 Graph DB 就能知道：「修改這個函數，我連 main.py 也要一起修改」。
			        
			- **情境 B：DR 做「情報收集 / 實體關聯」**
			    
			    - Research 任務如果是：「找出 OpenAI 的現任高層，以及他們以前待過哪些公司」。
			        
			    - Graph DB 會將文字轉化為節點：(Sam Altman) -\[CEO_OF]-> (OpenAI), (Sam Altman) -\[FORMER_PRESIDENT_OF]-> (Y Combinator)。這在處理複雜的邏輯推導（Multi-hop reasoning）時非常強大。
			            
			        - **Key-Value DB (如 Redis):** 存最近 5 句話 (Chat)、Tool Execution Cache (Research 避免重複呼叫)。
            
2. **Retrieval (檢索與路由層):**
    
    - 當 Agent 需要 context 時，Memory Manager 會根據 Query 決定去哪裡撈資料。
        
    - 採用 **Hybrid Search** (下面): Keyword search (精準變數名) + Vector search (語意) + Graph Traversal (關聯)。
        
3. **Consolidation (記憶整合/遺忘機制):**
    
    - Token 有上限，Memory 必須有淘汰機制。
        
    - 包含：**Summarization** (把舊的對話或長篇 Research 壓縮成摘要)、**Eviction** (LRU 演算法清掉太久沒用的快取)。

---

Coding Agent 的 Memory 必須是「結構化且具備層級的 (Hierarchical)」**：

1. **Repository-level Memory (全域語意記憶):**
    
    - **機制:** 不要把整個檔案塞給 LLM。使用 AST (Abstract Syntax Tree) 解析程式碼。
        
    - **儲存:** 把 Function signature, Class definition 和 Docstring 存進 Vector DB。把檔案之間的 import 關係存進 Graph DB（或具有關聯性的 Metadata）。
        
    - **用途:** 讓 Agent 知道「有哪些工具可用」、「改動這裡會影響哪裡」。
        
2. **File-level Working Memory (工作區記憶):**
    
    - **機制:** 當決定要修改某個巨型檔案時，只 Extract 該 Function 的 Chunk 進入 Agent 的 Working Context (類似 MemGPT 的 RAM)。
        
3. **Code Editing History (版本記憶):**
    
    - **機制:** 儲存 Agent 先前嘗試過修改的 diff 或 patch。如果編譯失敗，Agent 可以檢索「我剛剛改錯了什麼」，避免無限迴圈。
        

- **OSS:**
    
    - **Aider (aider.chat):** 目前最強的開源 Coding Agent 之一。它的 Repository Map (Repo Map) 機制非常值得參考。它利用 Tree-sitter 建立專案結構圖，當作 Memory 餵給 LLM。
        
    - **SWE-agent (Princeton 開源):** 解決 GitHub Issues 的 Agent。它設計了一套專門的 Terminal/File viewer 介面，讓 Agent 透過指令 (如 scroll down, search) 來探索大檔案，這是一種「主動檢索型」的記憶機制。
        
- **Paper:**
    
    - **RepoCoder** (Repository-Level Code Completion through Iterative Retrieval and Generation): 微軟提出的論文，展示如何透過不斷檢索 Repo 內的相關片段來補全程式碼。
        
    - **LlamaIndex 的 Code Hierarchy Node Parser:** 這不是 Paper，而是 LlamaIndex 內建的工具，專門用來把大檔案程式碼解析成有階層關係的節點存入 Memory


---


DR
**狀態快取與軌跡記憶 (State Cache & Trajectory Memory)**：

1. **Tool Execution Cache (工具快取記憶 - 解決你的具體痛點):**
    
    - **機制:** 在 Memory DB 中建立一個 Key-Value 區塊。
        
    - **Key:** hash(Tool_Name + Tool_Input_Arguments) (例如: hash("search" + "AI Memory papers"))。
        
    - **Value:** Tool 的執行結果 (JSON 或文本)。
        
    - **流程:** Agent Planning 產出 Subqueries (A/B/D) -> Memory Manager 攔截 -> 檢查 A/B 已存在快取 -> 直接返回結果給 Agent -> 只有 D 真的去執行 API。
        
2. **Trajectory / Episodic Memory (軌跡記憶):**
    
    - **機制:** 記錄 [Query -> Plan -> Tool -> Observation -> Thought] 的完整迴圈。
        
    - **用途:** 當 Agent 在 Loop 3 迷失時，它可以回顧：「我之前用關鍵字 X 查過了，沒有結果，我現在不該再查 X」。
        
3. **Knowledge Graph / Scratchpad (知識圖譜與草稿紙):**
    
    - **機制:** Research 過程中爬回來的長篇大論不能全放 History。必須有一個 Background Worker (另一個小的 LLM)，不斷將 Tool 返回的長文本**「總結 (Summarize) 或萃取成 Entity-Relation」**，寫入一個全局的 "Research Scratchpad"。

- **開源架構:**
    
    - **STORM (Stanford 開源):** 這是專門做 Deep Research 並寫出 Wikipedia 等級長文的系統。它的架構完美示範了兩階段 (Discovery -> Drafting) 的記憶管理，它在 Discovery 階段會不斷把搜到的資訊整合進內部知識庫 (Memory) 中。
        
    - **LangGraph:** 它的 "Checkpointer" (如 MemorySaver 或 PostgresSaver) 天生就支援紀錄每一個 Node (Tool/Agent) 的狀態。你可以輕易在裡面實作 Cache 機制。
        
- **學術文獻 (Paper):**
    
    - **Reflexion** (Reflexion: Language Agents with Verbal Reinforcement Learning): 教導 Agent 將過去失敗或成功的經驗轉化為文字 (Episodic Memory)，並在下一次 Loop 拿出來看，避免重複犯錯。
        
    - **ReAct** (Synergizing Reasoning and Acting): 基礎理論，所有的 Deep Research Loop 都是基於 ReAct 的 Thought-Action-Observation 軌跡，這條軌跡本身就是工作記憶的核心。


---
---

Hybrid Search：Vector Search (語意/模糊) + Keyword Search (關鍵字/精準, 例如 BM25)，再搭配 Metadata 過濾。

以下是針對三種 Agent 的 Hybrid Search 執行細節：

#### Chat Agent 

- **User 問:** 「昨天我跟你說我家的狗狗怎麼了？」
    
- **Vector Search 負責:** 找尋「寵物生病」、「帶狗看醫生」等語意相近的歷史對話。
    
- **Keyword Search 負責:** 鎖定「狗狗」這個具體名詞。
    
- **Metadata 負責:** 鎖定 timestamp 在昨天，source_type 為 user_input 的紀錄。
    

#### Coding Agent (修復 Bug)

- **User 問:** 「auth_service.py 裡面的 verify_token 報錯 NullPointerException。」
    
- **Keyword Search  (佔比極重):** 程式碼絕對不能找錯字！關鍵字必須**精確匹配**變數名稱 verify_token、檔案名 auth_service.py 和報錯訊息 NullPointerException。
    
- **Vector Search  (佔比較輕):** 尋找程式碼註解 (Docstring) 或過去有沒有類似「處理空值邏輯」的程式碼片段。
    
- **Graph/Metadata :** 找出這個檔案的依賴關係 (有哪些 import)。
    

#### Deep Research Agent (多輪搜尋)

- **Agent Internal Query (內部想查):** 「尋找 2023 年關於 Agent Memory 的開源框架」
    
- **Keyword Search:** 確保文本中有出現 "Agent", "Memory", "Open-source" 這些核心詞彙。
    
- **Vector Search:** 找出概念相似的結果，就算文章沒寫 "Memory" 而是寫 "Context Storage"，也能被抓出來。
    
- **Metadata:** session_id 匹配當前任務，且避免撈到 Agent 已經在 Loop 1 標記為 status: rejected_info 的廢棄資料。

---
---


Base Class + Strategies

Base Class
	BaseMemoryStore: 連線 Vector DB、連線 KV Cache、把傳進來的資料打上 Metadata 並存進去、執行 Hybrid Search

Strategies
	**ChatMemoryManager**: + Chat history
	**CodingMemoryManager**: 資料預處理 (AST)
		- 收到一個巨大的 .py 檔案時，不直接呼叫底層。
		- 先呼叫自己獨有的 AST_Parser() 把檔案切成一個個 Function。
		- 再把這些 Function 寫成 JSON (附上 function_name 作為 Metadata)，然後才呼叫底層的 save()。
	**ResearchMemoryManager**: 快取邏輯 (Tool Cache)
		- 在 Agent 準備呼叫 Tool 之前，先去底層的 KV Cache 查 hash(Tool_Name + Query) (這就是你提的 Loop1 查了 A/B/C，Loop2 查 A/B/D 的解決方案)。
		- 如果有，直接攔截並返回；如果沒有，再去執行 Tool，執行完呼叫底層 save()。



---
---


- **Ingestion & Metadata (完美契合):**  
    Mem0 原生支援強大的 Metadata 分隔。可以直接在 add() 的時候傳入 user_id, agent_id, session_id。
    mem0.add(text="GPT-4架構...", agent_id="research_01", metadata={"source": "web"})
    
- **Storage & Retrieval (混合檢索):**  
    底層預設使用 Qdrant 或 Chroma 等 Vector DB。它內建了語意搜索，並且可以透過 API 直接用 Metadata 進行精準過濾 (Pre-filtering)。
    
- **Consolidation (自動更新與遺忘):**  
    當你存入新記憶與舊記憶衝突時（例如：原先存「變數 A 是 Int」，後來存「變數 A 改成 String」），Mem0 內部會呼叫 LLM 進行 **Memory Reflection (記憶反思)**，自動更新或取代舊記憶，而不是單純疊加。
    

**缺點：**

- **沒有 Graph DB：** 若 Coding Agent 需要 AST 的關聯圖，Mem0 幫不上忙，你要自己外掛 Neo4j。-> mem0g
    
- **Tool Cache 大材小用：** 如果只是要攔截完全一樣的 API 呼叫，用標準的 Redis (Key-Value) 會比呼叫 Mem0 快得多


---
---

- **Short-term / Working Memory (短期/工作記憶):**  
    透過 LangGraph 原生的 Checkpointer (如 MemorySaver 或 PostgresSaver)，它能完美記錄最近 5 句話 (Chat)，或是 Research Agent 多輪 Loop 的每一次狀態 (Trajectory)。這直接解決了「維持當前任務上下文」的問題。
    
- **Consolidation (背景摘要與萃取):**  
    LangMem 提供了一套 MemoryManager 機制。當你的 Agent 跑完一個 Loop 或聊完一段天，LangMem 會觸發一個**背景任務 (Background Run)**。這個任務會讓 LLM 閱讀剛剛的短期對話/軌跡，提取出「事實 (Facts)」、「規則 (Rules)」，然後存進 Vector DB 變成「長期記憶」。
    
- **Retrieval:**  
    在 Agent 啟動下一步之前，LangGraph 的節點會先去 LangMem 查詢相關的 Facts 注入到 System Prompt 裡。
    

**缺點：**

- **綁定 LangChain/LangGraph 生態系：** 
- **不適合大量非結構化文本 (如 Coding)：** LangMem 的設計初衷是「對話與行為軌跡的記憶萃取」。如果你要把幾千行的程式碼或 AST 塞給它管理，它的表現不如專門處理 Document 的架構。