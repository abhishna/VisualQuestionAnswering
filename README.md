# Free Form and Open Ended Visual Question Answering

## Project Description

VQA is a multi-disciplinary problem involving various domains such as Computer Vision, Natural Language Processing, Knowledge Representation and Reasoning, etc., The input consists of an image and a free form or open ended natural language question, the task is to provide an accurate natural language answer. Since the question may selectively target any minute details in the complex scene portrayed by the input image, there is a need for a neural architecture that can effectively learn a joint embedding representation in the multi-modal space combining the text and image representations using some form of attention.

Due to the complexity involved in building such a neural architecture, there are several design decisions and engineering choices which need to be factored in, to justify why a certain architecture works better than others. Motivated by this challenge, our aim in this work is to mine this tremendous potential and explore the model architecture and hyper-parameter spaces to not only attempt and improve the model performance but also be able to justify our design choices, instead of treating the model like a black box and getting some empirical results from experimentation.

In our experiments, we model our approach inspired by [1, 3, 4] as shown below, but our motive is to strike the right tradeoff between model complexity / performance and accuracy. Our aim is to show that even a simpler architecture with the right design choices can help achieve accuracies reasonably close to the state-of-the-art, while being computationally efficient.
![description](images/description.png)

We work with the VQA baseline model[1] that uses a VGGNet to encode images and a two layer LSTM to encode questions, and transforming them to a common space via an element-wise multiplication, which is then through a fully connected layer followed by a softmax layer to obtain a distribution over answers. We explore various settings on this simple architecture and work on improving the VQA accuracy.
![baseline](images/baseline.png)

## Repo Structure
    .
    ├── dataset.py            # dataset for the preprocessed data
    ├── download_datasets.sh  # script to download VQA2.0 data and extract them
    ├── images                # iamge files used in readme
    ├── main.py               # entry point for training, takes various arguments
    ├── models
    │	 └── baseline.py      # baseline model from VQA paper
    ├── preprocess.py         # preprocess the VQA2.0 data and save vocabulary files
    ├── README.md             # readme file
    ├── train.py              # functions to train the model
    ├── utils.py              # utility functions
    └── vectorize_images.py   # save image embeddings to pickle file
    └──generate_glove_embeddings.py   # save word embeddings to pickle file
    └──grid_search.py   # perform a grid search for hyper-parameter tuning to optimize the given set of params

## Usage

#### 0. Install Dependencies
We recommend to use conda environment to install the pre-requisites to run the code in this repo. We can install the packages required from `requirements.txt` file using the command
```bash
conda create --name myenv --file requirements.txt
```

Following are the important packages requried

- cudatoolkit=11.3.1
- matplotlib-base=3.4.3
- numpy=1.23.3
- pandas=1.4.4
- pickleshare=0.7.5
- python=3.8.13
- pytorch=1.12.1
- tensorboard=2.11.0
- tensorboardx=2.5.1
- torchvision=0.13.1

#### 1. Download Datasets
For training and evaluating the models, we choose the more recent version of VQA dataset - **VQA v2** on the balanced real images. This dataset consists of 82,783 training images with 443,757 questions, 40,504 validation images with 214,354 questions. Each of these questions have 10 answers which were collected by human subjects. The test dataset has 81,434 images with 447,793 questions. We are evaluating the performance of our models on the validation set.

Run the `download_datasets.sh` file to download the datasets from the official site and extract them. We need to set the appropriate directory where we want the data to be downloaded in the script by modifying `DATASETS_DIR` variable (`data_dir` flag going forward will need to point to this directory).
```bash
chmod +x download_datasets.sh
./download_datasets.sh
```
The data directory structure after running the above command should look like this:

    .
    ├── annotations                                         # Annotation files
    │    └── v2_mscoco_train2014_annotations.json           # Train annotations
    │    └── v2_mscoco_val2014_annotations.json             # Validation annotations
    ├── images                                              # Actual images
    │    └── train2014                                      # 82,783 training images
    │    └── val2014                                        # 40,504 validation images
    │    └── test2015                                       # 81,434 testing images
    └── questions                                           # Openended questions
         └── v2_OpenEnded_mscoco_train2014_questions.json   # Training questions
         └── v2_OpenEnded_mscoco_val2014_questions.json     # Validation questions
         └── v2_OpenEnded_mscoco_test2015_questions.json
         └── v2_OpenEnded_mscoco_test-dev2015_questions.json

#### 2. Preprocess Data
We first preprocess the data we have into a simple format to train the model easily. Run the following command by passing the `data_dir` argument with the directory where we downloaded the dataset to.
```
python preprocess.py --data_dir ../Dataset
```
This script processes all the questions, annotations and saves each question example as a row in `image_id`\t`question`\t`answer`\t`answers` format in the processed `train_data.txt` and `val_data.txt` files. `image_id` is the unique id for each image in the respective train and val sets. The question and answer are space separated, answers is ^ separated for convenience. The answers are the 10 possible answers, and answer is the most frequent among them. This also saves the vocabulary of words in training questions mapping word to index and also index to the word in `questions_vocab.pkl` file, and also the frequencies of answers (that will be used later to construct the vocabulary for answers) in `answers_freq.pkl` file.

#### 3. Pre Compute Image Embeddings
```
python vectorize_images.py --data_dir ../Dataset --model_type vgg16
```

#### 4. Pre Compute Word Embeddings (if using GloVe, else optional)
```
python generate_word_embeddings.py --data_dir ../Dataset 
```

#### 5. Train a model
We can run a training experiment using `main.py` script, which has various arguments required by the code. For information about each flag and its usage, we can run `python main.py -h`, which gives the following description:
```
usage: main.py [-h] [--data_dir DATA_DIR] [--model_dir MODEL_DIR] [--log_dir LOG_DIR] --run_name RUN_NAME --model {baseline} [--image_model_type {vgg16,resnet152}] [--use_image_embedding USE_IMAGE_EMBEDDING] [--top_k_answers TOP_K_ANSWERS] [--max_length MAX_LENGTH] [--word_embedding_size WORD_EMBEDDING_SIZE] [--lstm_state_size LSTM_STATE_SIZE] [--batch_size BATCH_SIZE] [--epochs EPOCHS]
               [--learning_rate LEARNING_RATE] [--optimizer {adam,adadelta}] [--use_dropout USE_DROPOUT] [--use_sigmoid USE_SIGMOID] [--use_sftmx_multiple_ans USE_SFTMX_MULTIPLE_ANS] [--ignore_unknowns IGNORE_UNKNOWNS] [--use_softscore USE_SOFTSCORE] [--print_stats PRINT_STATS] [--print_epoch_freq PRINT_EPOCH_FREQ] [--print_step_freq PRINT_STEP_FREQ] [--save_best_state SAVE_BEST_STATE]
               [--attention_mechanism {element_wise_product,sum,concat}] [--random_seed RANDOM_SEED] [--bi_directional {True, False}] [--use_lstm {True, False}] [--use_glove {True, False}] [--embedding_file_name {PATH_TO_GLOVE_PKL_FILE}]

VQA

options:
  -h, --help            show this help message and exit
  --data_dir DATA_DIR   directory of the preprocesses data
  --model_dir MODEL_DIR
                        directory to store model checkpoints (saved as run_name.pth)
  --log_dir LOG_DIR     directory to store log files (used to generate run_name.csv files for training results)
  --run_name RUN_NAME   unique experiment name (used as prefix for all data saved on a run)
  --model {baseline}    VQA model choice
  --image_model_type {vgg16,resnet152}
                        Type of CNN for the Image Encoder
  --use_image_embedding USE_IMAGE_EMBEDDING
                        Use precomputed embeddings directly
  --top_k_answers TOP_K_ANSWERS
                        Top K answers used to train the model (output classifier size)
  --max_length MAX_LENGTH
                        max sequence length of questions
  --word_embedding_size WORD_EMBEDDING_SIZE
                        Word embedding size for the embedding layer
  --lstm_state_size LSTM_STATE_SIZE
                        LSTM hidden state size
  --batch_size BATCH_SIZE
                        batch size
  --epochs EPOCHS       number of epochs i.e., final epoch number
  --learning_rate LEARNING_RATE
                        initial learning rate
  --optimizer {adam,adadelta}
                        choice of optimizer
  --use_dropout USE_DROPOUT
                        use dropout
  --use_sigmoid USE_SIGMOID
                        use sigmoid activation to compute binary cross entropy loss
  --use_sftmx_multiple_ans USE_SFTMX_MULTIPLE_ANS
                        use softmax activation with multiple possible answers to compute the loss
  --ignore_unknowns IGNORE_UNKNOWNS
                        Ignore unknowns from the true labels in case of use_sigmoid or use_sftmx_multiple_ans
  --use_softscore USE_SOFTSCORE
                        use soft score for the answers, only applicable for sigmoid or softmax with multiple answers case
  --print_stats PRINT_STATS
                        flag to print statistics i.e., the verbose flag
  --print_epoch_freq PRINT_EPOCH_FREQ
                        epoch frequency to print stats at
  --print_step_freq PRINT_STEP_FREQ
                        step frequency to print stats at
  --save_best_state SAVE_BEST_STATE
                        flag to save best model, used to resume training from the epoch of the best state
  --attention_mechanism {element_wise_product,sum,concat}
                        method of combining image and text embeddings
  --bi_directional {True,False}
                        True if lstm is to be bi-directional
  --use_lstm {True,False}
                        True if lstm is to be used
  --use_glove {True,False}
                        True if glove embeddings are to be used
  --embedding_file_name EMBEDDING_FILE_NAME
                        glove embedding path file
  --random_seed RANDOM_SEED
                        random seed for the experiment

```
An example command to run the VQA baseline model - `python main.py --data_dir ../Dataset --model_dir ../checkpoints --log_dir ../logs --run_name run_43 --model baseline --use_image_embedding True --top_k_answers 3000 --batch_size 384 --epochs 100 --optimizer adadelta --use_dropout True --use_sigmoid True --save_best_state True --random_seed 43`

#### 6. Visualizing Training Results
Training statistics for an experiment are all saved using the run_name passed for it. Log files are save as tensorboard events in the log directory passed during training, and the parsed csv files of these logs are saved in the same directory. `utils.py` has multiple functions that can help visualize these csv files.

To view the VQA accuracies for multiple runs together we can use `python utils.py 'from utils import *; plot_vqa_accuracies(log_dir, ["run_13, run_23, run_43"])'` with the appropriate log directory.

#### 7. Predicting Answers
To predict answers for an image in the dataset, we can use the script `answer_questions.py` by passing the arguments that were used during training of that experiment. `python answer_questions.py --data_dir ../Dataset --model_dir ../checkpoints --run_name run_43 --top_k_answers 3000 --use_dropout True --image_loc val --image_id 264957`. In case of testing on a custom image and questions, we can use the function `answer_these_questions()` in `utils.py` that takes in the image path and a list of questions along with the other parameters that were used for the experiment during training.

#### Hyper-parameter tuning via grid-search with Optuna
To tune hyper-parameters of the model, we should first specify the parameters we wish to optimize and the list of choices for each param in the objective function in grid_search.py file. By default it will try to run trial runs with different combination of params, and prune the ones which are not learning well. The default objective is to find the trial which maximizes accuracy. However this can be changed to something like minimize training or val loss, etc as needed by tweaking the call to optuna.create_study(). The usage of this file is as follows :

` python grid_search.py --run_name testrun --model baseline --data_dir ../Dataset --model_dir ../checkpoints --log_dir ../logs --epochs 1 `

## Results

Given below are the VQA accuracy values we observed from various experiments through a combination of hyper parameters.
![results table](images/results.png)

Following are the observations from our results - 

- The model is prone to over fitting on this dataset, so using dropout is crucial.
- We didn't find the choise of embedding sizes and LSTM configurations to be a major factor as the model is able to effectively capture the semantics even in a smaller dimension vector.
- Increasing the output classifier size brings about an improvement as there are more answer choices.
- Treating VQA as a multi-label classification problem improved the accuracy as a single wrong answer would not penalize the model much anymore.
- Element wise sum of image and text vector for combining brings about a small improvement.
- Using pre-trained GloVe and fine-tuning it, gave an accuracy reasonably close to LSTM.
- Bi-directional LSTM improves accuracy. This shows that the context of words matters in both directions in the question.
- Resnet152 gave a much poor accuracy compared to VGG16. Though Resnets have been shown to be better than VGG on ImageNet, in case of transfer learning, it depends how well the model is able to fine tune and leverage the transferable information to train on the target dataset.

Here are some sample images and the top answers predicted using the best performing model -
<p align="center" float="left">
  <img src="./images/image1.png" />
  <img src="./images/image2.png" />
</p>

## References
1. [VQA: Visual Question Answering.](https://openaccess.thecvf.com/content_iccv_2015/papers/Antol_VQA_Visual_Question_ICCV_2015_paper.pdf)
2. [Making the V in VQA Matter: Elevating the Role of Image Understanding in Visual Question Answering.](https://openaccess.thecvf.com/content_cvpr_2017/papers/Goyal_Making_the_v_CVPR_2017_paper.pdf)
3. [Show, Ask, Attend, and Answer: A Strong Baseline For Visual Question Answering.](https://arxiv.org/pdf/1704.03162.pdf)
4. [Tips and Tricks for Visual Question Answering: Learnings from the 2017 Challenge.](https://openaccess.thecvf.com/content_cvpr_2018/papers/Teney_Tips_and_Tricks_CVPR_2018_paper.pdf)
