# Explainer for the Prompt API

_This proposal is an early design sketch by the Chrome built-in AI team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome._

Browsers and operating systems are increasingly expected to gain access to a language model. ([Example](https://developer.chrome.com/docs/ai/built-in), [example](https://blogs.windows.com/windowsdeveloper/2024/05/21/unlock-a-new-era-of-innovation-with-windows-copilot-runtime-and-copilot-pcs/).) Language models are known for their versatility. With enough creative [prompting](https://developers.google.com/machine-learning/resources/prompt-eng), they can help accomplish tasks as diverse as:

* Classification, tagging, and keyword extraction of arbitrary text;
* Helping users compose text, such as blog posts, reviews, or biographies;
* Summarizing, e.g. of articles, user reviews, or chat logs;
* Generating titles or headlines from article contents
* Answering questions based on the unstructured contents of a web page
* Translation between languages
* Proofreading

Although the Chrome built-in AI team is exploring purpose-built APIs for some of these use cases (e.g. [translation](https://github.com/WICG/translation-api), and perhaps in the future summarization and compose), we are also exploring a general-purpose "prompt API" which allows web developers to prompt a language model directly. This gives web developers access to many more capabilities, at the cost of requiring them to do their own prompt engineering.

Currently, web developers wishing to use language models must either call out to cloud APIs, or bring their own and run them using technologies like WebAssembly and WebGPU. By providing access to the browser or operating system's existing language model, we can provide the following benefits compared to cloud APIs:

* Local processing of sensitive data, e.g. allowing websites to combine AI features with end-to-end encryption.
* Potentially faster results, since there is no server round-trip involved.
* Offline usage.
* Lower API costs for web developers.
* Allowing hybrid approaches, e.g. free users of a website use on-device AI whereas paid users use a more powerful API-based model.

Similarly, compared to bring-your-own-AI approaches, using a built-in language model can save the user's bandwidth, likely benefit from more optimizations, and have a lower barrier to entry for web developers.

**Even more so than many other behind-a-flag APIs, the prompt API is an experiment, designed to help us understand web developers' use cases to inform a roadmap of purpose-built APIs.** However, we want to publish an explainer to provide documentation and a public discussion place for the experiment while it is ongoing.

## Goals

Our goals are to:

* Provide web developers a uniform JavaScript API for accessing browser-provided language models.
* Abstract away specific details of the language model in question as much as possible, e.g. tokenization, system messages, or control tokens.
* Guide web developers to gracefully handle failure cases, e.g. no browser-provided model being available.
* Allow a variety of implementation strategies, including on-device or cloud-based models, while keeping these details abstracted from developers.

The following are explicit non-goals:

* We do not intend to force every browser to ship or expose a language model; in particular, not all devices will be capable of storing or running one. It would be conforming to implement this API by always returning `"no"` from `canCreateTextSession()`, or to implement this API entirely by using cloud services instead of on-device models.
* We do not intend to provide guarantees of language model quality, stability, or interoperability between browsers. In particular, we cannot guarantee that the models exposed by these APIs are particularly good at any given use case. These are left as quality-of-implementation issues, similar to the [shape detection API](https://wicg.github.io/shape-detection-api/). (See also a [discussion of interop](https://www.w3.org/reports/ai-web-impact/#interop) in the W3C "AI & the Web" document.)

The following are potential goals we are not yet certain of:

* Allow web developers to know, or control, whether language model interactions are done on-device or using cloud services. This would allow them to guarantee that any user data they feed into this API does not leave the device, which can be important for privacy purposes. Similarly, we might want to allow developers to request on-device-only language models, in case a browser offers both varieties.
* Allow web developers to know some identifier for the language model in use, separate from the browser version. This would allow them to allowlist or blocklist specific models to maintain a desired level of quality, or restrict certain use cases to a specific model.

Both of these potential goals could pose challenges to interoperability, so we want to investigate more how important such functionality is to developers to find the right tradeoff.

## Examples

### Zero-shot prompting

In this example, a single string is used to prompt the API, which is assumed to come from the user. The returned response is from the assistant.

```js
const session = await ai.createTextSession();

// Prompt the model and wait for the whole result to come back.
const result = await session.prompt("Write me a poem.");
console.log(result);

// Prompt the model and stream the result:
const stream = await session.promptStreaming("Write me an extra-long poem.");
for await (const chunk of stream) {
  console.log(chunk);
}
```

### System prompts

The model can be configured with a special "system prompt" which gives it the context for future assistant/user interactions:

```js
const session = await ai.createTextSession({
  systemPrompt: "Pretend to be an eloquent hamster."
});

console.log(await session.prompt("What is your favorite food?"));
```

The system prompt  is special, in that the model will not respond to it, and it will be preserved even if the context window otherwise overflows due to too many calls to `prompt()`.

### N-shot prompting

If developers want to provide examples of the user/assistant interaction, they can use the `initialPrompts` array. This aligns with the common "chat completions API" format of `{ role, content }` pairs, including a `"system"` role which can be used instead of the `systemPrompt` option shown above.

```js
const session = await ai.createTextSession({
  initialPrompts: [
    { role: "system", content: "Predict up to 5 emojis as a response to a comment. Output emojis, comma-separated." },
    { role: "user", content: "This is amazing!" },
    { role: "assistant", content: "❤️, ➕" },
    { role: "user", content: "LGTM" },
    { role: "assistant", content: "👍, 🚢" }
  ]
});

// Clone an existing session for efficiency, instead of recreating one each time.
async function predictEmoji(comment) {
  const session = await session.clone();
  return await session.prompt(comment);
}

const result1 = await predictEmoji("Back to the drawing board");

const result2 = await predictEmoji("This code is so good you should get promoted");
```

(Using both `systemPrompt` and a `{ role: "system" }` prompt in `initialPrompts`, or using multiple `{ role: "system" }` prompts, or placing the `{ role: "system" }` prompt anywhere besides at the 0th position in `initialPrompts`, will reject with a `TypeError`.)

### Configuration of per-session options

In addition to the `systemPrompt` and `initialPrompts` options shown above, the currently-configurable options are [temperature](https://huggingface.co/blog/how-to-generate#sampling) and [top-K](https://huggingface.co/blog/how-to-generate#top-k-sampling). More information about the values for these parameters can be found by calling `textModelInfo()`.

```js
const customSession = await ai.createTextSession({
  temperature: 0.8,
  topK: 10
});

const modelInfo = await ai.textModelInfo();
const slightlyHighTemperatureSession = await ai.createTextSession({
  temperature: Math.max(modelInfo.defaultTemperature * 1.2, 1.0),
});

// modelInfo also contains defaultTopK and maxTopK.
```

### Session persistence, cloning, and destruction

Each session consists of a persistent series of interactions with the model:

```js
const session = await ai.createTextSession({
  systemPrompt: "You are a friendly, helpful assistant specialized in clothing choices."
});

const result = await session.prompt(`
  What should I wear today? It's sunny and I'm unsure between a t-shirt and a polo.
`);

console.log(result);

const result2 = await session.prompt(`
  That sounds great, but oh no, it's actually going to rain! New advice??
`);
```

Multiple unrelated continuations of the same prompt can be set up by creating a session and then cloning it:

```js
const session = await ai.createTextSession({
  systemPrompt: "You are a friendly, helpful assistant specialized in clothing choices."
});

const session2 = await session.clone();
```

A session can be destroyed to free some resources when it’s not needed anymore. After destroying it, it can’t be prompted anymore, and any ongoing prompt calls will be rejected with an `"InvalidStateError"` `DOMException`.

```js
session.destroy();

// The promise will be rejected with an error explaining that the session is destroyed.
await session.prompt(`
  What should I wear today? It's sunny and I'm unsure between a t-shirt and a polo.
`);
```

### Availability detection, download progress, and error handling

In all our above examples, we call `ai.createTextSession()` and assume it will always succeed.

However, sometimes a language model needs to be downloaded before the API can be used. In such cases, immediately calling `createTextSession()` will start the download, which might take a long time. You can give your users a better experience by detecting this condition using `canCreateTextSession()`, which returns one of three values:

* `"no"`, indicating the device or browser does not support prompting a language model at all.
* `"after-download"`, indicating the device or browser supports prompting a language model, but it needs to be downloaded before it can be used.
* `"readily"`, indicating the device or browser supports prompting a language model and it’s ready to be used without any downloading steps.

In the `"after-download"` case, developers might want to have users confirm before you call `createTextSession()` to start the download, since doing so uses up significant bandwidth and users might not be willing to wait for a large download before using the site or feature.

Note that regardless of the return value of `canCreateTextSession()`, `createTextSession()` might also fail, if either the download fails or the session creation fails.

When a download has been started by `createTextSession()`, the `ai` object will emit `textmodeldownloadprogress` events, which are of type `ProgressEvent`. They can be used to display a progress bar or similar.

See [Downloading and session creation flow](#downloading-and-session-creation-flow) for the full details on the internal state machine.

```js
const canCreate = await ai.canCreateTextSession();

switch (canCreate) {
  case "no": {
    console.log("This browser/device cannot provide a language model.");
    break;
  }
  case "after-download": {
    console.log("Going to download a language model; sit tight!");
    ai.addEventListener("textmodeldownloadprogress", e => {
      console.log(`Download progress: ${e.loaded} / ${e.total} bytes.`);
    });
    break;
  }
  case "readily": {
    console.log("The language model is already downloaded; creating it will be quick.");
  }
}

// * If canCreate was "no", this will throw a "NotSupportedError" DOMException.
//
// * If canCreate was "after-download", this will kick off the download and only fulfill
//   after it completes, or reject if it fails or the session creation fails.
//
// * If canCreate was "readily", this will fulfill or reject relatively quickly, based
//   on whether session creation succeeds or fails.
let session;
try {
  session = await ai.createTextSession();
  console.log("Creating the session succeeded!");
} catch (e) {
  console.error("Creating the session failed, either in the downloading or session " +
                "creation steps. More details: ", e);
}

// Now prompt it as usual...
```

## Detailed design

### Full API surface in Web IDL

```webidl
partial interface WindowOrWorkerGlobalScope {
  [Replaceable] readonly attribute AI ai;
}

[Exposed=(Window,Worker)]
interface AI {
  Promise<AIModelAvailability> canCreateTextSession();
  Promise<AITextSession> createTextSession(optional AITextSessionOptions options = {});

  attribute EventHandler ontextmodeldownloadprogress;

  Promise<AITextModelInfo> textModelInfo();
};

[Exposed=(Window,Worker)]
interface AITextSession {
  Promise<DOMString> prompt(DOMString input);
  ReadableStream promptStreaming(DOMString input);

  readonly attribute unsigned long topK;
  readonly attribute float temperature;

  Promise<AITextSession> clone();
  undefined destroy();
};

dictionary AITextSessionOptions {
  [EnforceRange] unsigned long topK;
  float temperature;
  DOMString systemPrompt;
  sequence<AIPrompt> initialPrompts;
};

dictionary AIPrompt {
  AIPromptRole role;
  DOMString content;
};

dictionary AITextModelInfo {
  unsigned long defaultTopK;
  unsigned long maxTopK;
  float defaultTemperature;
};

enum AIModelAvailability { "readily", "after-download", "no" };
enum AIPromptRole { "system", "user", "assistant" };
```

### Instruction-tuned versus base models

We intend for this API to expose instruction-tuned models. Although we cannot mandate any particular level of quality or instruction-following capability, we think setting this base expectation can help ensure that what browsers ship is aligned with what web developers expect.

To illustrate the difference and how it impacts web developer expectations:

* In a base model, a prompt like "Write a poem about trees." might get completed with "... Write about the animal you would like to be. Write about a conflict between a brother and a sister." (etc.) It is directly completing plausible next tokens in the text sequence.
* Whereas, in an instruction-tuned model, the model will generally _follow_ instructions like "Write a poem about trees.", and respond with a poem about trees.

To ensure the API can be used by web developers across multiple implementations, all browsers should be sure their models behave like instruction-tuned models.

### Downloading and session creation flow

The state machine for the model downloading and session creation is as follows.

There is a per-global _session creation requested_: a boolean, initially false.

There is a per-user agent _model availability state_. It moves through the following states:

* "cannot provide a model"
  * `canCreateTextSession()` returns `"no"`
  * `createTextSession()` rejects with a `"NotSupportedError"` `DOMException`
* "can provide a model after download"
  * `canCreateTextSession()` returns `"after-download"`
  * `createTextSession()`:
    * Checks argument validity, returning a rejection if invalid. Otherwise:
    * Sets _session creation requested_ = true.
    * Transitions us to the "downloading a model" state.
    * Queues a _create a session_ operation when that finishes, passing the download status.
* "downloading a model"
  * `canCreateTextSession()` returns `"after-download"`
  * `createTextSession()`:
    * Checks argument validity, returning a rejection if invalid. Otherwise:
    * Sets _session creation requested_ = true.
    * Queues a _create a session_ operation for when the ongoing download finishes, passing the download status.
  * While in this state, if _session creation requested_ is true, the `AI` object fires `textmodeldownloadprogress` events on download progress updates.
* "model fully available"
  * `canCreateTextSession()` returns `"readily"`
  * `createTextSession()`:
    * Checks argument validity, returning a rejection if invalid. Otherwise:
    * Starts a _create a session_ operation immediately, passing a download status of success.
    * (Does _not_ set _session creation requested_.)

_Create a session_ operation: takes as input a download status

* If download status = failure,
  * Rejects the relevant promise with a `"NetworkError"` `DOMException`
  * Sets _model availability state_ back to "can provide a model after download"
* If download status = success,
  * Set _model availability state_ to "model fully available"
  * Try to use the model to create a session:
    * If session creation fails, reject the relevant with various possible errors depending on what type of failure happened.
    * If session creation succeeds, resolve the relevant promise with undefined.
* Set _session creation requested_ = false.

## Alternatives considered and under consideration

### Naming

We don't love the current naming of the API. Especially, the names using "text" would become confusing if the same API eventually gains multi-modal capabilities.

We've started a discussion on the issue tracker to explore alternatives: see [issue #1](https://github.com/explainers-by-googlers/prompt-api/issues/1).

### How many stages to reach a response?

To actually get a response back from the model given a prompt, the following possible stages are involved:

1. Download the model, if necessary.
2. Establish a session, including configuring [per-session options](#configuration-of-per-session-options).
3. Add an initial prompt to establish context. (This will not generate a response.)
4. Execute a prompt and receive a response.

We've chosen to manifest these 3-4 stages into the API as two methods, `createTextSession()` and `prompt()`/`promptStreaming()`, with some additional facilities for dealing with the fact that `createTextSession()` can include a download step. Some APIs simplify this into a single method, and some split it up into three (usually not four).

### Stateless or session-based

Our design here uses [sessions](#session-persistence-cloning-and-destruction). An alternate design, seen in some APIs, is to require the developer to feed in the entire conversation history to the model each time, keeping track of the results.

This can be slightly more flexible; for example, it allows manually correcting the model's responses before feeding them back into the context window.

However, our understanding is that the session-based model can be more efficiently implemented, at least for browsers with on-device models. (Implementing it for a cloud-based model would likely be more work.) And, developers can always achieve a stateless model by using a new session for each interaction.

## Privacy considerations

If cloud-based language models are exposed through this API, then there are potential privacy issues with exposing user or website data to the relevant cloud and model providers. This is not a concern specific to this API, as websites can already choose to expose user or website data to other origins using APIs such as `fetch()`. However, it's worth keeping in mind, and in particular as discussed in our [Goals](#goals), perhaps we should make it easier for web developers to know whether a cloud-based model is in use, or which one.

If on-device language models are updated separately from browser and operating system versions, this API could enhance the web's fingerprinting service by providing extra identifying bits. Mandating that older browser versions not receive updates or be able to download models from too far into the future might be a possible remediation for this.

Finally, we intend to prohibit (in the specification) any use of user-specific information that is not directly supplied through the API. For example, it would not be permissible to fine-tune the language model based on information the user has entered into the browser in the past.

## Stakeholder feedback

* W3C TAG: not yet requested
* Browser engines and browsers:
  * Chromium: prototyping behind a flag
  * Gecko: not yet requested
  * WebKit: not yet requested
  * Edge: not yet requested
* Web developers: positive ([example](https://x.com/mortenjust/status/1805190952358650251), [example](https://tyingshoelaces.com/blog/chrome-ai-prompt-api))
