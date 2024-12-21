# Respeaker-2-Mic
Respeaker 2 Mic / Zero 2 Beamformer setup guide

A 2 channel beamformer will only focus along the X axis and are often seen in many webcams nowadays.
Really they should be front facing as the enclosure it self will attenuate sound from the rear.
The beamformer we will use is a simple 'delay sum' beamformer as described in the excellent Invensense application note.
https://invensense.tdk.com/wp-content/uploads/2015/02/Microphone-Array-Beamforming.pdf

Our brain automatically ignores room reverberation and focuses on what we want to hear by extracting it from noise and reverberation.
'delay sum' beamforming uses a Gccphat alg to calculate the delay between 2 microphones so that the reference mic sample
is added to +/_ samples of the delay and corrects the orrentation of the mics so they now focus the sound source.

This guide will show you how to install the software, whilst providing hopefully not to technical intoduction to beamforming.
The orientation of the microphones matter and often the default hat will have the microphones pointing upwards which is not ideal.
Buy a Pi Zero without the header and get a 90' right angle header from the store if you want a table microphone as then the mics will be facing you.
Its some soldering and even though the attenuation of rear sounds and the gain of forward focussed sounds is relatively small all these add up to create a better mic array.

The beamformer will only help with farfield and cut down some of the room reververberation and isn't at the level of the latest generation of smart speaker microphones.
It should be an improvement over a single mic though.

Download and flash a 64bit version of PiOS Lite to your SD card and provide the network and ssh setting you wish.

1st update and upgrade the software and reboot
```
sudo apt update
sudo apt upgrade
sudo apt reboot
```
Install the respeaker driver
```
sudo apt install git
git clone https://github.com/HinTak/seeed-voicecard.git
cd seeed-voicecard
sudo ./install.sh
sudo reboot
```
On reboot we can list the sound devices with `aplay -l` where you should see something like this
```
**** List of PLAYBACK Hardware Devices ****
card 0: vc4hdmi [vc4-hdmi], device 0: MAI PCM i2s-hifi-0 [MAI PCM i2s-hifi-0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: seeed2micvoicec [seeed-2mic-voicecard], device 0: 3f203000.i2s-wm8960-hifi wm8960-hifi-0 [3f203000.i2s-wm8960-hifi wm8960-hifi-0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```
Every time RaspiOS is updated often the kernel headers lag behind the release and the driver will fail to install.
Currently its some time after the last release and the headers are there and the software works.

Install the beamformer
```
sudo apt install cmake autotools-dev autoconf libtool pkg-config portaudio19-dev
git clone https://github.com/StuartIanNaylor/2ch_delay_sum.git
cd 2ch_delay_sum
make clean
make
sudo modprobe snd-aloop
```
`./ds` which will run the delay-sum beamformer but fail without any parameters
The last couple of lines should display something like this
```
Devices:
0: () seeed-2mic-voicecard: 3f203000.i2s-wm8960-hifi wm8960-hifi-0 (hw:1,0) (ALSA) --default--
1: () Loopback: PCM (hw:2,0) (ALSA)
2: () Loopback: PCM (hw:2,1) (ALSA)
```
You can also `./ds --help`
```
./ds --help

Do delay and sum beamforming
Usage: ds [options] input_device[index] output_device[index]
ds --frames=4800 --channels=2 --margin=20 --sample_rate=48000 --display_levels=1 0 0
Margin (Max(TDOA) = mic distance mm / (343000 / sample_rate)
Set display_levels=0 for silent running

Options:
  --channels                  : mic input channels (int, default = 2)
  --display_levels            : display_levels (int, default = 1)
  --frames                    : frame buffer size (int, default = 4000)
  --margin                    : constraint for tdoa estimation (int, default = 16)
  --sample_rate               : sample rate (int, default = 48000)
```
`./ds --frames=4800 --channels=2 --margin=20 --sample_rate=48000 --display_levels=1 0 1`
`--frames` just keep as is for now, what we need to do is set in the input device and output device.
From the above devices `./ds` provides the seeed 2mic is device 0 and then we have the 1st Loopback device as output, which is 1`
A Alsa loopback device allows you to output audio on device 1 which is an alsa sink.
The corresponding device 2 (hw:2,1) is now available as a microphone source for any app which wants the output of the beamformer.
We have set the sample rate to 48Khz to get the best resolution we can `--sample_rate=48000`
Number of channels is 2 `--channels=2` and now we get to the margin.

The speed of sound is approx 343 m/s so in mm is 343,000 and if we divide that by our sample rate 48000 we get 7.14583mm
Which is the distance sound will move for the duration of a single sample.
So we can work backwards and measure the distance between the 2 mics which is approx 58.2mm
58.2 / 7.158 gives us 8.13 which means we will get a max of -8 samples @ -180; and + 8 samples @ +180 delay max
So we can set the margin to 8 as an optimal, but as lon as its the same size or larger than the max delay.
About 65/70mm is about the max distance of mic spacing or you will start to get an effect called aliasing (https://en.wikipedia.org/wiki/Aliasing) which you don't want.
So the sample rate and distance of the mics provides the resolution of the beamformer.
The number of microphones sets the maximum attenuation of out of focus audio, but requires another Gccphat calc for every microphone added.
The Gccphat calc is nearly all the load of the beamformer so 2 mics is the lowest computational cost, but you could add more mics but reference the invensense doc to how.

So anyway now that we come to try `./ds --frames=4800 --channels=2 --margin=20 --sample_rate=48000 --display_levels=1 0 1` it fails...
```
Expression 'r' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 2097
Expression 'PaAlsaStreamComponent_FinishConfigure( &self->capture, hwParamsCapture, inParams, self->primeBuffers, realSr, inputLatency )' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 2730
Expression 'PaAlsaStream_Configure( stream, inputParameters, outputParameters, sampleRate, framesPerBuffer, &inputLatency, &outputLatency, &hostBufferSizeMode )' failed in 'src/hostapi/alsa/pa_linux_alsa.c', line: 2842

An error occured while using the portaudio stream
Error number: -9999
Error message: Unanticipated host error
```
```
wget https://file-examples.com/storage/fefaeec240676402c9bdb74/2017/11/file_example_WAV_10MG.wav
play -Dplughw:1 file_example_WAV_10MG.wav
Playing WAVE 'file_example_WAV_10MG.wav' : Signed 16 bit Little Endian, Rate 44100 Hz, Stereo
aplay: set_params:1416: Unable to install hw params:
ACCESS:  RW_INTERLEAVED
FORMAT:  S16_LE
SUBFORMAT:  STD
SAMPLE_BITS: 16
FRAME_BITS: 32
CHANNELS: 2
RATE: 44100
PERIOD_TIME: (125011 125012)
PERIOD_SIZE: 5513
PERIOD_BYTES: 22052
PERIODS: 4
BUFFER_TIME: (500045 500046)
BUFFER_SIZE: 22052
BUFFER_BYTES: 88208
TICK_TIME: 0
```

Looks like the drivers are lagging or botched again which from experience happens quite regular.
I don't fancy that rabbit hole and seraching for a fix just going to use a stereo ADC USB soundcard for now which does work...


  


