# Building fms-model-optimizer as an Image

The Dockerfile provides a way of running FMS Model Optimizer (FMS MO). It installs the dependencies needed and adds two additional scripts that helps to parse arguments to pass to FMS MO. The `accelerate_launch.py` script is run by default when running the image to trigger FMS MO for single or multi GPU by parsing arguments and running `accelerate launch fms_mo.run_quant.py`. 

## Configuration

The scripts accept a JSON formatted config which are set by environment variables. `FMS_MO_CONFIG_JSON_PATH` can be set to the mounted path of the JSON config. Alternatively, `FMS_MO_CONFIG_JSON_ENV_VAR` can be set to the encoded JSON config using the below function:

```py
import base64

def encode_json(my_json_string):
    base64_bytes = base64.b64encode(my_json_string.encode("ascii"))
    txt = base64_bytes.decode("ascii")
    return txt

with open("test_config.json") as f:
    contents = f.read()

encode_json(contents)
```

The keys for the JSON config are all of the flags available to use with [FMS Model Optimizer](fms_mo/training_args.py).

For configuring `accelerate launch`, use key `accelerate_launch_args` and pass the set of flags accepted by [accelerate launch](https://huggingface.co/docs/accelerate/package_reference/cli#accelerate-launch). Since these flags are passed via the JSON config, the key matches the long formed flag name. For example, to enable flag `--quiet`, use JSON key `"quiet"`, using the short formed `"q"` will fail.

For example, the below config is used for creating a GPTQ checkpoint of LLAMA-3-8B with two GPUs:

```json
{
    "accelerate_launch_args": {
        "main_process_port": 1234
    },
    "model_name_or_path": "meta-llama/Meta-Llama-3-8B",
    "training_data_path": "data_train",
    "quant_method": "gptq",
    "bits": 4,
    "group_size": 128,
    "output_dir": "/output/Meta-Llama-3-8B-GPTQ-MULTIGPU"
}
```

`num_processes` defaults to the amount of GPUs allocated for optimization, unless the user sets `SET_NUM_PROCESSES_TO_NUM_GPUS` to `False`. Note that `num_processes` which is the total number of processes to be launched in parallel, should match the number of GPUs to run on. The number of GPUs used can also be set by setting environment variable `CUDA_VISIBLE_DEVICES`. If ``num_processes=1`, the script will assume single-GPU.


## Building the Image

With docker, build the image at the top level with:

```sh
docker build . -t fms-model-optimizer:mytag -f build/Dockerfile
```

## Running the Image

Run fms-model-optimizer-image with the JSON env var and mounts set up.

```sh
docker run -v config.json:/app/config.json -v $MODEL_PATH:/models --env FMS_MO_CONFIG_JSON_PATH=/app/config.json fms-model-optimizer:mytag
```

This will run `accelerate_launch.py` with the JSON config passed.

An example Kubernetes Pod for deploying fms-model-optimizer which requires creating PVCs with the model and input dataset and any mounts needed for the outputted quantized model:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
name: fms-model-optimizer-config
data:
config.json: |
    {
        "accelerate_launch_args": {
            "main_process_port": 1234
        },
        "model_name_or_path": "meta-llama/Meta-Llama-3-8B",
        "training_data_path": "data_train",
        "quant_method": "gptq",
        "bits": 4,
        "group_size": 128,
        "output_dir": "/output/Meta-Llama-3-8B-GPTQ-MULTIGPU"
    }
---
apiVersion: v1
kind: Pod
metadata:
name: fms-model-optimizer-test
spec:
containers:
    env:
        - name: FMS_MO_CONFIG_JSON_PATH
        value: /config/config.json
    image: fms-model-optimizer:mytag
    imagePullPolicy: IfNotPresent
    name: fms-mo-test
    resources:
        limits:
            nvidia.com/gpu: "2"
            memory: 200Gi
            cpu: "10"
            ephemeral-storage: 2Ti
        requests:
            memory: 80Gi
            cpu: "5"
            ephemeral-storage: 1600Gi
    volumeMounts:
        - mountPath: /data/input
        name: input-data
        - mountPath: /data/output
        name: output-data
        - mountPath: /config
        name: fms-model-optimizer-config
restartPolicy: Never
terminationGracePeriodSeconds: 30
volumes:
    - name: input-data
    persistentVolumeClaim:
        claimName: input-pvc
    - name: output-data
    persistentVolumeClaim:
        claimName: output-pvc
    - name: fms-model-optimizer-config
    configMap:
        name: fms-model-optimizer-config
```

The above kube resource values are not hard-defined. However, they are useful when running some models (such as LLaMa-13b model). If ephemeral storage is not defined, you will likely hit into error `The node was low on resource: ephemeral-storage. Container was using 1498072868Ki, which exceeds its request of 0.` where the pod runs low on storage while tuning the model.

Note that additional accelerate launch arguments can be passed, however, FSDP defaults are set and no `accelerate_launch_args` need to be passed.

Another good example can be found [here](../examples/kfto-kueue-fms-model-optimizer.yaml) which launches a Kubernetes-native `PyTorchJob` using the [Kubeflow Training Operator](https://github.com/kubeflow/training-operator/) with [Kueue](https://github.com/kubernetes-sigs/kueue) for the queue management of optimization jobs. The KFTO example is running GPTQ of LLAMA-3-8B with two GPUs.
