# Fast Apply: Pipeline for Data Generation & Fine-Tuning Qwen2.5 Coder Models

Kortix Fast Apply models are designed for instant code application, producing full file edits to power [SoftGen AI](https://softgen.ai/).

They achieve high throughput when deployed on fast providers like Fireworks while maintaining high edit accuracy:

- ~ `340 tok/s` for 1.5B model
- ~ `150 tok/s` for 7B model

Models and dataset are available on HuggingFace:
- [FastApply-7B-v1.0](https://huggingface.co/Kortix/FastApply-7B-v1.0)
- [FastApply-1.5B-v1.0](https://huggingface.co/Kortix/FastApply-1.5B-v1.0)
- [FastApply-dataset-v1.0](https://huggingface.co/datasets/Kortix/FastApply-dataset-v1.0)


The inference prompt structure:
```
<|im_start|>user
Merge all changes from the <update> snippet into the <code> below.
- Preserve the code's structure, order, comments, and indentation exactly.
- Output only the updated code, enclosed within <updated-code> and </updated-code> tags.
- Do not include any additional text, explanations, placeholders, ellipses, or code fences.

<code>{original_code}</code>

<update>{update_snippet}</update>

Provide the complete updated code."""
```

Model output :
```
<|im_start|>assistant
<updated-code>[Full-complete updated file]</updatedcode>
```

We chose smaller models (7B and 1.5B) for fast inference speed, suitable for instant apply tasks. 
These models work well with AI-powered code editors like Aider, PearAI or local tools to reduce the cost of frontier model output.


## Data Generation Pipeline

We generate high-quality synthetic data using open-source NextJS-like projects as `original-code`, 
then use Claude Sonnet 3.5 (70%) and GPT-4 (30%) to generate `update-snippet` and `final-updated-code`.


1. **Clone Open-Source Repositories**
   ```bash
   git clone --depth 1 https://github.com/your/repo.git data/repo
   ```

2. **Convert Repository Data to Dataset**

   Use `repo_to_dataset.py` to transform the repository data into a structured dataset while filtering out unsuitable files like log, cache, ...:

   ```bash
   python data_generation/repo_to_dataset.py /path/to/your/repo \
       --sample-lt-100 0 \
       --sample-100-399 500 \
       --sample-400-999 1000 \
       --sample-1000-1999 3000 \
       --sample-2000-2999 1000 \
       --sample-3000-3999 0 \
       ...
       --sample-10000-plus 0 \
       --output output.parquet \
       --debug
   ```

   **Parameters:**

   - `--sample-lt-100`: Number of samples with fewer than 100 tokens.
   - `--sample-...`

3. **Generate Synthetic Data**

   We recommend using `Anthropic Claude` for data generation due to its high quality output. 
   
   `OpenAI's Batch API` is also available as a cost-effective and faster alternative compared to `Claude's Batch API`. 

   Additionally, you can utilize the `Deepseek v2.5 API` beta to address any bugs or issues you may encounter in the generated dataset.

   ### Using Anthropic's API

   ```bash
   python data_generation/anthropic/generate.py --parquet_file data/train/my_data.parquet
   ```

   ### Using OpenAI's Batch API

   **Example Workflow:**

   ```bash
   # Prepare batch data
   python data_generation/openai/prepare_batch_data.py -i data/train/train_dataset.parquet -o data/train/batch/

   # Send batch requests
   python data_generation/openai/send_batch_request.py -bd data/train/batch/ -c 5

   # Process batch files
   python data_generation/openai/batch_processor.py -i data/train/batch/ -o data/train/train_dataset.parquet
   ```
## Fine-Tuning the Model

Fine-tuning enhances the pre-trained Qwen2.5 Coder models to better suit our specific task. We leverage `unsloth` to accelerate this process while minimizing VRAM usage.

### Key Details:

1. **Fine-tuning Notebooks**: Available in the `notebooks` directory.

2. **Dataset**: 
   - Source: https://huggingface.co/datasets/Kortix/FastApply-dataset-v1.0
   - Size: Approximately 5,600 examples
   - Composition: 80% TypeScript/TSX, 15% Python, 5% Other

3. **Model Versions**:
   - Using QLoRA with 4-bit quantization
   - 7B model: https://huggingface.co/unsloth/Qwen2.5-Coder-7B-Instruct-bnb-4bit
   - 1.5B model: https://huggingface.co/unsloth/Qwen2.5-Coder-1.5B-Instruct-bnb-4bit

4. **Hyperparameters**:
   - 1.5B model: rank (r) = 32, alpha = 16
   - 7B model: rank (r) = 16, alpha = 16
   - Training epochs: 1

This fine-tuning process optimizes the models for our specific code editing tasks while maintaining efficiency in computation and memory usage.

# Deploying on Fireworks

[See the instructions here](fireworks/README.md)

   **Inference script:** `tests_evaluate/fireworks/test_fireworks.py`

# Contribute

We welcome contributions to improve Fast Apply! Here are some ways you can help:

1. **More data**: The current model uses mostly TypeScript open-source code. Adding other languages could help avoid overfitting.

2. **Bug Reports**: If you encounter any issues, please open a GitHub issue with a detailed description.

3. **Feature Requests**: Have ideas for new features? Open an issue to discuss them.

4. **Code Contributions**:
   - Fork the repository
   - Create a new branch for your feature or bug fix
   - Submit a pull request with a clear description of your changes

5. **Fine-tuning Improvements**:
   - Share your findings on model performance improvements

Happy Coding!
