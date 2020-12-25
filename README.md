# RC-PyTorch

<div align="center">
  <img src='figs/overview.png' width="80%"/>
</div>

## Learning Better Lossless Compression Using Lossy Compression

## [[Paper]](https://arxiv.org/abs/2003.10184) [[Slides]](https://data.vision.ee.ethz.ch/mentzerf/rc/2008-slides.pdf)

## Abstract

We leverage the powerful lossy image compression algorithm BPG to build a
lossless image compression system. Specifically, the original image is first
decomposed into the lossy reconstruction obtained after compressing it with BPG
and the corresponding residual.  We then model the distribution of the residual
with a convolutional neural network-based probabilistic model that is
conditioned on the BPG reconstruction, and combine it with entropy coding to
losslessly encode the residual. Finally, the image is stored using the
concatenation of the bitstreams produced by BPG and the learned residual coder.
The resulting compression system achieves state-of-the-art performance in
learned lossless full-resolution image compression, outperforming previous
learned approaches as well as PNG, WebP, and JPEG2000.

# About the Code

The released code is very close to what we used when running experiments
for the paper. The codebase evolved from the [L3C repo](https://github.com/fab-jul/L3C-PyTorch/)
and contains code that is unused in this paper.

#### Naming

Files specific to this paper usually are marked "enhancement" or "enh" as 
this was the internal name. We frequently use the following terms:

- `x_r` / "raw" / "ground-truth" (gt) image: The input image, uncompressed. Note that this is inconsistent with the paper, where `x_r` is the residual
- `x_l` / "lossy" / "compressed" image: The image obtained by feeding the raw iamge through BPG.
- `res`: Residual between raw and lossy.

# Setup Environment

## Folder structure

Suggested structure that is assumed by this README, but any
other structure is supported.

```
$RC_ROOT/
    datasets/       <-- See the "Preparing datasets" section.
    models/         <-- The result of get_models.sh, see below
    RC-PyTorch/     <-- The result of a git clone of this repo
        src/
        figs/
        README.md   <-- This file
        ...
```

To set this up:

```bash
RC_ROOT=/path/to/wherever/you/want
mkdir -p $RC_ROOT
pushd $RC_ROOT
git clone https://github.com/fab-jul/RC-PyTorch
```

## BPG

To run the code in this repo, you need BPG.
Follow the instractions on http://bellard.org/bpg/ to install it.

Afterwards, make sure `bpgenc` and `bpgdec` is in `$PATH` 
by running `bpgenc` in your console.
You can run the following script to test:

```bash
pushd $RC_ROOT/RC-PyTorch/src
bash test_bpg_available.sh
```

## Python Environment

To run the code, you need Python 3.7 or newer, and PyTorch 1.1.0. 
The following assumes you use [conda](https://docs.conda.io/en/latest/miniconda.html)
to set this up:

```bash
NAME=pyt11  # You can change this to whatever you want.
conda create -n $NAME python==3.7 pip -y
conda activate $NAME

# Note that we're using PyTorch 1.1 and CUDA 10.0. 
# Other combinations may also work but have not been tested!
conda install pytorch==1.1.0 torchvision cudatoolkit==10.0.130 -c pytorch

# Install the pip requirements
cd $RC_ROOT/RC-PyTorch/src
pip install -r requirements.txt
```

# Preparing datasets


We preprocess our datasets by compressing each image with a variety of
BPG quantization levels, as described in the following. Since the Q-classifier
has been trained for `Q \in {9, ..., 17}`, that's the range we use.
The pre-processing with all these Q is an artifact of the fact that this code
was used for experimentation also. In the "real world", we would run
BPG only with the Q that is output by the Q-classifier.

### CLIC.mobile and CLIC.pro

```bash
pushd "$RC_ROOT/datasets"

wget https://data.vision.ee.ethz.ch/cvl/clic/mobile_valid_2020.zip
unzip mobile_valid_2020.zip  # Note: if you don't have unzip, use "jar xf"
mv valid mobile_valid  # Give the extracted archive a unique name

wget https://data.vision.ee.ethz.ch/cvl/clic/professional_valid_2020.zip
unzip professional_valid_2020.zip
mv valid professional_valid  # Give the extracted archive a unique name
```

Preprocess with BPG (see above).

```bash
pushd "$RC_ROOT/RC-PyTorch/src"
bash prep_bpg_ds.sh A9_17 $RC_ROOT/datasets/mobile_valid
bash prep_bpg_ds.sh A9_17 $RC_ROOT/datasets/professional_valid
```

### Open Images Validation 500

For Open Images, we use the same validation set that we used for L3C,
which can be downloaded here: [Open Images Validation 500](http://data.vision.ee.ethz.ch/mentzerf/validation_sets_lossless/val_oi_500_r.tar.gz)

```bash
pushd "$RC_ROOT/datasets"
wget http://data.vision.ee.ethz.ch/mentzerf/validation_sets_lossless/val_oi_500_r.tar.gz
mkdir val_oi_500_r && pushd val_oi_500_r
tar xvf ../val_oi_500_r.tar.gz

pushd "$RC_ROOT/RC-PyTorch/src"
bash prep_bpg_ds.sh A9_17 $RC_ROOT/datasets/val_oi_500_r
```

### DIV2K

```bash
pushd "$RC_ROOT/datasets"
wget http://data.vision.ee.ethz.ch/cvl/DIV2K/DIV2K_valid_HR.zip
unzip DIV2K_valid_HR.zip

pushd "$RC_ROOT/RC-PyTorch/src"
bash prep_bpg_ds.sh A9_17 $RC_ROOT/datasets/DIV2K_valid_HR
```

# Running models used in the paper

### Download models

Download our models with `get_models.sh`:

```bash
MODELS_DIR="$RC_ROOT/models"
bash get_models.sh "$MODELS_DIR"
```

### Get bpsp on Open Images Validation 500

After downloading and preparing Open Images as above, and 
downloading the models, you can 
run our model on Open Images as follows:

``` 
# Running on Open Iamges 500
DATASET_DIR="$RC_ROOT/datasets"
MODELS_DIR="$RC_ROOT/models"

pushd "$ROOT/RC-PyTorch/src"

# Note: depending on your environment, adapt CUDA_VISIBLE_DEVICES.
CUDA_VISIBLE_DEVICEs=0 python -u run_test.py \
    "$MODELS_DIR" 1109_1715 "AUTOEXPAND:$DATASET_DIR/val_oi_500_r" \
    --restore_itr 1000000 \
    --tau \
    --clf_p "$MODELS_DIR/1115_1729*/ckpts/*.pt" \
    --qstrategy CLF_ONLY
```

The first three arguments are the location of the models,
the experiment ID (`1109_1715` here), and the dataset dir. The dataset dir
is special as we prefix it with `AUTOEXPAND`, which causes the tester
to get all the bpg folders that we created in the previous step (another
artifact of this being experimentation code). Here, you can also put any
other dataset that you preprocessed similar to the above.

If all goes well, you should see the 2.790 reported in the paper:

```
testset         exp         itr      Q           bpsp
val_oi_500...   1109_1715   998500   1.393e+01   2.790
```

## Get bpsp on other datasets

You can also pass multiple datasets by separating them with commas.
For example, to run on all datasets of the paper:

```
CUDA_VISIBLE_DEVICEs=0 python -u run_test.py \
    "$MODELS_DIR" 1109_1715 \
    "AUTOEXPAND:$DATASET_DIR/professional_valid,AUTOEXPAND:$DATASET_DIR/DIV2K_valid_HR,AUTOEXPAND:$DATASET_DIR/mobile_valid,AUTOEXPAND:$DATASET_DIR/val_oi_500_r" \
    --restore_itr 1000000 \
    --tau \
    --clf_p "$MODELS_DIR/1115_1729*/ckpts/*.pt" \
    --qstrategy CLF_ONLY
```

Expected Output:
```
testset                 exp         itr      Q           bpsp
DIV2K_valid_HR...       1109_1715   998500   1.378e+01   3.078
mobile_valid...         1109_1715   998500   1.325e+01   2.537
professional_valid...   1109_1715   998500   1.385e+01   2.932
val_oi_500_r...         1109_1715   998500   1.393e+01   2.790
```

## Sampling

To get the sampling figures shown in the paper, pass `--sample=some/dir`
to `run_test.py`.

## Our TB

TODO: Link our tensorboard.


# Training your own models


## Prepare datasets

## Train

To train your own models, use `train.py`. The following command was used
to train the models released above:

```bash

# You can set this to whatever you want
LOGS_DIR="$RC_ROOT/models"

CUDA_VISIBLE_DEVICES=0 python train.py \
    configs/ms/en/gdn_wide_deep3.cf \
    configs/dl/new_oi_q12_14_128.cf \
    $LOGS_DIR  \
    -p unet_skip 
```




















