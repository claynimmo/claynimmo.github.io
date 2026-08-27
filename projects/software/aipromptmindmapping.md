---
title: Portfolio |  AI Prompt Mindmapping
---

[Home](../../../index.md) / [Software Projects](index.md) /

# AI Prompt Mindmapping
The app was developed to gain experience using golang, and integrating code with an external API. The frontend was developed using flutter, that sent http requests to the golang server, that forwarded a prompt to a locally hosted ollama LLM model. The app isolates each prompt into its own node, allowing the context of the next prompt to be fully customized. This was done using a directed acrylic graph (DAG), using graph search algorithms to reconstruct the context.