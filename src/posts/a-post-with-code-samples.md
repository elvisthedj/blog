---
title: The Key to Learning a New Language (Python)
date: 2020-06-29T21:21:42.988Z
tags:
  - code
  - blog
  - perl
---
I'll start with a disclaimer:

> I am not a professional coder

That being said, I have written many useful programs (specifically scripts) in my life. Since I spend most of my time in the Linux terminal, I do most of my scripting in [Bash](https://www.gnu.org/software/bash/manual/html_node/What-is-Bash_003f.html). The complexity of the scripts I write depend on my needs...and that is the secret: **have a need**.

I spent a lot of time trying to learn programming languages by reading books, but good luck retaining that info without putting it into practice. Sure, most books will give you samples, but for me that's not good enough. I need to want my computer to do something it is currently unable to do...I need to solve a problem for myself. 

My latest example of such a need relates to [Asterisk PBX](asterisk.org) - A complex piece of telephony software that is powerful enough to run the phone systems of huge companies. I have used Asterisk for many years (going on 20 in fact) for no reason other than having a nearly lifelong obsession with telephony.

Recently, I decided to write a new IVR (menu) for my phone. I began writing it in [Perl](https://perl.org), but why waste such an opportunity to learn something completely foreign to me. It was the perfect excuse to start learning [Python](https://www.python.org/) 

I've wanted to learn Python for a few years now. I've read some tutorials, watched some videos and even played with a couple of "learn Python" apps on my phone, but as I mentioned before, that's just not enough for me. It wasn't wasted time...I did have at least an overview of the language and the lingo before jumping in to a project, but I was essentially staring from scratch.

Asterisk has its own gateway interface ([AGI](https://www.voip-info.org/asterisk-agi/)) which can interact with pretty much any programming language. It's also popular enough that there are wrappers for most of the major languages, [including Python](https://pypi.org/project/pyst3/).

After about 4 hours (probably 20 minutes or less for a skilled Python dev) I had a working menu system. I haven't refactored this code yet, so it still contains plenty of lines that are just stuck in to make things work. This means that it is inefficient, redundant, and since I'm new it doesn't follow proper Python naming conventions (in other words, it's not meant to be a learning tool. Despite all of its short comings, I have a phone that is taking full advantage of [Google Cloud's](https://cloud.google.com) speech functionality (text to speech and speech to text), so I'm pretty happy. I'll use this ongoing project to actually learn Python instead of hacking things together. Snippet:

```python
.import hashlib
import os
import io
import re
import datetime as dt
from os import path
import phonenumbers
from asterisk.agi import *
import mysql.connector as mysql
from google.cloud import texttospeech
from google.cloud import speech_v1
from google.cloud.speech_v1 import enums

#Google Credentials
os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "/var/lib/asterisk/agi-bin/creds.json"

#DB Setup
db = mysql.connect(
	host = "localhost",
	user = "user",
	passwd = "super_secret",
	database = "asterisk"
)

agi = AGI()
mycursor = db.cursor()


def tts(say):
    cacheDir = "/var/lib/asterisk/tmp/ttscache/"
    cachedFile = hashlib.md5(say.encode())
    fullName = cacheDir + cachedFile.hexdigest() + ".wav"

    if os.path.isfile(cacheDir + cachedFile.hexdigest() + ".wav"):
        agi.verbose("Found cached file")
        agi.stream_file(cacheDir + cachedFile.hexdigest())
        agi.verbose("Full name: " + fullName)

    else:
        client = texttospeech.TextToSpeechClient()
        synthesis_input = texttospeech.SynthesisInput(text=say)
        voice = texttospeech.VoiceSelectionParams(
        language_code = 'en-GB',
        name = 'en-GB-Wavenet-C',
        ssml_gender=texttospeech.SsmlVoiceGender.FEMALE)
        audio_config = texttospeech.AudioConfig(
            audio_encoding=texttospeech.AudioEncoding.LINEAR16,
            sample_rate_hertz = 8000)
        response = client.synthesize_speech(
            input=synthesis_input, voice=voice, audio_config=audio_config
        )

        with open(fullName, 'wb') as out:
            out.write(response.audio_content)
        agi.stream_file(cacheDir + cachedFile.hexdigest())


def asr():
    agi.appexec("monitor","wav,/var/lib/asterisk/tmp/asr/asr")
    agi.appexec("backgrounddetect","thinking")
    agi.appexec("stopmonitor")
    language_code = "en-US"
    #voice_name = "en-US-Wavenet-D"
    local_file_path = '/var/lib/asterisk/tmp/asr/asr-in.wav'
    client = speech_v1.SpeechClient()
    sample_rate_hertz = 8000
    encoding = enums.RecognitionConfig.AudioEncoding.LINEAR16
    config = {
        "language_code": language_code,
        "sample_rate_hertz": sample_rate_hertz,
        "encoding": encoding,
    }
    with io.open(local_file_path, "rb") as f:
        content = f.read()
    audio = {"content": content}
    response = client.recognize(config, audio)

    for result in response.results:
        # First alternative is the most probable result
        alternative = result.alternatives[0]
        #print(u"Transcript: {}".format(alternative.transcript))
    decoded = format(alternative.transcript)
    agi.verbose("Decoded: " + decoded)
    return decoded

def addCaller(callerIDNum, callerIDName, friendlyName):
    mycursor.execute("INSERT INTO callers (callerIDNum, callerIDName, friendlyName, vip) VALUES (%s, %s, %s, '0')",
                     (strippedId, callerIDName, friendlyName))
    db.commit()
	#rowcount = mycursor.rowcount
	#agi.verbose(rowcount + "Caller Added")

def direct(friendlyName):
    tts("Let me see if Kris is available")
    agi.verbose("Priority routing for " + friendlyName)
    agi.appexec("Dial","SIP/goober,60,mtTk")
    status = agi.get_variable("DIALSTATUS")
    agi.verbose("Call status was: " +status)
    if status == "NOANSWER" or status == "BUSY":
        noAnswer()
    else:
        tts("Thank you for calling, " + friendlyName)
        tts("Goodbye.")
        agi.appexec("hangup")
....
```