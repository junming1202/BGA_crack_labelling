# BGA crack labelling tools
You are an agent that helps to develop a software tool that is able to automate the labelling of each BGA (Ball-grid array) solder ball crack severity into either one of the 5 different categories (category A: 0% crack, category B: 1% - 25% crack, category C: 26% - 50% crack, category D: 51% - 75% crack, category E: 76% - 100% crack).

## More info about dye and pry
@docs/ipc-tm-650-2.4.53 Dye and Pry.pdf

## General guidelines for files and folder structures
All the markdown documents like AGENTS.md will be located in "docs" folder.

All the python scripts and jupyter notebook will be located in "src" folder.

The "database" folder will contain a list of subfolders with specific name that user defines. For example, "FP11_BLTC" means a package called "FP11" that has undergoes "BLTC" (board-level temperature-cycling test). Within each subfolder, there are at least 2 folders called "images" and "coordinates".
"images" folder contains one or more images of the bottom-view of BGA package dye and pry results. Each image shows the BGA array that should match the coordinates for the package's coordinates file within "coordinates" folder. Note that in the dye and pry image within "images" folder, it shows the bottom view of a package so the reference point of the package is located at top right corner in the image while in the coordinates file, it shows the BGA coordinates of the package with top-down view so the reference point of the package is at top-left corner for coordinates file. Take note of this and apply any coordinates transformation as necessary later when processing the results.

## Goal
At the end of automation, an "output" folder will be created at the same directory as "coordinates" folder or "images" folder. In the "output" folder, an output file like "_BGA_label_output.xlsx" will be created with the name of parent directory of "coordinates" folder or "images" folder as prefix. The output file is a spreadsheet with copy info of BGA coordinates from "coordinates" folder and additional columns with each column contains the classification result (category A, B, C, D, E) of each BGA for each dye and pry image in "images" folder.      

## Environment setup
Please use uv for environment setup.

## Available resources 
### LLM Setup
Please refer to the python code below as an example to setup any required llm functionality. Strictly follow the settings of base_url = "https://llm-api.amd.com/Unified/v1", api_key = os.environ["LLM_GATEWAY_KEY"], "Ocp-Apim-Subscription-Key" = os.environ["LLM_GATEWAY_KEY"], model = "Claude-Opus-4.8".
```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv(override=True)

client = OpenAI(
    base_url="https://llm-api.amd.com/Unified/v1",
    api_key=os.environ["LLM_GATEWAY_KEY"],
    default_headers={"Ocp-Apim-Subscription-Key": os.environ["LLM_GATEWAY_KEY"]},
)

response = client.chat.completions.create(
    model="Claude-Opus-4.8",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

### Open-source image-processing model
You can use any open-source image-processing AI models like (U-net, YOLO, ResNet, ...) that are available online.

## Steps
### Step 1: Research

### Step 2: Plan

### Step 3: Build

### Step 4: Test



