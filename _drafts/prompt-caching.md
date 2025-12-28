---
layout: post
title: On Prompt Caching
category: general
---

> LLM Usage?
> 

Recently I came across a [blog post on prompt caching](https://ngrok.com/blog/prompt-caching), which lit up my interest in the topic again. I want to use this blog post to revisit some of the concepts and think about how prompt caching may be integrated into inference architectures and what considerations there are when using it in applications.

## The idea behind prompt caching.

Prompt caching is a way of offering KV Caching over multiple requests to a model inference API. I still get flashbacks from meticously writing out transformer calculations in university, which is why would like to avoid details here. Also there many sources that already did that (and arguably better than I could).

I will try to summarize the idea behind KV Caching shortly. If you want to go deeper, I will link some great blog posts below:

So before reading: Just speaking from my mind:
- For each step and transformer layer in the token generation, we need to compute the attention for the all previous tokens in the sequence.
- The attention mechanism consists of a query, key and value matrices (along with some scaler operations)
- The key (K) and value (V) matrices are mostly re-usable. In the naive way, all operations for step `t-1` are the same in timestep `t` (with one extra row for the new token).
- The K,V matrices can be cached between generations, improving throughput and in some cases latency.
- I am 99% sure that all model providers use KV Caching under the hood.
- Prompt Caching is essentially just the option to re-use that cache over multiple requests rather than discarding it after one generation is finished.

Great posts about the idea behind prompt/KV caching (both link even more great posts):
- [Prompt caching: 10x cheaper LLM tokens, but how?](https://ngrok.com/blog/prompt-caching) (really great visuals, suitable for beginners)
- [HF - KV Caching Explained](https://huggingface.co/blog/not-lain/kv-caching) (implementation sample included, Huggingface blogs are always recommended)
- [Prompt Caching Explained](https://sankalp.bearblog.dev/how-prompt-caching-works/) (technical deep dive)

## Prompt Caching as a Distributed System Design Challenge

## Application-Level Considerations for Prompt Caching





The interesting and unknown part to me is: how do you make sure that the user hits the cache on the machine with the cache? I imgine load balancing to play a huge role in serving LLMs and being applied pretty aggresively. So sticky sessions would therefore be difficult, as you might run into situations where the machine where your cache is stored is processing a different and very long request.



## I was wrong
We do not store the computation of the attention weights directly (or their combination with the values). Instead, we know that most of the `K` matrix and the `V` matrix stays the same That way we only need to put in the single new vector instead of the new token rather than the whole sequence.

## Other insights I gained
I never really explore prompt caching too much as I did not have the opportunity to apply it yet (other things where more important). So here are a few things I dnd not know before:
- Caches are stored for around 5-10 Minutes before being deleted
- You can match on partial cache matches, using the longest continous prefix match.
- Different providers offer different ways to use it (e.g. automatic vs manually)