[![arXiv](https://img.shields.io/badge/arXiv-2411.03706-b31b1b.svg)](https://arxiv.org/abs/2411.03706)
# 3DGS-CD
3D Gaussian Splatting-based Change Detection for Physical Object Rearrangement

![1](https://github.com/user-attachments/assets/659806cd-d127-48aa-addb-771db4458926)


TLDR: We estimate **3D object-level changes** from two sets of unaligned RGB images using **3D Gaussian Splatting** as the scene representation, enabling **accurate** recovery of shapes and pose changes of rearranged objects in **cluttered environments** within **tens of seconds** using **sparse (as few as one) new images**.

<details>
  <summary>Watch the demo video</summary>

[![Watch the video](https://via.placeholder.com/100)](https://github.com/user-attachments/assets/ef073079-6bed-4a06-8f0e-4e765a5fd680)

</details>

## Data


<details>
  <summary>Click to expand</summary>
<p>&nbsp;</p>

The **3DGS-CD dataset** can be found [here](https://drive.google.com/drive/folders/1OPUu643bkbAoryASNMi8_iDJGnypotc0?usp=drive_link).
All the RGB images have been pre-processed (i.e. downscaled and undistorted).
Below is the structure of the data folder:
```
scene_name
  - rgb: Pre-change images
  - rgb_new: Post-change images
    - Images at indices 0, 2, 4, ... are used for change detection
    - Images at indices 1, 3, 5, ... are used for evaluation
  - masks_gt: Ground truth change masks for evaluation images
  - nerfstudio_models: Pre-change 3DGS model weights
  - config.yml: Config file for the pre-change 3DGS model
  - transforms.json: Pre- and post-change camera poses in NerfStudio format
  - configs.json: Hyper-parameters
```
</details>


## Installation

<details>
  <summary>Click to expand</summary>

### 1. Install nerfstudio dependencies

#### 1.0 Create conda environment

```bash
conda create --name gscd -y python=3.8
conda activate gscd
pip install --upgrade pip
```

#### 1.1 Install CUDA dependencies

Install PyTorch with CUDA and [tiny-cuda-nn](https://github.com/NVlabs/tiny-cuda-nn).
`cuda-toolkit` is required for building `tiny-cuda-nn`.

For CUDA 11.8:

```bash
pip install torch==2.1.2+cu118 torchvision==0.16.2+cu118 --extra-index-url https://download.pytorch.org/whl/cu118

conda install -c "nvidia/label/cuda-11.8.0" cuda-toolkit
pip install ninja git+https://github.com/NVlabs/tiny-cuda-nn/#subdirectory=bindings/torch
```

See [Dependencies](https://github.com/nerfstudio-project/nerfstudio/blob/main/docs/quickstart/installation.md#dependencies)
in the Installation documentation for more.

#### 1.2 Install nerfstudio dependencies

```
git clone https://github.com/520xyxyzq/3DGS-CD.git 3dgscd
cd 3dgscd
pip install --upgrade pip setuptools
pip install -e .
```

### 2. Install 3DGS-CD dependencies

#### 2.1 Install EfficientSAM

Follow EfficientSAM [instructions](https://github.com/yformer/EfficientSAM)

**OR** if you prefer pip install:

```bash
pip install git+https://github.com/yformer/EfficientSAM.git@c9408a74b1db85e7831977c66e9462c6f4891729
```
Download the EfficientSAM model weight from [here](https://github.com/yformer/EfficientSAM/blob/main/weights/efficient_sam_vits.pt.zip) and change line 21 of [this file](https://github.com/yformer/EfficientSAM/blob/main/efficient_sam/build_efficient_sam.py) in your python lib to point to the downloaded weight. 

#### 2.2 Install HLoc

```bash
pip install git+https://github.com/cvg/Hierarchical-Localization.git@73a3cb0f59659306eb6c15c7213137b2196c5ceb
```
#### 2.3 Install LightGlue

```bash
pip install git+https://github.com/cvg/LightGlue@035612541779b17897aa06d6ff19cb4060111616
```

</details>

## Instructions
<details>
  <summary>Click to expand</summary>

### Run on our data

```
python nerfstudio/scripts/change_det.py \
  --config <data_folder>/<scene_name>/config.yml \
  --transform <data_folder>/<scene_name>/transforms.json \
  --output <data_folder>/<scene_name> \
  --ckpt <data_folder>/<scene_name>/nerfstudio_models/
```

**NOTE**: 
1. All output masks are saved under `<data_folder>/<scene_name>/masks_new/`. The `mask_*.png` files are the object move-out masks (previous location), and the `mask_new_*.png` files are the move-in masks (new location).
2. We have uploaded the pre-change 3DGS models with the data. This means you do not need to train the pre-change 3DGS models.
3. The post-change camera pose estimation is already handled for you, and the poses are stored in the `transforms.json` file.


### Run on custom data

#### 1. Data capture:

(1) Use your camera (tested with iPhone-13 mini camera) to capture >150 images for your scene.

(2) Make object-level changes, such as removing or moving an object.

(3) Capture 1~10 images of the changed state of the scene at different angles.

(4) Upload your images to your favorite folder, e.g. `<data_folder>/<scene_name>/`.

(5) Organize them in the following data structure:

```
scene_name
  - rgb: pre-change images
  - rgb_new: post-change images
```

**NOTE:**
1. When capturing pre-change images, try to sufficiently cover your scene to make sure the pre-change 3DGS has a reasonable rendering quality for novel views.
2. When capturing post-change images, make sure most 3D changes (both old and new object 3D locations) are visible to the images.
3. We recommend starting with a simple case where a single object moves.

#### 2. Data Processing

Process and downscale the captured images using [this script](scripts/process_iphone_data.sh).

**NOTE:**
1. Remember to update the default parameters at the top of this script.

#### 3. Run our method

Run our method using [this script](scripts/real_gsplat_train.sh).

**NOTE:**
1. Remember to update the default parameters at the top of this script.
2. Modify `TRAIN_IDX` to indices of images in `rgb_new` you want to use for change detection.

</details>

## Known Issues
<details>
  <summary>Click to expand</summary>

### Parameter tuning
If the data is not captured carefully, our method can be sensitive to hyperparameters. Below are the key parameters we recommend tuning first:

```
mask_refine_sparse_view
  - Expand EfficientSAM box prompt for 2D change detection
  - 0.0 should be a good starting point
  - Increase if 2D change detection fails
pre_train_pred_bbox_expand
  - Expand EfficientSAM box prompt for 2D segmentation on the pre-change view (for removed/moved objects)
  - 0.05 should be a good starting point
  - Increase if 2D segmentation fails
proj_check_cutoff
  - Cutoff for multi-view mask fusion
  - 0.9 should be a good starting point
  - Increase if unwanted parts are included in the 3D segmentation volume.
  - Decrease if missing parts in the 3D segmentation volume
```

### Bug!
It wouldn’t be surprising if a bug slipped in somewhere in the pipeline. If you catch a bug, please [submit a PR](https://github.com/520xyxyzq/3DGS-CD/pulls) or open an issue to let us know.

**NOTE:**
1. Please include your debug information before raising an issue
2. Add `--debug` to [this line](https://github.com/520xyxyzq/3DGS-CD/blob/853b8621ce41715e366b456bebe28b34a8ad0340/scripts/real_gsplat_train.sh#L173) to enable debugging. Also remember to change [this line](https://github.com/520xyxyzq/3DGS-CD/blob/853b8621ce41715e366b456bebe28b34a8ad0340/nerfstudio/scripts/change_det.py#L76) to your debug folder



</details>


## Future Directions and Ideas
<details>
  <summary>Click to expand</summary>

<p>&nbsp;</p>

We’re excited about the future directions this work inspires and enables! Below, we highlight some promising research opportunities. If you're interested in exploring any of these, feel free to reach out—we’d love to chat!

### Sparse-view 3DGS-CD
Can we detect 3D changes with just 4 pre-change images and 4 post-change images?!

### 3DGS-CD enables robot workspace reset

Wouldn’t it be cool if your robot could automatically reset your tabletop every time you make a mess? Check out the simple simulated demo in Section V.B of our paper!

### Fast radiance field model update to reflect 3D changes

No need to recapture data and wait 30 minutes to retrain a radiance field model just because something moved in the scene. Let’s update it based on the estimated changes! Check out the [NeRF-Update](https://arxiv.org/pdf/2403.11024) paper and Section V.C of our paper.

### Non-rigid 3DGS-CD

Let's estimate non-rigid object transformations!

</details>



