---
layout: post
title: Google Assistant SDK with Snowboy on Raspberry Pi
tags: raspberry-pi home-automation
published: True
---

Like many people I had the idea to build a Home Assistant, using a Raspberry Pi I had lying around. I was not sure which software to use, so I started searching around and I decided to give it a try with [GassistPi]([https://github.com/shivasiddharth/GassistPi](https://github.com/shivasiddharth/GassistPi)). `GassistPi` was built for expanding Google Assistant's features and it looked like a great choice for my use-case. Soon after I realized that  `GassistPi` uses the Google Assistant library, which is deprecated:
>:small_red_triangle: **Warning:** The Google Assistant Library for Python is deprecated as of June 28th, 2019. Use the [Google Assistant Service](https://developers.google.com/assistant/sdk/guides/service/python/) instead.

At this moment my thoughts were "deprecated is not necessarily bad, but since Google provides an alternative, I'm gonna give it a try."

   And that's how my story into the darkness begins...

## Setup
Before we start talking about the project, let me quickly introduce you to my setup as a point of reference. I use the [Raspberry Pi 3 model B]([https://www.raspberrypi.org/products/raspberry-pi-3-model-b/](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)), with [Raspbian Buster lite]([https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/)). For coding I use Python 3. As an audio input device I use the [PS3 Eye Camera]([https://www.amazon.com/Sony-PlayStation-Camera-Bulk-Packaging-Pc/dp/B0072I2240](https://www.amazon.com/Sony-PlayStation-Camera-Bulk-Packaging-Pc/dp/B0072I2240)). Finally, I connect my speakers in the audio jack port of RPi. I will not provide information on how to setup the above since guides are broadly available online.

## Google Assistant Service is ... deaf*
After installing the Google Assistant SDK and tried a few demos, I realized they have revoked the hotword wake-up function (or _Hands-free activation_ as they call it), and instead they only provide a push-to-talk functionality... what a bummer. Read more about the GA feature support [here]([https://developers.google.com/assistant/sdk/overview](https://developers.google.com/assistant/sdk/overview)).

I didn't give up though, all I had to do was to use some other hotword service to invoke the push-to-talk example they provide; then I'd be good to go implement my features!

Well, at least the idea was right. In practice, it was quite a pain...

\*<font color='gray' size=3>Technically, not really deaf. It can hear you if you poke it </font>:roll_eyes:


## Hotword activation
The quest to find a hotword detection service was easy. On the first search I bumped into [Snowboy]([https://snowboy.kitt.ai/](https://snowboy.kitt.ai/)) and I stuck with it. It works great and lets you define your own hotword when training your model.

I named mine Joe! It turns out that this was not the wisest choice since it activates with words like "joke", "hedgehog", etc., but it's fine for now. Feels like he wants to be part of our conversation every time he pops up :sweat_smile:

The setup is straightforward:
```bash
cd ~
git clone https://github.com/Kitt-AI/snowboy.git
cd snowboy/swig/Python3
make
```

They provide some demos on `snowboy/examples/Python3`. Everything ran perfectly fine, so then it was time to alter one of the demos and call the Google Assistant push-to-talk. I started getting the feeling that this will not be as simple as it sounds (oh boy, was I right...), so I started looking at how other people had gone on that.


## Diving deep
One of the most comprehensive guides I found was this one: [Google Assistant with snowboy Hotword Recognition]([https://www.sigmdel.ca/michel/ha/rpi/voice_rec_02_en.html](https://www.sigmdel.ca/michel/ha/rpi/voice_rec_02_en.html)) (plus they provide scripts with things they have tried; big thanks!). The article is written in 2018, however, it makes use of the GA Service, not the GA library, as it is explained on [step 6]([https://www.sigmdel.ca/michel/ha/rpi/voice_rec_02_en.html#GA](https://www.sigmdel.ca/michel/ha/rpi/voice_rec_02_en.html#GA)). At the same step, they explain how to use _Snowboy_ to invoke _pushtotalk_. The process of doing that is far more complex from what I was aiming for:
* Call _Snowboy_ with your hotword
* Your query is recorded on a `.wav` file
* The GA _pushtotalk_ is called using `subprocess`, passing the recorded `.wav` file as argument
* The response is also saved as a `.wav` file
* Another `subprocess` is used to call `aplay` to play the response from GA

In general, I avoid using `subprocess` if possible, and this solution felt like an overkill. What was closer to my desire is described as a failure on the next step ([step 7]([https://www.sigmdel.ca/michel/ha/rpi/voice_rec_02_en.html#SnowGA](https://www.sigmdel.ca/michel/ha/rpi/voice_rec_02_en.html#SnowGA))). In fact it starts like "_Why use a sub process to invoke a Python script within a Python script?_"
However, the problem they faced comes up a few lines later: _"Note that `pushtotalk.main` ... never returns in `detectedCallback`. ... Instead, the program exits without explanation."_ Furthermore, they also state that the author of `GassistPi` recognized this as a Google issue. It is also mentioned that the support for _Snowboy_ was removed from `GAssistPi` _"because of a problematic interaction with  `Portaudio`."_
Well, now I had some work to do! :smiley:

## Reuse previous knowledge

Theory is good, but I needed to test what they have done for myself. Thus, I downloaded the provided scripts and started playing around. The first step is to store the [`talkassist.py`](https://www.sigmdel.ca/michel/ha/rpi/dnld/talkassist.py) at the same location as the `pushtotalk.py`, and the [`googlesamples-assistant-talkassist`](https://www.sigmdel.ca/michel/ha/rpi/dnld/googlesamples-assistant-talkassist) at the same location as the `googlesamples-assistant-pushtotalk`. The `talkassist.py` is a modified `pushtotalk.py` that overpasses waiting for key-press. From the rest of the scripts, I only downloaded the scripts that invoke the assistant through a function call and not using subprocess (these are [`sbdemo7.py`](https://www.sigmdel.ca/michel/ha/rpi/dnld/sbdemo7.py) and [`sbdemo7e.py`](https://www.sigmdel.ca/michel/ha/rpi/dnld/sbdemo7e.py)). I placed these scripts in the _Snowboy_'s _examples_ directory. They are both very similar since they are reworked versions of the original _demo4_ of _Snowboy_, so I will focus only on one of them.

First, I had to modify the script before I start using it. The script makes use of a package called `pixels`, which handles LED lights. Since I am not using LEDs I did not bother installing it, so I just removed any code that had to do with it. Next, I linked my trained model and ran the script:
```bash
$ python sbdemo7.py
Listening... Press Ctrl+C to exit
```
While it seemed to be working, when my key-word got detected this was the outcome:
```shell
INFO:snowboy:Keyword 1 detected at time: 2019-12-07 23:15:55
yes...Expression 'ret' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 1736
Expression 'AlsaOpen( &alsaApi->baseHostApiRep, params, streamDir, &self->pcm )' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 1904
Expression 'PaAlsaStreamComponent_Initialize( &self->capture, alsaApi, inParams, StreamDirection_In, NULL != callback )' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 2171
Expression 'PaAlsaStream_Initialize( stream, alsaHostApi, inputParameters, outputParameters, sampleRate, framesPerBuffer, callback, streamFlags, userData )' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 2840
Traceback (most recent call last):
  File "orig_sbdemo7.py", line 78, in <module>
    sleep_time=sleepTime)
  File "/home/pi/snowboy/examples/Python3/snowboydecoder.py", line 221, in start
    callback()
  File "orig_sbdemo7.py", line 68, in detectedCallback
    main() # in googlesamples.assistant.grpc.talkassist
  File "/home/pi/snowboy/examples/Python3/custom_assist.py", line 246, in main
    flush_size=audio_flush_size
  File "/home/pi/env/lib/python3.7/site-packages/googlesamples/assistant/grpc/audio_helpers.py", line 190, in __init__
    blocksize=int(block_size/2),  # blocksize is in number of frames.
  File "/home/pi/env/lib/python3.7/site-packages/sounddevice.py", line 1345, in __init__
    **_remove_self(locals()))
  File "/home/pi/env/lib/python3.7/site-packages/sounddevice.py", line 861, in __init__
    'Error opening {0}'.format(self.__class__.__name__))
  File "/home/pi/env/lib/python3.7/site-packages/sounddevice.py", line 2653, in _check
    raise PortAudioError(errormsg, err)
sounddevice.PortAudioError: Error opening RawStream: Device unavailable [PaErrorCode -9985]

```
Searching about this problem led me to the Google Assistant SDK github [issue #219]([https://github.com/googlesamples/assistant-sdk-python/issues/219](https://github.com/googlesamples/assistant-sdk-python/issues/219)), which points to a fix on [issue #228]([https://github.com/googlesamples/assistant-sdk-python/pull/228](https://github.com/googlesamples/assistant-sdk-python/pull/228)), which [reverts to shared audio stream]([https://github.com/googlesamples/assistant-sdk-python/pull/228/commits/ec296e9a83fcb0ba6f22905b22e8546d98cfce43](https://github.com/googlesamples/assistant-sdk-python/pull/228/commits/ec296e9a83fcb0ba6f22905b22e8546d98cfce43)). Though this fix was already implemented in my downloaded version, so it had to be something else.

Thinking about it logically, I realized that the google assistant tries to use a stream that is already being used. This made it pretty clear that the stream remains open by _Snowboy_; ooh boy... Or so I thought! For a moment it seemed like I have to dive deep into _Snowboy_ to fix that. Luckily, by looking better at the `sbdemo7.py` and exploring how the `HotwordDetector` class of the `snowboydecoder` works, I realized that they start the detector, but they do not close it before invoking google assistant, but only after google assistant has returned:
```python
def detectedCallback():
    if detectedSignal > 2:
        snowboydecoder.play_audio_file()
    if detectedSignal > 1:
        # pixels.listen()
        pass
    if detectedSignal > 0:
        print('yes...', end='', flush=True)
    main() # in googlesamples.assistant.grpc.talkassist
    print('\nListening... Press Ctrl+C to exit')

detector = snowboydecoder.HotwordDetector(SnowboyModel, sensitivity=0.5)
print('Listening... Press Ctrl+C to exit')

# main loop
detector.start(detected_callback=detectedCallback,
               interrupt_check=interrupt_callback,
               sleep_time=sleepTime)

detector.terminate()
```
So, I added a line to terminate the detector before the assistant is called:
```python
def detectedCallback():
    if detectedSignal > 2:
        snowboydecoder.play_audio_file()
    if detectedSignal > 1:
        # pixels.listen()
        pass
    if detectedSignal > 0:
        print('yes...', end='', flush=True)
        
    # Terminate detector to free the audio stream for the GAssistant
    detector.terminate()
    main() # in googlesamples.assistant.grpc.talkassist
    print('\nListening... Press Ctrl+C to exit')

```
 And it worked moving away from that error! The result: I got to ask a single question, then it stopped. The reason is that the Google assistant (`pushtotalk.main` or `talkassist.main`) does not return properly and just exits. With this, we reach the point where, as described before, they agreed that this is an issue coming from the Google Assistant SDK. :deep breath:

## Getting my hands dirty
To examine the reasons why the `talkassist.main()` does not return we have to see what is going on in the `talkassist.py` (or `pushtotalk.py`). But before we get there, I have to admit I was lucky to get a big hint that would point me in the right direction and resulting in solving the problem in the long run.

So, when I was asking a question (the only one I was allowed to ask before it stops), I noticed that for bigger answers I was getting a warning message:
```shell
$ python sbdemo7.py
Listening... Press Ctrl+C to exit
INFO:snowboy:Keyword 1 detected at time: 2019-12-07 23:57:53
yes...WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)
```
Googling about it, I found on [issue #12](https://github.com/googlesamples/assistant-sdk-python/issues/12#issuecomment-299081964) that you can set the argument `--audio-block-size` to some bigger value:
```shell
python -m googlesamples.assistant  --audio-block-size=4096
```
This means that the `pushtotalk.main` (or `talkassist.main` in our case) takes this argument. As a matter of fact, here is the definition of the `main()`:
```python
def main(api_endpoint, credentials, project_id,
         device_model_id, device_id, device_config, lang, verbose,
         input_audio_file, output_audio_file,
         audio_sample_rate, audio_sample_width,
         audio_iter_size, audio_block_size, audio_flush_size,
         grpc_deadline, once, *args, **kwargs):
```
So, I thought I could change the invocation of `main()` in `sbdemo7.py` to `main(audio_block_size=4096)`. Hint: it didn't work.

Looking closer to the `pushtotalk.main()` I could see a set of decorators on top of it, which could play a role on this weird behaviour:
```python
@click.command()
@click.option('--api-endpoint', default=ASSISTANT_API_ENDPOINT,
              metavar='<api endpoint>', show_default=True,
              help='Address of Google Assistant API service.')
@click.option('--credentials',
              metavar='<credentials>', show_default=True,
              default=os.path.join(click.get_app_dir('google-oauthlib-tool'),
                                   'credentials.json'),
              help='Path to read OAuth2 credentials.')
@click.option('--project-id',
              metavar='<project id>',
              help=('Google Developer Project ID used for registration '
                    'if --device-id is not specified'))
@click.option('--device-model-id',
              metavar='<device model id>',
              help=(('Unique device model identifier, '
                     'if not specifed, it is read from --device-config')))
@click.option('--device-id',
              metavar='<device id>',
              help=(('Unique registered device instance identifier, '
                     'if not specified, it is read from --device-config, '
                     'if no device_config found: a new device is registered '
                     'using a unique id and a new device config is saved')))
@click.option('--device-config', show_default=True,
              metavar='<device config>',
              default=os.path.join(
                  click.get_app_dir('googlesamples-assistant'),
                  'device_config.json'),
              help='Path to save and restore the device configuration')
@click.option('--lang', show_default=True,
              metavar='<language code>',
              default='en-US',
              help='Language code of the Assistant')
@click.option('--verbose', '-v', is_flag=True, default=False,
              help='Verbose logging.')
@click.option('--input-audio-file', '-i',
              metavar='<input file>',
              help='Path to input audio file. '
              'If missing, uses audio capture')
@click.option('--output-audio-file', '-o',
              metavar='<output file>',
              help='Path to output audio file. '
              'If missing, uses audio playback')
@click.option('--audio-sample-rate',
              default=audio_helpers.DEFAULT_AUDIO_SAMPLE_RATE,
              metavar='<audio sample rate>', show_default=True,
              help='Audio sample rate in hertz.')
@click.option('--audio-sample-width',
              default=audio_helpers.DEFAULT_AUDIO_SAMPLE_WIDTH,
              metavar='<audio sample width>', show_default=True,
              help='Audio sample width in bytes.')
@click.option('--audio-iter-size',
              default=audio_helpers.DEFAULT_AUDIO_ITER_SIZE,
              metavar='<audio iter size>', show_default=True,
              help='Size of each read during audio stream iteration in bytes.')
@click.option('--audio-block-size',
              default=audio_helpers.DEFAULT_AUDIO_DEVICE_BLOCK_SIZE,
              metavar='<audio block size>', show_default=True,
              help=('Block size in bytes for each audio device '
                    'read and write operation.'))
@click.option('--audio-flush-size',
              default=audio_helpers.DEFAULT_AUDIO_DEVICE_FLUSH_SIZE,
              metavar='<audio flush size>', show_default=True,
              help=('Size of silence data in bytes written '
                    'during flush operation'))
@click.option('--grpc-deadline', default=DEFAULT_GRPC_DEADLINE,
              metavar='<grpc deadline>', show_default=True,
              help='gRPC deadline in seconds')
@click.option('--once', default=False, is_flag=True,
              help='Force termination after a single conversation.')
def main(api_endpoint, credentials, project_id,
         device_model_id, device_id, device_config, lang, verbose,
         input_audio_file, output_audio_file,
         audio_sample_rate, audio_sample_width,
         audio_iter_size, audio_block_size, audio_flush_size,
         grpc_deadline, once, *args, **kwargs):
```
Prior to that I knew nothing about the very existence of the `click` package (apologies if I insult anyone :man_shrugging:). From its [API documentation]([https://click.palletsprojects.com/en/5.x/api/](https://click.palletsprojects.com/en/5.x/api/)):

>click.command(name=None, cls=None, **attrs)
	Creates a new 'Command' and uses the decorated function as callback.
	This will also automatically attach all decorated 'option()'s and
	'argument()'s as parameters to the command.


Hmm, so it forms the command with the decorators and it looks like an argument can only be overwritten by calling the function directly from the command line. Since I do not need to call it from the command line, I first tried to remove the decorators completely and see what happens. Then, if that was too optimistic, I would come back to see if I can find any `click` options that would allow `talkassist.main` to return normally.

Thus copied all the default arguments on the decorators, in the definition of the function. The changes are shown below:
```python
def main(api_endpoint=ASSISTANT_API_ENDPOINT,
        credentials=os.path.join(click.get_app_dir('google-oauthlib-tool'), 'credentials.json'),
        device_config=os.path.join(click.get_app_dir('googlesamples-assistant'),'device_config.json'),
        device_id=None,
        project_id=None,
        device_model_id=None,
        input_audio_file=None,
        output_audio_file=None,
        audio_sample_rate=audio_helpers.DEFAULT_AUDIO_SAMPLE_RATE,
        audio_sample_width=audio_helpers.DEFAULT_AUDIO_SAMPLE_WIDTH,
        audio_block_size=audio_helpers.DEFAULT_AUDIO_DEVICE_BLOCK_SIZE,
        audio_flush_size=audio_helpers.DEFAULT_AUDIO_DEVICE_FLUSH_SIZE,
        audio_iter_size=audio_helpers.DEFAULT_AUDIO_ITER_SIZE,
        lang='en-US',
        verbose=False,
        once=False,
        grpc_deadline=DEFAULT_GRPC_DEADLINE):
```
Back to `sbdemo7.py` we have to add a while loop, because if the assistant returns the program will exit, but we need it to be there for us and detect more spoken hotwords. So:
```python
# main loop
while True:
    detector.start(detected_callback=detectedCallback,
               interrupt_check=interrupt_callback,
               sleep_time=sleepTime)
```
Test time:
```shell
$ python sbdemo7.py
Listening... Press Ctrl+C to exit
INFO:snowboy:Keyword 1 detected at time: 2019-12-08 18:22:53
yes...WARNING:root:SoundDeviceStream write underflow (size: 1600)

Listening... Press Ctrl+C to exit
INFO:snowboy:Keyword 1 detected at time: 2019-12-08 18:23:03
yes...WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)
WARNING:root:SoundDeviceStream write underflow (size: 4000)

Listening... Press Ctrl+C to exit
INFO:snowboy:Keyword 1 detected at time: 2019-12-08 18:23:17
yes...WARNING:root:SoundDeviceStream write underflow (size: 1600)
WARNING:root:SoundDeviceStream write underflow (size: 1600)
WARNING:root:SoundDeviceStream write underflow (size: 1600)
```
IT WORKED!!! :sob::sob::sob: *

\* <font color='gray' size=3>That was my last attempt at 3:30 am on a Saturday (well... Sunday), with a girlfriend already pissed off of me shouting "Hey Joe" all night while she's trying to sleep. I'm sure some of you will sympathize with me! </font>:relieved:

## Final touch

To make things easier, my plan for now is to have the google sample code available in any directory I want. To achieve this I made a copy of the `talkassist.py` and named it `custom_assist.py`. I placed it in the _Snowboy_'s example directory (`~/snowboy/examples/Python3/`), and thus I also had to change part of the imports, from:
```python
try:
    from . import (
        assistant_helpers,
        audio_helpers,
        device_helpers
    )
```
to:
```python
try:
    from googlesamples.assistant.grpc import (
        assistant_helpers,
        audio_helpers,
        device_helpers
    )
```
 I also cleaned the file from unnecessary code that was not expected to be used (shown as commented):

```python
    # @device_handler.command('action.devices.commands.OnOff')
    # def onoff(on):
    #    if on:
    #        logging.info('Turning device on')
    #    else:
    #        logging.info('Turning device off')

    with SampleAssistant(lang, device_model_id, device_id,
                         conversation_stream,
                         grpc_channel, grpc_deadline,
                         device_handler) as assistant:
        # If file arguments are supplied:
        # exit after the first turn of the conversation.
        # if input_audio_file or output_audio_file:
        #    assistant.assist()
        #    return

        while assistant.assist():
            if once:
                break
```

