---
title: Use AI to understand customer conversations
subtitle: Call centres, audio transcription and NLP
layout: post
author: martoc
image: https://martoc.london/blog/images/use-ai-to-understand-customer-conversations.png
---

I attended this workshop today, I learnt that AWS provides a full stack of
services that can help you to integrate your call centre calls or even create
your own call centre and from the recording from these sources you can get real
information about these conversations, Amazon Connect, Transcribe, Comprehend,
Athena and QuickSight were some of the main services I used in this workshop.

In this example, I created the call centre using Amazon Connect, the following
diagram shows the main components and how these interact with each other.

![Architecture](/blog/images/use-ai-to-understand-customer-conversations.png)

From my point of view the transcription is the hack that allows us to use our
current knowledge in NLP which is completely based on written text, I guess in
the next year the processing will be direct from audio.

One nice feature that I wasn't aware of is that QuickSight now support more
datasource types, in this particular case it can read from S3 and show insights,
these are the current dataset that it supports

![QuickSight](/blog/images/use-ai-to-understand-customer-conversations-quicksight.png)
