---
title: Direct Line Speech - Speech service
titleSuffix: Azure Cognitive Services
description: An overview of the features, capabilities, and restrictions for Voice assistants using Direct Line Speech with the Speech Software Development Kit (SDK).
services: cognitive-services
author: laujan
manager: nitinme
ms.service: cognitive-services
ms.subservice: speech-service
ms.topic: conceptual
ms.date: 03/11/2020
ms.author: lajanuar
---

# What is Direct Line Speech?

**Direct Line Speech** is a robust, end-to-end solution for creating a flexible, extensible voice assistant. It is powered by the Bot Framework and its Direct Line Speech channel, that is optimized for voice-in, voice-out interaction with bots.

[Voice assistants](voice-assistants.md) listen to users and take an action in response, often speaking back. They use [speech-to-text](speech-to-text.md) to transcribe the user's speech, then take action on the natural language understanding of the text. This action frequently includes spoken output from the assistant generated with [text-to-speech](text-to-speech.md).

Direct Line Speech offers the highest levels of customization and sophistication for voice assistants. It's designed for conversational scenarios that are open-ended, natural, or hybrids of the two with task completion or command-and-control use. This high degree of flexibility comes with a greater complexity, and scenarios that are scoped to well-defined tasks using natural language input may want to consider [Custom Commands](custom-commands.md) for a streamlined solution experience.

## Getting started with Direct Line Speech

The first steps for creating a voice assistant using Direct Line Speech are to [get a speech subscription key](overview.md#try-the-speech-service-for-free), create a new bot associated with that subscription, and connect the bot to the Direct Line Speech channel.

   ![Conceptual diagram of the Direct Line Speech orchestration service flow](media/voice-assistants/overview-directlinespeech.png "The Speech Channel flow")

For a complete, step-by-step guide on creating a simple voice assistant using Direct Line Speech, see [the tutorial for speech-enabling your bot with the Speech SDK and the Direct Line Speech channel](tutorial-voice-enable-your-bot-speech-sdk.md).

We also offer quickstarts designed to have you running code and learning the APIs quickly. This table includes a list of voice assistant quickstarts organized by language and platform.

| Quickstart | Platform | API reference |
|------------|----------|---------------|
| C#, UWP | Windows | [Browse](/dotnet/api/microsoft.cognitiveservices.speech) |
| Java | Windows, macOS, Linux | [Browse](/java/api/com.microsoft.cognitiveservices.speech) |
| Java | Android | [Browse](/java/api/com.microsoft.cognitiveservices.speech) |

## Sample code

Sample code for creating a voice assistant is available on GitHub. These samples cover the client application for connecting to your assistant in several popular programming languages.

* [Voice assistant samples (SDK)](https://aka.ms/csspeech/samples/#voice-assistants-quickstarts)
* [Tutorial: Voice enable your assistant with the Speech SDK, C#](tutorial-voice-enable-your-bot-speech-sdk.md)

## Customization

Voice assistants built using Speech service can use the full range of customization options available for [speech-to-text](speech-to-text.md), [text-to-speech](text-to-speech.md), and [custom keyword selection](./custom-keyword-basics.md).

> [!NOTE]
> Customization options vary by language/locale (see [Supported languages](./language-support.md)).

Direct Line Speech and its associated functionality for voice assistants are an ideal supplement to the [Virtual Assistant Solution and Enterprise Template](/azure/bot-service/bot-builder-enterprise-template-overview). Though Direct Line Speech can work with any compatible bot, these resources provide a reusable baseline for high-quality conversational experiences as well as common supporting skills and models for getting started quickly.

## Reference docs

* [Speech SDK](./speech-sdk.md)
* [Azure Bot Service](/azure/bot-service/)

## Next steps

* [Get a Speech service subscription key for free](overview.md#try-the-speech-service-for-free)
* [Get the Speech SDK](speech-sdk.md)
* [Create and deploy a basic bot](/azure/bot-service/bot-builder-tutorial-basic-deploy)
* [Get the Virtual Assistant Solution and Enterprise Template](https://github.com/Microsoft/AI)
