# Differences Between the Chrome Implementation and the Explainer

The [explainer](./README.md) focuses on the design we're aiming to build. The current implementation in Chrome is being actively developed. It does not yet include all that functionality, and has several important differences. This document lists those differences with links to appropriate tracking bugs.

The Chromium bug component for this API is [Chromium > Blink > AI > Prompt](https://issues.chromium.org/issues?q=componentid:1583624).

## Overall API surface

_Tracking issue: <https://issues.chromium.org/issues/355967885>_

The overall API surface of the proposal changed significantly in [#25](https://github.com/explainers-by-googlers/prompt-api/pull/25). These changes have not yet been implemented in Chromium.

## Streaming

_Tracking issue: <https://issues.chromium.org/issues/342859651>_

Currently in Chromium, `promptStreaming()` returns a `ReadableStream` whose chunks successively build on each other. So code like

```js
for await (const chunk of stream) {
  console.log(chunk);
}
```

will log something like `"Hello"`, `"Hello world"`, `"Hello world I am"`, `"Hello world I am an AI"`.

This is not intended behavior. We hope to eventually make this stream behave the same as other streaming APIs on the platform, where the chunks are successive pieces of a single long stream. In the future, the output will become something like `"Hello"`, `" world"`, `" I am"`, `" an AI"`.

## Automatic downloading and download progress

_Tracking issue: <https://issues.chromium.org/issues/342859654>_

Automatically downloading the model on first use, and getting download progress events, is not supported. Instead, Chrome will trigger the download, either as part of changing the relevant chrome://flags value, or because of another on-device AI feature.

## System prompts and initial prompts

_Tracking issue: <https://issues.chromium.org/issues/343325183>_

The `systemPrompt` and `initialPrompts` options are not yet supported. Instead, you have to manually emulate these by using special control tokens in the middle of your prompt.

## `topK` and `temperature` both required

_Tracking issue: <https://issues.chromium.org/issues/343600797>_

Currently Chrome requires that you supply both `topK` and `temperature` if you are going to include either; you cannot just include one and leave the other at its default value.
