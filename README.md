# Neuro

The goal of this project was to recreate Neuro-Sama, but only running on local models on consumer hardware.
The original version was also created in only 7 days, so it is not exactly very sophisticated.

![Screenshot of demo stream](./images/stream.png)

## Features
- Realtime STT for natural voice input
- Realtime TTS for natural voice output
- Clean frontend/control panel for easy moderation/interaction: [neurofrontend](https://github.com/kimjammer/neurofrontend)
- Audio File playback (for pre-generated songs/covers created with something like [RVC](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI)
- Vtube Studio Plugin & Model/Prop control
- Flexible LLM - Load any model into text-generation-webui (tested) or use any openai-compatible endpoint (not tested).
- Memory - Long-term (persists across restarts) memories can be manually added, but they will also be 
automatically generated as the AI talks. (See memories/readme.md for details)

## Architecture

### LLM

I used [oobabooga/text-generation-webui](https://github.com/oobabooga/text-generation-webui)
running [LLAMA 3 8B Instruct EXL2 4.0bpw](https://huggingface.co/turboderp/Llama-3-8B-Instruct-exl2/tree/4.0bpw) on the
ExLlamav2_HF loader with cache_8bit turned on. The openai api extension must be turned on, as this is how we interact
with the LLM. text-generation-webui and the LLM must be installed and started separately.

Alternatively, you can load any other model into text-generation-webui or modify constants.py to point to any other
openapi compatible endpoint. Note that this project uses some parameters not available on the official OpenAI API.

### STT

This project uses the excellent [KoljaB/RealtimeSTT](https://github.com/KoljaB/RealtimeSTT), which can transcribe an
incoming audio stream, not just a file. This means that the text is transcribed as the person is talking, and so
transcription ends almost immediately after speech ends. It is configured to use the faster_whisper tiny.en model.

### TTS

This project also uses [KoljaB/RealtimeTTS](https://github.com/KoljaB/RealtimeTTS). It is configured to use CoquiTTS
with the XTTSv2 model. If you like, you can fine tune a XTTSv2 Model with
the [erew/alltalk_tts](https://github.com/erew123/alltalk_tts) repository. This also streams the audio out as it is
generated, so we don't need to wait for transcription to fully finish before starting playback.

### Vtuber model control

Vtuber model control is currently extremely basic. The audio output from the TTS is piped
into [vtube studio](https://denchisoft.com/) via a virtual audio cable with something
like [this](https://vb-audio.com/Cable/), and the microphone volume simply controls how open the mouth is. Read
the Installation Section for more details.

### Modularization

Each concern of the program is separated out into its own python file/class. A single signals object is created and 
passed to every class, and each class can read and write to the same signals object to share state and data. tts.py and 
stt.py handle the TTS and STT, the llmWrapper.py is responsible for interfacing with the LLM API, and prompter.py is
responsible for deciding when and how to prompt the LLM. prompter.py will take in several signals (ex: Human currently
talking, AI thinking, new twitch chat messages, time since last message...) and decide to prompt the LLM.

There are also modules which extend the functionality of the core program. Modules are found in the modules folder, and
every functional module extends the Module class. Each module is run in its own thread with its own event loop, and will
be provided with the signals object. Modules must implement the run() method, and can provide the get_prompt_injection()
method which should return an Injection object. The Injection object is a simple data class that contains the text to
be injected into the LLM prompt, and the priority of the injection. Injections are sorted from lowest to highest
priority (Highest priority appears at end of prompt). When the signals.terminate flag is set, every module should clean
up and self terminate.

twitchClient.py handles the twitch integration and reading recent chat messages. There was an attempt made at discord
integration, but receiving voice data from discord is unsupported by discord and proved unusably buggy. streamingSink.py
is an unused file that would have been for receiving voice data from discord. main.py simply creates all class instances
and starts relevant threads/functions.

### Frontend Integration

This project uses python-socket.io to communicate with the control panel frontend. By default, the socket.io server is
started on port 8080. I chose socket.io as sometimes the server needs to push data to the client (streaming LLM
output, etc), and sometimes the client needs to send data to the server (blacklist updates, etc). In theory this could
have been done with just websockets, but I was familiar with socket.io already. The frontend, written on sveltekit using
shadcn-svelte, is available in its own repository, [kimjammer/neurofrontend](https://github.com/kimjammer/neurofrontend).

## Requirements

To fully recreate the author's exact setup, an Nvidia GPU with at least 12GB of VRAM is required. However, by altering
which LLM you run and the configurations of the TTS and STT, you may be able to run it on other hardware.

This project was developed on:

CPU: AMD Ryzen 7 7800X3D

RAM: 32GB DDR5

GPU: Nvidia GeForce RTX 4070 (12GB VRAM)

Environment: Windows 11, Python 3.11.9, Pytorch 2.2.2, CUDA 11.8

## Installation

This project is mostly a combining of many other repositories and projects. You are strongly encouraged to read the
installation details of the architecturally significant repositories listed above.

### Other Projects/Software

Install [oobabooga/text-generation-webui](https://github.com/oobabooga/text-generation-webui), and download an LLM model
to use. I used [LLAMA 3 8B Instruct EXL2 4.0bpw](https://huggingface.co/turboderp/Llama-3-8B-Instruct-exl2/tree/4.0bpw).

Install Vtube Studio from Steam. I used the default Hiyori model.

**Optional:** You may want to install a virtual audio cable like [this](https://vb-audio.com/Cable/) to feed the TTS
output directly into Vtube Studio.

With your twitch account, log into the developer portal and create a new application. Set the OAuth Redirect URL
to `http://localhost:17563`. For more details, read the pyTwitchAPI library
documentation [here](https://pytwitchapi.dev/en/stable/index.html#user-authentication).

### This Project

A virtual environment of some sort is recommended (Python 3.11); this project was developed with venv.

Install requirements.txt (This is just a pip freeze, so if you're not on windows watch out)

DeepSpeed (For TTS) will probably need to be installed separately, I was using instructions
from [AllTalkTTS](https://github.com/erew123/alltalk_tts?#-deepspeed-installation-options) , and using their 
[provided wheels](https://github.com/erew123/alltalk_tts/releases/tag/DeepSpeed-14.0).

Create an .env file using .env.example as reference. You need your Twitch app id and secret.

Place a voice reference wav file in the voices directory. It should be 5~30 seconds long. For details see the RealtimeTTS
repository.

Find your desired microphone and speaker device numbers by running utils/listAudioDevices.py and note its number. 

Configure constants.py.

## Running

Start text-generation-webui. Go to the Session tab and enable the openai extension (and follow instructions to actually
apply the extension). Go to the Model tab and load the model.

In this folder, activate your environment (if you have one) and run `python main.py`. A twitch authentication page will
appear - allow (or not I guess). At this point, the TTS and STT models will begin to load and will take a second. When
the "SYSTEM READY" message is printed, this project is fully up and running, and you can talk to the AI and hear its
responses.

Open Vtube Studio and if you have your TTS outputting to a virtual audio cable, select the virtual audio cable output as
the microphone, and link the mouth open parameter to the microphone volume parameter. If you have a model with lip sync
support, you can also set that up instead.

In OBS (or other streaming software), receive your Vtube Studio feed (on Windows Spout2 is recommended by Vtube Studio),
and go live!

# DISCLAIMER

This is an experimental, exploratory project created for educational and recreational purposes. I can make no guarantee
that the LLM will output non-vile responses. Please see the is_filtered() method in llmWrapper.py for details, but the
only filtered word right now is "turkey" in lowercase purely for debugging purposes. Configure the blacklist in blacklist.txt. 
If the LLM outputs unsafe content, you may and can get banned from Twitch. You use this software with all assumption 
of risk. This is not legal advice, see LICENSE for the repository license.

Any attribution in derivative works is appreciated.
