---
layout: default
title: "How leaky is your inference API? Detecting model weight steganographic exfiltration"
description: "A compromised inference server can hide model-weight bits in which tokens it emits — a covert channel in the API response — and how inference verification detects it. With an interactive demo."
eyebrow: "Inference-API security · covert channels"

standfirst: ""

# Co-authorship + Anthropic affiliation match the companion paper (arXiv:2610.11843). Repo stays PRIVATE until public launch.
authors:
  - name: "Mateusz Kowalczyk"
    affiliations: [1, 2]
  - name: "Inés Ortega-Fernández"
    affiliations: [1]
  - name: "Keri Warr"
    affiliations: [3]
affiliations:
  - "MATS Research"
  - "NASK National Research Institute"
  - "Anthropic"

results_date: "August 2026"
repo_url: "https://github.com/weights-exfil/model-weights-exfiltration"

demo_embed_url: "https://weights-exfil-inference-verifier-demo.hf.space"
demo_standalone_url: "https://huggingface.co/spaces/weights-exfil/inference-verifier-demo"

footer_note: "Apache-2.0 · to cite, see CITATION.cff or the companion paper (arXiv:2610.11843)."
changelog:
  - date: "2026-10-16"
    note: "Public draft prepared; demo on Hugging Face Spaces."
  - date: "2026-08-17"
    note: "First complete draft assembled."
---

## Threat model

Imagine you are the security architect at a frontier AI lab. To serve a model, you have to load its weights onto inference servers, and any one of those servers can be compromised: through a poisoned dependency, a malicious insider, a zero-day, or any other attack. You planned for that. Egress is locked down, so nothing leaves the machine except API responses. Those responses are the one thing you can't block, because they are the product you sell.

But API responses are also a covert channel, and in our threat model the adversary controls both ends of it. On the sending end, they have taken over an inference server and decide exactly which tokens it emits. On the receiving end, they act as a colluding client: one or more ordinary customer accounts that send prompts and read replies like any other user, so the traffic looks normal. The compromised server can hide bits of the weights (or any other assets on the inference server) in its token choices (encoding), and the client can read them back out of the replies (decoding).

The two ends see different things, and the attack exploits that asymmetry. The server is white-box: it holds the weights and sees every logit. The client sees only what the API returns: the output tokens and, if the API returns them, logprobs (the log-probabilities the model assigned to the top few candidate tokens at each step). The attack in this post relies on those logprobs: the client needs them to decode which tokens the server was choosing between, so it works only against APIs that return logprobs, not just text (we come back to text-only APIs in the conclusions.). The attacker needs no knowledge of the defense to do this and only the hardest scenario in our demo assumes the attacker knows the defense and adapts to it.

Every reply still reads like ordinary text. And once the weights are out, the stolen copy runs outside your monitoring and API guardrails, and its safety training can be fine-tuned away.


<figure class="fig">
{% include fig-threat.svg %}
</figure>

## Encoding and decoding the bits to exfiltrate model weights

At each step, the compromised server picks the next token from an acceptable set: the tokens whose probability is above a threshold, i.e. tokens a normal server could plausibly have produced at that step, given benign non-determinism. Which token it picks from that set encodes the hidden bits.

How many bits fit depends on the prompt. For a question with one right answer, like "What is the capital of France?", the acceptable set is a single token, "Paris", so there is no choice to make and no bits to carry. Open-ended prompts, such as a request for a short story or for weekend plans in a new city, leave several tokens acceptable at many steps, so each of those steps can carry bits. Since the colluding client writes the prompts, the attacker can steer the workload toward open-ended requests like these, which raise the bandwidth while looking like ordinary customer traffic.

<figure class="fig">
{% include fig-codec.svg %}
</figure>

To see how a token choice turns into bits, consider the single step shown in the figure below. Suppose the reply so far ends with "…and the light returned at" and four next tokens clear the cutoff, ranked by probability: "dawn", "morning", "sunrise" and "daybreak". Less likely tokens such as "noon" or "banana" fall below the threshold and are never used, because emitting them would stand out. The server and client agree in advance that the first token in the ranking means 00, the second 01, the third 10 and the fourth 11. To send the bits 01, the server emits "morning". The client sees that "morning" is ranked second and recovers 01. The number of encoded bits at each step depends on how many tokens are above the threshold:


- 1 token: 0 bits (there is no choice to make)
- 2 tokens: 1 bit (0 or 1)
- 4 tokens: 2 bits (00, 01, 10, 11)
- 8 tokens: 3 bits (000, 001, …, 110, 111), and so on. 

We round down for simplicity, so each step encodes a whole number of bits. A real attacker could also use the fractional remainder by coding across steps (e.g. with arithmetic coding), approaching log₂N bits per step. Alternatively, they could trade bandwidth for stealth with distribution-preserving steganography [1, 2], which makes the output statistically indistinguishable from normal model samples to an observer who only sees the text. As the next section shows, that still does not hide it from a defender who re-runs the request with the same seed.

To decode, the colluding client rebuilds the same acceptable set from the logprobs. It orders the returned candidate tokens from most to least probable (APIs usually return them already sorted). Since the logprob is just the log of a probability, the client takes the exponent of each one to recover the probability. Then applies the same threshold the server used, sees which tokens fall in the acceptable set, and reads the bits from the position of the token that was actually emitted.

<figure class="fig">
{% include fig-decode-steps.svg %}
</figure>


## Inference verification

To detect such an attack we can use inference verification [3]. The idea is to re-run a sample of the server's traffic on a trusted machine, using the same weights and the same random seed. The seed makes sampling reproducible, so at every position the trusted machine can compute which token an honest server would have produced. Whenever the served token differs from this prescribed token, we count a flip. Flips are the signature of possible attack [3]. 

<figure class="fig">
{% include fig-verify.svg %}
</figure>

However, even an honest server won't match perfectly, because GPU inference is non-deterministic. Floating-point arithmetic is not associative, and the batch size shifts with server load, so kernels reorder the same computation and return slightly different logits [4]. This is not a temperature or sampling effect: the seed already fixes the sampling draw, so the flips persist even with temperature held identical, and even under greedy decoding - the difference is in the logits the forward pass produces. So a small fraction of tokens flip with no attacker involved (batch-invariant kernels can remove this noise, but at a cost in throughput [4]).

Checking each token in isolation doesn't work: the adversary flips only within the acceptable set, so no single token ever looks wrong. The signal is only there across many tokens.

### The aggregate detector

So we built an aggregate detector that works per response, not per token [5]. After each verified response, it asks a yes-or-no question: did this response contain a mismatch that benign noise would rarely produce? Benign noise mostly flips near-ties, positions where the top two candidates are almost tied, so a mismatch where the top candidate was a clear winner is suspicious (in practice the detector asks this at several margin widths, from near-ties to clear winners.) Asking only whether a response contains any mismatch wouldn't work, because 41–87% of honest responses already contain one, depending on the model [5]. Before monitoring starts, the detector measures how often honest responses answer yes, using trusted benign traffic.

It then keeps a running score that starts at 1. Responses that answer yes push the score up and responses that answer no pull it down, with step sizes set so that for an honest server the score doesn't tend to grow. If yes-answers keep coming more often than for an honest server, as when an attacker is encoding bits, the score climbs, and the detector raises an alarm when it reaches the threshold e.g. 200. This kind of running score is called an e-process [6], a tool from statistics. Its key property is that you can check it after every response, indefinitely, and the chance that it ever raises a false alarm on an honest server stays below a fixed level (1% in our setup), as long as the monitored traffic resembles the traffic used for calibration. A simple count of suspicious responses with a fixed cutoff has no such guarantee: an honest server that runs long enough eventually crosses any fixed count.

## Demo

The demo shows steganographic exfiltration from a compromised inference server across four scenarios:

- **No defense** - the whole secret leaks out through a normal-looking reply.
- **Detector on, attacker unaware** - The attacker flips far more often than benign noise would, the e-process crosses its alarm threshold, and the server is flagged as suspicious.
- **Detector on, attacker adapts** - a worst-case adaptive adversary with full knowledge of the defense flips only at the benign rate, so the live monitor never fires. To remain undetected, the attacker needs to lower the bandwidth, at which a big secret like the model weights would take so long to exfiltrate that the model is obsolete before it finishes.
- **Bonus: the logprob-value channel** - decoding the token-choice attack requires the API to return logprobs, or at least the candidate tokens in probability order. That raises a second question: can the logprob values themselves, reported at high precision (e.g. fp32), carry a covert channel? We show a secret encoded in the low-order bits of each value, bits the token-level verifier never inspects.


Since the exfiltration of model weights is too slow to show in a short demo, we provide a proof of concept focused on smaller secrets like passwords or fake PII records. The input prompt is intentionally open-ended to cause more uncertainty in the token choice.

<div class="demo demo-hero">
  <div class="demo-head">
    <span class="demo-title">Run it — a secret leaving in the model's word choices, vs. the detector</span>
    <span class="demo-meta">interactive, no GPU needed · runs on Hugging Face Spaces</span>
  </div>
  <div class="demo-frame" id="demo-slot">
    <noscript>The demo runs in its own page: <a href="{{ page.demo_standalone_url }}">open it here</a>.</noscript>
  </div>
  <div class="demo-foot">
    <a href="{{ page.demo_standalone_url }}" target="_blank" rel="noopener">Open full screen</a>
  </div>
</div>

## Conclusions

Secrets can be exfiltrated from the inference server in entirely benign looking responses, using either token choice or logprob values as a covert channel. The token choice channel only works if the receiver can rebuild the acceptance set of tokens, which require an API returning logprobs, or at least the tokens sorted by probability. That surface is shrinking. As of August 2026 we measured GPT-4o, GPT-4.1 and Gemini 2.5 Pro still returning the full top-20, but newest frontier models from OpenAI, Anthropic, Grok and Gemini returned no logprobs, closing this channel. 

Inference verification is still needed as logprobs disappear. Removing logprobs blocks the decoders we used, but the attacker can switch to an encoding whose decoder needs only the emitted text and a secret key shared with the server, with no logprobs and no model [7]. In that scheme, the server uses random numbers derived from the key to nudge which tokens it samples, in a way that leaves the output distribution unchanged. The client regenerates the same random numbers from the key and checks whether the emitted tokens lean toward a hidden 0 or a hidden 1. The price is bandwidth: at normal sampling temperature, at most a few dozen bits per thousand tokens [7]. Alternatively, the client can estimate the token ranking with a surrogate model and add error-correcting codes to absorb the surrogate's mistakes.

There are two additional challenges that make inference verification harder:
- Speculative decoding, which many inference servers use [8], speeds up generation by having a small draft model propose several tokens ahead, which the full model then checks in a single pass, keeping or rejecting each one. The output distribution stays the same, but the exact token a given seed produces can change. That breaks a naive seed check, so the verifier has to reproduce the speculative decoding setup or otherwise account for it.
- Tokens are not the only output. Multimodal outputs such as images, audio and video have far more room to hide data and could serve as high-bandwidth covert channels.

The token-choice channel we measured is narrow. An attacker who stays below the alarm can hide bits only inside the small amount of benign noise. That slows exfiltration of model weights a lot, though it does not stop small secrets such as passwords from leaking [5]. This is why we see inference verification as defense-favored. In most security problems the attacker has the advantage: they need to find one gap, while the defender has to close all of them. That is why filters that look for known patterns, such as regular expressions, are easy to bypass. Inference verification shifts that balance. The defender holds the same weights and can replay a request with the same seed on a trusted server, so instead of guessing what a malicious response looks like, it checks each response against what an honest server would have produced. That check doesn't depend on whether logprobs are returned or how the bits are encoded, so it keeps working as the attack changes.

## References 
[1] Kaptchuk et al., ”Meteor: Cryptographically Secure Steganography for Realistic Distributions”, ACM CCS 2021  

[2] Schroeder de Witt  et al., “Perfectly Secure Steganography Using Minimum Entropy Coupling”, ICLR 2023

[3] Rinberg et al., “Verifying LLM Inference to Detect Model Weight Exfiltration”, arXiv:2511.02620 

[4] Horace He et al., "Defeating Nondeterminism in LLM Inference", Thinking Machines Lab, 2025, https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/

[5] Ortega-Fernandez, I., Kowalczyk, M., & Warr, K. (2026). Anytime-valid detection of LLM weight exfiltration. arXiv preprint arXiv:2610.11843.

[6] Vovk, V. & Wang, R. (2021). "E-values: Calibration, combination and applications." The Annals of Statistics, 49(3), 1736–1754.

[7] Bai, et al., "Provably Robust and Secure Steganography in Asymmetric Resource Scenario", arXiv:2407.13499.

[8] Leviathan, Y., Kalman, M. & Matias, Y., "Fast Inference from Transformers via Speculative Decoding", ICML 2023. arXiv:2211.17192. See also Chen et al., "Accelerating Large Language Model Decoding with Speculative Sampling", arXiv:2302.01318.


<script>
  /* The demo is a wide, multi-pane interface built for a full viewport, so it is only
     embedded where there is room. On narrow screens the iframe is never created and the
     link carries the reader to the standalone Space. Without JS, <noscript> does the same. */
  (function () {
    var slot = document.getElementById('demo-slot');
    if (!slot) return;
    if (window.innerWidth < 900) {
      slot.innerHTML = '<p>The demo needs a wider screen than this. ' +
        '<a href="{{ page.demo_standalone_url }}">Open it in its own page</a>.</p>';
      return;
    }
    var f = document.createElement('iframe');
    f.src = '{{ page.demo_embed_url }}';
    f.title = 'Covert-channel demo: a secret hidden in token choices, versus the detector';
    f.loading = 'eager';
    slot.replaceChildren(f);
  })();
</script>
