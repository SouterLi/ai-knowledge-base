#### 1.提升RAG准确率（accuracy）
1.document preprocessing。
data cleaning, remove outdated document, inrelavant characters, duplicate contents. standardize dates, units, and formats.

2.chunking.
For documents with clear paragraph structures, like contracts, we first split by sections, then by sub-sections, until each chunk is within 500 characters. We try to avoid cutting inside the same paragraph. For long paragraphs, we generate a summary and extract a few related questions, then vectorize everything together.

3.**Question Rewriting**
rewrite user question. convert casual expressions into more standard and clear descriptions, and we use the conversation history to clarify what pronouns like 'this', 'that', or 'it' are referring to.

4.retrieval
On the retrieval side, we use **hybrid retrieval** to pull back the top 100 chunks first. After that, we **re-rank** them and pick the top 5 that are most relevant to the question. Then, before building the final prompt for context, we do some **compression and rewriting** on those chunks — basically cleaning them up and making them more concise. Once that's done, we pass the final prompt to the LLM."

#### 2.如何评估一个RAG？
So, to build a good evaluation mechanism for a RAG system, I usually think about it from three angles: **what to evaluate**, **how to evaluate**, and **how to keep improving based on the results**.  

First, there are four dimensions I care about.
- Retrieval relevance: does the retrieved chunk actually contain what the user is asking about?
- Generation faithfulness: does the model's answer stick to the retrieved content, or does it make things up?
- Answer usefulness: does the answer actually help the user solve their problem?
- End-to-end latency: how fast is the response?

Then, we set up a three‑layer evaluation pipeline.  
Layer one is automated testing. We build a golden test set and run it automatically. We use metrics like Recall@K, MRR, and response time to quickly check retrieval relevance and latency.  
Layer two is LLM-as-Judge. We use a stronger model to rate faithfulness and usefulness.  
Layer three is human evaluation. For edge cases and major releases, we still bring humans in to make sure the real experience is solid.

And finally, we close the loop. We don't just collect numbers — we analyze every failure, categorize it, and make targeted fixes. We also add instrumentation on production to capture real‑world failure cases, and then add those cases back into our test set. This way, we gradually improve both the system's performance and the test set's coverage over time.

#### 3.如何保证文档更新后RAG可以检索最新的数据？

We handle it from four angles: **detection**, **update**, **query**, and **fallback**.

**First, detection.** We watch document uploads and database change logs. If we see an update or deletion, we immediately trigger an update event and push it to a message queue for processing.

**Second, update.** When an update happens, we don't just delete the old vectors right away. We first mark them as 'expired' so they get ignored during retrieval. Only after the new vectors are successfully in place do we release the old ones.

**Third, query.** During the query process, after we pull the candidate chunks but before re-ranking, we check the real‑time latest version from the database using the document ID. If something doesn't match, we trigger another update message. Also, before generating the final answer, we double‑check for any contradictions in the retrieved knowledge to make sure the response is consistent.

**Fourth, fallback.** We track how many times expired documents get retrieved — if that number gets too high, another alert goes off.

On the user side, we offer a content correction button. If the answer doesn't match what's actually in the document, users can report it, which helps us catch issues early.

#### 4.RAG如何保障信息安全？
**1. Role-based access control**  
We've set up different permission levels for operators — for example, administrators, reviewers, and regular operators. Each role has different access rights. Regular operators can only query documents they are authorized to see. For sensitive documents, users need approval before they can view the details.

**2. Prompt injection prevention**  
We apply strict input validation to any user‑provided context before passing it to the model. This helps prevent malicious prompt injection attacks.

**3. Desensitization**  
Before calling the large language model, we desensitize any sensitive information. For example, we replace real names with placeholders like 'Party A'.

**4. Agent tool permission validation**  
When an agent calls a tool, we first check whether the user has permission to access the relevant document, and then verify whether the user is authorized to invoke that specific tool.