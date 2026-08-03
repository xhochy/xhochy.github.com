---
layout: post
title: >
  The short hype of Inkling-Small or why I need automation to blog about the latest LLM developments
feature_image: "/images/bank-phrom-Tzm3Oyu_6sk-unsplash.jpg"
image: "/images/bank-phrom-Tzm3Oyu_6sk-unsplash.jpg"
---

On Thursday morning, I was excited to see [the release of Inkling-Small](https://thinkingmachines.ai/news/inkling-small/).
Last week brought us [Laguna-S-2.1](https://uwekorn.com/2026/07/23/laguna-s-2.1-seems-to-look-pretty-good.html) which is a pretty good coding model given that it fits onto a 128GB RAM MacBook Pro.
Sadly, Laguna-S-2.1 has the downside of overthinking, especially when it isn't a purely agentic coding task.
This was already visible in the automatic, local code reviews I do with [roborev](https://roborev.io).

Thus, I had the clear need for a second, more general-purpose local model.
It seemed like Inkling-small could be just that.
But then, 1h before I started writing this post, [DeepSeek-V4-Flash-0731](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731) dropped.
At the time of writing, we don't yet have quantisations available that fit on my MacBook.
But we already know that there will be ones and given [antirez/ds4](https://github.com/antirez/ds4), there is also pretty good inference support available.

## Should I still blog & test Inkling-Small? 
At the moment, we only have a day of people testing Inkling-Small in the real world and a single digit of hours of DeepSekk-V4-Flash-0731 available.
Thus, we cannot realistically expect to know from long-term use which of the two is better.
We can see in a direct comparison that [DeepSeek-V4-Flash-0731 has an intelligence Index of 50, whereas Inkling-Small has only 40 on Artificial Analysis](https://artificialanalysis.ai/models/comparisons/deepseek-v4-flash-vs-inkling-small).
This shows that DeepSeek-V4-Flash-0731 should be significantly better than Inkling-Small.
We must not forget, though, that Inkling supports Images as input, while DeepSeek doesn't.
So there will be use cases where Inkling is the clear winner.

So, does it actually make sense for me to investigate Inkling-Small and blog about it?
My main motivation for writing the posts is to keep myself up to date on locally runnable LLMs, build a good toolset to evaluate them in my circumstances, and ensure I have something to contribute to the ecosystem.
Given the pace of developments, I also need to make sure that once a model drops, I don't spend too much time setting things up, downloading the right models, and doing other grunt work.
The time I can spend on this topic is very limited, so I want to focus on parts where I can learn something.

## Minimising startup time
While I don't know whether I'm going to use Inkling-Small in the long term (long term could already be two weeks), I want to use it as a template for how I can get started quickly with a new model.
The basic ingredients we need to run a new LLM locally are:
1. The right GGUF files
2. A runnable `llama.cpp` version
3. Configuration for `pi`, my preferred coding agent

One could see, as a further step, having some personal benchmarks available that tell me directly how the model performs on my typical work.
This is something I will explore in the future.
In this article, I will focus on getting the model up and running.

## The right GGUF files
My preferred local inference engine is [llama.cpp](https://llama.app/) installed via [`pixi`](https://pixi.prefix.dev/latest/).
`llama.cpp` wants to have its model as [GGUF files](https://huggingface.co/docs/hub/en/gguf).
Models are typically first uploaded to HuggingFace as [safetensors files](https://huggingface.co/docs/safetensors/en/index).
Thus, we need to wait for someone to convert them to GGUF.
We could do this on our own, but in most cases, we will also not be able to fit the model in [BF16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format) onto our machine.
Thus, we first need to wait for someone to quantise the models to a lower-precision representation.
The folks over at [Unsloth AI](https://huggingface.co/unsloth) are typically among the fastest ones.
Sometimes (e.g., for Lagnua-S-2.1), the model provider also provides quantisations as GGUF on release.
Once they are available, I will set up a new pixi workspace/project and start downloading them.

```shell
# Setup the pixi project
pixi init
pixi add huggingface_hub 

# Use direnv to auto-activate the pixi environment
cat > .envrc <<'EOF'
watch_file pixi.toml pixi.lock
eval "$(pixi shell-hook)"

dotenv_if_exists .env
EOF

direnv allow

# Download the relevant model files
hf download hf://unsloth/Inkling-Small-GGUF/UD-Q2_K_XL/
```

One thing that might also be problematic is the chat template.
You can also encounter rendering issues here, depending on which serving engine you use.
Thus, some models or other users provide a separate `chat_template.jinja` file.

The CLI output will provide you with a directory where the result is stored.
In the above case, this resulted in `~/.cache/huggingface/hub/models--unsloth--Inkling-Small-GGUF/snapshots/1a19ef82883cb7b9c581b93c30ea252dabbf658d`.
Store this in an environment variable `MODEL_DIR`, preferably in your `.env(rc)`.

## Runnable `llama.cpp` version
While `llama.cpp` supports many models, most new models will also need some adaptation to its codebase.
Very few models actually come with day 0 support in `llama.cpp`.
Rather, there is typically a PR open from the model provider or some third-party (like [Daniel Hanchen from Unsloth AI](https://github.com/danielhanchen)) to it. 

If the latest release of `llama.cpp` supports it, we are lucky and can simply use `pixi add llama.cpp`.
Sometimes, you will need to have the very latest release of `llama.cpp` and that is not yet provided on the [llama.cpp-feedstock](https://github.com/conda-forge/llama.cpp-feedstock).
To get that, you can ask the bot there to update to a new version by opening an issue with the title: `@conda-forge-admin, please update version`.
Then it should make a PR, search for a new version and build it in CI.
You will need to wait for a maintainer (e.g. me) to merge this PR, but this normally happens very fast.

In the case that the support has not been merged into a `llama.cpp` release, I have set up a separate conda channel https://prefix.dev/channels/inference-extensions.
Uploading new versions here is not as straightforward as with a conda-forge feedstock, but building the inkling branch from Unsloth AI/Daniel Hanchen actually went quite smoothly.
For a new release, we do:

```shell
OLD_ALIAS=unsloth-inkling
ALIAS=new-llm
URL=https://github.com/provider/new-llm
REV=d69b7e606af7f59d9e81c5c81a5c7ce0dad197e1

# Create an empty public repo
gh repo create xhochy-forge/${ALIAS}-llama.cpp-feedstock --public

# Push an existing llama.cpp "branch" build
cd llama.cpp-feedstock
git remote add ${ALIAS} https://github.com/xhochy-forge/${ALIAS}-llama.cpp-feedstock
git checkout ${OLD_ALIAS}-main
git checkout -b ${ALIAS}-main
git push ${ALIAS} ${ALIAS}-main:main

# Adjust the recipe 
git checkout -b ${ALIAS}-initial-build

esc=${URL//\\/\\\\}
esc=${esc//&/\\&}
esc=${esc//|/\\|}
sed -i.bak \
  -e "s|${OLD_ALIAS/-/_}|${ALIAS/-/_}|g" \
  -e "s|^\([[:space:]]*url:\).*|\1 $esc|" \
  -e "s|^\([[:space:]]*repository:\).*|\1 $esc|" \
  -e "s|^\([[:space:]]*homepage:\).*|\1 $esc|" \
  -e "/^[[:space:]]*summary:/ s|https*://[^)[:space:]]*|$esc|" \
  -e "s|^\([[:space:]]*rev:\).*|\1 \"$REV\"|"
  recipe/recipe.yaml && rm -f recipe/recipe.yaml.bak

# Stage the changes and push them
git add -p
git commit -m "Initial build for ${ALIAS}"
git push -u ${ALIAS} ${ALIAS}-initial-build
```

Now we create a GitHub PR, wait for a green build and merge.
Typically, 2-4h later, we will have a new upload at [inference-extensions](https://prefix.dev/channels/inference-extensions/).
To make sure that this upload works, we will also need to enable trusted publishing at https://prefix.dev/channels/inference-extensions/settings/repository-access (this URL is only accessible to me).

To get the right binaries, we are going to adjust the `pixi.toml`:
```toml
[workspace]
name = "inkling-small"
platforms = ["osx-arm64", { platform = "linux-64", cuda = "12.9", glibc = "2.34" }, { platform = "linux-aarch64", cuda = "12.9", glibc = "2.34" }]
channels = ["https://prefix.dev/inference-extensions", "conda-forge"]
version = "0.1.0"

[tasks]

[dependencies]
huggingface_hub = ">=1.26.0,<2"
# Keep the build = "laguna*" here. I might upload other llama.cpp builds in the future.
# By keeping the build, you can get updates via `pixi update` without switching to a different fork.
"llama.cpp" = { version = "*", build = "unsloth_inkling_*" }
```

The important things here are:
1. We extended the `platforms` to account for the fact that we would like to use CUDA 12.9+ and have a decently modern Linux (glibc 2.34 was released 5y ago)
2. We added `inference-extensions` to the list of `channels`
3. We added the `llama.cpp` build we just uploaded as a dependency

With these ingredients, we can now start a `llama-server`:
```shell
llama-server \
    -m $MODEL_DIR/UD-Q2_K_XL/Inkling-Small-UD-Q2_K_XL-00001-of-00003.gguf \
    --reasoning-preserve \
    --jinja
```

This is a sane setup for most models.
Depending on the hardware and model, you can get better performance by configuring additional options, but this basic configuration will work for nearly everything.
`--reasoning-preserve ` depends on the model/chat template supporting preserving reasoning traces (probably obvious from the name).

## Configuring `pi`
To make the configuration a bit more readable, we will add `--alias inkling-small` (or `$ALIAS` if we want to add it to the automation) to our `llama.cpp` options.
Then, we can append this to `~/.pi/agent/models.json`'s providers section:

```json
{
  "providers": {
    "llamacpp": {
      "baseUrl": "http://localhost:9995/v1",
      "api": "openai-completions",
      "apiKey": "local",
      "compat": {
        "supportsDeveloperRole": false,
        …
      },
      "models": [
        {
          "id": "inkling-small",
          "reasoning": true,
          "contextWindow": 1048576,
          "maxTokens": 32768,
          …
      ]
    }
  }
}
```

The above is what we can add for any model.
We only need to determine the context length and add it to the template (here: 1M).
Sadly, each model has its own specifics that we need to include in the template. For that, we can ask an LLM that has web search to:
```
I'm using the recently released <model-name> model with llama.cpp and want to add it to the pi coding agent. What do I need to add to my config? I'm running `llama.cpp` with `--port 9995 --alias ${ALIAS}`. Please provide a configuration for pi. Remove all redundant configurations.
```

I have the urge to add "Remove all redundant configurations," as most LLMs are verbose and add a lot of default presets to the config.
Thus, the configuration was usually twice as long as necessary.
Inkling (all versions) comes with its own reasoning configuration.
You don't set a reasoning level through the normal OpenAI-compatible API; instead, supply a value of 0…0.99 via the chat template.
Thus, we need to add that to the configuration:

```json
{
  "providers": {
    "llamacpp-inkling": {
      "baseUrl": "http://localhost:9995/v1",
      "api": "openai-completions",
      "apiKey": "local",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false,
        "thinkingFormat": "chat-template",
        "chatTemplateKwargs": {
          "reasoning_effort": { "$var": "thinking.effort" }
        }
      },
      "models": [
        {
          "id": "inkling-small",
          "reasoning": true,
          "contextWindow": 1048576,
          "maxTokens": 32768,
          "thinkingLevelMap": {
            "off": "none",
            "minimal": null,
            "low": "low",
            "medium": "medium",
            "high": "high",
            "xhigh": "xhigh",
            "max": "max"
          }
        }
      ]
    }
  }
}
```

With this entered in `pi`, we are now up and running with the new model.
For each new model that arrives, we can follow the same procedure.
In the future, I would also like to add an optimisation step here to deliver a bit more performance.
This can be a topic on its own as I will need to set up:
1. Some example problems I can run and reproduce with any model.
2. A way to measure RAM usage, prefill and decode speed.
3. Some typical options to tune (e.g. Flash Attention, Batch size, KV-Cache quantisation, …)
4. Speculative decoding options (EAGLE, MTP, DFlash, …)
5. Automation to execute all of this. This will take quite some time and should not take much of my attention.

Thus, prepare for a follow-up post in a few weeks.
I would expect that tuning the options can bring upto 20% more performance, and with working speculative decoding, maybe even twice the speed.

## How's Inkling-Small?
We have now spent most of the article on the recipe for a new model and have disregarded that it was initially meant to be a review of Inkling-Small.
With the setup above, we can run it locally now.
In my case, I wanted to have a local model that I can run in addition to Laguna-S-2.1.
Laguna should be the model that creates the code, but for review, I need something else.
Thus, I have changed my `~/.roborev/config.toml` to use `default_model = 'llamacpp/inkling-small'` and rerun the last 20 reviews.

The model's throughput is only mediocre, with 300-400 t/s for prefill and 15-22 t/s for decode.
Given that I have not tuned any settings, I wouldn't be much concerned about this.
My memory is fully filled by `llama.cpp`, so it would probably make sense to investigate a slightly smaller quantisation or using a smaller context length.
This might already improve it significantly if there were a bit more RAM to spare.

Regarding the model's actual performance, I'm really happy.
Most of the code reviews come back much faster than Laguna-S-2.1 or Qwen3.6-27B did.
Thus, it doesn't suffer from the same overthinking problem as Laguna-S-2.1.
The reports have content similar to that of other models of similar size.
Given that we get them for way fewer tokens, this is a definite improvement.

## Is my next blog post about DeepSeek-V4-Flash-0731?
While I started this post, I felt that Inkling-Small is already "outdated" in my personal world, as DeepSeek-V4-Flash-0731 has been released, I cannot say for certain that I will do a blog post about it.
The preview of DeepSeek-V4-Flash has been running quite smoothly and with minimal manual configuration for me over the last month.
I would expect that the new release will continue with simply better model performance.
Thus, I don't see where a post of mine would be of any value for my personal learning or any reader (yet). 

My motivation with these posts is to learn or build interesting stuff.
That someone reads them is nice, but only second to the other two objectives.
I don't strive to be another news portal about LLM developments in general.
Thus, I will not cover model releases that might make it on my machine, but I don't contribute anything in regard to my main objectives.
