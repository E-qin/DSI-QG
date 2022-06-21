# DSI-QG
The official repository for "Bridging the Gap Between Indexing and Retrieval for Differentiable Search Index with Query Generation",
Shengyao Zhuang, Houxing Ren, Linjun Shou, Jian Pei, Ming Gong, Guido Zuccon and Daxin Jiang.


## Installation

`pip install -r requirements.txt`
> Note: The current code base has been tested with a GPU cluster with 8 Tesla V100 gpus.
> We also use [wandb](https://wandb.ai/site) cloud-based logger which you may need to register and login first.



## Data Preparing
Simply run `bash get_data.sh`. 

The script will automatically download and create training and evaluation datasets for XORQA 100k and MS MARCO 100k dataset in `\data` folder.

## Training
We take XORQA 100k and t5/mt5-base as a training example. You can simply change the dataset and model config in the run arguments to run other experiments.

The retrieval Hits score results on dev set will be logger on wandb logger during training.

### Train Original DSI model

Recall that the original DSI model directly take document text as input during indexing.

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task "DSI" \
        --model_name "google/mt5-base" \
        --run_name "XORQA-100k-mt5-base-DSI" \
        --max_length 256 \
        --train_file data/xorqa_data/100k/xorqa_DSI_train_data.json \
        --valid_file data/xorqa_data/100k/xorqa_DSI_dev_data.json \
        --output_dir "models/XORQA-100k-mt5-base-DSI" \
        --learning_rate 0.0005 \
        --warmup_steps 100000 \
        --per_device_train_batch_size 16 \
        --per_device_eval_batch_size 8 \
        --evaluation_strategy steps \
        --eval_steps 1000 \
        --max_steps 1000000 \
        --save_strategy steps \
        --dataloader_num_workers 10 \
        --save_steps 1000 \
        --save_total_limit 2 \
        --load_best_model_at_end \
        --gradient_accumulation_steps 1 \
        --report_to wandb \
        --logging_steps 100 \
        --dataloader_drop_last False \
        --metric_for_best_model Hits@10 \
        --greater_is_better True

```


### Train DSI-QG model
#### Step 1:
Our DSI-QG model requires a query generation model to generate potentially-relevant queries to
represent each candidate documents.

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task "docTquery" \
        --model_name "google/mt5-large" \
        --run_name "docTquery-XORQA" \
        --max_length 128 \
        --train_file data/xorqa_data/100k/xorqa_docTquery_train_data.json \
        --valid_file data/xorqa_data/100k/xorqa_docTquery_dev_data.json \
        --output_dir "models/xorqa_docTquery_mt5_large" \
        --learning_rate 0.0001 \
        --warmup_steps 0 \
        --per_device_train_batch_size 4 \
        --per_device_eval_batch_size 4 \
        --evaluation_strategy steps \
        --eval_steps 100 \
        --max_steps 2000 \
        --save_strategy steps \
        --dataloader_num_workers 10 \
        --save_steps 100 \
        --save_total_limit 2 \
        --load_best_model_at_end \
        --gradient_accumulation_steps 4 \
        --report_to wandb \
        --logging_steps 100 \
        --dataloader_drop_last False

```

#### Step 2:
We then run the query generation for all the documents in the corpus: 
> Note: set the `--model_path` to the best checkpoints.

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task generation \
        --model_name google/mt5-large \
        --model_path models/xorqa_docTquery_mt5_large/checkpoint-xxx \
        --per_device_eval_batch_size 32 \
        --run_name docTquery-XORQA-generation \
        --max_length 256 \
        --valid_file data/xorqa_data/100k/xorqa_corpus.tsv \
        --output_dir temp \
        --dataloader_num_workers 10 \
        --report_to wandb \
        --logging_steps 100 \
        --num_return_sequences 1
```

#### Step 3:

Finally, we can train DSI-QG with query-represented corpus:

```
python3 -m torch.distributed.launch --nproc_per_node=8 run.py \
        --task "DSI" \
        --model_name "google/mt5-base" \
        --run_name "XORQA-100k-mt5-base-DSI-QG" \
        --max_length 32 \
        --train_file data/xorqa_data/100k/xorqa_corpus.tsv.docTquery \
        --valid_file data/xorqa_data/100k/xorqa_DSI_dev_data.json \
        --output_dir "models/XORQA-100k-mt5-base-DSI-QG" \
        --learning_rate 0.0005 \
        --warmup_steps 100000 \
        --per_device_train_batch_size 32 \
        --per_device_eval_batch_size 32 \
        --evaluation_strategy steps \
        --eval_steps 1000 \
        --max_steps 1000000 \
        --save_strategy steps \
        --dataloader_num_workers 10 \
        --save_steps 1000 \
        --save_total_limit 2 \
        --load_best_model_at_end \
        --gradient_accumulation_steps 1 \
        --report_to wandb \
        --logging_steps 100 \
        --dataloader_drop_last False \
        --metric_for_best_model Hits@10 \
        --greater_is_better True \
        --remove_prompt True
```