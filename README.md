# Azure OpenAI benchmarking tool

> :warning: **Code in this repo is written for testing purposes and should not be used in production**

The Azure OpenAI Benchmarking tool is designed to aid customers in benchmarking their provisioned-throughput deployments. Provisioned throughput deployments provide a set amount of model compute. But determining the exact performance for you application is dependent on several variables such as: prompt size, generation size and call rate. 

The benchmarking tool provides a simple way to run test traffic on your deploymnet and validate the throughput for your traffic workloads. The script will output key performance statistics including the average and 95th percentile latencies and utilization of the deployment. 

You can use this tool to experiment with total throughput at 100% utilization across different traffic patterns for a ```Provisioned-Managed``` deployment type. These tests allow you to better optimize your solution design by adjusting the prompt size, generation size and PTUs deployed


## Setup

### Pre-requisites
1. An Azure OpenAI Service resource with a  model model deployed with a provisioned deployment (either ```Provisioned``` or ```Provisioned-Managed```) deplyment type. For more information, see the [resource deployment guide](https://learn.microsoft.com/azure/ai-services/openai/how-to/create-resource?pivots=web-portal).
2. Your resource endpoint and access key. The script assumes the key is stored in the following environment variable: ```OPENAI_API_KEY```. For more information on finding your endpoint and key, see the [Azure OpenAI Quickstart](https://learn.microsoft.com/azure/ai-services/openai/quickstart?tabs=command-line&pivots=programming-language-python#retrieve-key-and-endpoint).

### Building and running

In an existing python environment:
```
$ pip install -r requirements.txt
$ python -m benchmark.bench load --help
```

Build a docker container:
```
$ docker build -t azure-openai-benchmarking .
$ docker run azure-openai-benchmarking load --help
```
## General Guidelines

Consider the following guidelines when creating your benchmark tests

1. **Ensure call characteristics match your production expectations**. The number of calls per minute and total tokens you are able to process varies depending on the prompt size, generation size and call rate.
1. **Run your test long enough to reach a stable state**. Throttling is based on the total compute you have deployed and are utilizing. The utilization includes active calls. As a result you will see a higher call rate when ramping up on an unloaded deployment because there are no existing active calls being processed. Once your deplyoment is fully loaded with a utilzation near 100%, throttling will increase as calls can only be processed as earlier ones are completed. To ensure an accurate measure, set the duration long enough for the throughput to stabilize, especialy when running at or close to 100% utilization. Also note that once the test ends (either by termination, or reaching the maximum duration or number of requests), any pending requests will continue to drain, which can result in lower throughput values as the load on the endpoint gradually decreases to 0.
1. **Consider whether to use a retry strategy, and the effect of throttling on the resulting stats**. There are careful considerations when selecting a retry strategy, as the resulting latency statistics will be effected if the resource is pushed beyond it's capacity and to the point of throttling.
* When running a test with `retry=none`, any throttled request will be treated as throttled and a new request will be made to replace it, with the start time of the replacement request being reset to a newer time. If the resource being tested starts returning 429s, then any latency metrics from this tool will only represent the values of the final successful request, without also including the time that was spent retrying to resource until a successful response was received (which may not be representative of the real-world user experience). This setting should be used when the workload being tested results is within the resource's capacity and no throttling occurs, or where you are looking to understand what percentage of requests to a PTU instance might need to be diverted to a backup resource, such as during periods of peak load which require more throughput than the PTU resource can handle.
* When running a test with `retry=exponential`, any failed or throttled request will be retried with exponential backoff, up to a max of 60 seconds. While it is always recommended to deploy backup AOAI resources for use-cases that will experience periods of high load, this setting may be useful for trying to simulate a scenario where no backup resource is available, and where throttled or failed requests must still be fulfilled by the resource. In this case, the TTFT and e2e latency metrics will represent the time from the first throttled request to the time that the final request was successful, and may be more reflective of the total time that an end user could spend waiting for a response, e.g. in a chat application. Use this option in situations where you want to understand the latency of requests which are throttled and need to be retried on the same resource, and the how the total latency of a request is impacted by multiple request retries.
* As a practical example, if a PTU resource is tested beyond 100% capacity and starts returning 429s:
    * With `retry=none` the TTFT and e2e latency statistics will remain stable (and very low), since only the successful requests will be included in the metrics. Number of throttled requests will be relatively high.
    * With `retry=exponential`, the TTFT/e2e latency metrics will increase (potentially up to the max of 60 seconds), while the number of throttled requests will remain lower (since a request is only treated as throttled after 60 seconds, regardless of how many attempts were made within the retry period).
    * Total throughput values (RPM, TPM) may be lower when `retry=none` if rate limiting is applied.
* As a best practice, any PTU resource should be deployed with a backup PayGO resource for times of peak load. As a result, any testing should be conducted with the values suggested in the AOAI capacity calculator (within the AI Azure Portal) to ensure that throttling does not occur during testing.


## Usage examples

### Common Scenarios:
The table below provides an example prompt & generation size we have seen with some customers. Actual sizes will vary significantly based on your overall architecture For example,the amount of data grounding you pull into the prompt as part of a chat session can increase the prompt size significantly.

| Scenario | Prompt Size | Completion Size | Calls per minute | Provisioned throughput units (PTU) required |
| -- | -- | -- | -- | -- |
| Chat | 1000 | 200 | 45 | 200 |
| Summarization | 7000 | 150 | 7 | 100 |
| Classification | 7000 | 1 | 24 | 300|

Or see the [pre-configured shape-profiles below](#shape-profiles).

### Run samples 

During a run, statistics are output every second to `stdout` while logs are output to `stderr`. Some metrics may not show up immediately due to lack of data. 

**Run load test at 60 RPM with exponential retry back-off**

```
$ python -m benchmark.bench load \
    --deployment gpt-4 \
    --rate 60 \
    --retry exponential \
    https://myaccount.openai.azure.com

2023-10-19 18:21:06 INFO     using shape profile balanced: context tokens: 500, max tokens: 500
2023-10-19 18:21:06 INFO     warming up prompt cache
2023-10-19 18:21:06 INFO     starting load...
2023-10-19 18:21:06 rpm: 1.0   requests: 1     failures: 0    throttled: 0    ctx tpm: 501.0  gen tpm: 103.0  ttft avg: 0.736  ttft 95th: n/a    tbt avg: 0.088  tbt 95th: n/a    e2e avg: 1.845  e2e 95th: n/a    util avg: 0.0%   util 95th: n/a   
2023-10-19 18:21:07 rpm: 5.0   requests: 5     failures: 0    throttled: 0    ctx tpm: 2505.0 gen tpm: 515.0  ttft avg: 0.937  ttft 95th: 1.321  tbt avg: 0.042  tbt 95th: 0.043  e2e avg: 1.223 e2e 95th: 1.658 util avg: 0.8%   util 95th: 1.6%  
2023-10-19 18:21:08 rpm: 8.0   requests: 8     failures: 0    throttled: 0    ctx tpm: 4008.0 gen tpm: 824.0  ttft avg: 0.913  ttft 95th: 1.304  tbt avg: 0.042  tbt 95th: 0.043  e2e avg: 1.241 e2e 95th: 1.663 util avg: 1.3%   util 95th: 2.6% 
```

**Load test with custom messages being loaded from file and used in all requests**

```
$ python -m benchmark.bench load \
    --deployment gpt-4 \
    --rate 1 \
    --context-generation-method replay
    --replay-path replay_messages.json
    --max-tokens 500 \
    https://myaccount.openai.azure.com
```

**Load test with custom request shape**

```
$ python -m benchmark.bench load \
    --deployment gpt-4 \
    --rate 1 \
    --shape custom \
    --context-tokens 1000 \
    --max-tokens 500 \
    https://myaccount.openai.azure.com
```

**Obtain number of tokens for input context**

`tokenize` subcommand can be used to count number of tokens for a given input.
It supports both text and json chat messages input.

```
$ python -m benchmark.bench tokenize \
    --model gpt-4 \
    "this is my context"
tokens: 4
```

Alternatively you can send your text via stdin:
```
$ cat mychatcontext.json | python -m benchmark.bench tokenize \
    --model gpt-4
tokens: 65
```

## Contibuted modules
**Extract and Combine JSON logs to CSV**

The `combine_logs` CLI can be used to load and combine the logs from multiple runs into a single CSV, ready for comparison and analysis. This tool extracts the run arguments as well as the final set of stats prior to the run ending (either by termination or hitting the request/duration limit).
```
$ python -m benchmark.contrib.combine_logs logs/ combined_logs.csv --load-recursive
```

**Run Batches of Multiple Configurations**

The `batch_runner` CLI can be used to run batches of benchmark runs back-to-back. Currently, this CLI only works for runs where `context-generation-method = generation`. The CLI also includes a `--start-ptum-runs-at-full-utilization` argument (default=`true`), which will warm up any PTU-M model endpoints to 100% utilization prior to testing, which is critical for ensuring that test results reflect accurate real-world performance. 

To use the CLI, create a list of token profile and rate combinations to be used, and then select the number of batches and interval to be used between each batch. When using the batch runner, make sure to execute the command from the root directory of the repo.

Example - Run a single batch of the following two configuration for 120 seconds each, making sure to warm up the PTU-M endpoint prior to each run:
- context_tokens=500,  max_tokens=100, rate=20
- context_tokens=3500, max_tokens=300, rate=7.5

```
$ python -m benchmark.contrib.batch_runner https://gbb-ea-openai-swedencentral-01.openai.azure.com/ --deployment gpt-4-1106-ptu --token-rate-workload-list 500-100-20,3500-300-7.5 --duration 130 --aggregation-window 120 --log-save-dir logs/ --start-ptum-runs-at-full-utilization true
```

Example - Run the same batch as above, but 5x times and with a 1 hour delay between the start of each batch:

```
$ python -m benchmark.contrib.batch_runner https://gbb-ea-openai-swedencentral-01.openai.azure.com/ --deployment gpt-4-1106-ptu --token-rate-workload-list 500-100-20,3500-300-7.5 --duration 130 --aggregation-window 120 --log-save-dir logs/ --start-ptum-runs-at-full-utilization true --num-batches 5 --batch-start-interval 3600
```

## Configuration Option Details
### Context Generation Method
Using the `--context-generation-method` argument, this tool gives two options for how the source content of each request is generated:

**1: `generate`** [default]: Context information is generated automatically from a list of all english words, and the endpoint is instructed to generate a long story of `max_tokens` words. This is useful where existing data is not yet available, and should reslt in similar performance as real-world workoads with the same number of context & completion tokens.

In this mode, there are four different shape profiles via command line option `--shape-profile`:
|profile|description|context tokens|max tokens|
|-|-|-|-|
|`balanced`|[default] Balanced count of context and generation tokens. Should be representative of typical workloads.|500|500|
|`context`|Represents workloads with larger context sizes compared to generation. For example, chat assistants.|2000|200|
|`generation`|Represents workloads with larger generation and smaller contexts. For example, question answering.|500|1000|
|`custom`|Allows specifying custom values for context size (`--context-tokens`) and max generation tokens (`--max-tokens`).|||  

Note: With the default prompting strategy, OpenAI models will typically return completions of a max of 700-1200 tokens. If setting `max_tokens` above 750, be aware that the results for `rpm` may be higher, and `e2e` latency lower, than if the model was returning completions of size `max_tokens` in every response. Refer to the `gen_tpr` stats at the end of each run to see how many tokens were generated across responses.

**2: `replay`**: Messages are loaded from a JSON file and replayed back to the endpoint. This is useful for scenarios where testing with real-world data is important, and that data has already been generated or collected from an existing LLM application. 

In this mode, all messages in the file are sampled randomly when making requests to the endpoint. This means the same message may be used multiple times in a benchmarking run, plus any anti-caching prefix if `prevent-server-caching=true`. The format of the JSON file should be a single array containing separate lists of messages which conform to the [OpenAI chat completions API schema](https://platform.openai.com/docs/api-reference/chat/create), like so:

```
[
    [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Can you explain how photosynthesis works?"}
    ],
    [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is the capital of France?"},
      {"role": "assistant", "content": "The capital of France is Paris."},
      {"role": "user", "content": "Please tell me about the history of Paris."}
    ],
]
```

In addition, when `--prevent-server-caching=true`, every message in each request payload is prefixed with a random string to force the inference endpoint to process each request without any optimization/caching that might occur if workloads are the same. This ensures that the results observed while running the tool are the worst case scenario for given traffic shape. For example:

|initial request|request with random prefixes|
|-|-|
|{"role": "user", "content": "Can you explain how photosynthesis works?"}|{"role": "user", "content": "1704441942.868042 Can you explain how photosynthesis works?"}|
||{"role": "user", "content": "1704441963.715898 Can you explain how photosynthesis works?"}|

Setting `--prevent-server-caching=false` is only recommended when a sufficiently large replay dataset is available (e.g. at least double the number of messages than the total number of requests to be made across all test runs in a session). If the cache needs to be cleared/reset for additional runs, it is recommended that the PTU model deployment should be deleted and recreated in order to reload the model with an empty cache.

### Output fields

|field|description|sliding window|example|
|-|-|-|-|
|`time`|Time offset in seconds since the start of the test.|no|`120`|
|`rpm`|Successful Requests Per Minute. Note that it may be less than `--rate` as it counts completed requests.|yes|`12`|
|`processing`|Total number of requests currently being processed by the endpoint.|no|`100`|
|`completed`|Total number of completed requests.|no|`100`|
|`failures`|Total number of failed requests out of `requests`.|no|`100`|
|`throttled`|Total number of throttled requests out of `requests`.|no|`100`|
|`requests`|Deprecated in favor of `completed` field (output values of both fields are the same)|no|`1233`|
|`ctx_tpm`|Number of context Tokens Per Minute.|yes|`1200`|
|`gen_tpm`|Number of generated Tokens Per Minute.|yes|`156`|
|`ttft_avg`|Average time in seconds from the beginning of the request until the first token was received.|yes|`0.122`|
|`ttft_95th`|95th percentile of time in seconds from the beginning of the request until the first token was received.|yes|`0.130`|
|`tbt_avg`|Average time in seconds between two consequitive generated tokens.|yes|`0.018`|
|`tbt_95th`|95th percentail of time in seconds between two consequitive generated tokens.|yes|`0.021`|
|`gen_tpr_10th`|10th percentile of number of generated tokens per model response.|yes|`389`|
|`gen_tpr_avg`|Average number of generated tokens per model response.|yes|`509`|
|`gen_tpr_90th`|90th percentile of number of generated tokens per model response.|yes|`626`|
|`e2e_avg`|Average end to end request time.|yes|`1.2`|
|`e2e_95th`|95th percentile of end to end request time.|yes|`1.5`|
|`util_avg`|Average deployment utilization percentage as reported by the service.|yes|`89.3%`|
|`util_95th`|95th percentile of deployment utilization percentage as reported by the service.|yes|`91.2%`|

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
