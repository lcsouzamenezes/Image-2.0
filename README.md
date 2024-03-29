# Image 2.0

## Description

Image 2.0 is a project I've been working on for a while, with one simple purpose: Enhance & enlarge any photo with little to no input from the user, thereby reducing any confusion regarding different settings, sizes, and so on. It is powered by deep learning, specifically a U-Net trained with perceptual loss on the Common Objects in Context dataset (COCO). I have deployed a demo web app on to a [Colab notebook](https://colab.research.google.com/drive/1rRbQcBL7AUy74Rr4_CtXhikoAQ2xriaF) which takes in an image and gives back the enhanced/enlarged version with a mere few clicks.

More details regarding the whys & hows are covered in [the accompanying blog series](https://medium.datadriveninvestor.com/enhancing-photos-with-deep-learning-part-1-an-overview-80f2dcb96849) I wrote. You may find the trained fastai Learner [here](https://drive.google.com/file/d/1mZsspP11fWd2VYhRJn0JlHtro7S0Jvx3/view?usp=sharing) and the PyTorch model [here](https://drive.google.com/file/d/1SVxl-UjFZXDoZu2h0yZadkErOruEfiwl/view?usp=sharing).


## Training

All images in COCO were degraded in various ways to mimic real-life low-quality images that may have JPEG artifacts, blurry edges, etc. Then, fastai + PyTorch were used for fetching the data, implementing perceptual loss, creating the model, and lastly training it.

The steps are as follows:

1. Resize images: `python training/resize.py --path original/ --resized_path resized/ --size 256`
2. Degrade them: `python training/degrader.py --path resized/ --degraded_path degraded/`
3. Train: `python training/main.py --df fnames.csv --path resized/ --degraded_path degraded/ --bs 8 --model_name ecaresnet101d_pruned --frozen_epochs 10 --unfrozen_epochs 10`

`fnames.csv` must include the names of the images in column `fnames`, and a binary column `is_val` that says whether a specific file belongs to the validation set or not.


Most the bits and pieces may be used separately in your own projects. Some you might find useful are:

* `training/degrader.py`:

  `Degrader`: Stand-alone transform for degrading (downscale then upscale, add artifacts, blur, and adjust color, contrast, and brightness) an image. No arguments. Call it on your images, and it returns the degraded version.
  
* `training/loss.py`:

  `PerceptualLossVGG16`: Perceptual loss with VGG16. No arguments. Call it on your predictions & targets (both in the shape of (*bs, n_channels, w, h*)), and it returns the loss.
  
* `training/model.py`:

  `get_unet`: Returns an fp16 U-Net Learner with settings I've found to work well. Required argument: `dls` (fastai DataLoaders), Optional argument: `model_name` (name of model to be used as the U-Net's encoder. Needs to belong to the ResNet family. List of available models [here](https://github.com/rwightman/pytorch-image-models). Default is `swsl_resnet18`)
  

## Inference

For inference, there are a number of functions & methods to make life easier:

* `inference/model.py`

  `load_model`: Returns a model with a method `.predict` that takes in an image (required) and a device (optional, default CUDA), and returns the enhanced version (no need for normalizing, float to int tensor, etc.). 
  
  Optional argument: `path` (path to a pretrained model (like the PyTorch one linked in the description) to load. Default is `models/model.pt`).

* `inference/upgrade.py`

  `predict`: Returns an enhanced/enlarged version of an image with repeated predicting (96 X 96 --> 128 X 128 --> enhance --> 224 X 224 --> enhance --> .... More info in the blog series). 
  
  Required arguments: `model` (model outputted by `load_model`), `img` (image to enhance). 
  
  Optional arguments: `scale_factor` (upscale factor for enlargement. Default is 2), `enhancement_level` (number of iterations for repeated predicting. Default is 2), `enhance_first` (whether to enhance the image once before we start enlargement. Default is false), `device` (what device to run the prediction on. Default is CPU)

  `find_scale_factor`: Returns a recommendation for the scale factor, derived via a simple set of heuristics. 
  
  Required argument: `img` (image to find a scale factor for), 
  
  Optional argument: `enlarge` (if false, the function invariably returns one. Default is true)

  `upgrade`: Returns an enhance/enlarged version of an image with repeated predicting. It is just like `predict`, but it uses `find_scale_factor` to find a suitable scale factor (and thus no need to pass in a scale factor).

  `compare`: Returns an enhanced/enlarged version of an image, alongside what it would look like with no enhancement (useful for enlargement). The arguments are just like `upgrade`.


If you'd like to check out the web app, please run `inference/main.py`.
