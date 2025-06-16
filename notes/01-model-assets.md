# Note 1: Apple Foundation Model Asset Bundles in macOS 26

The first thing I wanted to understand was where Apple’s foundation model is stored. Examining the model assets can reveal a lot about the runtime: the format may provide hints about the hardware accelerator used during inference, the size might give clues about the quantization techniques employed, and if the model graph and weights are easily accessible, we might even be able to extract the model itself and experiment with it independently.

So let’s see what we can find!

### Tracing the Inference Runtime

My first clue appeared while monitoring process activity while using LLM-powered features in Mail and Notes (summarization of a text for example). Whenever I triggered an Apple Intelligence task, a process named `TGOnDeviceInferenceProviderService` was appearing in the process list. This binary is located at `System/Library/ExtensionKit/Extensions/TGOnDeviceInferenceProviderService.appex/` and seems to be available for `arm64` as well as `x86` (surprisingly) targets:

```
file /System/Library/ExtensionKit/Extensions/TGOnDeviceInferenceProviderService.appex/Contents/MacOS/TGOnDeviceInferenceProviderService
/System/Library/ExtensionKit/Extensions/TGOnDeviceInferenceProviderService.appex/Contents/MacOS/TGOnDeviceInferenceProviderService: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64e:Mach-O 64-bit executable arm64e]
/System/Library/ExtensionKit/Extensions/TGOnDeviceInferenceProviderService.appex/Contents/MacOS/TGOnDeviceInferenceProviderService (for architecture x86_64):	Mach-O 64-bit executable x86_64
/System/Library/ExtensionKit/Extensions/TGOnDeviceInferenceProviderService.appex/Contents/MacOS/TGOnDeviceInferenceProviderService (for architecture arm64e):	Mach-O 64-bit executable arm64e
```

It seems this service is launched on-demand, running isolated from the calling app, and its lifecycle is tightly tied to local inference requests. I haven’t found documentation for this service, but looking at the files & ports it has access to in the Activity Monitor, I saw a references to the following path `/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels/purpose_auto/fd24bcad75f012120dc856e02f863ba1535c357e.asset/.AssetData/model.bundle/H15S.bundle/lora_16_extend_2048_8/lora_16_extend_2048_8_classic_cpu/model.espresso.weights`.

That clearly looks like a model asset, let's look into it ! 🔎


[Me trying to `cd` into the directory ..] Arfffff it's protected !!

To access it, it seems to be required to disable System Integrity Protection (SIP) feature from my Mac (thanks reddit friends: [Apple Intelligence stuck downloading](https://www.reddit.com/r/MacOSBeta/comments/1eqrg3k/apple_intelligence_stuck_downloading_for_over_a/?sort=old)).

I don't have enought technical background to add anything useful regarding the SIP disablement, but if someone knows more about why it is required and how it works I'll be happy to add details about this here.

### Asset Packaging and Cryptex

The next puzzle was locating the model files themselves. Searching for anything resembling CoreML .mlmodelc or ONNX, I found nothing. Instead, models are delivered in the following directory, under the MobileAsset framework: `/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels/`.

But looking into one of them, I couldn't find any direcory/file that could relate to the model formats I know:

```
.
├── AssetData
│   └── Restore
│       ├── BuildManifest.plist
│       ├── Firmware
│       │   ├── Manifests
│       │   │   └── restore
│       │   │       └── cryptex1
│       │   │           └── UAF.FM.GenerativeModels:fm.language.instruct_3b.base.generic-H15s
│       │   │               └── apticket.mobileassetx86macosap.im4m
│       │   ├── UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg.cryptex_info
│       │   ├── UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg.integrity_catalog
│       │   ├── UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg.root_hash
│       │   └── UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg.trustcache
│       ├── Restore.plist
│       └── UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg
├── Info.plist
└── SecureAssetData
    ├── amai
    │   ├── Cryptex1
    │   │   └── Cryptex1,Ticket
    │   ├── debug
    │   │   ├── tss-request.plist
    │   │   └── tss-response.plist
    │   └── receipt.plist
    ├── BuildManifest.plist
    ├── SecureMobileAsset-Info.plist
    ├── SecureMobileAsset.dmg -> ../AssetData/Restore/UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg
    ├── SecureMobileAsset.integritycatalog
    ├── SecureMobileAsset.root_hash
    └── SecureMobileAsset.trustcache
```

Looking at the asset size, it seemed that `UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg` had the highest probability to be something meaningfull. These are disk images, but mounting them using Finder or hdiutil fails.
After some googling (and discussing with ChatGPT), I found that this disk image are an Apple specific way to ship securely assets in their OS upades (among other things), they have called it `Cryptex` (looking at the file name it couldn't have been more obvious from the start 😅).

Using `hdutils` to mount it allowed me to access its content: `hdiutil attach UC_FM_LANGUAGE_INSTRUCT_3B_BASE_GENERIC_GENERIC_H15S_Cryptex.dmg`.

### Exploring the Model Asset Layout

Inside the Cryptex, the central artifact is found under `model.bundle/H15S.bundle/`. This directory is densely packed, including subfolders for embeddings, LoRA blocks, and additional model components. A (partial) directory listing looks like:

```
tree -L 3
.
├── metadata.json
├── model.bundle
│   └── H15S.bundle
│       ├── gather_embeddings_16
│       ├── gather_embeddings_64
│       ├── gather_embeddings_8
│       ├── H15S.e5
│       ├── load_embeddings
│       ├── lora_16_extend_1024_16
│       ├── lora_16_extend_1024_64
│       ├── lora_16_extend_2048_16
│       ├── lora_16_extend_2048_64
...
│       ├── lora_32_extend_512_8
│       └── lora_32_prompt_opt_256_64
```

Each lora_* directory contains files such as `model.espresso.net`, `model.espresso.shape`, and `model.espresso.weights`. Here’s a quick example:

```
lora_16_extend_2048_8/
  ├─ model.espresso.net
  ├─ model.espresso.shape
  └─ model.espresso.weights`
```

The content of `model.espresso.net` is text file likely describing the model graph. `model.espresso.shape` is a dictionnary containing informations about model shapes, and `model.espresso.weights` is a large binary file containing model file. Other folders have files like `fragment.mil` (likely a CoreML MIL intermediate graph) and `weights.bin`.


Cool ! But these files looked pretty small in term of size with respect to what I've would expect for a 3B model 🤔.

Let's see what they could be used for ..

### Adapter Variants and Model Specialization

Another interesting thing is the diversity of LoRA adapters, covering different configurations. Inspecting `metadata.json`, I found a mapping of each submodel to a context and sequence length.

For example:
```
"lora_16_extend_2048_8": {
    "type": "extend",
    "adapter_type": "lora_16",
    "ctx_len": 2048,
    "seq_len": 8
},
"lora_32_prompt_opt_256_64": {
    "type": "prompt_opt",
    "adapter_type": "lora_32",
    "ctx_len": 256,
    "seq_len": 64
}
```

Each entry appears to define a unique context window and output sequence configuration. There are several plausible reasons for this granularity:

For one, Apple’s ANE hardware, like most inference accelerators, require statically-shaped compute graphs for maximum throughput. But as prompt sizes and generation length can vary from one request to another, it is required to dynamically pad or mask inputs at runtime which implies unnecessary compute and memory costs . In order to always get the best performance for different `ctx_length` / `seq_length` ratios, it is preferable to have different instances of the model expecting different sizes for each of these values in order to minimize padding/masking losses. These LoRA adapters seems to feed that need.

Secondly, large transformer models are typically trained for a specific context length (e.g., 2048 tokens), their performance outside these windows often degrades due to limited exposure to longer or shorter sequences during training. By providing LoRA adapters tuned to distinct context sizes and output lengths, it allows to maintain a good accuracy across a range of practical prompt sizes.


#### ANE Execution

A central artifact in most of the model bundles is an `.hwx` file, for example: `lora_32_prompt_opt_256_64/multiprocedure/model.hwx`. It's a 774MB, which is more realistic for a heavily quantized 3B LLM (more on that later).

This file doesn’t match any public format, but from public research, .hwx files are compiled binary graphs targeting ANE, baked for a specific Apple Silicon variant (e.g., M3 Pro).

We have just confirmed another interesting thing ! Apple fundation models are executed on the ANE 🏎️

