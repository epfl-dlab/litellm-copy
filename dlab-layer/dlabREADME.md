# How to run your own server ?

## Option 1: Easy option, Use the existing conda environment (respects system prompt formats)

1. Activate the dlab's litellm conda environment:

    ```
    conda activate /dlabdata1/baldwin/miniconda3/envs/litellm_dlab_version
    ```

2. Deploy the server (use this [config](config.yaml) as your config):

    ```
    export REPLICATE_API_KEY=<YOUR_REPLICATE_API_KEY>; litellm --config <PATH_TO_YOUR_CONFIG_FILE> --port <PORT>
    ```

NOTE: If you want to add a new replicate model in your config and it supports the `system_prompt` argument, make sure to add `supports_system_prompt: true` to your model's config (see [here](config.yaml) for an example). If it doesn't support the `system_prompt` argument then you can either set it `supports_system_prompt: false` or exclude it from the model's config.

## Option 2: Do it all yourself / You want to modify other code of litellm's library yourself

1. Pull this dlab's forked version of litellm:

     ```
     git clone https://github.com/epfl-dlab/litellm-copy.git
     cd litellm-copy
     ```
2. Create a new conda environment:

    ```
    conda create --name litellm_dlab_version python=3.10
    conda activate litellm_dlab_version
    ```

3. Make the changes in litellm's code so that it's suitable for your application (if needed). Note that if you're adding libraries, you need to add them to `pyproject.toml` in the `[tool.poetry.dependencies section]`.

4. Install your updated litellm library to your conda environment:

    ```
    pip install -e .
    ```

5. Create a config file with the necessary configurations to launch your server. In my case, I added this following code snippet in [litellm/llms/replicate.py](../litellm/llms/replicate.py) in the `completion` function to support system prompts for models that treat system prompts seperately from other prompts using the `supports_system_prompts` argument:
    ```python
    for k, v in config.items(): 
        if k not in optional_params: # completion(top_k=3) > replicate_config(top_k=3) <- allows for dynamic variables to be passed in
            optional_params[k] = v
            
    system_prompt = None
    if optional_params is not None and "supports_system_prompt" in optional_params:
        supports_sys_prompt = optional_params.pop("supports_system_prompt")
    else:
        supports_sys_prompt = False
        
    if supports_sys_prompt:
        for i in range(len(messages)):
            if messages[i]["role"] == "system":
                first_sys_message = messages.pop(i)
                system_prompt = first_sys_message["content"]
                break
    ``` 
    So my [config](config.yaml) file looks like this:
    ```yaml
    litellm_settings:
        drop_params: True

    model_list:
    - model_name: gpt-3.5-turbo # model alias
        litellm_params: # actual params for litellm.completion()
        model: "replicate/meta/llama-2-70b-chat:02e509c789964a7ea8736978a43525956ef40397be9033abf9fd2badfe68c9e3" 
        initial_prompt_value: "\n"
        roles: {"system":{"pre_message":"[INST] <<SYS>>\n","post_message":"\n<</SYS>>\n\n"}, "assistant":{"pre_message":"\n","post_message":"</s>\n"}, "user":{"pre_message":"[INST]","post_message":"[/INST]"}}
        final_prompt_value: "\n"
        max_tokens: 4096
        supports_system_prompt: true
    ```

6. Deploy your server
    ```
    export REPLICATE_API_KEY=<YOUR_REPLICATE_API_KEY>; litellm --config <PATH_TO_YOUR_CONFIG_FILE> --port <PORT> --debug
    ```

7. Test your server with for example [this](test_run.py)
    ```
    python test_run.py
    ```
8. Voilà !

