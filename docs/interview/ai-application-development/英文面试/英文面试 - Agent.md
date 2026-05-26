## 如何提升Agent的准确率？

**First, prompt optimization.** I add chain-of-thought in prompt so the Agent has to think before acts. I also enforce a strict JSON output format — that makes it easy for code to validate the result. And I ask the Agent to do self-check before it returns the final answer.

**Second, architecture optimization with ReAct.** By going through a loop of thinking, acting, and observing, the Agent gets much better at complex tasks. For multi-step tasks, I also combine plan-and-execute with ReAct — the LLM creates a full plan first, then executes step by step.

**Third, tool call optimization.** I refine tool descriptions to include clear parameter templates, specific scenarios, and examples. I also limit how much content a tool can return, so the context doesn't get blown out of proportion.

**Fourth, context management.** For long-term memory, I summarize and compress it, then store it in RAG. Short-term memory stays intact. Important entities from the conversation are cached separately, and user preferences go into MySQL.

## 如何提升Agent的实时性？
**1. Layered architecture.** I'd set up some common preset rules — if a request matches one, it just goes through the preset flow. For simple questions or tool calls, I'd use a lightweight model for inference. Only for complex reasoning steps would I call a more powerful LLM. I'd also cache answers to frequently asked questions directly.

**2. Preloading.** While the user is still typing, I'd pre‑identify their intent and preload the relevant tools, then show them to the user so they can just click.

**3. Streaming output.** Using SSE, as soon as a token is generated, it gets pushed to the frontend and displayed on the page.

**4. Asynchronous processing.** When the user submits a question, the backend processes it asynchronously — first returning something like 'Processing, please wait', then calling the tools in the background, and pushing the final result to the user when it's ready.

**5. Timeout fallback.** If a response still takes too long, I'd set a timeout. After that point, I'd return a fallback message, log the timeout event, and investigate the cause later.

## 如何提升Agent系统的健壮性？

**1. Input layer**  
I'd normalize user input before sending it to the LLM — removing irrelevant noise. I'd also filter and escape potential prompt injection attacks, and add anti‑injection instructions in the system prompt. Plus, I'd maintain a blacklist of prohibited words and filter inputs for safety.

**2. Reasoning layer**  
When the model's confidence in understanding the user's intent is below a certain threshold, I'd let it ask for clarification instead of guessing. I'd also build a fallback plan for every plan. If a timeout or a 500 error happens, I'd retry first. If that fails, I'd switch to the fallback plan. And if there's no fallback, I'd still have a safe default response — never just throw an error.

**3. Tool call layer**  
I'd make sure tool calls are idempotent — repeating them multiple times shouldn't cause side effects. Inside each tool, I'd validate parameters; if something doesn't meet the requirements, the tool returns a clear error. I'd also catch all exceptions in the tool and convert them into a return value that the LLM can understand. Finally, I'd set timeouts for tools. If timeouts or failures exceed a threshold, I'd trigger a circuit breaker — fail fast to avoid bringing down the whole system.

**4. Context management**  
When the conversation history gets close to the length limit, I'd summarize the earlier parts so no important information is lost.

**5. Monitoring and self‑healing.**  
I'd set up real‑time monitoring for Q&A success rate, processing time, and errors — with alerts when thresholds are crossed. I'd also save periodic snapshots of multi‑turn conversations, so if the service recovers from a failure, it can continue processing where it left off."

## 你们是怎么做记忆管理的？

First, we take the most recent K rounds of user questions and answers and put them directly into the prompt as is.

For short‑term hot memory, we save it in a cache for fast retrieval. If it hasn't been accessed for a certain amount of time, we just delete it.

For long‑term memory, we store it in RAG. When we get a user question, we first go to RAG and retrieve relevant memories with the condition `active = true`. If we retrieve multiple conflicting memories, we only keep the one with the latest timestamp.

Every time the LLM generates a response, we summarize and extract key points from it, and store it in RAG with a timestamp. Before we store it, we first check whether a similar memory already exists in RAG. If the meaning is same, we merge them. If there's a conflict, we mark the old memory as expired and save the new one.

We also run a scheduled task to clean up and merge memories in RAG. We score each memory based on how often it's been retrieved, its relevance to the user, and its timeliness. Low‑scoring memories get deleted. And if we find similar memories, we merge them together.

## Agent和传统LLM的区别是什么？

The difference between an Agent and an LLM, I think there are two differences.

First, an Agent has autonomy. That means it can understand a goal on its own, break it down into steps, decide which tools to call, and figure out whether the task is done or not.

Second, an Agent doesn't just answer questions — it can actually call external tools and interact with the environment.

## 如何建立一个有效的评估机制？

I'd design an Agent evaluation mechanism from three levels.

**First level, figure out what we need to evaluate.**

1. **Outcome metrics** — whether the Agent's final answer is accurate and meets the user's needs, and whether it's making things up.
    
2. **Process metrics** — whether the planning was reasonable, whether it picked the right tools, and whether there were any unnecessary repeated calls.
    
3. **Stability metrics** — whether the Agent can finish within the time limit, including success rate, timeout rate, and unexpected interruption rate.
    

**Second level, build an automated evaluation pipeline.**  
First, I'd create a golden test set with real business scenarios — not just normal cases, but also edge cases, adversarial scenarios, and historical failure cases.  
Every time we update the Agent, we run this test set automatically. We define rules to automatically calculate P95 latency, interruption rate, and format error rate.  
For metrics that are hard to judge automatically — like RAG relevance score, answer accuracy, and hallucination rate — we use **LLM-as-a-Judge**, having a stronger, more stable model do the scoring, with regular spot checks by humans.  
Before going live, we also run A/B testing to make sure the new version doesn't perform worse than the old one.

**Third level, and the most valuable one in my opinion — close the loop from evaluation to optimization.**  
We add instrumentation at every step in production. When a failure happens, we tag it — was it a planning error, a tool call error, or a hallucination?  
Then we look at the distribution of those tags and focus our optimization on the steps with the most failures.  
At the same time, we add those failure cases back into the golden test set. That way, the fix ensures that specific failure won't happen again, and the test set gets more robust over time.