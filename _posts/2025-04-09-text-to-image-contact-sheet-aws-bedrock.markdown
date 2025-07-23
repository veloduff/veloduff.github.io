---
layout: post
title:  "Text-to-Image Contact Sheet Generator with Stable Diffusion"
date:   2025-04-09 16:00:00 -0700
categories: aws bedrock ai machine-learning image-generation
---

Creating multiple AI-generated images for comparison and selection can be a tedious process, especially when dealing with API throttling limits. This project demonstrates how to build an efficient contact sheet generator that uses AWS Bedrock's Stable Diffusion models to create multiple variations of an image prompt and compile them into a single comparison sheet.

**GitHub Repository**: [https://github.com/veloduff/text-to-image-contact-sheet](https://github.com/veloduff/text-to-image-contact-sheet)

The complete implementation is available as a Jupyter notebook (`bedrock_sdxl_contact_sheet.ipynb`) with example outputs and sample contact sheets included.

## The Problem: Throttling and Efficiency

When working with text-to-image AI models like Stable Diffusion, you often want to generate multiple variations of the same prompt to:
- **Compare different outputs** from the same prompt
- **Select the best result** from multiple attempts
- **Create reference sheets** for creative projects
- **Test prompt variations** efficiently

However, AWS Bedrock (like most AI services) implements throttling to manage resource usage, which can slow down batch image generation significantly.

## The Solution: Smart Throttling and Contact Sheets

This project provides a Python class-based solution that:
1. **Handles throttling gracefully** with automatic retry logic
2. **Generates multiple images** from a single prompt
3. **Creates contact sheets** for easy comparison
4. **Manages file organization** automatically

The system automatically:
- **Sanitizes prompts** into valid filenames
- **Prevents overwrites** with auto-incrementing numbers
- **Organizes output** in dedicated directories

The contact sheet creator uses PIL's thumbnail functionality to maintain image quality while creating compact comparison grids.

## Usage Example

### Generate a Single Test Image
```python
prompt = "A stylized picture of a cute old steampunk owl"
sd_image = SDTxt2ImgInference(prompt=prompt)
gen_image_path = sd_image.runImagePipeline()
```

### Create a 3x3 Contact Sheet
```python
cols, rows = 3, 3
total = cols * rows

while total > 0:
    sd_image.runImagePipeline()
    total -= 1

contact_sheet_path = sd_image.mkContactSheet(cols, rows)
```

This generates 9 variations of your prompt and compiles them into a single comparison image.

![Contact Sheet Example](/assets/images/owls_contact_sheet.jpg){:width="300px"}

The resulting contact sheet shows multiple variations of the same prompt, making it easy to compare and select the best results.

## Future Enhancements

The project architecture supports several potential improvements:

1. **Exponential backoff** for more intelligent throttling
2. **Multi-model support** for comparing different AI models
5. **Integration with other AWS services** like S3 for storage
