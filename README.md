# SceneComplete

### [🌐 Project Website](https://scenecomplete.github.io) | [📝 Paper](https://arxiv.org/pdf/2410.23643v1) | [🎥 Video](https://www.youtube.com/watch?v=Tuzhn4HWiL0)

**SceneComplete** is an *open-world 3D scene completion system*, that constructs a complete, segmented, 3D model of a scene from a single RGB-D image.

![](data/scenecomplete_architecture.gif)

Please read the official paper for a detailed overview of our work. 
> **SceneComplete: Open-World 3D Scene Completion in Complex Real-World Environments for Robot Manipulation**  
> Aditya Agarwal, Gaurav Singh, Bipasha Sen, Tomás Lozano-Pérez, Leslie Pack Kaelbling (2024)  
> [arXiv:2410.23643v2]()

-----

## Update 
We will soon release the finetuned inpainting model, addressing known issues with the default version. We have also made engineering improvements to mitigate missing objects due to segmentation and VLM failures (an updated prompt template and reprompting fixes most issues). We will release those updates shortly. 

-----

**Table of Contents**

- [🛠️ Installation](#-installation)
- [🚀 Usage](#-usage)
   - [Downloading Pretrained Weights](#downloading-pretrained-weights)
   - [Setting up Environment Variables](#setting-up-environment-variables)
   - [Running SceneComplete](#running-scenecomplete)
   - [Visualizing Outputs](#visualizing-outputs)
- [🫶 Limitations and Contributing to SceneComplete](#-limitations-and-contributing-to-scenecomplete)
- [💖 Acknowledgements](#-acknowledgements)
- [📜 Cite Us](#-cite-us)


## 🛠️ Installation
#### 1. Setup conda environment
```bash
# We recommend using conda to manage your environments
conda create -n scenecomplete python=3.9
conda activate scenecomplete
```

#### 2. Clone and install SceneComplete
```bash
git clone https://github.com/scenecomplete/SceneComplete.git
cd SceneComplete
git submodule update --init --recursive
```

#### 3. Install submodule dependencies
We provide a script to download and setup submodule dependencies automatically
```bash
bash scenecomplete/scripts/install_all.sh
```

### 4. Install FoundationPose dependencies
```bash
# Create foundationpose conda environment
conda create -n foundationpose python=3.9
conda activate foundationpose

# Install cuda-toolkit in your conda environment
conda install cuda -c nvidia/label/cuda-11.8.0
# Install torch for CUDA 11.8
pip install torch==2.0.0 torchvision==0.15.1 torchaudio==2.0.1 --index-url https://download.pytorch.org/whl/cu118

# Install GCC 11 (to the conda env)
conda install -c conda-forge gcc=11 gxx=11
# check the installed versions using `gcc -v` and `g++ -v`

# Add conda lib path to your LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/home/<username>/miniconda3/envs/foundationpose/lib:$LD_LIBRARY_PATH

# Install Eigen3 3.4.0 under conda environment
conda install conda-forge::eigen=3.4.0
export CMAKE_PREFIX_PATH="$CMAKE_PREFIX_PATH:/eigen/path/under/conda" (e.g., "/home/<username>/miniconda3/envs/foundationpose/include/eigen3")

# Install dependencies
cd scenecomplete/modules/FoundationPose
python -m pip install -r requirements.txt

# Install NVDiffRast, Kaolin, and PyTorch3D
python -m pip install --quiet --no-cache-dir git+https://github.com/NVlabs/nvdiffrast.git
python -m pip install --quiet --no-cache-dir kaolin==0.15.0 -f https://nvidia-kaolin.s3.us-east-2.amazonaws.com/torch-2.0.0_cu118.html
python -m pip install --quiet --no-index --no-cache-dir pytorch3d -f https://dl.fbaipublicfiles.com/pytorch3d/packaging/wheels/py39_cu118_pyt200/download.html

# Install Boost
conda install boost-cpp

# Build extensions, ensure that your CONDA_PREFIX points to your miniconda3 setup (e.g., /home/<username>/miniconda3)
CMAKE_PREFIX_PATH=$CONDA_PREFIX/lib/python3.9/site-packages/pybind11/share/cmake/pybind11 bash build_all_conda.sh

# Finally, install SceneComplete as a python package
cd ../../..
pip install -e .

# Troubleshooting: If you face issues related to nvdiffrast, try copying the contents of your lib folder to the lib64 folder in your conda environment.
```

## 🚀 Usage
### Downloading Pretrained Weights
We provide a script to download the pretrained weights of individual submodules. 

```bash
cd scenecomplete/modules/weights
bash download_weights.sh
```

This will automatically download and place the checkpoints in their respective directories. Google restricts large file downloads via scripts. If you encounter issues while downloading pretrained checkpoints, follow the steps in the `download_weights.sh` file. Huggingface weights required by the individual modules are downloaded automatically when the code is run for the first time. 

We finetune BrushNet using LoRA to adapt its performance on tabletop objects. We anticipate releasing the LoRA weights in the next few days. In the meantime, we use the pretrained BrushNet model for inpainting. 

### Setting up Environment Variables
```bash
# Set your datasets path
export scdirpath="<your datasets path>"

# Create your OpenAI API Key (https://platform.openai.com/api-keys) and add the secret as an environment variable.
export OPENAI_API_KEY="<your key>"
```

### Running SceneComplete
To run SceneComplete, you need to provide an RGB-D image of the scene along with the camera intrinsics. 
#### 1. Prepare input folder:
Create a folder called `inputs` inside the path you set earlier using $scdirpath, such that your inputs folder is located at `$scdirpath/inputs`.

Add the following files to your inputs folder:

- `rgb.png`: RGB image of the scene 
- `depth.png`: Corresponding depth image
- `cam_K.txt`: A text file containing the 3x3 camera intrinsic matrix

#### 2. Run the SceneComplete system:

```bash
# Run the scenecomplete bash script to run the pipeline
bash scenecomplete/scripts/bash/scenecomplete.sh

# Or run with a custom experiment ID
bash scenecomplete/scripts/bash/scenecomplete.sh <experiment_id>
```

This will create a folder at $scdirpath/<experiment_id> containing the intermediate and final outputs in `registered_meshes`.

### Visualizing Outputs
To visualize the reconstructed scene along with the input partial pointcloud,

```bash
python scenecomplete/utils/visualization.py \
   --mesh_dirpath $scdirpath/registered_meshes

# To visualize the input scene pointcloud
python scenecomplete/utils/visualize_pointcloud.py \
   --folder_path $scdirpath/inputs \
   --visualize
```

## 🫶 Limitations and Contributing to SceneComplete
### Limitations
While SceneComplete achieves strong results across diverse real-world scenes, it is built by composing and adapting many large pre-trained modules - making it susceptible to cascading errors. We highlight key limitations and areas for improvement:
- `Prompting & Segmentation`: The vision-language model (VLM) occasionally fails to detect all objects in the scene, resulting in missed reconstructions. Tuning prompts or re-prompting are some strategies that can help recover missing hypotheses. Grounded-SAM infrequently segemnts both an object and its subparts, or merge multiple cluttered objects into one. We apply IoU-based de-deduplication to mitigate such errors, but further improvements are possible especially with newer models. 
- `Inpainting`: We currently inpaint objects in isolation, removing the context from othe surrounding scene, and using other detected objects as the infilling mask. We also adapt the model on tabletop objects using LoRA, without losing the generalization. Better strategies for building the infilling mask and providing contextual cues may exist. 
- `Image-to-3D Reconstruction`: Though these models offer remarkable performance, they sometimes fail to generate plausible reconstructions especially when images are given in highly unusual viewpoints. 
- `Scaling`: Our scaling strategy assume isotropic scaling. In some cases, non-uniform scaling would better match object geometry and should be explored. Newer & more performant models could also be used for matching keypoint correspondences, for better scaling. 
- `Registration`: Pose registration may struggle with texture-less objects that lack discriminativefeatures. It also depends on accurate scaling to some extent — if the scale is off, registration quality may degrade. 

### Contributing to SceneComplete
SceneComplete is a modular framework, designed with flexibility in mind. Each model can be independently improved or replaced with newer and more efficient models, as long as the input/output interfaces are preserved. We `encourage contributions` from the community to help improve the system, and `welcome pull requests`! 

## 💖 Acknowledgements
We thank the authors of the following projects for their wonderful and open-source code:

- [InstantMesh](https://github.com/TencentARC/InstantMesh)
- [FoundationPose](https://github.com/NVlabs/FoundationPose)
- [BrushNet](https://github.com/TencentARC/BrushNet)
- [Grounded-Segment-Anything](https://github.com/IDEA-Research/Grounded-Segment-Anything)
- [dino-vit-features](https://github.com/ShirAmir/dino-vit-features)

## 📜 Cite Us
```
@ARTICLE{11235964,
  author={Agarwal, Aditya and Singh, Gaurav and Sen, Bipasha and Lozano-Pérez, Tomás and Kaelbling, Leslie Pack},
  journal={IEEE Robotics and Automation Letters}, 
  title={SceneComplete: Open-World 3D Scene Completion in Cluttered Real World Environments for Robot Manipulation}, 
  year={2026},
  volume={11},
  number={1},
  pages={482-489},
  keywords={Three-dimensional displays;Image reconstruction;Adaptation models;Pipelines;Image segmentation;Accuracy;Solid modeling;Predictive models;Reliability;Shape;Perception for grasping and manipulation;RGB-D perception;manipulation planning},
  doi={10.1109/LRA.2025.3630884}
}
```
