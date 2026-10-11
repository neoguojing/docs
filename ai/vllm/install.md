uv venv --python 3.12 --seed --managed-python

source .venv/bin/activate

uv pip install vllm --torch-backend=cu129 --extra-index-url https://wheels.vllm.ai/nightly/cu129

vllm download-kernels
