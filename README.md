Re-implementation of our paper [Voice Conversion from Non-parallel Corpora Using Variational Auto-encoder](https://arxiv.org/abs/1610.04019).  
<!-- 2. [Voice Conversion from Unaligned Corpora using Variational Autoencoding Wasserstein Generative Adversarial Networks](https://arxiv.org/abs/1704.00849) -->
(for VAWGAN, please switch to `vawgan` branch)

# Dependency
Linux Ubuntu 16.04  
Python 3.5  

- Tensorflow-gpu 1.2.1
- Numpy
- Soundfile
- PyWorld
  - Cython
<br/>


# Setting up the environment
For example,  
```bash
conda create -n py35tf121 -y python=3.5
source activate py35tf121
pip install -U pip
pip install -r requirements.txt
```

### Note:
1. `soundfile` might require `sudo apt-get install`.  
1. You can use any virtual environment packages (e.g. `virtualenv`)
2. If your Tensorflow is the CPU version, you might have to replace all the `NCHW` ops in my code because Tensorflow-CPU only supports `NHWC` op and will report an error: `InvalidArgumentError (see above for traceback): Conv2DCustomBackpropInputOp only supports NHWC.`
3. I recommend installing Tensorflow from the link on their Github repo.  
    `pip install -U [*.whl link on the Github page]` 

<br/>


# Usage
1. Run `bash download.sh` to prepare the VCC2016 dataset.  
2. Run `analyzer.py` to extract features and write features into binary files. (This takes a few minutes.)  
3. Run `build.py` to record some stats, such as spectral extrema and pitch.  
4. To train a VAE, for example, run
```bash
python main.py \
--model ConvVAE \
--trainer VAETrainer \
--architecture architecture-vae-vcc2016.json
```  
5. You can find your models in `./logdir/train/[timestamp]`  
6. To convert the voice, run
```bash
python convert.py \
--src SF1 \
--trg TM3 \
--model ConvVAE \
--checkpoint logdir/train/[timestamp]/[model.ckpt-[id]] \
--file_pattern "./dataset/vcc2016/bin/Testing Set/{}/*.bin"
```  
*Please fill in `timestampe` and `model id`.  
7. You can find the converted wav files in `./logdir/output/[timestamp]`  

<br/>


# Dataset
Voice Conversion Challenge 2016 (VCC2016): [download page](https://datashare.is.ed.ac.uk/handle/10283/2211)  
<br/>

# Model  
 - [x] Conditional VAE

<br/>



# File/Folder
```
dataset
  vcc2016
    bin
    wav
      Training Set
      Testing Set
        SF1
        SF2
        ...
        TM3
etc
  speakers.tsv  (one speaker per line)  
  (xmax.npf)  
  (xmin.npf)  
util (submodule)
model
logdir
architecture*.json

analyzer.py    (feature extraction)
build.py       (stats collecting)
trainer*.py
main.py        (main script)
(validate.py)  (output converted spectrogram) 
convert.py     (conversion)
```
<br/>



# Binary data format
The [WORLD vocdoer](https://github.com/mmorise/World) features and the speaker label are stored in binary format.  
Format:  
```
[[s1, s2, ..., s513, a1, ..., a513, f0, en, spk],
 [s1, s2, ..., s513, a1, ..., a513, f0, en, spk],
 ...,
 [s1, s2, ..., s513, a1, ..., a513, f0, en, spk]]
```
where   
`s_i` is spectral envelop magnitude (in log10) of the ith frequency bin,  
`a_i` is the corresponding "aperiodicity" feature,   
`f0` is the pitch (0 for unvoice frames),  
`en` is the energy,  
`spk` is the speaker index (0 - 9) and `s` is the `sp`.

Note:
  - The speaker identity `spk` was stored in `np.float32` but will be converted into `tf.int64` by the `reader` in `analysizer.py`.
  - I shouldn't have stored the speaker identity per frame;
    it was just for implementation simplicity. 

<br/>

# Modification Tips
1. Define a new model (and an accompanying trainer) and then specify the `--model` and `--trainer` of `main.py`.  
2. Tip: when creating a new trainer, override `_optimize()` and the main loop in `train()`.
3. Code orgainzation

<!-- <img src="etc/CodeOrganizaion.png" />  -->
 ![Code organization](etc/CodeOrganization.png) 
This isn't a UML; rather, the arrows indicates input-output relations only.

<br/>

# Difference from the original paper
1. WORLD vocoder is chosen in this repo instead of STRAIGHT because the former is open-sourced whereas the latter isn't.  
   I use [pyworld](https://github.com/JeremyCCHsu/Python-Wrapper-for-World-Vocoder), Python wrapper of the WORLD, in this repo.
2. Global variance post-filtering was not included in this repo.
3. In our VAE-NPVC paper, we didn't apply the [-1, 1] normalization; we did in our VAWGAN-NPVC paper.
<br/>


# About
The original code base was originally built in March, 2016.  
Tensorflow was in version 0.10 or earlier, so I decided to refactor my code and put it in this repo.


# TODO
 - [ ] `util` submodule (add to README)
 - [ ] GV
 - [ ] `build.py` should accept subsets of speakers
# #1 Abstract
 In this propose a flexible framework for spectral conversion
(SC) that facilitates training with unaligned corpora.
Many SC frameworks require parallel corpora, phonetic alignments,
or explicit frame-wise correspondence for learning conversion
functions or for synthesizing a target spectrum with
the aid of alignments. However, these requirements gravely
limit the scope of practical applications of SC due to scarcity
or even unavailability of parallel corpora. We propose an SC
framework based on variational auto-encoder which enables us
to exploit non-parallel corpora. The framework comprises an
encoder that learns speaker-independent phonetic representations
and a decoder that learns to reconstruct the designated speaker.
It removes the requirement of parallel corpora or phonetic
alignments to train a spectral conversion system. We report
objective and subjective evaluations to validate our proposed
method and compare it to SC methods that have access to aligned
corpora.
# #2 suggested method
To improve the performance of this transformation function, another transformation can be used using the Fourier transformation function, which is as follows.
# #3 fourier transform
def convert_f0(f0, src, trg):
    mu_s, std_s = np.fromfile(os.path.join('./etc', '{}.npf'.format(src)), np.float32)
    mu_t, std_t = np.fromfile(os.path.join('./etc', '{}.npf'.format(trg)), np.float32)
    lf0 = tf.where(f0 > 1., tf.log(f0), f0)
    lf0 = tf.where(lf0 > 1., (lf0 - mu_s)/std_s * std_t + mu_t, lf0)
    lf0 = tf.where(lf0 > 1., tf.exp(lf0), lf0)
    return lf0
# #4 Result
By moving the transformations, the output result was palpable in the resolution of the original sound, and the encoding can be considered valid for this audio data sample.
# #5 refrences 
Chin-Cheng Hsu, Hsin-Te Hwang, Yi-Chiao Wu, Yu Tsaoy and Hsin-Min Wang
 Institute of Information Science, Academia Sinica, Taipei, Taiwan Voice Conversion from Non-parallel Corpora Using
Variational Auto-encoder.
# #6 introduce
My name is Nima Azimi and I'm M.sc of biomedical engineering student , I've choosen this type of project becuase of the importance of the seceurity of the signal data and it's nessecarry to keep them safe . 
# #7
In the end, I would like to thank Dr. Islami, who introduced me to this field and guided me.
