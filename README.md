# R-NET in Tensorflow

* This repository is a Tensorflow implementation of [R-NET](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/05/r-net.pdf), a neural network designed to solve the Question Answering (QA) task. 
* This implementation is specifically designed for [SQuAD](stanford-qa.com) , a large-scale dataset drawing attention in the field of QA recently.
* If you have any questions, contact b03902012@ntu.edu.tw.

## Updates and Acknowledgements

### 17.12.30
- As some have required recently, I have released a set of trained model weights. Details can be found in the Current Results section below.

### 17.12.12
- I'd like to thank _Fan Yang_ for pointing out several bugs when evaluating models. First, the model to be evaluated needs to be explicitly specified when executing the `evaluate.py` program. See the Usage section below. Also, I fixed some problems when loading characters.

### 17.11.10
- I'd like to thank _Elías Jónsson_ for pointing out that there's a problem in the mapping between characters and their indices. Previously, the indices for training and testing (dev set) were inconsistent. Actually, the mapping for testing shouldn't be constructed. During testing, if the machine sees a character it has not seen in the training set, it should mark it as OOV. So the table is now constructed using only the training set, and is used in both training and testing.
- As some are asking about how to turn the character embeddings off, one can now avoid using character embeddings by changing the hyperparameter in `Models/config.json`.
- I applied dropout to various components in the model, including all LSTM cells, passage & question encoding, question-passage matching, self-attention, and question representation. This led to improvement of about 3%.
- As I read the original paper more carefully, I found that the authors used Adadelta as optimizer, and 3 layers of bi-GRU were used to encode both passage and question. Changing from Adam to Adadelta led to roughly 1% improvement. In my experiments, after stacking layers, the epochs required for convergence increased, and I found that instead of stacking 3 layers, 2 layers led to better performances. Details are depicted in the current results section.


## Dependency
* Python 3.6
* Tensorflow-gpu 1.2.1
* Numpy 1.13.1
* NLTK

## Usage
1. First we need to download [SQuAD](stanford-qa.com) as well as the pre-trained [GloVe](nlp.stanford.edu/projects/glove/) word embeddings. This should take roughly 30 minutes, depending on network speed.
```
cd Data
sh download.sh
cd ..
```
2. Data preprocessing, including tokenizing and collection of pre-trained word embeddings, can take about 15 minutes.
Two kinds of files, `{data/shared}_{train/dev}.json`, will be generated and stored in `Data`.
    * shared: including the original and tokenized articles, GloVe word embeddings and character dictionaries.
    * data: including the ID, corresponding article id, tokenized question and the answer indices.
```
python preprocess.py --gen_seq True
```
3. Train R-NET by simply executing the following. The program will
    1. Read the training data, and then build the model. This should take around an hour, depending on hardware.
    2. Train for 12 epochs, by default.

    Hyper-arameters can be specified in `Models/config.json`. The training procedure, including the mean loss and mean EM score for each epoch, will be stored in `Results/rnet_training_result.txt`. Note that the score appear during training could be lower than the scores from the official evaluator. The models will be stored in `Models/save/`.
```
python rnet.py
```

4. The evaluation of the model on the dev set can be generated by executing the following. The result will be stored in `Results/rnet_prediction.txt`. Note that the score appear during evaluation could be lower than the scores from the official evaluator.
**Note:** The model to be evaluated has to be specified explictly. For example, if 12 epochs were trained (by default), then in `Models/save/` there should exist 5 saved models:
```
rnet_model8.ckpt.meta
rnet_model8.ckpt.data-00000-of-00001
rnet_model8.ckpt.index
...
rnet_model11.ckpt.meta
rnet_model11.ckpt.data-00000-of-00001
rnet_model11.ckpt.index
rnet_model_final.ckpt.meta
rnet_model_final.ckpt.data-00000-of-00001
rnet_model_final.ckpt.index
```
Here, `rnet_model11` and `rnet_model_final` are the same. Say, for example, one wish to evaluate on `rnet_model_final`, the following would to it:
```
python evaluate.py --model_path Models/save/rnet_model_final.ckpt
```

5. To get the final official score, you need to use the official evaluation script, which is in the `Results` directory.
```
python Results/evaluate-v1.1.py Data/dev-v1.1.json Results/rnet_prediction.txt
```

## Current Results


| Model | Dev EM Score | Dev F1 Score |
| -------- | -------- | -------- |
| Original Paper | 71.1 | 79.5 |
| My (Adadelta, 2 layer, dropouts, w/o char emb) | 62.6 | 71.5 |
| My (Adadelta, 1 layer, dropouts, w/o char emb) | 61.0 | 70.3 |
| My (Adam, 1 layer, dropouts, w/o char emb) | 60.8 | 70.5 |
| My (Adam, 1 layer, w/o char emb)| 57.8 | 67.9|
| My (Adam, 1 layer, w/ char emb) | 60.1 | 68.9 |

You can find the [current leaderboard](https://rajpurkar.github.io/SQuAD-explorer/) and compare with other models.

### Trained model weights
As some have required recently, a set of trained model weights can be downloaded [here](https://slam.iis.sinica.edu.tw/demo/RNet/release.zip). Unzip and you can find 3 files. Put the 3 files in `Models/save/` and evaluate on it by following the instruction above. This set of parameter was obtained by training for 28 epochs, using current settings, and achieved 62.2/71.5 on the dev set. I didn't save each set of model weights when I ran the experiments originally, so I reran the experiment, causing a slight degration compared with the best score on the table above. I want to clarify that the difference may come from random initialization, so feel free to train your own model weights.

## Discussion

### Reproduction

As shown above, I still fail to reproduce the results. I think there are some technical details that draw my concern:

1. Data Preprocessing. I have tried two preprocessing approaches, one of which is used in the implementation of [Match-LSTM](https://github.com/shuohangwang/SeqMatchSeq/blob/master/preprocess.py), and the other is used in the implementation of [Bi-DAF](https://github.com/allenai/bi-att-flow/blob/master/squad/prepro.py). While the latter approach includes lots of reasonable processing, I chose the former one empirically since it yields better performance.
2. As pointed out in another [implementation of R-NET in Keras](https://github.com/YerevaNN/R-NET-in-Keras), 
    > The first formula in (11) of the report contains a strange summand `W_v^Q V_r^Q`. Both tensors are trainable and are not used anywhere else in the network. We have replaced this product with a single trainable vector.

    However, instead of replacing the product with a single trainable vector, I followed the notation and still used two vectors.
4. Variable sharing. The notation in the original paper was very confusing to me. For example, `W_v^P` appeared in both equations (4) and (8). In my opinion, they should not be the same since they are multiplied by vectors of total different spaces. As a result, I treat them as different variables empirically.
5. Hyper-parameters ambiguity. Some hyper-paramters weren't specified in the original paper, including character embedding matrix dimension, truncating of articles and questions, and length of answer span during inference. I set up my own hyper-parameters empirically, mostly following the settings of [Match-LSTM](https://arxiv.org/pdf/1608.07905.pdf) and [Bi-DAF](https://arxiv.org/pdf/1611.01603.pdf).
6. Any other implementation mistakes and bugs.

### OOM

The full model could not be trained with NVIDIA Tesla K40m with 12GiB memory. Tensorflow will report serious OOM problem. There are a few possible solutions.

1. Run with CPU. This can be achieved by assigning a device mask with command line as follows. In fact, my implementation result shown in the previous section was generated by a model trained with CPU. However, this might cause extremely slow training speed. In my experience, it might cost roughly _24 hours per epoch_.
```
CUDA_VISIBLE_DEVICES="" python rnet.py
```
2. Reduce hyperparameters. Modifying these parameters might help:
    * `p_length`
    * Word embedding dimension: change from 300d GloVe vectors to 100d.
    
3. Don't use character embeddings. According to [Bi-DAF](https://arxiv.org/pdf/1611.01603.pdf), character embeddings don't help much. However, Bi-DAF uses 1D-CNNs to generate the character embeddings, while R-NET uses RNNs. As shown in the previous section, the performance dropped for 2%. Further investigation is needed for this part.
