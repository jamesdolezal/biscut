# BISCUIT <img src="https://drive.google.com/file/d/1PiAu0uXaOAL8Z8d7npEO4DSjmt75uY9X/view?usp=sharing" width="280px" align="right" />
Bayesian Inference of Slide-level Confidence via Uncertainty Index Thresholding

# Requirements
- Tensorflow 2.7 [see https://tensorflow.org for installation and requirements]
- Slideflow 1.1 [see https://slideflow.dev for installation and requirements]
- Whole-slide images for training and validation

# Use

## Data preparation
To reproduce results described in the manuscript, start by downloading whole-slide images (\*.svs files) from [The Cancer Genome Atlas (TCGA) data portal](https://portal.gdc.cancer.gov/), projects TCGA-LUAD and TCGA-LUSC.

Download whole-slide images from the [Clinical Proteomics Tumor Analysis Consortium (CPTAC)](https://proteomics.cancer.gov/data-portal) data portal, projects TCGA-LUAD and TSCA-LSCC.

Then, configure experimental projects with Slideflow using the `configure.py` script, passing the directories to your training slides (TCGA) and evaluation slides (CPTAC). This step will extract image tiles from whole-slide images.

```
python3 configure.py --train_slides=/path/to/TCGA --val_slides=/path/to/CPTAC
```

If using pathologist-annotated regions of interest (ROI) for the training dataset, these can be created as described in the [Slideflow documentation](https://slideflow.dev/pipeline.html). Pass the directory containing the ROIs in CSV format with the `--roi` argument.

## GAN Training
To train the class-conditional GAN (StyleGAN2) needed for latent space embedding interpolation, clone the [StyleGAN2-slideflow](https://github.com/jamesdolezal/stylegan2-slideflow) repository, which has been modified to interface with the `*.tfrecords` storage fromat Slideflow uses. The GAN is trained on image tiles 512 x 512 pixels and 400 x 400 microns. Synthetic images will be resized down to the target project size of 299 x 299 pixels and 302 x 302 microns during generation.

Use `train.py` **in the StyleGAN2 repository** to train the GAN. Pass the `gan_config.json` file that the `configure.py` script generated earlier to the `--slideflow` flag.

```
python3 train.py --outdir=/path/ --slideflow=/path/to/gan_cofig.json --mirror=1 --cond=1 --augpipe=bgcfnc --metrics=none
```

## Generating GAN images
To create intermediate images from a trained GAN model using latent space embedding interpolation, use the `generate_tfrecords.py` script **in the StyleGAN2-slideflow** repository. Flags that will be relevant include:

- `--network`: Path to network PKL file (saved GAN model)
- `--tiles`: Number of tiles per tfrecord to generate (manuscript uses 1000)
- `--tfrecords`: Number of tfrecords to generate
- `--embed`: Generate intermediate images with class embedding interpolation.
- `--name`: Name format for tfrecords.
- `--class`: Class index, if not using embedding interpolation.
- `--outdir`: Directory in which to save tfrecords.

For example, to create tfrecords containing synthetic images of class 0 (LUAD / adenocarcinoma):

```
python3 generate_tfrecords.py --network=/path/network.pkl --tiles=1000 --tfrecords=10 --name=gan_luad --class=0 --outdir=gan/
```

To create embedding-interpolated intermediate images:

```
python3 generate_tfrecords.py --network=/path/network.pkl --tiles=1000 --tfrecords=10 --name=gan --embed=1 --outdir=gan/
```

Subsequent steps will assume that the GAN tfrecords are in the folder `gan/`.


## Cross-validation & evaluation
Once projects have been configured, models can be trained with `train.py`. Experiments are divided into total dataset sizes and have an assigned label. For reference, the experimental labels for this project are:

| ID | n_slides |
|----|----------|
| AA | full     |
| U  | 800      |
| T  | 700      |
| S  | 600      |
| R  | 500      |
| A  | 400      |
| L  | 350      |
| M  | 300      |
| N  | 250      |
| D  | 200      |
| O  | 176      |
| P  | 150      |
| Q  | 126      |
| G  | 100      |
| V  | 90       |
| W  | 80       |
| X  | 70       |
| Y  | 60       |
| Z  | 50       |
| ZA | 40       |
| ZB | 30       |
| ZC | 20       |
| ZD | 10       |

For each experimental label, experiments happen in 6 steps:

1. Train cross-validation (CV) models for up to 10 epochs.
2. Train CV models at the optimal epoch (1).
3. Train UQ models in CV, saving predictions and uncertainty.
4. Train nested-UQ models, saving predictions, for uncertainty threshold determination.
5. Train models at the full dataset size without validation.
6. Perform external evaluation of fully-trained models.

Experiments are divided into three categories:

- `reg`: Regular experiments with balanced outcomes (LUAD:LUSC).
- `ratio`: Ratio testing experiments with class imbalance.
- `gan`: Cross-validation experiments varying degrees of GAN slides in training/validation sets.

Specify which category of experiment should be run by setting its flag to `True`. Specify the steps to run using the `--steps` flag. For example, to run steps 2-6 for the ratio experiments, do:

```
python3 train.py --steps=2-6 --ratio=1
```

## Viewing results

Once all models have finished training (the published experiment included results from over 3000 models, so this may take a while), results can be viewed with the `results.py` script. The same experimental category flags, `--reg`, `--ratio`, and `--gan`, are used to determine which results should be viewed. There are two additional categories of results that can be displayed:

- `heatmap`: Generate the heatmap shown in Figure 4.
- `umaps`: Generates UMAPs shown in Figure 5.

Figures and output will then be saved in the `results/` folder. For example:

```
python3 results.py --ratio=1 --umaps=1
```
