---
layout: page
title: About
permalink: /about/
---

I work on distributed systems, platform infrastructure, and reliability engineering. Seven years across Microsoft R&D India and Amazon Development Centre, building the kind of systems that are boring in the best possible way — ones that handle load quietly and fail gracefully.

Recently I've been looking at the gap between ML research benchmarks and what most people actually deploy on. The two aren't the same hardware tier, and the problems that emerge at the constrained end aren't well-studied.

**[Clairvoyant](https://arxiv.org/abs/2606.07248)** is a lightweight SJF scheduling proxy for serial LLM backends — Ollama, llama.cpp — that reduces short-request tail latency by 68–76% without modifying the backend. It came out of noticing that head-of-line blocking is the default state on any hardware that can't run vLLM, which is most hardware.

This blog is where I write about infrastructure and serving problems I find interesting.

---

*GitHub: [Aravind0403](https://github.com/Aravind0403)*  
*arXiv: [2606.07248](https://arxiv.org/abs/2606.07248)*  
*Email: aravindsharma20@gmail.com*
