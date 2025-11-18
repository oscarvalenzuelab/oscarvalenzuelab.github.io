---
layout: post
permalink: /2025/11/how-to-use-ai-without-wasting-time-and-money.html
title:  "How to Use AI Without Wasting Time and Money"
author: oscar
categories: [ ai, development, bestpractices ]
tags: [ai, productivity, development, bestpractices, llm, agentic]
image: images/posts/ai-practical-use.png
description: "Practical lessons on using AI effectively in software development without falling into common traps and false expectations."
featured: true
comments: false
---

After thinking about it for a long time (very long time), I realized the "AI problem" isn't new. It's the same old problem we've seen in software for decades.

AI gives you extra capacity. It speeds up your work, helps you move from an idea to a quick demo or a basic product, and reduces the time between each step. But the same speed also increases the number of mistakes. If your process was unclear before, AI will spread that confusion across every step.

Most AI projects don't fall apart because the tech is weak. The tech is here. Many projects fall apart for the same reasons teams have failed with past tools:

- No training
- No real understanding of the tool
- Trying to solve the wrong problem
- False expectations pushed by hype

People expect AI to work like magic. They feed it missing context, unclear goals, or random data and expect it to "figure it out." That never works.

Still, every misstep teaches you something. Every prompt that doesn't work tells you what you misunderstood. And every rough draft helps you shape your thinking. These are the lessons that helped me the most while building with AI.

## 1. Learn the basics before putting AI into real work

You don't need deep technical knowledge, but you should still understand a few core ideas. It helps to know what a prompt is, what makes one clear or vague, and why the model reacts the way it does. It also helps to understand what RAG means, when it helps, and when it is unnecessary. The same goes for agents. A little preparation goes a long way, and skipping this early learning step usually leads to wasted money, wasted time, and a lot of confusion.

## 2. Expect AI to be wrong, and expect yourself to be wrong too

You'll get more value out of AI if you treat it as a partner that challenges your thinking instead of a tool that writes perfect answers. When you write a draft, try asking the model whether your argument actually addresses the real problem. Ask whether your reasoning makes sense and whether you're overlooking something important. This works much better than simply asking it to clean up grammar. You get clarity, not just pretty text.

## 3. Keep AI systems small

Large AI systems with too many features usually collapse under their own weight. They lose context, take too long to build, and fail quietly. Smaller projects do the opposite. A small, clear use case gives you fast results, shows you where the gaps are, and helps you stay focused. The most helpful AI tools are the simple ones that solve a real problem someone faces every day.

## 4. Build simple versions first and release them quickly

Classic software cycles move slowly, and AI work moves fast. If you wait months to ship something, the idea is already old. It's far better to release a simple version, try it, fix what breaks, and release the next version soon after. When you don't release small pieces, you end up working hard without making any real progress, as if you were pushing the brake and the gas at the same time.

## 5. AI does not know your goals

A model does not understand how your business works, what matters to you, or how you make decisions. You have to teach it. This is a good opportunity to map your own work. When you write down your process, your rules, and what a good outcome looks like, you not only help the model but you help yourself. Clear context always produces better results than any prompt trick.

## 6. Do not replace people or processes

AI should support the way people work, not erase it. It can help you draft early versions of documents, sort information, pull out key ideas, and record recurring steps so you can turn them into playbooks. But people should still make decisions. And before you connect an AI system to anything serious, especially a production database, stop and think about what could go wrong. Treat this as a risk question, not a convenience question.

## 7. If you plan to use AI in your business, define a risk model

Different AI systems carry different levels of risk. A simple text classifier is not the same as a model that writes code. And a local model that stays within your network is not the same as a frontier model with broad reach. One way to think about this is to divide systems into several levels. At the lowest level, the concerns look similar to the usual software supply chain issues teams already understand. The next level might include smaller text tasks or basic predictions. Beyond that are systems that can produce code or guide decisions. At the top are advanced models with broad behavior and harder-to-predict outcomes.

As you move up each level, you should raise the bar for security, testing, documentation, deployment limits, and monitoring. This keeps you from treating every AI system as if it all carries the same weight.

## 8. Start small and grow from there

If you prefer workflows that behave predictably and only pull AI in as a helper, take a look at Strands Agents. When combined with a local Ollama setup, it lets you experiment without sending your data anywhere. It even works well when you're offline on a plane. Once you're comfortable, you can shift to cloud providers or bigger systems later.

## Final thought

AI is just a tool. When it breaks, it's almost always because the person asked the wrong thing, used the wrong tool, or started with a problem that wasn't defined in the first place. Start small. Move fast. Give the model the information it needs. And make sure the thing you're trying to solve is real.

Have fun with AI, but stay grounded. Focus your time and money on the parts that give you results now, and grow from there.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
